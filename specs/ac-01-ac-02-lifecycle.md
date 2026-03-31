# AC-01/AC-02: Lifecycle State Machine Reconciliation Specification

**Status:** Approved  
**Type:** Architectural Decision  
**Version:** 1.0.0  
**Approved Date:** 2026-03-31  
**Author:** STRATT/Choco Lifecycle Team  
**Impact:** Status enum, CRDT merge ordering, state transitions, Phase 2 bridge layer

---

## Executive Summary

This specification resolves the unified lifecycle state machine for STRATT and Choco content: **9-state model with explicit transitions and CRDT priority ordering**.

**AC-01 Decision**: Adopt 9 status states [draft → review → approved → published → active → deprecated → tombstoned] + [tampered] + [archived] with defined transition rules.

**AC-02 Decision**: CRDT merge ordering prioritizes restriction level: tombstoned > deprecated > archived > published > active > approved > review > draft. Conflicts resolved via highest-restriction-wins.

**Scope**: Unifies lifecycle semantics across STRATT (prompt units) and Choco (all content types) for consistent governance.

---

## Motivation: Why Lifecycle Unification Matters

**Problem**: STRATT has 6 statuses (draft, review, stable, deprecated, tampered, tombstoned). Choco will evolve its own lifecycle. Cross-org imports need reconciliation.

**Example Conflict**:
- STRATT unit is published as `stable` (STRATT status)
- Choco imports it; Choco's own status model has no `stable` state
- On merge: Is the unit "published" (Choco) or "stable" (STRATT)?
- CRDT needs deterministic rule to resolve

**Solution**: Define unified 9-state model. All content (STRATT, Choco, future) uses the same states. No mapping required.

---

## Unified Lifecycle: 9 States

### State Definitions

**1. DRAFT** — Work in progress, not ready for use
- **Meaning**: Incomplete, unstable, breaking changes expected
- **Publication**: CAN be published to R2 (for team review)
- **Imports**: CANNOT be imported by stable/published units (FM-07)
- **Transition**: DRAFT → REVIEW, DRAFT → DRAFT (no op)
- **Semver**: 0.x.x convention
- **Example**: A new prompt role before internal review

**2. REVIEW** — Submitted for approval, awaiting decision
- **Meaning**: Complete enough for peer review, but not approved
- **Publication**: Published (with review status badge)
- **Imports**: CAN be imported by review+ units (not by stable)
- **Transition**: REVIEW → DRAFT, REVIEW → APPROVED, REVIEW → REVIEW
- **Semver**: 0.x.x or 1.x.x (depends on approval decision)
- **Example**: A prompt rule during PR review cycle

**3. APPROVED** — Approved by domain authority, ready for deployment
- **Meaning**: Passed all gates and reviews; ready for production
- **Publication**: Published with approved status
- **Imports**: CAN be imported by approved+ units
- **Transition**: APPROVED → REVIEW, APPROVED → PUBLISHED, APPROVED → APPROVED
- **Semver**: 1.x.x (stable)
- **Example**: A chain that passed all FM checks and stakeholder sign-off

**4. PUBLISHED** — Live in production, stable contract
- **Meaning**: Active production artifact; breaking changes require major bump
- **Publication**: Published with published status
- **Imports**: CAN be imported freely
- **Transition**: PUBLISHED → ACTIVE, PUBLISHED → DEPRECATED, PUBLISHED → PUBLISHED
- **Semver**: 1.x.x, 2.x.x, ... (strict semver enforced)
- **Example**: A prompt chain running in production

**5. ACTIVE** — Actively in use, high dependency count
- **Meaning**: Published state + high usage/integration
- **Publication**: Published with active status (metadata only, no state change)
- **Imports**: CAN be imported freely
- **Transition**: ACTIVE → DEPRECATED, ACTIVE → ACTIVE (no op)
- **Semver**: 1.x.x+ (strict semver enforced)
- **Example**: A core rule or widely-used fragment

**6. DEPRECATED** — Scheduled for removal, prefer alternatives
- **Meaning**: Will be removed; new imports discouraged; migration path documented
- **Publication**: Published with deprecation banner + successor URI
- **Imports**: CAN be imported (but flagged as deprecated in dependency graph)
- **Transition**: DEPRECATED → PUBLISHED (revoke deprecation), DEPRECATED → TOMBSTONED, DEPRECATED → DEPRECATED
- **Semver**: 1.x.x+ (patch/minor/major for content updates)
- **Notifications**: All direct dependents notified (Phase 2)
- **Grace Period**: Typically 90 days before tombstoning
- **Example**: An old prompt template being replaced by better version

**7. ARCHIVED** — No longer used, not removed
- **Meaning**: Deprecated → ARCHIVED after grace period; accessible for historical reasons
- **Publication**: Published with archived status (read-only)
- **Imports**: CANNOT be imported by new content (blocks at validation)
- **Transition**: ARCHIVED → DEPRECATED (revoke archive), ARCHIVED → TOMBSTONED, ARCHIVED → ARCHIVED
- **Semver**: No updates (frozen)
- **Example**: A legacy prompt chain no longer used but kept for audit

**8. TOMBSTONED** — Deleted but addressable (irreversible)
- **Meaning**: Permanently removed; URI remains resolvable but content is empty
- **Publication**: Published with tombstone record (reason, successor)
- **Imports**: CANNOT be imported (FM-02 broken import)
- **Transition**: TOMBSTONED → TOMBSTONED only (irreversible)
- **Semver**: No updates
- **Retention**: Forever (immutable record)
- **Example**: A deprecated rule finally removed; URI 404s with "see also" link

**9. TAMPERED** — Fingerprint mismatch detected
- **Meaning**: Content integrity violation (not a normal state; indicates an error)
- **Publication**: Cannot publish (blocks at validate)
- **Imports**: CANNOT be imported (FM-01 failure)
- **Transition**: TAMPERED → DRAFT (only after re-publish with fingerprint)
- **Cause**: Post-publish mutation (rare, indicates infra failure)
- **Recovery**: Author re-runs `stratt publish` to recompute fingerprint
- **Example**: Cloudflare R2 corruption detected; unit marked tampered until fixed

### State Transition Diagram

```
        DRAFT
         ↓ ↑
       REVIEW ← ↔ → APPROVED
         ↓ ↑        ↓ ↑
       PUBLISHED
         ↓ ↑
       ACTIVE
         ↓ ↑
     DEPRECATED
         ↓ ↑
     ARCHIVED
         ↓ ↑
    TOMBSTONED (terminal)

    TAMPERED (error state)
         ↓
       DRAFT (recovery only)
```

### Transition Rules

**Allowed Transitions**:

| From | To | Requires | Semver | Conditions |
|------|----|-----------|---------|----|
| DRAFT | REVIEW | Author | 0.x.x | Any time |
| DRAFT | DRAFT | Author | — | No op |
| REVIEW | DRAFT | Author | — | Revert for changes |
| REVIEW | APPROVED | Gate Authority | 1.x.x | All gates passed |
| REVIEW | REVIEW | Author | — | Update PR |
| APPROVED | REVIEW | Author | — | Need more review |
| APPROVED | PUBLISHED | Author | 1.x.x | Ready for production |
| APPROVED | APPROVED | — | — | No op |
| PUBLISHED | ACTIVE | Metadata | 1.x.x+ | Auto (high usage) or manual |
| PUBLISHED | DEPRECATED | Author + gate | 1.x.x+ | Announce deprecation |
| PUBLISHED | PUBLISHED | — | — | No op (patch/minor/major allowed) |
| ACTIVE | DEPRECATED | Author + gate | 1.x.x+ | High-impact deprecation |
| ACTIVE | ACTIVE | — | — | No op |
| DEPRECATED | PUBLISHED | Author | 1.x.x+ | Revoke deprecation (rare) |
| DEPRECATED | ARCHIVED | Auto | 1.x.x+ | 90-day grace period expires |
| DEPRECATED | TOMBSTONED | Author + gate | — | Immediate removal (urgent) |
| DEPRECATED | DEPRECATED | — | — | No op |
| ARCHIVED | DEPRECATED | Author | 1.x.x+ | Un-archive (rare) |
| ARCHIVED | TOMBSTONED | Author + gate | — | Final removal |
| ARCHIVED | ARCHIVED | — | — | No op |
| TOMBSTONED | TOMBSTONED | — | — | Irreversible |
| TAMPERED | DRAFT | Author | — | Recover after re-publish |

---

## CRDT Merge Ordering (AC-02)

When concurrent edits change status field on different branches, **highest-restriction-wins**.

### Restriction Hierarchy

```
Highest Restriction
       |
       ↓
  TOMBSTONED       (permanently deleted)
       ↓
  ARCHIVED         (no new imports allowed)
       ↓
  DEPRECATED       (discourage use)
       ↓
  PUBLISHED        (production)
       ↓
  ACTIVE           (high usage)
       ↓
  APPROVED         (ready)
       ↓
  REVIEW           (under review)
       ↓
  DRAFT            (work in progress)
       ↓
Lowest Restriction
```

### Merge Rule

**If side-A has status S_A and side-B has status S_B**:
- **Merged status** = Higher restriction of S_A and S_B
- **Rationale**: Restriction intent wins. If one branch deprecates, merged state is deprecated.

### Examples

**Example 1: Deprecation wins**
- Side-A: Edits description, status stays PUBLISHED
- Side-B: Edits description + sets status DEPRECATED (decides to deprecate)
- Merged state: DEPRECATED (higher restriction)
- Audit: CRDT history shows both versions + why DEPRECATED was chosen

**Example 2: Archive wins**
- Side-A: Updates content, status PUBLISHED
- Side-B: Decides to archive, status ARCHIVED
- Merged state: ARCHIVED (higher restriction)
- Consequence: New imports blocked after merge

**Example 3: Tombstone (terminal)**
- Side-A: Status DEPRECATED
- Side-B: Status TOMBSTONED
- Merged state: TOMBSTONED (highest, irreversible)
- Consequence: Unit becomes unresolvable; imports fail

**Example 4: Draft reverts to draft**
- Side-A: Status DRAFT
- Side-B: Status DRAFT (both agree draft)
- Merged state: DRAFT
- No conflict

---

## Contract Changes and Status Constraints

### Contract Breaking Changes

When APPROVED+ (approved, published, active, deprecated, archived), breaking contract changes require:

1. **Major version bump** (required)
2. **Gate authority approval** (required)
3. **Dependents notification** (required in Phase 2)

Examples of breaking changes:
- Remove required input
- Change output type
- Remove a previously-protected agent from chain
- Remove a gate checkpoint

### Draft-Stability Rule (FM-07)

**PUBLISHED+ units cannot import DRAFT units**.

- DRAFT: Can import DRAFT only
- REVIEW: Can import DRAFT + REVIEW
- APPROVED+: Can import APPROVED+ only

Enforced at validation time (FM-07 check).

### Deprecation Grace Period

**Recommended**: 90 days from DEPRECATED → ARCHIVED

Allows time for dependents to migrate. Notification system (Phase 2) reminds direct dependents.

---

## Comparison to STRATT v1.x and Choco

### STRATT v1.x Lifecycle (Current)

- **States**: draft, review, stable, deprecated, tampered, tombstoned
- **Mapping to AC-01/AC-02**:
  - draft → DRAFT (0.x.x)
  - review → REVIEW (0.x.x or 1.x.x)
  - stable → PUBLISHED (1.x.x+)
  - deprecated → DEPRECATED (1.x.x+)
  - tampered → TAMPERED (error)
  - tombstoned → TOMBSTONED (permanent)
- **Missing**: APPROVED, ACTIVE, ARCHIVED
- **Change**: Rename `stable` → `published` for clarity

### Choco Lifecycle (TBD)

Choco will adopt AC-01/AC-02's 9-state model to align with STRATT.

### Cross-System Imports

When STRATT imports Choco (IC-02 bridge):
- Choco content's status is resolved via AC-01 states
- Merge uses AC-02 CRDT ordering
- No translation layer needed (unified model)

---

## Implementation: Status Enum and Validators

### Updated @stratt/schema

```typescript
// constants.ts
export const STATUS = {
  DRAFT: 'draft',
  REVIEW: 'review',
  APPROVED: 'approved',
  PUBLISHED: 'published',
  ACTIVE: 'active',
  DEPRECATED: 'deprecated',
  ARCHIVED: 'archived',
  TOMBSTONED: 'tombstoned',
  TAMPERED: 'tampered'
} as const;

export const STATUS_RESTRICTION_ORDER = [
  'tombstoned',   // 8 (highest)
  'archived',     // 7
  'deprecated',   // 6
  'published',    // 5
  'active',       // 4
  'approved',     // 3
  'review',       // 2
  'draft'         // 1 (lowest)
];

export const STATUS_CRDT_ORDER = STATUS_RESTRICTION_ORDER; // For CRDT merge
```

### Transition Validation

```typescript
// lifecycle.ts
interface Transition {
  from: Status;
  to: Status;
  requiresGate?: boolean;
  requiresMajorBump?: boolean;
}

const ALLOWED_TRANSITIONS: Transition[] = [
  { from: 'draft', to: 'review' },
  { from: 'review', to: 'approved', requiresGate: true },
  { from: 'approved', to: 'published' },
  { from: 'published', to: 'active' },
  { from: 'published', to: 'deprecated', requiresGate: true },
  { from: 'deprecated', to: 'archived' },
  { from: 'archived', to: 'tombstoned', requiresGate: true },
  // ... etc
];

export function isTransitionAllowed(from: Status, to: Status): boolean {
  return ALLOWED_TRANSITIONS.some(t => t.from === from && t.to === to);
}

export function transitionRequiresGate(from: Status, to: Status): boolean {
  const transition = ALLOWED_TRANSITIONS.find(t => t.from === from && t.to === to);
  return transition?.requiresGate ?? false;
}
```

### Draft Isolation (FM-07) Update

```typescript
export function canImport(importer: Status, imported: Status): boolean {
  // PUBLISHED+ cannot import DRAFT+REVIEW
  if (['published', 'active', 'deprecated', 'archived'].includes(importer)) {
    return !['draft', 'review'].includes(imported);
  }
  // REVIEW can import DRAFT+REVIEW
  if (importer === 'review') {
    return !['published', 'active', 'deprecated', 'archived'].includes(imported);
  }
  // DRAFT can only import DRAFT
  if (importer === 'draft') {
    return imported === 'draft';
  }
  return true; // APPROVED imports like PUBLISHED+
}
```

---

## Phase 1 vs Phase 2 Implementation

### Phase 1 (Current)

- ✓ Define 9-state model
- ✓ Implement status enum in @stratt/schema
- ✓ Implement transition validation
- ✓ Implement draft isolation (FM-07)
- ✓ Implement CRDT ordering (AC-02)
- ✓ No Choco integration yet (just STRATT)

### Phase 2

- ☐ Implement notifications (deprecation alerts)
- ☐ Auto-transition from DEPRECATED → ARCHIVED (90-day grace)
- ☐ Choco integration (import Choco content as AC-01 statuses)
- ☐ Cross-org notifications
- ☐ Lifecycle analytics (track state transition patterns)

---

## Diagram: Unified Lifecycle Model

```
┌─────────────────────────────────────────────────────────────────┐
│              AC-01/AC-02 Unified Lifecycle States               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Work-In-Progress     Approval Flow      Production            │
│  ─────────────────    ──────────────     ──────────            │
│                                                                 │
│  DRAFT ─────┐                                                   │
│    ↑        │                                                   │
│    └─ REVIEW ─────→ APPROVED ─────→ PUBLISHED ─→ ACTIVE       │
│             ↓           ↓             ↓             ↓           │
│             └─ Back to review        |             |           │
│                                       |             |           │
│                    Deprecation Flow   |             |           │
│                    ─────────────      |             |           │
│                       ↙              ↙              |           │
│                    DEPRECATED ←────────             |           │
│                       ↓                             ↓           │
│                    ARCHIVED ──────────────────────→ |           │
│                       ↓                             |           │
│                    TOMBSTONED (Terminal)            |           │
│                                                     ↓           │
│                    Error State: TAMPERED            |           │
│                    ↓                                 |           │
│                    └──→ DRAFT (Recovery)            |           │
│                                                     |           │
└─────────────────────────────────────────────────────────────────┘

    1. Restriction increases downward (higher = more restricted)
    2. Transitions left-to-right (normal flow)
    3. Backward transitions allowed (reviewer feedback, revoke deprecation)
    4. CRDT merge: highest restriction wins (cross-branch)
    5. TAMPERED: error state, recovery via re-publish
```

---

## Rationale: Why These 9 States?

**Draft, Review, Approved**: Governance gates before production
- DRAFT: Author's personal branch
- REVIEW: Peer review (gate check happens here)
- APPROVED: Authority sign-off

**Published, Active**: Production lifecycle
- PUBLISHED: Live, initial deployment
- ACTIVE: High usage/critical dependency (auto or manual promotion)

**Deprecated, Archived, Tombstoned**: Retirement phases
- DEPRECATED: Announced; grace period for migration
- ARCHIVED: No longer used; accessible for history
- TOMBSTONED: Deleted; permanent record

**Tampered**: Error indicator (not a working state)

**Why not fewer?** Because each state has distinct implications:
- Can I import this? (DRAFT vs REVIEW vs PUBLISHED)
- Is this in production? (PUBLISHED vs ACTIVE)
- Can I still use this? (DEPRECATED vs ARCHIVED vs TOMBSTONED)
- What must happen next? (REVIEW requires gate authority approval; DEPRECATED requires notification)

---

## Decision Record

**Decision (AC-01)**: 9-state unified lifecycle model  
**Decision (AC-02)**: Highest-restriction-wins CRDT merge ordering  
**Date Approved**: 2026-03-31  
**Approved By**: STRATT/Choco Architecture Team  
**Rationale**: Unifies STRATT and Choco governance; enables cross-org imports without state translation  
**Risk**: Requires status rename (stable → published) in TAD (low, cosmetic)  
**Next Steps**: Implement Phase 1 changes to @stratt/schema, Phase 2 notifications + Choco integration

---

**References**:
- TAD v1.1.0: specs/tad-v1.1.0.md (Layer 1, status enum section)
- IC-01: specs/ic-01-crdt-boundary.md (CRDT merge strategies)
- IC-02: specs/ic-02-namespace-policy.md (cross-org imports)

---

**Document Version**: 1.0.0  
**Status**: Approved  
**Maintained By**: STRATT/Choco Architecture Team
