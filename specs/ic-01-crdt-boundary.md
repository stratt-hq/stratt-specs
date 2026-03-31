# IC-01: CRDT Serialisation Boundary Specification

**Status:** Approved  
**Type:** Architectural Decision  
**Version:** 1.0.0  
**Approved Date:** 2026-03-31  
**Author:** STRATT Architecture Team  
**Impact:** Layer 2 (CRDT Resolution Model), Phase 2 implementation

---

## Executive Summary

This specification resolves the CRDT library choice for STRATT Layer 2: **Yjs (not Automerge)** is the authoritative CRDT engine for prompt unit field-level merge resolution.

**Decision**: Adopt Yjs v13.x as the CRDT foundation for @stratt/crdt package.

**Rationale**: Yjs is already in production within Choco via Tiptap (the-river, fork-node). Adopting Yjs means zero translation cost at the Choco boundary, native interoperability with existing real-time collaboration infrastructure, and proven scalability at production load.

**Risk Mitigation**: STRATT's existing lifecycle ordering (status priority merge model) is algorithm-agnostic and orthogonal to CRDT choice. The interface contract defined below isolates CRDT implementation from higher layers, enabling future swaps if justified.

---

## Decision: Why Yjs Over Automerge

### Evaluation Matrix

| Criterion | Yjs | Automerge | Winner |
|-----------|-----|-----------|--------|
| **Production Status in Choco** | In use via Tiptap | Not in use | Yjs ✓ |
| **Tiptap Integration** | Native (same library ecosystem) | Requires bridge layer | Yjs ✓ |
| **Real-Time Collab Support** | Built-in, proven at scale | Supported but less mature | Yjs ✓ |
| **Merge Semantics (Text)** | Rich, with Quill/Monaco integration | Richer, better for text | Automerge ~ |
| **Merge Semantics (Structured)** | Good (CRDT for any JSON type) | Excellent (graph-based) | Automerge ~ |
| **Performance** | Fast (O(1) ops), low memory | Fast (O(1) ops), higher memory | Yjs ~ |
| **Ecosystem Maturity** | Large, well-maintained | Smaller, slower updates | Yjs ✓ |
| **Learning Curve** | Moderate (Y.Array, Y.Map, Y.Text) | Moderate (actors, lamport clocks) | Tie |
| **License** | ISC (permissive) | Apache 2.0 (permissive) | Tie |

### Rationale

**Choco Alignment**: Choco's the-river document editor and fork-node collaboration infrastructure both use Yjs via Tiptap. Adopting Yjs means:
- Zero translation surface between STRATT and Choco
- Tiptap-to-MERIDIAN transformer (stratt-run#7) operates natively on Yjs docs
- Shared operational expertise within team
- Single dependency version to manage (Yjs, not Yjs + Automerge)

**Architecture Independence**: STRATT's field-level merge strategies (defined in TAD v1.1.0) are CRDT-agnostic. Strategies like "last-write-wins" (metadata), "manual-resolution" (contract), "append-wins" (composition), and "union-merge" (imports) work identically with Yjs or Automerge. The CRDT library is pluggable; the interface is what matters.

**Risk Profile**: Automerge's superior text merge semantics don't apply to STRATT because:
- STRATT units are small structured YAML documents, not rich text
- Contract changes are rare and always require manual review
- Text bodies are Markdown strings (no concurrent inline editing in Phase 2)
- Conflicts surface and resolve at the field level, not character level

**Production Readiness**: Yjs v13+ is battle-tested at scale in Figma-like systems. Automerge is excellent but less mature at high-concurrency, high-throughput scales.

---

## L2 Package Specification: @stratt/crdt

### Package Overview

**Name**: @stratt/crdt  
**Version**: 1.0.0  
**Scope**: Layer 2 (CRDT Resolution Model)  
**Dependencies**: yjs ^13.0, yjs-protocols (optional, for network transport)  
**Runtime**: Node.js 18+, Bun  
**License**: ISC

### Interface Contract

The @stratt/crdt package exports a minimal interface for resolving concurrent edits to STRATT units.

#### Core Type: `PromptUnitDoc`

```typescript
interface PromptUnitDoc {
  root: Y.Doc;                    // Underlying Yjs document
  unit: Y.Map<unknown>;           // Root map containing all unit fields
  id: Y.Text;                     // strat:// address (immutable)
  type: Y.Text;                   // unit type (immutable)
  domain: Y.Text;                 // domain (immutable)
  slug: Y.Text;                   // slug (immutable)
  version: Y.Text;                // semver version (immutable)
  status: Y.Text;                 // current lifecycle status
  meta: Y.Map<unknown>;           // metadata fields
  contract: Y.Map<unknown>;       // contract (inputs, outputs, failure_modes)
  composition: Y.Array<unknown>;  // chain steps
  imports: Y.Array<string>;       // import URIs
  fingerprint: Y.Text;            // Blake3 hash
  modified: Y.Text;               // ISO8601 timestamp
}
```

#### Core Functions

**`createPromptUnitDoc(yaml: string): PromptUnitDoc`**

Parse a YAML unit file and create a Yjs document with field structure.

- **Input**: Raw YAML string
- **Output**: PromptUnitDoc with Yjs structure
- **Validation**: Delegates to @stratt/schema (Zod validators)
- **Error**: Throws on invalid YAML or schema violation

```typescript
const unitYaml = fs.readFileSync('role-hostile-reviewer@1.0.0.yaml', 'utf8');
const doc = createPromptUnitDoc(unitYaml);
// doc.unit.get('meta').get('title') -> Y.Text for title field
```

**`applyEdit(doc: PromptUnitDoc, edits: Edit[]): void`**

Apply a batch of field-level edits to a Yjs document with automatic merge strategy selection.

- **Input**: 
  - `doc`: PromptUnitDoc
  - `edits`: Array of `{ field: string, value: unknown, strategy?: MergeStrategy }`
- **Behavior**: 
  - Metadata fields (meta.*): Last-write-wins
  - Contract fields (contract.*): Manual resolution required (throws if conflict)
  - Composition (composition): Append-wins for new steps; LWW for edits
  - Imports: Union merge
  - Status: Highest-restriction-wins (CRDT_ORDER from lifecycle.ts)
- **Error**: Throws if manual resolution required; returns conflict metadata

```typescript
applyEdit(doc, [
  { field: 'meta.title', value: 'Updated Title' },  // OK: LWW
  { field: 'contract.inputs', value: [...] },       // ERROR: conflict, manual needed
]);
```

**`resolveConflict(doc: PromptUnitDoc, field: string, resolved: unknown): void`**

Manually resolve a contract field conflict with documented justification.

- **Input**:
  - `doc`: PromptUnitDoc
  - `field`: Path (e.g., 'contract.inputs')
  - `resolved`: The resolved value chosen by reviewer
- **Behavior**: Marks conflict as resolved, stores both sides + resolution in CRDT history
- **Audit**: Resolution is attributed to author and timestamped

```typescript
resolveConflict(doc, 'contract.inputs', mergedInputs, {
  author: 'reviewer-handle',
  justification: 'Merged side-A and side-B inputs; side-A had correct required flags'
});
```

**`getHistory(doc: PromptUnitDoc, field?: string): FieldEdit[]`**

Retrieve the full CRDT merge history for the document or a specific field.

- **Input**: 
  - `doc`: PromptUnitDoc
  - `field` (optional): If provided, return history for just this field
- **Output**: Array of `{ author, timestamp, operation, before, after, strategy }`
- **Access**: Public read (immutable)

```typescript
const allEdits = getHistory(doc);
const metaEdits = getHistory(doc, 'meta.title');
// [
//   { author: 'null0', timestamp: '2026-03-31T09:00:00Z', operation: 'create', ... },
//   { author: 'dev-team', timestamp: '2026-03-31T09:15:00Z', operation: 'edit', ... },
// ]
```

**`getConflicts(doc: PromptUnitDoc): Conflict[]`**

Retrieve all unresolved conflicts in the document.

- **Input**: PromptUnitDoc
- **Output**: Array of `{ field, sideA, sideB, timestamp, authors }`
- **Behavior**: If no conflicts, returns empty array

```typescript
const conflicts = getConflicts(doc);
// [
//   { field: 'contract.inputs', sideA: [...], sideB: [...], timestamp: ... },
// ]
```

**`toYAML(doc: PromptUnitDoc): string`**

Serialize a Yjs document back to YAML for publication.

- **Input**: PromptUnitDoc (possibly with unresolved conflicts)
- **Output**: YAML string
- **Behavior**: Includes conflict markers if unresolved; throws if conflicts prevent serialization
- **Format**: Matches canonical serialisation spec (NFC-normalised, UTF-8)

```typescript
const yaml = toYAML(doc);
fs.writeFileSync('unit.yaml', yaml);
```

**`recomputeFingerprint(doc: PromptUnitDoc): string`**

Recompute Blake3 fingerprint from merged state (post-resolution).

- **Input**: PromptUnitDoc
- **Output**: `blake3:{64hex}`
- **Behavior**: Uses canonical serialisation pipeline from @stratt/fingerprint
- **Error**: Throws if document has unresolved conflicts

```typescript
const fingerprint = recomputeFingerprint(doc);
doc.unit.get('fingerprint').delete(0, doc.unit.get('fingerprint').length);
doc.unit.get('fingerprint').insert(0, fingerprint);
```

### Field Merge Strategies

All strategies are defined in this package; higher layers (@stratt/cli, @stratt/graph) refer to them by name.

**Strategy: `LAST_WRITE_WINS`** (Metadata fields)
```
Applies to: meta.title, meta.description, meta.tags, meta.author, meta.modified
Conflict: Neither side rejects; most recent write wins
Used by: Y.Text or Y.Map with timestamps
```

**Strategy: `MANUAL_RESOLUTION`** (Contract fields)
```
Applies to: contract.inputs, contract.outputs, contract.failure_modes
Conflict: Auto-merge blocked; requires human review
Behavior: Gate fires on PR; reviewer must approve merge strategy
History: Both versions stored; resolution documented with justification
```

**Strategy: `APPEND_WINS`** (Composition steps)
```
Applies to: composition.steps (additions only)
Conflict: New steps from both sides are appended
Edits: Existing step edits: LWW
Deletions: Explicit tombstone required (no auto-remove)
Used by: Y.Array
Rationale: Adding a step should not conflict with metadata changes; removing is intentional
```

**Strategy: `UNION_MERGE`** (Imports)
```
Applies to: imports[]
Conflict: Both sides' imports are retained
Removal: Explicit with documented justification
Used by: Y.Array (treated as set for uniqueness)
Rationale: Imports should never silently shrink
```

**Strategy: `HIGHEST_RESTRICTION_WINS`** (Status field)
```
Applies to: status
Conflict: Uses CRDT_ORDER from @stratt/schema/lifecycle.ts
Order: [tombstoned, deprecated, stable, review, draft]
Behavior: If side-A is 'stable' and side-B is 'review', merged state is 'review'
Rationale: Deprecation intent (restriction) wins over maintenance intent (advancement)
History: Both versions stored; final state documented
```

**Strategy: `RECOMPUTE_POST_MERGE`** (Fingerprint)
```
Applies to: fingerprint
Conflict: Never merged directly; always recomputed
Trigger: After any field merges resolve
Used by: @stratt/fingerprint.recomputeFingerprint()
Rationale: Fingerprint is a function of content; it must reflect the merged state
```

**Strategy: `IMMUTABLE`** (Identity fields)
```
Applies to: id, slug, domain, type, created
Conflict: Rejects merge if either side differs
Behavior: Throws error; manual intervention required (revert one branch)
Rationale: Identity fields define the unit's address and cannot change
```

---

## Integration with Tiptap / the-river

STRATT unit bodies are Markdown strings in Phase 1. In Phase 2, when task/chain prompts become rich-text documents editable in Tiptap:

1. The Y.Text field for `prompt_body` is compatible with Tiptap's Y.Text binding
2. Tiptap edits sync directly into the Yjs structure
3. Concurrent edits resolve via Yjs's built-in text merge
4. The broader unit structure (meta, contract, composition) remains structured (Y.Map, Y.Array)

**No bridge layer required**. Tiptap and STRATT share the same Yjs substrate.

---

## Offline/Online Sync Handshake

Yjs supports offline editing with sync on reconnect. The handshake:

1. **Offline Phase**: Author edits unit locally; Yjs buffers changes
2. **Online Phase**: Connect to remote; sync protocol exchanges updates
3. **Merge**: Incoming remote edits merge with local edits via CRDT
4. **Conflict Surface**: If conflicts exist, reviewer is prompted
5. **Publish**: Author resolves conflicts and publishes merged state

Implementation detail: Yjs handles the protocol; @stratt/crdt exposes conflict status via `getConflicts()`.

---

## Version Vector Alignment

Yjs maintains a version vector (lamport clock) for causal ordering. STRATT semantic versioning is independent:

- **Yjs version vector**: Tracks edit causality for merge resolution
- **STRATT semver**: Tracks user-facing contract changes (patch/minor/major)

A single STRATT version bump may contain multiple Yjs edits (e.g., editing description + adding input). The version vector ensures edits merge in correct causal order; semver indicates the nature of the change to downstream consumers.

---

## Phase 1 → Phase 2 Transition

**Phase 1** (current): No CRDT needed. Git handles versioning. Status-level lifecycle ordering from @stratt/schema/lifecycle.ts provides merge strategy.

**Phase 2** (planned): @stratt/crdt deployed. Multi-author concurrent editing enabled. Yjs handles field-level resolution per this spec.

**No breaking changes** to @stratt/schema or @stratt/cli. The interface defined here is pure addition.

---

## Comparison to TAD v1.0.0

TAD v1.0.0 spec'd Automerge. This decision changes that:

| Aspect | TAD v1.0.0 | IC-01 Decision |
|--------|-----------|----------------|
| **CRDT Library** | Automerge | Yjs |
| **Package** | @stratt/crdt | Same |
| **Field Merge Rules** | Same (LWW, manual, append, union, etc.) | Same |
| **History Storage** | Automerge binary | Yjs binary |
| **MERIDIAN Rendering** | Automerge history viewer | Yjs history viewer (Phase 2) |
| **Tiptap Integration** | Bridge layer needed | Native (no bridge) |

The merge strategies and interface are spec-agnostic. Only the library and serialization format change.

---

## Risk Mitigation: Implementation Isolation

To allow future CRDT swaps if needed:

1. **Minimal Public API**: Only the functions in "Interface Contract" above are public
2. **No Direct Yjs Exposure**: Callers use PromptUnitDoc, not Y.Doc
3. **Pluggable History Renderer**: MERIDIAN's history viewer is registered via callback
4. **Conflict Resolution Protocol**: Independent of CRDT library (works with any version vector)

If a future decision favors Automerge, the swap requires:
- Reimplement the internal Y.* structures with Automerge equivalents
- Reimplement merge strategies with Automerge's API
- Keep the public interface identical
- Update history renderer

**No callers change**.

---

## Decision Record

**Decision**: Yjs  
**Alternative Considered**: Automerge  
**Date Approved**: 2026-03-31  
**Approved By**: STRATT Architecture Team  
**Rationale**: Choco alignment, zero translation surface, production maturity  
**Risk**: Lower text merge sophistication (mitigated: structured docs, not rich text)  
**Next Steps**: IC-02 (namespace), AC-01/AC-02 (lifecycle), then L2 implementation

---

**References**:
- TAD v1.1.0: specs/tad-v1.1.0.md (Layer 2, Phase 2 section)
- Yjs Documentation: https://docs.yjs.dev/
- stratt-run#7: Tiptap-to-MERIDIAN transformer (Phase 2)

---

**Document Version**: 1.0.0  
**Status**: Approved  
**Maintained By**: STRATT Architecture Team
