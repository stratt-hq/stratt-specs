# STRATT Canonical Serialisation Specification

> **Spec identifier:** `stratt-canonical-v1`
> **Status:** Normative
> **Version:** 1.0.0
> **Date:** 2026-03-28
> **Author:** null0
> **Resolves:** UA-06 (Deterministic YAML serialisation underspecified)
> **Blocks:** Critical path dependency #1 — this spec MUST be frozen before any `stratt publish`

---

## 1. Purpose and Scope

### 1.1 Problem Statement

The STRATT Technical Architecture Document (TAD, Layer 0) defines Blake3 fingerprinting with the following prose:

> *Canonical serialisation of the prompt unit YAML excluding the fingerprint field itself. Serialisation is deterministic: keys sorted alphabetically, no trailing whitespace, UTF-8 encoding enforced.*

This prose is insufficient for reproducible fingerprinting. Two independent implementations parsing the same YAML file could produce different byte sequences — and therefore different Blake3 digests — due to ambiguity in YAML type resolution, key ordering semantics, Unicode normalisation form, null-handling, and whitespace treatment. If any fingerprint in the system becomes unreproducible, the entire identity layer (Layer 0) is compromised.

This document replaces the prose definition with a concrete, machine-verifiable algorithm and a reference test suite of 14 input/output pairs.

### 1.2 Design Goals

1. **Reproducibility.** Any conforming implementation MUST produce byte-identical output for the same YAML input, on any platform, in any language.
2. **Simplicity.** The algorithm uses widely-available libraries and standard operations. No custom parsers or bespoke encodings.
3. **Cross-implementation verifiability.** The reference test suite (Section 10) provides concrete input/output pairs that any implementation can validate against.
4. **Alignment with standards.** The JSON serialisation stage is compatible with RFC 8785 (JSON Canonicalization Scheme). Divergences are documented and motivated (Section 8).

### 1.3 Normative Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119.

---

## 2. Algorithm Overview

### 2.1 Pipeline

```
YAML source bytes
    │
    ▼
┌──────────────────────────────┐
│  Stage 1: YAML Parsing       │  yaml npm ^2.x, YAML 1.2 core schema
│  UTF-8 input → JS object     │  No BOM. Reject duplicates. No merge keys.
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│  Stage 2: Object Transform   │  Remove `fingerprint`. Remove nulls.
│  JS object → JS object       │  NFC-normalise all strings.
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│  Stage 3: Canonical JSON     │  Recursive key sort (UTF-16 code unit order).
│  JS object → JSON string     │  Compact JSON.stringify — no whitespace.
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│  Stage 4: Encoding           │  UTF-8. No BOM. No trailing newline.
│  JSON string → byte[]        │  Exactly: new TextEncoder().encode(json)
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│  Stage 5: Blake3 Hashing     │  256-bit digest → 64-char lowercase hex
│  byte[] → fingerprint        │
└──────────────────────────────┘
```

### 2.2 Stage Summary

| Stage | Input | Output | Key Operation |
|-------|-------|--------|---------------|
| 1 | UTF-8 bytes (YAML source) | JavaScript object | YAML 1.2 core schema parse |
| 2 | JavaScript object | JavaScript object (cleaned) | Field exclusion, null removal, NFC normalisation |
| 3 | JavaScript object (cleaned) | JSON string | Recursive key sort + compact serialisation |
| 4 | JSON string | UTF-8 byte array | Encoding (no BOM, no trailing newline) |
| 5 | UTF-8 byte array | 64-char hex string | Blake3 256-bit hash |

---

## 3. Stage 1: YAML Parsing

### 3.1 Required Library

The YAML parser MUST be the `yaml` npm package version ^2.x (currently 2.8.x). This is the reference parser for STRATT.

Non-JavaScript implementations MUST use a YAML 1.2 parser that implements the core schema identically to `yaml` ^2.x. The test suite in Section 10 serves as the conformance check.

### 3.2 Required Configuration

```javascript
import { parse } from 'yaml';

const parsed = parse(yamlString, {
  schema: 'core',    // YAML 1.2 core schema
  version: '1.2',    // YAML 1.2
  merge: false,       // Disable merge keys (<<)
  uniqueKeys: true,   // Reject duplicate keys
});
```

| Option | Value | Rationale |
|--------|-------|-----------|
| `schema` | `'core'` | YAML 1.2 core schema. `yes`/`no`/`on`/`off` are strings, not booleans. Only `true`/`false` are boolean. Timestamps are not auto-resolved to Date objects. |
| `version` | `'1.2'` | YAML 1.2 is the required version. YAML 1.1 behaviour (e.g., `yes` → `true`) is prohibited. |
| `merge` | `false` | Merge keys (`<<`) inject keys from other mappings non-deterministically. Prohibited. |
| `uniqueKeys` | `true` | Duplicate keys create ambiguity. Input with duplicate keys MUST be rejected as a parse error. |

### 3.3 Input Encoding

- The YAML source MUST be valid UTF-8.
- If the input begins with a UTF-8 BOM (bytes `0xEF 0xBB 0xBF`), the BOM MUST be stripped before parsing.
- CRLF line endings (`\r\n`) in the YAML source are normalised to LF (`\n`) by the YAML parser. This is standard YAML behaviour and does not affect the output.

### 3.4 Type Resolution Rules

The YAML 1.2 core schema resolves types as follows. Implementations MUST match this behaviour:

| YAML Value | JSON Type | Example |
|------------|-----------|---------|
| `true`, `false` | boolean | `gate: true` → `true` |
| `null`, `~`, empty value | null (then removed in Stage 2) | `martian_name:` → removed |
| Integer (no decimal point) | number (integer) | `42` → `42` |
| Quoted string | string | `"1.0.0"` → `"1.0.0"` |
| Unquoted non-reserved string | string | `null0` → `"null0"` |
| `yes`, `no`, `on`, `off` | string (NOT boolean) | `yes` → `"yes"` |
| Timestamp-like values | string (NOT Date) | `2026-03-27` → `"2026-03-27"` |
| Block scalar `\|` | string (newlines preserved) | See TV-07 |
| Block scalar `>` | string (newlines folded to spaces) | See TV-08 |

**Critical:** The `version` field (e.g., `"1.0.0"`) and `created`/`modified` fields (e.g., `"2026-03-27"`) MUST always be quoted in YAML source to ensure they are parsed as strings. The canonical serialisation algorithm does not validate quoting — it relies on the YAML core schema parsing them correctly.

### 3.5 Error Handling

Any YAML parse error is fatal. The canonical serialisation function MUST NOT produce output for malformed YAML. No fallback parsing, no error recovery.

---

## 4. Stage 2: Object Transformation

Three transformations are applied to the parsed JavaScript object, in this order:

### 4.1 Field Exclusion

Remove the `fingerprint` key at the top level of the parsed object.

```javascript
delete parsed.fingerprint;
```

The `fingerprint` field is a computed output of this algorithm — it cannot be an input to itself. No other fields are excluded. Specifically, the `modified` field (also computed) IS included because it carries meaningful metadata about when the content was last changed.

**Exclusion list (exhaustive):** `fingerprint`. This list has exactly one entry.

### 4.2 Null Removal

Any key whose value is `null` MUST be removed, recursively, at all nesting levels. This applies to:
- Explicit YAML `null` values
- YAML `~` values
- YAML empty values (bare key with no value)

```javascript
function removeNulls(obj) {
  if (Array.isArray(obj)) return obj.map(removeNulls);
  if (obj !== null && typeof obj === 'object') {
    const result = {};
    for (const [key, value] of Object.entries(obj)) {
      if (value !== null && value !== undefined) {
        result[key] = removeNulls(value);
      }
    }
    return result;
  }
  return obj;
}
```

**Rationale:** In the STRATT schema, absent optional fields are semantically "not present". Serialising them as `null` would create a different byte sequence from omitting them entirely, breaking reproducibility when authors add optional fields to some units but not others.

### 4.3 Empty Arrays and Objects

Empty arrays (`[]`) and empty objects (`{}`) MUST be preserved. They are NOT removed.

An empty `imports: []` is semantically distinct from the absence of an `imports` field. An empty `tags: []` means "explicitly no tags", not "tags field omitted".

### 4.4 Unicode NFC Normalisation

All string values MUST be normalised to Unicode NFC (Canonical Decomposition followed by Canonical Composition) per Unicode Standard Annex #15.

This normalisation is applied recursively:
- To all string **values** at every nesting level
- To all string **keys** at every nesting level (STRATT keys are ASCII, but this rule ensures correctness if non-ASCII keys are ever introduced)
- To string elements within arrays

```javascript
function normaliseNFC(obj) {
  if (typeof obj === 'string') return obj.normalize('NFC');
  if (Array.isArray(obj)) return obj.map(normaliseNFC);
  if (obj !== null && typeof obj === 'object') {
    const result = {};
    for (const [key, value] of Object.entries(obj)) {
      result[key.normalize('NFC')] = normaliseNFC(value);
    }
    return result;
  }
  return obj;
}
```

**Rationale:** The same visual string can have multiple valid Unicode byte representations. For example, `é` can be encoded as:
- U+00E9 (precomposed: LATIN SMALL LETTER E WITH ACUTE) — 2 UTF-8 bytes
- U+0065 U+0301 (decomposed: `e` + COMBINING ACUTE ACCENT) — 3 UTF-8 bytes

Without NFC normalisation, these produce different Blake3 digests. NFC normalisation collapses both to the precomposed form, ensuring identical output. See TV-13a/TV-13b in the test suite.

---

## 5. Stage 3: Canonical JSON Serialisation

### 5.1 Recursive Key Sorting

At every nesting level, object keys MUST be sorted in **UTF-16 code unit order**. This is the default sort order of ECMAScript's `Array.prototype.sort()` with no comparator.

```javascript
function sortKeysDeep(obj) {
  if (obj === null || obj === undefined) return obj;
  if (Array.isArray(obj)) return obj.map(sortKeysDeep);
  if (typeof obj === 'object') {
    const result = {};
    for (const key of Object.keys(obj).sort()) {
      result[key] = sortKeysDeep(obj[key]);
    }
    return result;
  }
  return obj;
}
```

**Key rules:**
- Object keys are sorted. Array element order is preserved (arrays are NOT sorted).
- Each element within an array that is an object has its keys sorted recursively.
- Sort order is UTF-16 code unit order, NOT locale-aware collation. For ASCII-only keys (which covers all current STRATT schema keys), this is identical to byte-order sorting.
- This matches RFC 8785 Section 3.2.3.

**Why UTF-16 code unit order instead of UTF-8 byte order:** UTF-16 code unit order is the native sort order of JavaScript and matches RFC 8785. For all characters in the Basic Multilingual Plane (U+0000 to U+FFFF), UTF-16 code unit order and UTF-8 byte order produce identical results. They diverge only for supplementary characters (U+10000+), which do not appear in STRATT schema keys. Using the JS-native order means `Object.keys(obj).sort()` is the correct and complete implementation — no custom comparator is needed.

### 5.2 JSON Serialisation

The sorted object is serialised to JSON using `JSON.stringify` with no `space` argument, producing compact JSON with no whitespace between tokens.

```javascript
const canonicalJson = JSON.stringify(sortedObject);
```

**Rules:**
- No whitespace between tokens (no spaces after `:` or `,`)
- String escaping follows ECMAScript default (`JSON.stringify` behaviour), which matches RFC 8785
- Boolean values serialise as `true` or `false`
- Integer values serialise without decimal point or exponent (e.g., `42`, not `42.0`)
- Unicode characters in strings are NOT escaped to `\uXXXX` unless the ECMAScript spec requires it (control characters U+0000–U+001F, `"`, `\`). All other Unicode characters appear as literal UTF-8 in the output. This matches `JSON.stringify` default behaviour.

### 5.3 Implementation Strategy

The recommended approach is **pre-sort then stringify**: recursively rebuild the object with sorted keys, then call `JSON.stringify()`. This works because ECMAScript (ES2015+) mandates that `Object.keys()` returns own enumerable string-keyed properties in insertion order. Since we control insertion order via the sorted rebuild, `JSON.stringify` output is deterministic.

Do NOT use a `JSON.stringify` replacer function for key ordering. The replacer approach has subtle edge cases with array elements and nested objects that make it harder to verify correctness.

---

## 6. Stage 4: Encoding

The canonical JSON string from Stage 3 is encoded to a byte array as UTF-8.

```javascript
const utf8Bytes = new TextEncoder().encode(canonicalJson);
```

**Rules:**
- Encoding MUST be UTF-8.
- No UTF-8 BOM (U+FEFF / bytes `0xEF 0xBB 0xBF`) SHALL be prepended.
- No trailing newline (`\n`) SHALL be appended. The byte sequence is **exactly** the UTF-8 encoding of the compact JSON string — nothing more, nothing less.
- In Node.js/Bun, `Buffer.from(canonicalJson, 'utf-8')` is an equivalent alternative.

**Clarification on trailing newlines:** The original TAD prose mentions "no trailing whitespace". This spec clarifies: the byte sequence fed to Blake3 has no trailing whitespace of any kind — no spaces, no tabs, no newlines. If the canonical JSON is written to a file for human inspection, a trailing newline MAY be added for POSIX compliance, but that file is NOT the hash input.

---

## 7. Stage 5: Blake3 Hashing

### 7.1 Algorithm

The hash algorithm is **Blake3** (not Blake2, not SHA-256, not SHA-3).

### 7.2 Input

The exact byte array from Stage 4. No additional framing, length prefixing, or domain separation.

### 7.3 Output Format

- 256-bit digest (32 bytes)
- Encoded as lowercase hexadecimal: 64 characters, `[0-9a-f]`
- Stored in the unit's `fingerprint` field as `blake3:{hex_digest}`

Example: `blake3:a1d94a025f820f161ce61bbc6c6a7e04d3a0d3bbf2ad289a061c89067b21dcfc`

### 7.4 Recommended Libraries

| Environment | Library | Notes |
|-------------|---------|-------|
| Node.js (native) | `blake3` npm (v3.x) | Uses native bindings. Fastest. |
| Bun | `blake3` npm or `blake3-wasm` | Check Bun compatibility. |
| Browser / WASM | `blake3-wasm` (v3.x) | Pure WebAssembly. Portable. |
| Python | `blake3` PyPI (v1.x) | Reference implementation bindings. |
| Rust | `blake3` crate | The Blake3 reference implementation. |

### 7.5 SPUH Header Truncation

The STRATT Protocol Unit Header (SPUH) stores only the first 64 bits (16 hex characters) of the Blake3 digest for fast routing and filtering. Verification MUST always use the full 256-bit digest from the unit payload, never the SPUH prefix.

---

## 8. RFC 8785 (JCS) Divergence Analysis

RFC 8785 defines JSON Canonicalization Scheme (JCS), the only standardised algorithm for producing a deterministic JSON serialisation. STRATT's canonical serialisation is designed to align with JCS wherever possible, diverging only where the YAML-to-JSON bridge requires it.

### 8.1 Divergence Table

| Aspect | RFC 8785 (JCS) | STRATT Canonical (`stratt-canonical-v1`) | Divergence? | Rationale |
|--------|----------------|------------------------------------------|-------------|-----------|
| **Input format** | JSON text | YAML text | **Yes** | STRATT units are authored in YAML. The YAML-to-JSON conversion (Stage 1) is the novel contribution of this spec. |
| **Pre-processing** | None | Field exclusion + null removal + NFC (Stage 2) | **Yes** | The `fingerprint` field cannot hash itself. Null removal enforces the "absent = omitted" semantic. NFC prevents Unicode encoding drift. |
| **Key sort order** | UTF-16 code unit order | UTF-16 code unit order | No | Aligned. |
| **Compact format** | Yes (no whitespace) | Yes (no whitespace) | No | Aligned. |
| **String escaping** | ECMAScript default | ECMAScript default | No | Aligned. |
| **Number format** | IEEE 754 double → ES serialise | Integer only (schema constraint) | No | STRATT schema has no floating-point fields. This is a narrower input domain, not a divergence. |
| **Unicode in strings** | Literal (not escaped) | Literal (not escaped) + NFC | **Yes** | JCS does not normalise Unicode. STRATT applies NFC to prevent encoding-variant fingerprint drift. |
| **Trailing data** | None | None | No | Aligned. |
| **Null values** | Preserved as `null` | Removed | **Yes** | JCS preserves JSON null. STRATT removes null-valued keys because absent optional fields must not appear. |
| **Array ordering** | Preserved | Preserved | No | Aligned. |

### 8.2 Summary

All divergences from RFC 8785 occur in the **pre-JSON stages** (YAML parsing, field exclusion, null removal, NFC normalisation). Once the canonical JSON string is produced by Stage 3, it is a valid JCS output for the equivalent JSON input — with one exception: null-valued keys have been removed, which JCS would preserve.

The YAML-to-JSON conversion step (Stage 1) is the primary novel contribution. JCS assumes JSON input; STRATT starts from YAML. This creates three additional requirements that JCS does not address:
1. YAML parser selection and configuration (Section 3)
2. YAML-specific type resolution (Section 3.4)
3. Field exclusion for computed fields (Section 4.1)

---

## 9. Reference Implementation

The following TypeScript implements the complete canonical serialisation algorithm. This is the reference against which all implementations MUST be validated.

```typescript
import { parse as parseYAML } from 'yaml';      // yaml npm ^2.x
import { hash } from 'blake3';                    // blake3 npm ^3.x

/**
 * Canonical serialisation of a STRATT prompt unit.
 *
 * @param yamlSource - The raw YAML source as a UTF-8 string.
 * @returns The canonical JSON string and the Blake3 fingerprint.
 * @throws On YAML parse error, non-object root, or invalid input.
 */
export function canonicalise(yamlSource: string): {
  canonicalJson: string;
  fingerprint: string;
} {
  // ── Stage 1: Parse YAML ──────────────────────────────────
  const parsed = parseYAML(yamlSource, {
    schema: 'core',
    version: '1.2',
    merge: false,
    uniqueKeys: true,
  });

  if (parsed === null || typeof parsed !== 'object' || Array.isArray(parsed)) {
    throw new Error('YAML root must be a mapping (object)');
  }

  // ── Stage 2: Transform ───────────────────────────────────
  delete parsed.fingerprint;
  const transformed = normaliseNFC(removeNulls(parsed));

  // ── Stage 3: Canonical JSON ──────────────────────────────
  const canonicalJson = JSON.stringify(sortKeysDeep(transformed));

  // ── Stage 4 + 5: Encode and Hash ─────────────────────────
  const utf8Bytes = new TextEncoder().encode(canonicalJson);
  const digest = hash(utf8Bytes).toString('hex');

  return { canonicalJson, fingerprint: `blake3:${digest}` };
}

/** Recursively remove keys with null/undefined values. */
function removeNulls(obj: unknown): unknown {
  if (Array.isArray(obj)) return obj.map(removeNulls);
  if (obj !== null && typeof obj === 'object') {
    const result: Record<string, unknown> = {};
    for (const [key, value] of Object.entries(obj as Record<string, unknown>)) {
      if (value !== null && value !== undefined) {
        result[key] = removeNulls(value);
      }
    }
    return result;
  }
  return obj;
}

/** Recursively apply Unicode NFC normalisation to all strings. */
function normaliseNFC(obj: unknown): unknown {
  if (typeof obj === 'string') return obj.normalize('NFC');
  if (Array.isArray(obj)) return obj.map(normaliseNFC);
  if (obj !== null && typeof obj === 'object') {
    const result: Record<string, unknown> = {};
    for (const [key, value] of Object.entries(obj as Record<string, unknown>)) {
      result[key.normalize('NFC')] = normaliseNFC(value);
    }
    return result;
  }
  return obj;
}

/** Recursively sort object keys in UTF-16 code unit order. */
function sortKeysDeep(obj: unknown): unknown {
  if (obj === null || obj === undefined) return obj;
  if (Array.isArray(obj)) return obj.map(sortKeysDeep);
  if (typeof obj === 'object') {
    const result: Record<string, unknown> = {};
    for (const key of Object.keys(obj as Record<string, unknown>).sort()) {
      result[key] = sortKeysDeep((obj as Record<string, unknown>)[key]);
    }
    return result;
  }
  return obj;
}
```

### 9.1 Implementation Notes

- **Transformation order matters.** Null removal MUST happen before NFC normalisation. If nulls are not removed first, `normaliseNFC` would attempt to normalise `null` values (which it handles correctly, but removing nulls first is the specified order).
- **`JSON.stringify` determinism.** Both V8 (Node.js) and JavaScriptCore (Bun) produce deterministic output for objects with controlled insertion order. This is guaranteed by the ECMAScript specification (ES2015+).
- **No `JSON.stringify` replacer.** The pre-sort approach is simpler and avoids replacer edge cases with arrays.

---

## 10. Reference Test Suite

Each test vector specifies:
- **YAML Input:** The exact YAML source text
- **Canonical JSON:** The exact JSON string produced by Stages 1–3
- **Blake3 Digest:** The 64-character lowercase hex digest

An implementation is conforming if and only if it produces the exact canonical JSON and Blake3 digest for every test vector.

---

### TV-01: Minimal valid role

**YAML Input:**
```yaml
id: "strat://dev/role/code-reviewer@1.0.0"
type: role
domain: dev
slug: code-reviewer
version: "1.0.0"
fingerprint: "blake3:placeholder"
status: stable
meta:
  title: "Code Reviewer"
  description: "Reviews code for correctness."
  tags:
    - review
    - dev
  author: null0
  created: "2026-03-27"
persona:
  lens: "Software quality"
  tone: "Direct and precise"
  behaviour:
    - "Identify bugs before they ship"
    - "Suggest improvements"
```

**Canonical JSON:**
```json
{"domain":"dev","id":"strat://dev/role/code-reviewer@1.0.0","meta":{"author":"null0","created":"2026-03-27","description":"Reviews code for correctness.","tags":["review","dev"],"title":"Code Reviewer"},"persona":{"behaviour":["Identify bugs before they ship","Suggest improvements"],"lens":"Software quality","tone":"Direct and precise"},"slug":"code-reviewer","status":"stable","type":"role","version":"1.0.0"}
```

**Blake3:** `a1d94a025f820f161ce61bbc6c6a7e04d3a0d3bbf2ad289a061c89067b21dcfc`

**Validates:** Minimal role unit. Base schema fields + persona block. `fingerprint` field excluded. Keys sorted at all levels. Array order preserved in `tags` and `behaviour`.

---

### TV-02: Minimal valid rule

**YAML Input:**
```yaml
id: "strat://core/rule/never-fabricate-citation@1.0.0"
type: rule
domain: core
slug: never-fabricate-citation
version: "1.0.0"
fingerprint: "blake3:placeholder"
status: stable
meta:
  title: "Never Fabricate Citation"
  description: "All citations must be real and verifiable."
  tags:
    - safety
    - core
  author: null0
  created: "2026-03-27"
rule:
  polarity: never
  statement: "Never fabricate, invent, or hallucinate a citation, reference, URL, or source."
  scope: global
  protected: true
```

**Canonical JSON:**
```json
{"domain":"core","id":"strat://core/rule/never-fabricate-citation@1.0.0","meta":{"author":"null0","created":"2026-03-27","description":"All citations must be real and verifiable.","tags":["safety","core"],"title":"Never Fabricate Citation"},"rule":{"polarity":"never","protected":true,"scope":"global","statement":"Never fabricate, invent, or hallucinate a citation, reference, URL, or source."},"slug":"never-fabricate-citation","status":"stable","type":"rule","version":"1.0.0"}
```

**Blake3:** `475cb5cb0cf363e1195fd0034c356b1798abd00d4213bc07de99e5098415ef53`

**Validates:** Minimal rule unit. Boolean `protected: true` serialised correctly. `core` domain. `polarity` enum value as string.

---

### TV-03: Minimal valid task

**YAML Input:**
```yaml
id: "strat://dev/task/review-pull-request@1.0.0"
type: task
domain: dev
slug: review-pull-request
version: "1.0.0"
fingerprint: "blake3:placeholder"
status: stable
meta:
  title: "Review Pull Request"
  description: "Systematic PR review for code quality."
  tags:
    - review
  author: null0
  created: "2026-03-27"
council: "council/pathfinder"
contract:
  inputs:
    - name: pr_url
      type: string
      required: true
      description: "URL of the pull request to review"
    - name: focus_areas
      type: array
      required: false
      description: "Specific areas to focus the review on"
  outputs:
    - name: review_report
      type: document
      description: "Structured review with findings"
  failure_modes:
    - condition: "PR URL is inaccessible"
      handler: abort
      message: "Cannot access the pull request."
prompt_body: "Review the pull request at {{pr_url}}. Focus on correctness, security, and maintainability."
```

**Canonical JSON:**
```json
{"contract":{"failure_modes":[{"condition":"PR URL is inaccessible","handler":"abort","message":"Cannot access the pull request."}],"inputs":[{"description":"URL of the pull request to review","name":"pr_url","required":true,"type":"string"},{"description":"Specific areas to focus the review on","name":"focus_areas","required":false,"type":"array"}],"outputs":[{"description":"Structured review with findings","name":"review_report","type":"document"}]},"council":"council/pathfinder","domain":"dev","id":"strat://dev/task/review-pull-request@1.0.0","meta":{"author":"null0","created":"2026-03-27","description":"Systematic PR review for code quality.","tags":["review"],"title":"Review Pull Request"},"prompt_body":"Review the pull request at {{pr_url}}. Focus on correctness, security, and maintainability.","slug":"review-pull-request","status":"stable","type":"task","version":"1.0.0"}
```

**Blake3:** `50cf77c8a619656dac49e341c1a70ab9efb14d4e5ad519a5fbc9ca50a521df26`

**Validates:** Minimal task unit. Contract block with inputs, outputs, failure_modes. Nested objects within arrays have sorted keys. `council` field present. `prompt_body` with template variables.

---

### TV-04: Minimal valid chain

**YAML Input:**
```yaml
id: "strat://dev/chain/sol-1-boot@1.0.0"
type: chain
domain: dev
slug: sol-1-boot
version: "1.0.0"
fingerprint: "blake3:placeholder"
status: stable
meta:
  title: "Sol 1 Boot Sequence"
  description: "Initial boot chain for dev domain activation."
  tags:
    - boot
    - dev
  author: null0
  created: "2026-03-27"
council: "council/pathfinder"
contract:
  inputs:
    - name: project_context
      type: document
      required: true
  outputs:
    - name: boot_report
      type: document
  failure_modes:
    - condition: "Missing project context"
      handler: abort
      message: "Project context document is required."
composition:
  steps:
    - id: "step-01"
      unit: "strat://dev/task/scaffold-project@1.0.0"
      agent: "WATNEY-01"
      gate: false
    - id: "step-02"
      unit: "strat://dev/task/review-pull-request@1.0.0"
      agent: "WATNEY-02"
      gate: true
      depends_on:
        - "step-01"
imports:
  - "strat://dev/task/scaffold-project@1.0.0"
  - "strat://dev/task/review-pull-request@1.0.0"
```

**Canonical JSON:**
```json
{"composition":{"steps":[{"agent":"WATNEY-01","gate":false,"id":"step-01","unit":"strat://dev/task/scaffold-project@1.0.0"},{"agent":"WATNEY-02","depends_on":["step-01"],"gate":true,"id":"step-02","unit":"strat://dev/task/review-pull-request@1.0.0"}]},"contract":{"failure_modes":[{"condition":"Missing project context","handler":"abort","message":"Project context document is required."}],"inputs":[{"name":"project_context","required":true,"type":"document"}],"outputs":[{"name":"boot_report","type":"document"}]},"council":"council/pathfinder","domain":"dev","id":"strat://dev/chain/sol-1-boot@1.0.0","imports":["strat://dev/task/scaffold-project@1.0.0","strat://dev/task/review-pull-request@1.0.0"],"meta":{"author":"null0","created":"2026-03-27","description":"Initial boot chain for dev domain activation.","tags":["boot","dev"],"title":"Sol 1 Boot Sequence"},"slug":"sol-1-boot","status":"stable","type":"chain","version":"1.0.0"}
```

**Blake3:** `e9ec3e4021046ad33142af0bc36ac286a22724877931ea3573b8693de786f233`

**Validates:** Minimal chain unit. Composition block with 2 steps. `depends_on` array present on step-02 but absent on step-01 (absent field correctly omitted). `gate: false` and `gate: true` as booleans. Import array preserves declaration order. Steps within array preserve order while each step object has sorted keys.

---

### TV-05: Minimal valid fragment

**YAML Input:**
```yaml
id: "strat://shared/fragment/soap-note-schema@1.0.0"
type: fragment
domain: shared
slug: soap-note-schema
version: "1.0.0"
fingerprint: "blake3:placeholder"
status: stable
meta:
  title: "SOAP Note Schema"
  description: "Reusable schema for SOAP note format."
  tags:
    - schema
    - medical
  author: null0
  created: "2026-03-27"
fragment_body: "## SOAP Note\n\n- **S**ubjective: Patient-reported symptoms\n- **O**bjective: Measurable findings\n- **A**ssessment: Diagnosis\n- **P**lan: Treatment plan"
```

**Canonical JSON:**
```json
{"domain":"shared","fragment_body":"## SOAP Note\n\n- **S**ubjective: Patient-reported symptoms\n- **O**bjective: Measurable findings\n- **A**ssessment: Diagnosis\n- **P**lan: Treatment plan","id":"strat://shared/fragment/soap-note-schema@1.0.0","meta":{"author":"null0","created":"2026-03-27","description":"Reusable schema for SOAP note format.","tags":["schema","medical"],"title":"SOAP Note Schema"},"slug":"soap-note-schema","status":"stable","type":"fragment","version":"1.0.0"}
```

**Blake3:** `8d21c0991b10eda3aef2402e9ae09b028e31c9a3d6bf4414018cf58e8d4b2a5e`

**Validates:** Minimal fragment unit. `fragment_body` with embedded newlines and markdown formatting. `shared` domain. No contract block, no composition block, no council, no imports.

---

### TV-06: Empty imports array

**YAML Input:**
```yaml
id: "strat://dev/task/simple-task@1.0.0"
type: task
domain: dev
slug: simple-task
version: "1.0.0"
fingerprint: "blake3:placeholder"
status: draft
meta:
  title: "Simple Task"
  description: "A task with no imports."
  tags: []
  author: null0
  created: "2026-03-27"
council: "council/pathfinder"
imports: []
prompt_body: "Do the thing."
```

**Canonical JSON:**
```json
{"council":"council/pathfinder","domain":"dev","id":"strat://dev/task/simple-task@1.0.0","imports":[],"meta":{"author":"null0","created":"2026-03-27","description":"A task with no imports.","tags":[],"title":"Simple Task"},"prompt_body":"Do the thing.","slug":"simple-task","status":"draft","type":"task","version":"1.0.0"}
```

**Blake3:** `3b3d1b24a8e8778180779b01b976a363b02b5e32670ff9a987eb6d43a915c262`

**Validates:** Empty arrays are preserved, not removed. Both `imports: []` and `tags: []` appear in canonical JSON as empty arrays. `draft` status.

---

### TV-07: YAML literal block scalar (`|`)

**YAML Input:**
```yaml
id: "strat://dev/task/block-literal@1.0.0"
type: task
domain: dev
slug: block-literal
version: "1.0.0"
fingerprint: "blake3:placeholder"
status: stable
meta:
  title: "Block Literal Test"
  description: "Tests YAML literal block scalar."
  tags:
    - test
  author: null0
  created: "2026-03-27"
council: "council/pathfinder"
prompt_body: |
  Line one of the prompt.
  Line two of the prompt.

  Line four after blank line.
```

**Canonical JSON:**
```json
{"council":"council/pathfinder","domain":"dev","id":"strat://dev/task/block-literal@1.0.0","meta":{"author":"null0","created":"2026-03-27","description":"Tests YAML literal block scalar.","tags":["test"],"title":"Block Literal Test"},"prompt_body":"Line one of the prompt.\nLine two of the prompt.\n\nLine four after blank line.\n","slug":"block-literal","status":"stable","type":"task","version":"1.0.0"}
```

**Blake3:** `cdf13e845ed64d3528c0000fafc63d448f167c717f279b712f6028f7d5fbc58b`

**Validates:** YAML literal block scalar (`|`) preserves newlines. The resolved string includes a trailing newline (YAML default clip chomping). Blank lines within the block are preserved as `\n\n`. The JSON representation uses `\n` escape sequences within the string value.

---

### TV-08: YAML folded block scalar (`>`)

**YAML Input:**
```yaml
id: "strat://dev/task/block-folded@1.0.0"
type: task
domain: dev
slug: block-folded
version: "1.0.0"
fingerprint: "blake3:placeholder"
status: stable
meta:
  title: "Block Folded Test"
  description: >
    This description uses a folded block scalar.
    It should fold newlines into spaces.
  tags:
    - test
  author: null0
  created: "2026-03-27"
council: "council/pathfinder"
prompt_body: >
  This is a folded block.
  These lines become one.

  Blank lines become newlines.
```

**Canonical JSON:**
```json
{"council":"council/pathfinder","domain":"dev","id":"strat://dev/task/block-folded@1.0.0","meta":{"author":"null0","created":"2026-03-27","description":"This description uses a folded block scalar. It should fold newlines into spaces.\n","tags":["test"],"title":"Block Folded Test"},"prompt_body":"This is a folded block. These lines become one.\nBlank lines become newlines.\n","slug":"block-folded","status":"stable","type":"task","version":"1.0.0"}
```

**Blake3:** `cffd53efba2ab32796e30b8bc4b4164f30865a1c91dc7e4023ad65a3edaff415`

**Validates:** YAML folded block scalar (`>`) folds consecutive non-empty lines into a single line separated by spaces. Blank lines within the block become `\n`. Trailing newline present (clip chomping). Both `description` and `prompt_body` use folded blocks.

---

### TV-09: Unicode in string values

**YAML Input:**
```yaml
id: "strat://dev/task/unicode-test@1.0.0"
type: task
domain: dev
slug: unicode-test
version: "1.0.0"
fingerprint: "blake3:placeholder"
status: stable
meta:
  title: "Analyse des données"
  description: "Prüft die Qualität der Übersetzung."
  tags:
    - i18n
    - qualität
  author: null0
  created: "2026-03-27"
council: "council/pathfinder"
prompt_body: "Analysez le rapport. Résumé en français."
```

**Canonical JSON:**
```json
{"council":"council/pathfinder","domain":"dev","id":"strat://dev/task/unicode-test@1.0.0","meta":{"author":"null0","created":"2026-03-27","description":"Prüft die Qualität der Übersetzung.","tags":["i18n","qualität"],"title":"Analyse des données"},"prompt_body":"Analysez le rapport. Résumé en français.","slug":"unicode-test","status":"stable","type":"task","version":"1.0.0"}
```

**Blake3:** `bd4ec06c0cbca4a1a6fbc5b84850d680d7dfdcfd420d34bdd65aa666637134be`

**Validates:** Unicode characters (accented Latin) in title, description, tags, and prompt_body. Characters appear as literal UTF-8 in the JSON output, not `\uXXXX` escapes.

---

### TV-10: Absent optional fields

**YAML Input:**
```yaml
id: "strat://dev/rule/minimal-rule@1.0.0"
type: rule
domain: dev
slug: minimal-rule
version: "1.0.0"
fingerprint: "blake3:placeholder"
status: draft
meta:
  title: "Minimal Rule"
  description: "A rule with no optional fields."
  tags: []
  author: null0
  created: "2026-03-27"
rule:
  polarity: always
  statement: "Always validate input."
  scope: domain
```

**Canonical JSON:**
```json
{"domain":"dev","id":"strat://dev/rule/minimal-rule@1.0.0","meta":{"author":"null0","created":"2026-03-27","description":"A rule with no optional fields.","tags":[],"title":"Minimal Rule"},"rule":{"polarity":"always","scope":"domain","statement":"Always validate input."},"slug":"minimal-rule","status":"draft","type":"rule","version":"1.0.0"}
```

**Blake3:** `47a03907d92185c385799977f007640e8b405d96f66a5eaace0d63f19d55ef22`

**Validates:** Optional fields `martian_name`, `modified`, `protected`, `imports`, `council` are all absent from the YAML and correspondingly absent from the canonical JSON. The `tags: []` empty array IS present. No `null` values appear anywhere in the output.

---

### TV-11: Nested fragment references via imports

**YAML Input:**
```yaml
id: "strat://neuro/task/clinical-assessment@1.0.0"
type: task
domain: neuro
slug: clinical-assessment
version: "1.0.0"
fingerprint: "blake3:placeholder"
status: stable
meta:
  title: "Clinical Assessment"
  description: "Perform a clinical assessment using imported fragments."
  tags:
    - clinical
    - neuro
  author: null0
  created: "2026-03-27"
council: "council/crick"
imports:
  - "strat://shared/fragment/soap-note-schema@1.0.0"
  - "strat://shared/fragment/intake-checklist@1.0.0"
  - "strat://neuro/fragment/cranial-nerve-exam@2.1.0"
prompt_body: "Conduct assessment using {{soap-note-schema}} and {{cranial-nerve-exam}}."
```

**Canonical JSON:**
```json
{"council":"council/crick","domain":"neuro","id":"strat://neuro/task/clinical-assessment@1.0.0","imports":["strat://shared/fragment/soap-note-schema@1.0.0","strat://shared/fragment/intake-checklist@1.0.0","strat://neuro/fragment/cranial-nerve-exam@2.1.0"],"meta":{"author":"null0","created":"2026-03-27","description":"Perform a clinical assessment using imported fragments.","tags":["clinical","neuro"],"title":"Clinical Assessment"},"prompt_body":"Conduct assessment using {{soap-note-schema}} and {{cranial-nerve-exam}}.","slug":"clinical-assessment","status":"stable","type":"task","version":"1.0.0"}
```

**Blake3:** `df712f4333738161b666cd75db20de46d5e34980d4ae081c7b7fe3f2f83c86b2`

**Validates:** Import array with 3 fragment references across different domains and versions. Array order preserved (not sorted). `neuro` domain. `council/crick` council. Cross-domain imports (`shared` fragments imported by `neuro` task).

---

### TV-12: Maximum-complexity chain with 6 steps

**YAML Input:**
```yaml
id: "strat://dev/chain/full-release-pipeline@2.1.0"
type: chain
domain: dev
slug: full-release-pipeline
version: "2.1.0"
fingerprint: "blake3:placeholder"
status: stable
meta:
  title: "Full Release Pipeline"
  martian_name: "Hermes Launch Sequence"
  description: "Complete 6-step release pipeline with gates, dependencies, and all optional fields."
  tags:
    - release
    - pipeline
    - ci-cd
    - dev
  author: null0
  created: "2026-03-27"
  modified: "2026-03-28"
council: "council/pathfinder"
contract:
  inputs:
    - name: release_branch
      type: string
      required: true
      description: "The branch to release from"
    - name: version_bump
      type: string
      required: true
      description: "Semver bump type"
      default: "patch"
    - name: dry_run
      type: boolean
      required: false
      default: false
      description: "If true, simulate without publishing"
  outputs:
    - name: release_report
      type: document
      description: "Full release summary with all step results"
    - name: published_version
      type: string
      description: "The published semver string"
  failure_modes:
    - condition: "Tests fail"
      handler: abort
      message: "Release blocked by failing tests."
    - condition: "Security scan finds critical vulnerability"
      handler: gate
      message: "Human review required for security finding."
    - condition: "Publish to npm fails"
      handler: retry
      message: "Retrying npm publish."
composition:
  steps:
    - id: "step-01"
      unit: "strat://dev/task/lint-and-format@1.0.0"
      agent: "WATNEY-01"
      gate: false
    - id: "step-02"
      unit: "strat://dev/task/run-test-suite@1.0.0"
      agent: "WATNEY-02"
      gate: false
      depends_on:
        - "step-01"
    - id: "step-03"
      unit: "strat://dev/task/security-scan@1.0.0"
      agent: "WATNEY-03"
      gate: true
      depends_on:
        - "step-02"
    - id: "step-04"
      unit: "strat://dev/task/build-artifacts@1.0.0"
      agent: "WATNEY-01"
      gate: false
      depends_on:
        - "step-03"
    - id: "step-05"
      unit: "strat://dev/task/publish-npm@1.0.0"
      agent: "WATNEY-04"
      gate: true
      depends_on:
        - "step-04"
    - id: "step-06"
      unit: "strat://dev/task/post-release-verify@1.0.0"
      agent: "WATNEY-05"
      gate: false
      depends_on:
        - "step-05"
imports:
  - "strat://dev/task/lint-and-format@1.0.0"
  - "strat://dev/task/run-test-suite@1.0.0"
  - "strat://dev/task/security-scan@1.0.0"
  - "strat://dev/task/build-artifacts@1.0.0"
  - "strat://dev/task/publish-npm@1.0.0"
  - "strat://dev/task/post-release-verify@1.0.0"
```

**Canonical JSON:**
```json
{"composition":{"steps":[{"agent":"WATNEY-01","gate":false,"id":"step-01","unit":"strat://dev/task/lint-and-format@1.0.0"},{"agent":"WATNEY-02","depends_on":["step-01"],"gate":false,"id":"step-02","unit":"strat://dev/task/run-test-suite@1.0.0"},{"agent":"WATNEY-03","depends_on":["step-02"],"gate":true,"id":"step-03","unit":"strat://dev/task/security-scan@1.0.0"},{"agent":"WATNEY-01","depends_on":["step-03"],"gate":false,"id":"step-04","unit":"strat://dev/task/build-artifacts@1.0.0"},{"agent":"WATNEY-04","depends_on":["step-04"],"gate":true,"id":"step-05","unit":"strat://dev/task/publish-npm@1.0.0"},{"agent":"WATNEY-05","depends_on":["step-05"],"gate":false,"id":"step-06","unit":"strat://dev/task/post-release-verify@1.0.0"}]},"contract":{"failure_modes":[{"condition":"Tests fail","handler":"abort","message":"Release blocked by failing tests."},{"condition":"Security scan finds critical vulnerability","handler":"gate","message":"Human review required for security finding."},{"condition":"Publish to npm fails","handler":"retry","message":"Retrying npm publish."}],"inputs":[{"description":"The branch to release from","name":"release_branch","required":true,"type":"string"},{"default":"patch","description":"Semver bump type","name":"version_bump","required":true,"type":"string"},{"default":false,"description":"If true, simulate without publishing","name":"dry_run","required":false,"type":"boolean"}],"outputs":[{"description":"Full release summary with all step results","name":"release_report","type":"document"},{"description":"The published semver string","name":"published_version","type":"string"}]},"council":"council/pathfinder","domain":"dev","id":"strat://dev/chain/full-release-pipeline@2.1.0","imports":["strat://dev/task/lint-and-format@1.0.0","strat://dev/task/run-test-suite@1.0.0","strat://dev/task/security-scan@1.0.0","strat://dev/task/build-artifacts@1.0.0","strat://dev/task/publish-npm@1.0.0","strat://dev/task/post-release-verify@1.0.0"],"meta":{"author":"null0","created":"2026-03-27","description":"Complete 6-step release pipeline with gates, dependencies, and all optional fields.","martian_name":"Hermes Launch Sequence","modified":"2026-03-28","tags":["release","pipeline","ci-cd","dev"],"title":"Full Release Pipeline"},"slug":"full-release-pipeline","status":"stable","type":"chain","version":"2.1.0"}
```

**Blake3:** `4f33756ca1cac47d55123ff7c258d91848f7e933b6f17f545b1985489b36f254`

**Validates:** Maximum-complexity chain. 6 composition steps with mixed gates, multi-step dependency chains, agent reuse (WATNEY-01 in steps 01 and 04). 3 contract inputs (including `default` values of different types: string and boolean). 3 failure modes with different handlers. 6 imports matching the 6 step units. All optional meta fields present (`martian_name`, `modified`). Version `2.1.0` (not 1.0.0). step-01 has no `depends_on` (absent = omitted), other steps do.

---

### TV-13a: NFC normalisation — precomposed input

**YAML Input:**
```yaml
id: "strat://dev/task/nfc-test@1.0.0"
type: task
domain: dev
slug: nfc-test
version: "1.0.0"
fingerprint: "blake3:placeholder"
status: stable
meta:
  title: "Résumé"
  description: "Test NFC."
  tags:
    - test
  author: null0
  created: "2026-03-27"
council: "council/pathfinder"
prompt_body: "Café"
```

**Canonical JSON:**
```json
{"council":"council/pathfinder","domain":"dev","id":"strat://dev/task/nfc-test@1.0.0","meta":{"author":"null0","created":"2026-03-27","description":"Test NFC.","tags":["test"],"title":"Résumé"},"prompt_body":"Café","slug":"nfc-test","status":"stable","type":"task","version":"1.0.0"}
```

**Blake3:** `fe2a7d7a235abd952ef948c6451fd5814bbcd8a47994a1ab4548684a2c5bdfbb`

### TV-13b: NFC normalisation — decomposed input (MUST match TV-13a)

**YAML Input:**

This vector uses the same logical content as TV-13a, but the Unicode characters are in NFD (decomposed) form:
- `é` is encoded as U+0065 (e) + U+0301 (combining acute accent) instead of U+00E9 (precomposed é)
- This applies to `Résumé` in the title and `Café` in the prompt_body

**Canonical JSON:** Identical to TV-13a (NFC normalisation collapses decomposed to precomposed)

**Blake3:** `fe2a7d7a235abd952ef948c6451fd5814bbcd8a47994a1ab4548684a2c5bdfbb`

**Validates:** NFC normalisation ensures that precomposed (NFC) and decomposed (NFD) Unicode inputs produce identical canonical JSON and identical Blake3 digests. This is the primary motivation for Stage 2's NFC normalisation step. Without it, the same visual content would produce different fingerprints depending on the text editor or operating system used to author the YAML.

---

### TV-14: Reverse-alphabetical key order stress test

**YAML Input:**
```yaml
version: "1.0.0"
type: fragment
status: stable
slug: reverse-order
meta:
  title: "Reverse Order Test"
  tags:
    - test
  description: "Keys in reverse alphabetical order."
  created: "2026-03-27"
  author: null0
id: "strat://shared/fragment/reverse-order@1.0.0"
fragment_body: "Content here."
fingerprint: "blake3:placeholder"
domain: shared
```

**Canonical JSON:**
```json
{"domain":"shared","fragment_body":"Content here.","id":"strat://shared/fragment/reverse-order@1.0.0","meta":{"author":"null0","created":"2026-03-27","description":"Keys in reverse alphabetical order.","tags":["test"],"title":"Reverse Order Test"},"slug":"reverse-order","status":"stable","type":"fragment","version":"1.0.0"}
```

**Blake3:** `28b0c66e281b23636d076959298629d417052eaf5015391282a814b01c685aa8`

**Validates:** Keys in YAML are deliberately in reverse alphabetical order at both top-level and within `meta`. The canonical JSON has all keys sorted correctly: `domain` before `fragment_body` before `id`, and within meta: `author` before `created` before `description`. This verifies that key ordering in the YAML source does not affect the canonical output.

---

## 11. Migration Strategy

### 11.1 Problem

If this specification must be revised after content has been published with `stratt-canonical-v1` fingerprints, all existing fingerprints become invalid under the new algorithm. A "flag day" migration (switch everything at once) is unacceptable because:
- Downstream consumers may cache fingerprints
- CI pipelines verify fingerprints on every build
- MERIDIAN renders fingerprint verification status on every page

### 11.2 Spec Versioning

Every published fingerprint is implicitly associated with a canonical serialisation spec version. The initial version is `stratt-canonical-v1`.

The spec version is NOT stored in each unit's YAML (to avoid circular dependency — the spec version would change the fingerprint). Instead, the spec version is recorded:
- In the `@stratt/fingerprint` package's `SPEC_VERSION` constant
- In the STRATT schema package's `canonical_spec_version` metadata field
- In this document's header

### 11.3 Migration Procedure

When a new spec version (e.g., `stratt-canonical-v2`) is required:

**Step 1: Publish new spec.** Release `stratt-canonical-v2` as a new version of this document with a full changelog. The new spec MUST include its own reference test suite.

**Step 2: Implement dual-fingerprint support.** Update `@stratt/fingerprint` to compute both v1 and v2 fingerprints. Update the schema to accept a `fingerprint_migration` field:

```yaml
fingerprint: "blake3:{v2_digest}"
fingerprint_migration:
  previous_spec: "stratt-canonical-v1"
  previous_fingerprint: "blake3:{v1_digest}"
  migration_date: "2026-XX-XX"
```

**Step 3: Re-fingerprint all published units.** Run `stratt migrate fingerprints --from v1 --to v2` across all published content. This command:
1. Reads each published unit from R2
2. Computes the v2 fingerprint
3. Stores the v1 fingerprint in `fingerprint_migration`
4. Writes the v2 fingerprint to `fingerprint`
5. Re-publishes the unit

**Step 4: Transition window.** For a minimum of **90 days**, all verification tools (MERIDIAN, CLI `stratt verify`, CI pipeline) MUST accept either the v1 or v2 fingerprint. The `fingerprint_migration` block is the signal that a unit is in transition.

**Step 5: Cutover.** After the transition window:
1. Remove `fingerprint_migration` from all units
2. Update verification tools to accept only v2
3. Archive the v1 spec as superseded

### 11.4 Constraints on Spec Revisions

A spec revision SHOULD only be made when:
- A correctness bug is discovered (e.g., the algorithm produces different output on different platforms)
- A security issue requires changing the hash function (e.g., Blake3 is broken)
- A new YAML type is introduced that the current spec does not handle

A spec revision MUST NOT be made for cosmetic reasons (e.g., changing sort order, adding pretty-printing).

### 11.5 Tooling

| Command | Purpose |
|---------|---------|
| `stratt migrate fingerprints --from v1 --to v2` | Re-compute all fingerprints under new spec |
| `stratt verify --spec-version v1` | Verify against a specific spec version |
| `stratt fingerprint --show-spec-version` | Display which spec version would be used |

---

## 12. Security Considerations

### 12.1 Hash Collision Resistance

Blake3 provides 256-bit security against collision attacks. The birthday bound for 50% collision probability is 2^128 — effectively infinite for any practical number of STRATT units.

The SPUH header stores only 64 bits of the digest for routing. The birthday bound at 64 bits is ~2^32 (~4 billion). This is acceptable for routing/filtering but MUST NOT be used for identity verification. All verification MUST use the full 256-bit digest.

### 12.2 Input Validation

The canonical serialisation function MUST NOT produce a fingerprint for invalid input:
- YAML parse errors → fatal, no output
- Non-object YAML root → fatal, no output
- Duplicate YAML keys → fatal, no output

A fingerprint is a statement of identity. Producing a fingerprint for malformed content would allow invalid units to pass verification.

### 12.3 No Partial Fingerprinting

If any stage of the pipeline fails, no fingerprint is produced. There is no "best effort" mode. Either the full pipeline succeeds and a valid fingerprint is returned, or the function throws an error.

### 12.4 Fingerprint Replay Prevention

The fingerprint is computed from the full unit content including `id`, `version`, `domain`, `type`, and `slug`. This means:
- Two units with the same content but different addresses produce different fingerprints
- Two versions of the same unit with different content produce different fingerprints
- Copying a unit's YAML to a different address changes the fingerprint

This provides natural replay prevention: a valid fingerprint for `strat://dev/task/foo@1.0.0` is not valid for `strat://dev/task/foo@2.0.0`, even if the content is identical, because the `version` field differs.

---

## Appendix A: Test Vector Summary Table

| Vector | Type | Key Feature | Blake3 (first 16 hex) |
|--------|------|-------------|----------------------|
| TV-01 | role | Minimal role with persona | `a1d94a025f820f16` |
| TV-02 | rule | Core rule, boolean field | `475cb5cb0cf363e1` |
| TV-03 | task | Contract block, nested arrays | `50cf77c8a6196569` |
| TV-04 | chain | 2 steps, imports, depends_on | `e9ec3e4021046ad3` |
| TV-05 | fragment | Fragment body, shared domain | `8d21c0991b10eda3` |
| TV-06 | task | Empty imports `[]`, empty tags `[]` | `3b3d1b24a8e87781` |
| TV-07 | task | Literal block scalar `\|` | `cdf13e845ed64d35` |
| TV-08 | task | Folded block scalar `>` | `cffd53efba2ab327` |
| TV-09 | task | Unicode (accented Latin) | `bd4ec06c0cbca4a1` |
| TV-10 | rule | All optional fields absent | `47a03907d92185c3` |
| TV-11 | task | 3 cross-domain imports | `df712f4333738161` |
| TV-12 | chain | 6 steps, 6 imports, 3 failure modes | `4f33756ca1cac47d` |
| TV-13 | task | NFC: precomposed = decomposed | `fe2a7d7a235abd95` |
| TV-14 | fragment | Reverse key order → sorted output | `28b0c66e281b2363` |

---

## Appendix B: Changelog

### v1.0.0 — 2026-03-28

Initial release. Resolves UA-06. Establishes `stratt-canonical-v1` as the canonical serialisation algorithm for all STRATT Blake3 fingerprints.
