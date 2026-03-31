# IC-02: Namespace Collision Policy Specification

**Status:** Approved  
**Type:** Architectural Decision  
**Version:** 1.0.0  
**Approved Date:** 2026-03-31  
**Author:** STRATT/Choco Integration Team  
**Impact:** URI addressing, cross-org integration, Phase 2 bridge layer

---

## Executive Summary

This specification resolves the namespace strategy for STRATT and Choco content: **Coexistence with BridgeResolver** (not merger, not collision detection).

**Decision**: `strat://` and `choco://` are separate, non-overlapping URI schemes. A pluggable BridgeResolver interface enables cross-scheme imports without coupling the systems.

**Rationale**: Separate schemes preserve domain independence while enabling explicit bridging. No unified namespace reduces coupling risk and allows each org to evolve its addressing model independently.

**Mutual Support**: STRATT can import Choco content via bridge; Choco can consume STRATT units as documented services.

---

## Namespace Design: Coexistence

### The Schemes

**STRATT Namespace**: `strat://{domain}/{type}/{slug}@{version}`

- **Scheme**: strat (immutable)
- **Domain**: Registered set [dev, neuro, finance, nutrition, legal, film, artist, core, shared]
- **Type**: [role, rule, task, chain, fragment]
- **Slug**: kebab-case, immutable forever
- **Version**: Semver

Examples:
- `strat://dev/chain/sol-1-boot@1.0.0`
- `strat://shared/fragment/soap-note-schema@1.1.0`
- `strat://core/rule/never-fabricate-citation@1.0.0`

**Choco Namespace**: `choco://{org}/{content-type}/{path}@{version}`

- **Scheme**: choco (immutable)
- **Org**: Choco organization domain (e.g., choco, block-k, fork-node)
- **Content-Type**: API, guide, runbook, decision, sample, schema (extensible)
- **Path**: Hierarchical content path (e.g., block-k/nats/consumer/subscribe)
- **Version**: Semver

Examples:
- `choco://block-k/api/nats/consumer@1.0.0`
- `choco://fork-node/guide/realtime/conflict-resolution@2.1.0`
- `choco://choco/decision/adr-001-yjs-adoption@1.0.0`

### Key Properties

1. **No Overlap**: STRATT uses strat://; Choco uses choco://. No collision possible by design.
2. **Independent Evolution**: Each scheme can add fields/versions without affecting the other.
3. **Explicit Imports**: Cross-scheme imports require BridgeResolver (see below) — no implicit resolution.
4. **Coexistence**: Both schemes exist in the same knowledge graph. A chain can import from both strat:// and choco://.
5. **Trust Boundary**: Implicit trust within a scheme; explicit trust at bridge boundary.

---

## Cross-Scheme Imports: BridgeResolver

### Problem Statement

When a STRATT unit imports Choco content (or vice versa), how is that import resolved? The native @stratt/graph resolver understands strat:// URIs. It does not understand choco:// URIs.

### Solution: Pluggable BridgeResolver

A **BridgeResolver** is a pluggable interface that translates cross-scheme imports into resolvable references.

```typescript
interface BridgeResolver {
  /**
   * Attempt to resolve a cross-scheme URI.
   * Returns null if this resolver does not handle the scheme.
   */
  resolve(uri: string): ResolvedUnit | null;

  /**
   * Test whether this resolver handles a given scheme.
   */
  canResolve(scheme: string): boolean;

  /**
   * Optional: Verify cross-scheme import is permitted.
   * Enables trust policies (e.g., only certain Choco content can be imported).
   */
  verifyTrust(importingUnit: Unit, importedUri: string): boolean;
}

interface ResolvedUnit {
  uri: string;                    // The URI that was resolved
  content: object;                // Parsed/hydrated content (schema-dependent)
  metadata: {
    fetched: Date;
    schema_version: string;
    trust_level: 'public' | 'restricted' | 'internal';
  };
}
```

### Integration with @stratt/graph

The @stratt/graph package accepts a resolver registry:

```typescript
interface GraphOptions {
  resolvers: BridgeResolver[];    // Array of resolvers, tried in order
  defaultResolver?: BridgeResolver; // Fallback for unmapped schemes
}

function buildDag(registry: UnitRegistry, options: GraphOptions): Dag {
  // For each unit in registry:
  //   For each import in unit:
  //     If scheme is 'strat', use native resolver
  //     Else, try each resolver in order
  //     If no resolver handles it, report FM-02 (broken import)
}
```

### Resolver Chain Example

```typescript
const defaultOptions: GraphOptions = {
  resolvers: [
    new ChocoBlockKResolver(),      // Handles choco://block-k/*
    new ChocoForkNodeResolver(),    // Handles choco://fork-node/*
    new ChocoResolver(),            // Catch-all for choco://*
  ],
  defaultResolver: new ErrorResolver(), // Reject unknown schemes
};

const dag = buildDag(registry, defaultOptions);
```

### STRATT-to-Choco Bridge Implementation (Phase 2)

When STRATT units import Choco content:

```typescript
class ChocoResolver implements BridgeResolver {
  canResolve(scheme: string): boolean {
    return scheme === 'choco';
  }

  resolve(uri: string): ResolvedUnit | null {
    // Parse choco://org/type/path@version
    const parsed = parseChocoUri(uri);
    
    // Fetch from Choco's content service (e.g., Cloudflare R2)
    const content = await fetchFromChoco(parsed);
    
    // Return wrapped unit
    return {
      uri,
      content,
      metadata: {
        fetched: new Date(),
        schema_version: '1.0.0',
        trust_level: 'public'  // or 'restricted' based on policy
      }
    };
  }

  verifyTrust(importingUnit: Unit, importedUri: string): boolean {
    // Implement trust policy
    // E.g.: STRATT core domain can import any Choco content
    //       STRATT dev domain can only import public content
    return importingUnit.domain === 'core' || 
           this.isTrustworthy(importedUri);
  }
}
```

### Choco-to-STRATT Bridge Implementation (Phase 2)

Similarly, Choco content can resolve STRATT units:

```typescript
class StratResolver implements BridgeResolver {
  canResolve(scheme: string): boolean {
    return scheme === 'strat';
  }

  resolve(uri: string): ResolvedUnit | null {
    // Fetch from STRATT/Cloudflare R2
    const yaml = await fetchFromR2(`strat/${uri}`);
    const parsed = parseAndValidate(yaml); // Uses @stratt/schema
    
    return {
      uri,
      content: parsed,
      metadata: {
        fetched: new Date(),
        schema_version: '1.0.0',
        trust_level: 'public'
      }
    };
  }

  verifyTrust(importingUnit: object, importedUri: string): boolean {
    // Choco's trust policy for importing STRATT content
    return true; // STRATT is publicly available
  }
}
```

---

## Trust Boundaries

### Within STRATT (No Boundary)

All strat:// units are implicitly trusted within STRATT:
- Fingerprints verified at publish time
- Circular dependencies checked
- Protected agents enforced
- Draft isolation enforced

**Trust Model**: Authority-based (author's GitHub team membership + CI checks)

### Across Bridge (Explicit Boundary)

When importing across schemes:

1. **Verification**: The BridgeResolver's `verify Trust()` is called before import resolution
2. **Metadata**: Resolution includes `trust_level` annotation
3. **Annotation**: The importing unit can declare trust assumptions (metadata annotation, Phase 2)
4. **Enforcement**: CI can block imports from untrusted sources

**Trust Model**: Explicit declaration + policy-based enforcement

### Example: STRATT Importing Choco

```yaml
# strat://dev/task/call-choco-api@1.0.0
id: strat://dev/task/call-choco-api@1.0.0
imports:
  - strat://core/rule/never-fabricate-citation@1.0.0
  - choco://block-k/api/nats/consumer@1.0.0  # Cross-scheme!

# Metadata annotation (Phase 2):
trust:
  choco://block-k/api/nats/consumer:
    level: public
    verified_at: 2026-03-31
    verifier: LEWIS-06  # gate authority agent who approved
```

CI checks:
1. Can STRATT resolve choco://block-k/api/nats/consumer? (via ChocoBlockKResolver)
2. Is importingUnit.domain allowed to import from this Choco org? (verifyTrust)
3. Is the import declared in metadata.trust with appropriate approval? (FM-09, Phase 2)

---

## URI Scheme Registration

### Reserved Schemes

- `strat://` — STRATT prompt units (this spec)
- `choco://` — Choco platform content

### Extension Path (Phase 3)

New schemes can be registered by:
1. Creating a formal spec (like this document)
2. Implementing a BridgeResolver
3. Registering resolver in integration module

Example future schemes:
- `veritas://` — Legacy prompt library (for migration)
- `meridian://` — Generic MERIDIAN content (Phase 3)
- `custom://` — User-defined content types

Each scheme is self-describing: resolution, versioning, and trust model defined in its own spec.

---

## Namespace Migration Path (Phase 3)

If STRATT and Choco eventually unify their content model (unlikely but possible):

1. **Option A (Preserve Schemes)**: Keep strat:// and choco:// forever. Bridge layer becomes permanent infrastructure.
2. **Option B (Merge Schemes)**: Introduce meridian:// as the unified scheme. Create migration resolver that redirects old URIs to new.
3. **Option C (Metadata Bridge)**: Keep separate schemes but add lightweight bridging in schema (cross-reference by metadata, not URI).

**No decision needed now**. The coexistence model accommodates all three options.

---

## Comparison to TAD v1.0.0

TAD v1.0.0 did not explicitly address multi-namespace. IC-02 clarifies:

| Aspect | TAD v1.0.0 | IC-02 Decision |
|--------|-----------|----------------|
| **URI Scheme** | strat:// (implied) | strat:// + choco:// + extensible |
| **Cross-Org Imports** | Not addressed | Explicit via BridgeResolver |
| **Collision Policy** | Not addressed | Coexistence (no collision) |
| **Trust Model** | Authority-based (GitHub) | Authority within scheme; policy-based across |

---

## Alignment with Choco Architecture

**Choco's Content IR**: The-river document format + Tantivy indexing + ClickHouse analytics.

**Bridge Role**: STRATT's strat:// scheme coexists with Choco's content IR. The BridgeResolver translates between them.

**Data Flow**:
```
STRATT units (YAML) → Cloudflare R2
                   ↓
             BridgeResolver
                   ↓
           Choco content (IR format) → Tantivy index → ClickHouse
```

No data loss. Each system retains its native representation. The bridge is translate-on-access, not ETL.

---

## Implementation Phases

**Phase 1** (current): Single strat:// scheme. No BridgeResolver.

**Phase 2** (planned):
- Implement ChocoResolver for STRATT → Choco imports
- Implement StratResolver for Choco → STRATT imports
- Integrate with @stratt/graph resolver chain
- Add trust verification to CI pipeline (FM-09, Phase 2)

**Phase 3** (planned):
- Support additional schemes (veritas://, custom://)
- Advanced trust policies (fine-grained RBAC)
- Unified schema registration (all schemes, single registry)

---

## Specification for Bridge Resolver

(Details for implementation team)

### ChocoResolver (STRATT consuming Choco)

```typescript
class ChocoResolver implements BridgeResolver {
  private chocoClient: ChocoContentClient;
  private trustPolicy: TrustPolicy;

  constructor(chocoClient, trustPolicy) {
    this.chocoClient = chocoClient;
    this.trustPolicy = trustPolicy;
  }

  canResolve(scheme: string): boolean {
    return scheme === 'choco';
  }

  async resolve(uri: string): Promise<ResolvedUnit | null> {
    const parsed = parseChocoUri(uri);
    const content = await this.chocoClient.fetch(parsed);
    
    if (!content) return null;
    
    return {
      uri,
      content,
      metadata: {
        fetched: new Date(),
        schema_version: '1.0.0',
        trust_level: this.trustPolicy.evaluate(uri)
      }
    };
  }

  verifyTrust(importingUnit: Unit, importedUri: string): boolean {
    return this.trustPolicy.allows(importingUnit, importedUri);
  }
}
```

### StratResolver (Choco consuming STRATT)

(Inverse of ChocoResolver; Choco team implements)

---

## Decision Record

**Decision**: Coexistence with BridgeResolver  
**Alternative Considered**: Single unified namespace (rejected: too much coupling)  
**Alternative Considered**: Separate systems, no bridge (rejected: blocks integration)  
**Date Approved**: 2026-03-31  
**Approved By**: STRATT/Choco Architecture Team  
**Rationale**: Preserves independence, enables explicit bridging, extensible to future schemes  
**Next Steps**: AC-01/AC-02 (lifecycle), then bridge implementations in Phase 2

---

**References**:
- TAD v1.1.0: specs/tad-v1.1.0.md (Layer 3, Dependency Graph section)
- IC-01: specs/ic-01-crdt-boundary.md (complements this spec)
- AC-01/AC-02: specs/ac-01-ac-02-lifecycle.md (complements this spec)

---

**Document Version**: 1.0.0  
**Status**: Approved  
**Maintained By**: STRATT/Choco Architecture Team
