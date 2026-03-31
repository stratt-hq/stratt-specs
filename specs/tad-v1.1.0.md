# STRATT Technical Architecture Document (TAD) v1.1.0

**Status**: Stable  
**Version**: 1.1.0  
**Released**: 2026-03-31  
**Previous**: v1.0.0 (2026-03-27)  
**Changes**: 23 specification updates reflecting Phase 1 implementation

---

## Overview

STRATT is a prompt engineering infrastructure for versioned, validated, dependency-tracked prompt units with deterministic fingerprinting and multi-domain execution. This document is the authoritative technical reference.

---

## Change Summary (v1.0.0 → v1.1.0)

### What Changed

1. **Layer 2 (CRDT)**: Clarified as Phase 2 future work, removed from normative spec. Current implementation uses Git for version control, not Automerge.
2. **CLI Commands**: Documented 9 actual implemented commands; noted 5 commands planned for Phase 2 (graph, diff, export n8n, council list/show).
3. **Core Rules Auto-Injection**: Documented as Phase 2 feature; currently manual imports required.
4. **Agent Schema**: Added notes on validation gaps (capabilities enum, designation pattern).
5. **R2 Integration**: Clarified FM-08 as out-of-scope for Phase 1.

### What Stayed the Same

- ✓ All Layer 0 (Identity & Addressing) normative requirements
- ✓ All Layer 1 (Schema) enums and base field definitions
- ✓ All Layer 3 (DAG) operations and protection model
- ✓ All Layer 4 (Council) structure
- ✓ Canonical Serialisation (5-stage pipeline)
- ✓ All 7 Failure Modes (FM-01 through FM-07)

### Roadmap

**Phase 1** (Current, ✓ Complete):
- Layers 0, 1, 3, 4, 5, 6
- Canonical fingerprinting
- FM-01 through FM-07 (FM-08 deferred)
- 9 CLI commands

**Phase 2** (In Progress):
- Layer 2: CRDT merge engine (Yjs-based per IC-01 decision)
- 5 Missing CLI commands (graph, diff, export n8n, council list/show)
- Core rules auto-injection
- FM-08: R2 error handling

**Phase 3** (Planned):
- Execution IR and multiple backends
- Storage abstraction layer
- MCP bridge integration
- Full CRDT conflict resolution with manual review queue

---

## Architecture Layers

### Layer 0: Identity & Addressing

Every STRATT artefact has a canonical URI address and Blake3 fingerprint.

**URI Scheme**: `strat://{domain}/{type}/{slug}@{version}`

**Components**:
- `domain`: One of [dev, neuro, finance, nutrition, legal, film, artist, core, shared]
- `type`: One of [role, rule, task, chain, fragment]
- `slug`: Kebab-case identifier, max 64 chars, pattern `[a-z0-9-]+`
- `version`: Semantic version (e.g., `1.0.0`, `0.2.3`)

**Examples**:
- `strat://dev/role/hostile-reviewer@1.2.0`
- `strat://legal/chain/hab-inspection@2.0.1`
- `strat://core/rule/never-fabricate-citation@1.0.0`
- `strat://shared/fragment/soap-note-schema@1.1.0`

**Fingerprinting**:
- Algorithm: Blake3
- Format: `blake3:{64hex}`
- Computed over canonical serialisation (5-stage deterministic pipeline)
- Verified on every publish, render, and verify operation
- Mismatch sets status to `tampered`

**Versioning**:
- Strategy: Semantic versioning (major.minor.patch)
- Patch: Content edit (no contract change)
- Minor: New optional fields added
- Major: Breaking contract change
- Draft versions (0.x.x) cannot be referenced by stable units (FM-07)

---

### Layer 1: Prompt Unit Schema

All units conform to a base schema plus type-specific constraints.

**Base Schema** (all types):

```yaml
id: strat://domain/type/slug@version  # Canonical URI
type: role|rule|task|chain|fragment   # Unit type
domain: dev|neuro|finance|...         # Domain namespace
slug: kebab-case-identifier           # Human-readable slug
version: "1.0.0"                       # Semantic version (quoted)
fingerprint: blake3:...               # Computed at publish
status: draft|review|stable|deprecated|tampered|tombstoned

meta:
  title: string (max 80)
  martian_name: string (max 80, optional)
  description: string (max 500)
  tags: [string, ...]
  author: string
  created: ISO8601_date
  modified: ISO8601_date

imports:  # All dependencies declared here
  - strat://domain/type/slug@version
  - strat://...

council: council/council-slug  # Required for task, chain
```

**Type-Specific Constraints**:

**Role**:
- Required: meta, persona block
- Persona block: `lens`, `tone`, `behaviour[]`, `output_format?`

**Rule**:
- Required: meta, rule block
- Rule block: `polarity` (always|never), `statement`, `scope` (global|domain|chain|session), `protected` (bool)
- Protected rules cannot be overridden in composition

**Task**:
- Required: meta, contract, council, prompt_body
- Contract: `inputs[]`, `outputs[]`, `failure_modes[]`
- Prompt body: Markdown task definition

**Chain**:
- Required: meta, contract, council, composition
- Contract: `inputs[]`, `outputs[]`, `failure_modes[]`
- Composition: `steps[]` with step-NN ID pattern, unit refs, agent designation, gate flag, depends_on

**Fragment**:
- Required: meta, fragment_body
- No contract (reusable schema or template)
- Fragment body: Markdown reusable content

**Contract Block**:
```yaml
contract:
  inputs:
    - name: string
      type: string|integer|float|boolean|document|array|object
      required: bool
      default: any (optional)
      description: string (optional)
  outputs:
    - name: string
      type: string
      description: string (optional)
  failure_modes:
    - condition: string
      handler: gate|retry|abort|fallback
      message: string
```

**Composition Block** (chains only):
```yaml
composition:
  steps:
    - id: step-01  # Pattern: step-NN (two digits)
      unit: strat://domain/type/slug@version
      agent: AGENT-NN  # Designation format
      gate: false      # Default false
      depends_on: [step-00]  # Optional ordering
```

---

### Layer 2: CRDT Resolution (Phase 2)

**Status**: Phase 2 planning. Currently NOT implemented.

In Phase 2, multi-author concurrent editing will be supported via Yjs (per IC-01 decision). This section describes the design; implementation follows this release.

**Planned Approach**:
- Yjs for field-level CRDT resolution on top of Git
- Unit of merge: individual fields (not files)
- Field merge strategies:
  - Meta fields: Last-write-wins
  - Contract fields: Manual resolution (gates on conflict)
  - Composition steps: Append-wins (additions), LWW (edits), explicit tombstone (removals)
  - Imports: Union merge (both sides' imports retained)
  - Status: Highest restriction wins (tombstoned > deprecated > stable > review > draft)
  - Fingerprint: Recomputed post-merge
  - Protected fields: Immutable (id, slug, domain, type, created)

**Downstream Impact Scan**:
- Triggered on any change to stable unit
- Reverse dependency traversal via @stratt/graph
- Blast radius report: direct, transitive, gate-bearing, protected dependents
- PR approval required if any gate or protected dependent affected

---

### Layer 3: Dependency Graph

All units form a directed acyclic graph (DAG) defined by imports.

**Graph Operations**:

1. **Build DAG**: Construct adjacency lists from all units' imports
2. **Detect Cycles**: Kahn's algorithm + DFS for cycle path extraction
3. **Topological Sort**: Order units for sequential execution
4. **Transitive Resolution**: Full import tree for any unit
5. **Blast Radius**: Reverse BFS from changed unit, classify dependents

**Reserved Namespaces**:

**core**: Global rules auto-injected at chain execution time (Phase 2). Current: manual imports.
- Examples: `strat://core/rule/never-fabricate-citation@1.0.0`

**shared**: Cross-domain reusable Fragments. Must be explicitly imported.
- Examples: `strat://shared/fragment/soap-note-schema@1.1.0`

**Protected Units**:
- Definition: Units with `protected: true` in rule block or agent council designation
- Cannot be removed from chain composition without documented override
- Removal triggers FM-04 check (protected agent/rule presence)
- Examples: BECK-02 (reviewer), JOHANSSEN-03 (compliance guard)

**Gate Authority**:
- Definition: Agents with `gate_authority: true` in council registry
- Only these agents resolve gate checkpoints
- Gate resolution logged with timestamp and agent designation

---

### Layer 4: Agent Council Registry

Councils are versioned, first-class entities. Chains reference councils by ID (not hardcoded agents).

**Council Schema**:
```yaml
id: council/slug
domain: dev|neuro|finance|...
version: "1.0.0"
agents:
  - designation: AGENT-NN  # e.g., WATNEY-01
    role: string
    capabilities: [array of capability names]
    default_for: [chain|task|role]
    protected: bool
    gate_authority: bool
protectedAgents:
  domain-name: [AGENT-NN, ...]
gateAuthorityAgents: [AGENT-NN, ...]
```

**Registered Councils** (Phase 1):

**council/pathfinder** (dev domain, v1.0.0):
- WATNEY-01 (architect) — default for chain, task
- BECK-02 (reviewer) — protected
- JOHANSSEN-03 (documenter)
- MARTINEZ-04 (ci_cd_operator)
- VOGEL-05 (test_architect)
- LEWIS-06 (planner) — gate_authority

**council/crick** (neuro, planned)  
**council/ares-iv-finance** (finance, planned)  
**council/hab-nutrition** (nutrition, planned)  
**council/mission-control-legal** (legal, planned)  
**council/ares-film** (film, planned)  
**council/ares-arts** (artist, planned)

---

### Layer 5: CLI

The primary interface for all STRATT operations.

**Package**: @stratt/cli  
**Runtime**: Bun  
**Binary**: `stratt`  
**Global Flags**: `--json` (structured output), `--verbose` (debug logging)

**Implemented Commands** (Phase 1):

| Command | Purpose | Exit Codes |
|---------|---------|-----------|
| `stratt new <type> <domain> <slug>` | Scaffold new unit with valid skeleton | 0=success, 1=error |
| `stratt validate <path>` | Schema + import validation | 0=valid, 1=invalid, 2=tampered, 3=broken_import |
| `stratt fingerprint <path>` | Compute/verify Blake3 fingerprint | 0=verified, 2=tampered, 1=error |
| `stratt publish <path>` | Sign, fingerprint, push to R2 | 0=success, 1=error, 2=tampered, 3=broken_import |
| `stratt run <unit-address>` | Execute task/chain locally | 0=success, 1=gate_halt, 2=contract_violation, 3=model_error |
| `stratt impact <unit-address>` | Compute blast radius | 0=success, 1=error |
| `stratt deprecate <unit-address>` | Tombstone a unit | 0=success, 1=error |
| `stratt verify <unit-address>` | Fetch from R2, verify fingerprint | 0=verified, 2=tampered, 3=not_found |
| `stratt ci <paths...>` | Full 9-check CI pipeline (FM-01 through FM-07) | 0=pass, 1=fail, 2=tampered, 3=broken_import |

**Planned Commands** (Phase 2):

| Command | Purpose |
|---------|---------|
| `stratt graph <unit-address>` | Show dependency tree (or equivalence with impact) |
| `stratt diff <addr@v1> <addr@v2>` | Semantic diff between versions |
| `stratt export n8n <chain-address>` | Compile chain to n8n workflow JSON |
| `stratt council list` | List all registered councils |
| `stratt council show <council-id>` | Show council with agent capabilities |

**Command Details**:

**`stratt new <type> <domain> <slug>`**
- Scaffolds unit YAML with valid skeleton
- Flags: `--council <id>`, `--from <address>` (fork)
- All new units start at version 0.1.0 (draft)
- Validates slug uniqueness in domain

**`stratt validate <path>`**
- Validates schema via Zod
- Resolves imports
- Checks constraints (FOREBIDDEn blocks, draft isolation, etc.)
- Exit 0: PASS with field summary; Exit 1: FAIL with field-level errors

**`stratt fingerprint <path>`**
- Flags: `--verify` (compare), `--update` (write to file)
- Uses canonical serialisation pipeline from @stratt/fingerprint
- Deterministic: same unit always produces same hash

**`stratt publish <path>`**
- Validates first (aborts if invalid)
- Computes Blake3 fingerprint
- Writes fingerprint field to unit file
- Git commits fingerprint update
- Pushes unit to Cloudflare R2 at canonical path
- Triggers MERIDIAN rebuild
- Idempotent: false (append-only versioning)

**`stratt run <unit-address>`**
- Flags: `--dry-run` (validate without execution), `--gate-mode manual|auto`, `--model <string>`, `--input <json>`
- Fetches unit from R2, verifies fingerprint
- Resolves all imports
- Executes via Anthropic API (claude-sonnet-4-20250514)
- Manual gate mode: pauses at gates, requires human approval
- Auto gate mode: logs and continues
- Outputs match contract.outputs schema
- Execution log includes all gate records

**`stratt impact <unit-address>`**
- Computes reverse BFS from unit through dependency graph
- Output: direct, transitive, gate-bearing, protected dependents
- Use case: before editing shared Fragments or core Rules

**`stratt deprecate <unit-address>`**
- Required flags: `--reason <string>`, `--successor <address?>`
- Sets status to deprecated
- Writes tombstone record to unit
- MERIDIAN shows deprecation banner

**`stratt verify <unit-address>`**
- Fetches unit from R2
- Recomputes Blake3
- Compares to stored fingerprint
- Exit 0: verified, Exit 2: tampered, Exit 3: not found

**`stratt ci <paths...>`**
- Runs full 9-check CI pipeline on specified unit files
- Checks FM-01 through FM-07 (FM-08 deferred to Phase 2)
- Generates markdown blast radius report with tables and graphs

---

### Layer 6: MERIDIAN (Documentation Site)

MERIDIAN is the STRATT documentation platform at stratt.dev.

**URL**: stratt.dev  
**Framework**: Astro + Starlight  
**Hosting**: Vercel  
**Content Source**: Cloudflare R2 (published units + gists)

**Auto-Generated Pages** (from unit YAML):
- Route: `/units/{domain}/{type}/{slug}/{version}`
- Sections:
  - Unit header (id, type, status badge)
  - Fingerprint verification widget (live Blake3 check)
  - Meta block (title, description, tags, council)
  - Contract (if applicable)
  - Composition flowchart (for chains)
  - Imports dependency mini-graph
  - Version history (all published versions, diff links)
  - Deprecation banner (if deprecated)

**Manually Authored**:
- Architecture guide (this TAD)
- Council registry with agent capability matrices
- Changelog (auto-generated from git tags)
- Getting started guide

**Interactive Features** (Phase 2):
- Full dependency graph explorer (D3)
- Fingerprint verifier widget
- Chain flow visualiser
- Version diff viewer

---

## Canonical Serialisation Specification

**5-Stage Deterministic Pipeline**:

1. **YAML Parse**: Parse YAML 1.2 core schema (no merge, unique keys enforced)
2. **Remove Nulls**: Recursive null/undefined removal; keep empty arrays/objects
3. **Unicode Normalisation**: NFC on all string keys and values
4. **Key Sorting**: Recursive sort by UTF-16 code unit order (default JS `.sort()`)
5. **Encode & Hash**: UTF-8 encode, Blake3 hash, output as `blake3:{hex}`

**Implementation**: `/packages/fingerprint/src/canonicalise.ts` and `hash.ts`  
**Determinism**: Identical inputs always produce identical outputs  
**Test Vectors**: Reference vectors in `/packages/fingerprint/tests/`

---

## Failure Modes (FM-01 through FM-08)

The CI pipeline validates 8 failure modes:

| FM | Name | Trigger | Response |
|----|------|---------|----------|
| FM-01 | Fingerprint Mismatch | Computed ≠ stored hash | Status = tampered, exit 2 |
| FM-02 | Broken Import | Missing URI resolution | Exit 3, list unresolvable imports |
| FM-03 | Circular Dependency | DAG cycle detected | Exit 1, show cycle path |
| FM-04 | Protected Agent Missing | Chain missing domain's protected agent | Exit 1, require justification |
| FM-05 | Gate Removed | Gate step deleted without major bump | Exit 1, require major version |
| FM-06 | Contract Breaking | Required input removed / output type changed without major bump | Exit 1, require major version |
| FM-07 | Draft in Stable | Stable unit imports 0.x.x unit | Exit 1, require stable dependency |
| FM-08 | R2 Publish Failure | Cloudflare R2 write error | (Phase 2) Exit 4 |

**Enforcement**: All checks run on every CI push. Hard failures block PR merge.

---

## Design Principles

- **P-01 Schema First**: No unit exists without valid schema. Validation before content.
- **P-02 Identity Before Content**: Every unit gets a canonical address and fingerprint before publication.
- **P-03 Explicit Over Implicit**: All dependencies declared in imports. No ambient context.
- **P-04 Human Gates Are Architectural**: Gate checkpoints are invariants, not UX choices.
- **P-05 Blast Radius Before Merge**: Every change triggers impact scan. PR shows affected units.
- **P-06 Docs Are The Schema**: MERIDIAN renders directly from YAML. Zero documentation drift.
- **P-07 Deprecation Over Deletion**: Units never deleted. Tombstoned for audit trail.
- **P-08 Councils First-Class**: Agent teams are versioned, registered entities (not hardcoded).

---

## Roadmap & Future Work

**Phase 2** (In Progress):
- CRDT layer (Yjs-based, IC-01)
- 5 missing CLI commands
- Core rules auto-injection
- FM-08 (R2 error handling)
- Bridge specs (IC-02, IC-03, AC-01/AC-02)

**Phase 3** (Planned):
- Execution IR with multiple backends
- MCP bridge to Choco-HQ
- CRDT conflict resolution with manual queue
- Storage abstraction (Blake3+R2 vs SHA-256+PostgreSQL)

---

## References

- **Repository**: github.com/stratt-hq
- **Canonical Serialisation**: specs/canonical-serialisation-v1.md
- **Council Registry**: councils/*/council.yaml
- **Unit Examples**: packages/units/
- **DIVERGENCE_REPORT.md**: Comprehensive audit of v1.0.0 vs implementation

---

**Document Version**: 1.1.0  
**Last Updated**: 2026-03-31  
**Status**: Stable  
**Maintained By**: stratt-hq team
