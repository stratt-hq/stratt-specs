# MERIDIAN — Specification v1.1.0

**Status:** Stable  
**Version:** 1.1.0  
**Released:** 2026-03-31  
**Previous:** v1.0.0 (Draft, Gist-based)  
**Changes:** 6 scope clarifications introducing explicit phase boundaries

---

## Executive Summary

MERIDIAN is a **schema-driven, cryptographically verifiable, governance-enabled knowledge rendering and discovery platform**. This version clarifies Phase 1 scope (identity, schema, verification) and Phase 2 scope (interactive rendering, CRDT visualization, notifications).

**Key Insight**: MERIDIAN is not primarily a documentation site. It is a verifiable content protocol where the schema IS the documentation and every artifact carries a Blake3 fingerprint. The first implementation governs STRATT prompt units. The architecture scales to govern all content types within Choco.

---

## Change Summary (v1.0.0 → v1.1.0)

### What Changed

This release introduces explicit phase boundaries to clarify which requirements are Phase 1 (implemented) vs Phase 2 (planned).

**The 6 Pending Changes**:

1. **Layer 6 Rendering (P0 → Phase 2)** 
   - Move MER-FR-050 through 054: Interactive pages, diff viewer, chain visualiser, search, deep links
   - Phase 1: CLI output only; Phase 2: Astro/Starlight web site

2. **CRDT Merge & History (P0 → Phase 2)**
   - Move MER-FR-030 through 034: Automerge field-level resolution, merge conflict handling, history visualization
   - Phase 1: Status-level lifecycle ordering only; Phase 2: Full Yjs CRDT with manual review queue

3. **Fingerprint Verification on Render (P0 → Phase 2)**
   - Move MER-FR-003, MER-FR-004, MER-NFR-008: Render-time verification UI, tamper banners
   - Phase 1: Works at publish/CI time; Phase 2: Real-time render verification with banners

4. **Deprecation Notifications (P1 → Phase 2)**
   - Move MER-FR-042: Automated notifications to dependents on deprecation
   - Phase 1: deprecate command exists; Phase 2: Notification system

5. **Atomic Publish (P0 → Split into Phase 1+2)**
   - MER-FR-061: Local atomic operations in Phase 1; Distributed atomic transactions Phase 2
   - Phase 1: Local file write is atomic; Phase 2: Distributed write with tombstone delivery

6. **Multi-Namespace Support (P1 → Phase 2)**
   - Move MER-FR-012: Multiple namespace schemes (`strat://`, `choco://`, custom URIs)
   - Phase 1: STRATT only; Phase 2: Per IC-02 spec, support coexistence with BridgeResolver

### What Stayed the Same

- ✓ Layer 0 (Identity): Canonical URIs, Blake3, semver versioning
- ✓ Layer 1 (Schema): Zod validators, schema governance, field sovereignty
- ✓ Layer 3 (Graph): DAG operations, dependency transparency, protection model
- ✓ Layer 5 (CLI): 9 implemented commands with --json and --verbose
- ✓ Canonical Serialisation: 5-stage deterministic pipeline
- ✓ Failure Modes: FM-01 through FM-07 (FM-08 in Phase 2)

### Phase Designation

**Phase 1** (Current, ✓ ~80% Complete):
- Requirements: MER-FR-001, 002, 005 (partial), 010-022, 041, 043-060 (except 050-054), 061 (partial)
- Scope: Identity, schema, verification (at publish/CI), dependency graph, CLI interface
- Technology: CLI, Bun, Zod, @stratt/* packages, Git, Cloudflare R2

**Phase 2** (Planned):
- Requirements: MER-FR-003 (partial), 004, 012, 030-034, 042, 050-054, 061 (partial), all interactive features
- Scope: Web rendering, CRDT visualization, notifications, full-text search, distributed atomicity
- Technology: Astro, Starlight, D3.js, Yjs, Tantivy, notification system

**Phase 3** (Planned):
- Advanced features: Machine learning recommendations, AI-powered content generation, analytics, real-time collaboration

---

## System Definition (Updated)

### What MERIDIAN Is

A **verifiable, schema-sovereign, dependency-transparent knowledge layer**. Three defining properties:

1. **Verifiability** — Every content unit carries a Blake3 fingerprint computed at publish time. Fingerprints are verified on publish (Phase 1) and on render (Phase 2). Tampering is detected automatically.

2. **Schema Sovereignty** — The schema IS the content model. Documentation drift is impossible because rendering is generated directly from schema files, not authored separately.

3. **Dependency Transparency** — All dependencies declared explicitly. Changes trigger automated blast radius computation. Nothing changes silently.

### Current Scope: STRATT Prompt Units

Phase 1 governs STRATT prompt units (role, rule, task, chain, fragment) via:
- @stratt/schema (Zod validators)
- @stratt/fingerprint (Blake3 verification)
- @stratt/graph (dependency resolution)
- @stratt/cli (9 commands for authoring, validation, publication)

### Target Scope: Choco Documentation Platform

Phase 2+ will extend MERIDIAN to govern any content type:
- MDX documentation pages
- API references (OpenAPI)
- Code samples with verification
- Runbooks and procedures
- ADRs and decision records
- Stakeholder resources
- AI agent definitions (MCP)

The same three properties apply identically regardless of content type.

---

## Functional Requirements by Phase

### Phase 1: Identity & Verification

**MER-FR-001** (P0)  
Every content unit published to MERIDIAN shall have a canonical URI with scheme, domain, type, slug, and semver version components.

**MER-FR-002** (P0)  
Every published content unit shall carry a Blake3 fingerprint computed from its canonical serialisation at publish time.

**MER-FR-005** (P0, Partial)  
All content unit versions shall conform to semantic versioning with machine-enforced rules for patch/minor/major classification. **Phase 1**: Validation enforced. **Phase 2**: Dynamic patch/minor/major classification rules for contract changes.

**MER-FR-010** (P0)  
No content unit shall be published to MERIDIAN without passing validation against its registered JSON Schema.

**MER-FR-011** (P0)  
MERIDIAN shall auto-generate documentation from published YAML schemas via the CLI `stratt new` and schema validators. Schema changes → auto-updated docs (via @stratt/schema, not web rendering).

**MER-FR-013** (P0)  
Every content unit version shall be addressed immutably. New versions create new addresses; old addresses never change.

**MER-FR-014** (P0)  
All identity fields (id, slug, domain, type, created) shall be immutable post-creation.

**MER-FR-015** (P0, Partial)  
Unit modification shall create an append-only audit trail. **Phase 1**: Git commit log is the audit trail. **Phase 2**: Immutable CRDT history storage in R2.

**MER-FR-020** (P0)  
Dependencies between content units shall be declared explicitly in an imports field. Undeclared dependencies fail validation.

**MER-FR-021** (P0)  
MERIDIAN shall compute the transitive import tree for any unit and detect circular dependencies.

**MER-FR-022** (P0)  
MERIDIAN shall compute the blast radius for any change — all downstream consumers directly or transitively importing the changed unit.

**MER-FR-041** (P1)  
Units shall progress through a defined lifecycle: draft → review → stable → deprecated → tombstoned. Status changes enforced via semantic versioning and deprecation rules.

**MER-FR-043** (P1)  
Protected content (rules, agents) cannot be removed from a chain's composition without documented override.

**MER-FR-044** (P1)  
Gate checkpoints in chains are architectural invariants. Gate removal requires a major version bump.

**MER-FR-045** (P1)  
Stable units cannot import draft (0.x.x) units. Draft isolation enforced at publish time.

**MER-FR-060** (P1)  
Unit metadata (title, description, tags, author) shall be validated against schema constraints.

**MER-FR-061** (P0, Partial)  
Publish operations shall be atomic at the local file level. **Phase 1**: Single file write is atomic. **Phase 2**: Distributed atomic write to R2 + Git with tombstone delivery guarantee.

### Phase 2: Rendering & Discovery

**MER-FR-003** (P0)  
MERIDIAN shall recompute and verify the Blake3 fingerprint on every page render and expose verification status to the user. *(Phase 2: Requires web rendering layer)*

**MER-FR-004** (P0)  
Content units with fingerprint mismatch shall be displayed with a tamper banner. *(Phase 2: Requires web rendering layer)*

**MER-FR-012** (P1)  
MERIDIAN shall support multiple content namespace schemes (strat://, choco://, custom). *(Phase 2: Per IC-02 spec decision)*

**MER-FR-030** (P0)  
Concurrent edits to content units shall be resolved using Yjs-based CRDT with field-level merge strategies. *(Phase 2: Depends on L2 CRDT implementation)*

**MER-FR-031** (P0)  
Field-level merge strategies shall differentiate between metadata (LWW), contract (manual), composition (append), imports (union), status (highest-restriction), and fingerprint (recompute) fields. *(Phase 2)*

**MER-FR-032** (P1)  
Full CRDT merge history shall be publicly viewable in MERIDIAN with field-level attribution and timestamp. *(Phase 2: Requires web rendering + Yjs integration)*

**MER-FR-034** (P2)  
Units with unresolved CRDT conflicts shall not be publishable to stable status until manual resolution is documented. *(Phase 2)*

**MER-FR-042** (P1)  
Deprecation operation shall trigger automated notification to all direct dependents. *(Phase 2: Requires notification infrastructure)*

**MER-FR-050** (P0)  
MERIDIAN shall render interactive content pages with all schema fields, status badges, fingerprint verification, dependency mini-graph, version history, and full CRDT merge log. *(Phase 2: Requires Astro/Starlight web framework)*

**MER-FR-051** (P1)  
MERIDIAN shall provide an interactive version diff viewer showing field-level changes between any two versions with contract changes highlighted. *(Phase 2)*

**MER-FR-052** (P1)  
MERIDIAN shall provide a live chain flow visualiser rendering step composition as an interactive flowchart with gate step markers. *(Phase 2)*

**MER-FR-053** (P2)  
MERIDIAN shall expose a full-text search index covering all published content units with sub-50ms query latency. *(Phase 2: Requires Tantivy or Algolia)*

**MER-FR-054** (P2)  
MERIDIAN shall provide stable, shareable deep-link URLs for every unit, version, diff, and dependency graph view. *(Phase 2)*

---

## Non-Functional Requirements

**MER-NFR-001** (P0)  
Fingerprint computation time < 100ms per unit.

**MER-NFR-002** (P0)  
Dependency graph computation time < 500ms for unit with up to 1000 transitive dependents.

**MER-NFR-003** (P1)  
Schema validation latency < 50ms per unit.

**MER-NFR-004** (P0)  
All published units remain addressable indefinitely. URI links never 404.

**MER-NFR-005** (P1)  
Content discoverability: All published units appear in dependency graph and impact analysis automatically (no manual indexing).

**MER-NFR-006** (P1)  
Verification reproducibility: Blake3 fingerprint computation identical across all implementations (CLI, CI, Phase 2 web).

**MER-NFR-007** (P1)  
Immutability guarantee: Stable unit versions are write-once. Updates create new versions; no in-place mutations.

**MER-NFR-008** (P0, Phase 2)  
Tampered unit detection latency from R2 mutation to user notification < 1 render cycle. *(Phase 2)*

**MER-NFR-009** (P1)  
Authentication: Published units are publicly readable. Authoring requires GitHub team membership.

---

## Principles (Unchanged from v1.0.0)

1. **Verifiable Content**: Every unit carries a fingerprint. Tampering is detected.
2. **Schema Sovereignty**: Schema changes = docs change. No separate writing step.
3. **Dependency Transparency**: All imports declared. Impact analysis automatic.
4. **Deprecation Over Deletion**: Units never deleted. Tombstoned with audit trail.
5. **Atomic Publishing**: Publish is all-or-nothing at file level (Phase 1) or distributed (Phase 2).
6. **Explicit Conflicts**: Concurrent edits surface conflicts for manual resolution (Phase 2).
7. **Immutable Identity**: Addresses never change. New versions get new addresses.

---

## Architecture: The Three Layers

### Layer 0: Identity & Addressing

Every unit: `strat://{domain}/{type}/{slug}@{version}`

- Domain: [dev, neuro, finance, nutrition, legal, film, artist, core, shared]
- Type: [role, rule, task, chain, fragment]
- Slug: Kebab-case, immutable forever
- Version: Semver (e.g., 1.0.0, 0.2.3)

Blake3 fingerprinting via 5-stage canonical serialisation:
1. YAML parse (core schema, no merge, unique keys)
2. Remove nulls
3. NFC-normalise strings
4. Sort keys UTF-16 order
5. UTF-8 encode → Blake3 hash

**Phase 1**: Fully implemented ✓  
**Phase 2**: Render-time verification UI

### Layer 1: Schema Governance

All units conform to base schema (id, type, domain, slug, version, fingerprint, status, meta, imports, council) plus type-specific constraints (persona for role, rule for rule, contract for task/chain, body for fragment).

**Validation**: Zod via @stratt/schema  
**Schema Generation**: JSON Schema via zod-to-json-schema  
**Enforcement**: CLI rejects non-conforming units at publish time

**Phase 1**: Fully implemented ✓  
**Phase 2**: Dynamic contract-change detection for patch/minor/major rules

### Layer 3: Dependency Graph

All units form a DAG via explicit imports. Operations:
- **Build**: Construct adjacency lists
- **Cycles**: Detect and report
- **Topological Sort**: Order for execution
- **Blast Radius**: Reverse BFS showing impact
- **Protection**: Enforce protected unit/agent presence

**Phase 1**: Fully implemented ✓  
**Phase 2**: Multi-namespace support (strat://, choco://, custom)

---

## The Expansion Map (Phase 2+)

MERIDIAN's architecture applies to any content type:

| Content Type | Addressing | Schema | Governance | Phase |
|---|---|---|---|---|
| Prompt Unit | strat://dev/task/... | role/rule/task/chain/fragment | Council, gates, protected rules | 1 |
| MDX Document | meridian://docs/guide/... | frontmatter + markdown | Sections, deprecation, links | 2 |
| API Reference | meridian://api/endpoint/... | OpenAPI | Methods, versions, deprecation | 2 |
| Code Sample | meridian://code/pattern/... | Test-driven schema | Verification status, language, framework | 2 |
| Runbook | meridian://ops/procedure/... | Checklist schema | Steps, owners, SLA, escalation | 2 |
| ADR | meridian://decisions/adr/... | Status, stakeholders | Supersession, decision log | 2 |
| Agent Definition | meridian://ai/agent/... | Capability matrix, model schema | Permissions, gates | 2 |

All follow the same 3 principles: verifiable, schema-sovereign, dependency-transparent.

---

## Roadmap: 3-Phase Plan

**Phase 1** (Current, ~80% complete)
- CLI interface for authoring, validation, publishing
- Blake3 fingerprinting and verification at publish/CI time
- Dependency graph engine with blast radius
- 7 failure modes (FM-01 through FM-07) enforced
- Git-based version control and audit trail

**Phase 2** (In Progress)
- Web site rendering (Astro/Starlight at stratt.dev)
- Interactive features (diff viewer, chain visualizer, graph explorer)
- Full-text search (Algolia or Tantivy)
- CRDT merge with Yjs (IC-01 decision)
- Notification system for deprecation
- Multi-namespace support (IC-02 decision)
- Distributed atomic publish with tombstone delivery

**Phase 3** (Planned)
- Machine learning recommendations ("You might want to import...")
- AI-powered content generation and linting
- Real-time collaboration with CRDT
- Advanced analytics (usage patterns, dependency changes)
- Integration with Choco's broader content ecosystem

---

## Conformance Statement

**TAD v1.1.0 Alignment**: This specification is fully aligned with TAD v1.1.0. Both documents establish Phase 1 as identity + schema + verification, and Phase 2 as rendering + CRDT + discovery.

**Canonical Serialisation**: Conformant with specs/canonical-serialisation-v1.md ✓

**Failure Modes**: Implements FM-01 through FM-07 in Phase 1; FM-08 in Phase 2

---

## References

- **TAD v1.1.0**: specs/tad-v1.1.0.md
- **Canonical Serialisation**: specs/canonical-serialisation-v1.md
- **Implementation**: github.com/stratt-hq/stratt-run (Layer 5 CLI)
- **Bridge Specs (Phase 2)**: IC-01, IC-02, IC-03, AC-01/AC-02 (to follow)

---

**Document Version**: 1.1.0  
**Last Updated**: 2026-03-31  
**Status**: Stable  
**Maintained By**: stratt-hq team
