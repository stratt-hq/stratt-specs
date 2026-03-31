# IC-03: NATS Event Bridge for STRATT Lifecycle Events

**Status:** Approved (Wave 2B)  
**Date:** 2026-03-31  
**Scope:** Event streaming bridge between STRATT prompt orchestration and Choco workflow system via NATS JetStream

---

## Overview

IC-03 defines a standardized event bridge that allows STRATT to stream unit lifecycle events (publication, deprecation, validation, etc.) into Choco's JetStream infrastructure. This enables event-driven automation in Choco workflows: publishing a unit can trigger downstream chain executions, tamper events can trigger security audits, and import resolution can drive dependency tracking.

The bridge is designed to integrate with **Choco Block K consumer framework** (~26 items in flight) and maintain semantic versioning alignment across system boundaries.

---

## Lifecycle Events (7 types)

### 1. `unit.published`

A unit transitioned to `published` status (or created directly as published).

**Payload:**
```json
{
  "uri": "strat://dev/task/my-task@2.0.0",
  "previousStatus": "approved",
  "unit": {
    "id": "strat://dev/task/my-task@2.0.0",
    "type": "task",
    "status": "published",
    "version": "2.0.0",
    "fingerprint": "blake3:...",
    "contract": { ... }
  },
  "actor": "user:alice@example.com",
  "timestamp": "2026-03-31T10:15:00.000Z"
}
```

**Use Case:** Choco subscribers can trigger chains that depend on this unit, update documentation, or notify teams of new published versions.

---

### 2. `unit.deprecated`

A unit transitioned to `deprecated` status.

**Payload:**
```json
{
  "uri": "strat://dev/task/my-task@1.5.0",
  "unit": {
    "id": "strat://dev/task/my-task@1.5.0",
    "status": "deprecated"
  },
  "tombstone": {
    "reason": "Replaced by v2.0.0",
    "successor": "strat://dev/task/my-task@2.0.0",
    "deprecatedAt": "2026-03-31T10:15:00.000Z"
  },
  "actor": "user:alice@example.com",
  "timestamp": "2026-03-31T10:15:00.000Z"
}
```

**Use Case:** Choco can notify users about deprecated units, flag them in workflows, or automatically upgrade chains to newer versions.

---

### 3. `unit.tampered`

A unit's fingerprint failed verification (FM-01 violation detected post-publication).

**Payload:**
```json
{
  "uri": "strat://dev/task/my-task@2.0.0",
  "failureMode": "FM-01",
  "fingerprint": {
    "stored": "blake3:abc123...",
    "computed": "blake3:xyz789...",
    "timestamp": "2026-03-31T10:15:00.000Z"
  },
  "actor": "system:ci-pipeline",
  "timestamp": "2026-03-31T10:15:05.000Z"
}
```

**Use Case:** Choco security workflows can quarantine tampered units, alert admins, trigger audits, and prevent their use in new chains.

---

### 4. `unit.fingerprint.verified`

A unit's fingerprint was successfully verified during import or validation.

**Payload:**
```json
{
  "uri": "strat://dev/task/my-task@2.0.0",
  "fingerprint": "blake3:abc123...",
  "verifiedAt": "2026-03-31T10:15:00.000Z",
  "actor": "system:ci-pipeline",
  "timestamp": "2026-03-31T10:15:00.000Z"
}
```

**Use Case:** Choco auditing systems can maintain a verification log for compliance; enables trust chain tracking.

---

### 5. `unit.import.resolved`

An import reference was successfully resolved (FM-02 check passed).

**Payload:**
```json
{
  "importer": "strat://dev/task/root@1.0.0",
  "imported": "strat://shared/fragment/base@1.0.0",
  "resolvedAt": "2026-03-31T10:15:00.000Z",
  "actor": "system:ci-pipeline",
  "timestamp": "2026-03-31T10:15:00.000Z"
}
```

**Use Case:** Choco dependency tracking: build a live import graph, detect orphaned units, or alert when a unit is imported for the first time.

---

### 6. `chain.gate.fired`

A gate step in a chain was triggered (awaiting human approval).

**Payload:**
```json
{
  "chainUri": "strat://dev/chain/approval-flow@1.0.0",
  "stepId": "gate-01",
  "gateAgent": "REVIEWER-02",
  "firedAt": "2026-03-31T10:15:00.000Z",
  "context": {
    "precedingState": { ... }
  },
  "actor": "system:chain-executor",
  "timestamp": "2026-03-31T10:15:00.000Z"
}
```

**Use Case:** Choco can route gate approvals to human teams, log gate activity, or escalate based on SLA.

---

### 7. `chain.gate.resolved`

A gate step was resolved (approved or rejected).

**Payload:**
```json
{
  "chainUri": "strat://dev/chain/approval-flow@1.0.0",
  "stepId": "gate-01",
  "resolution": "approved",
  "resolvedBy": "user:alice@example.com",
  "resolvedAt": "2026-03-31T10:15:02.000Z",
  "context": {
    "feedback": "LGTM"
  },
  "timestamp": "2026-03-31T10:15:02.000Z"
}
```

**Use Case:** Choco resumes chain execution, records approval decision, updates audit logs.

---

## Event Envelope Format

All events follow a standard envelope structure (JSON):

```json
{
  "eventId": "evt_abc123def456",
  "eventType": "unit.published",
  "eventVersion": "1.0",
  "domain": "dev",
  "timestamp": "2026-03-31T10:15:00.000Z",
  "idempotencyKey": "evt_abc123def456",
  "source": "stratt:ci-pipeline:1",
  "payload": { ... },
  "meta": {
    "strattVersion": "0.1.0",
    "chocoVersion": "0.2.0"
  }
}
```

**Fields:**
- `eventId`: UUID v4, unique per event
- `eventType`: One of the 7 types (e.g., `unit.published`)
- `eventVersion`: Schema version for backward compatibility (currently `1.0`)
- `domain`: STRATT domain (`dev`, `neuro`, `finance`, etc.)
- `timestamp`: RFC 3339 UTC timestamp when event was emitted
- `idempotencyKey`: Must equal `eventId` for deduplication
- `source`: Identifying string for the event source (service:component:version)
- `payload`: Type-specific content (see event types above)
- `meta`: Version metadata for observability

---

## NATS Subject Mapping

Events are published to NATS JetStream subjects following this pattern:

```
strat.{domain}.{event-type}
```

**Examples:**
- `strat.dev.unit.published`
- `strat.dev.unit.deprecated`
- `strat.core.unit.tampered`
- `strat.shared.unit.fingerprint.verified`
- `strat.dev.unit.import.resolved`
- `strat.dev.chain.gate.fired`
- `strat.dev.chain.gate.resolved`

**Subject Wildcard Patterns:**
- `strat.dev.>` — all events in the `dev` domain
- `strat.*.unit.>` — all unit events across all domains
- `strat.>` — all STRATT events

**Forwarding to Choco:**
NATS will forward via a durable consumer to Choco's parallel subject structure:

```
choco.strat.{domain}.{event-type}
```

This allows Choco subscribers (Block K) to consume STRATT events without direct NATS connection; instead, they pull from a `choco.strat.*` stream backed by a mirror consumer.

---

## Delivery Guarantees

### At-Least-Once Semantics

- **JetStream Durable Consumer**: STRATT emits to a durable JetStream stream (`STRATT_EVENTS`)
- **Acknowledgment**: Subscribers must acknowledge message receipt; unacknowledged messages are redelivered after configurable timeout (default 30s)
- **Retention**: Stream retains events for 30 days or until acknowledged by all active consumers
- **Idempotency**: Subscribers **must** handle duplicate delivery via `idempotencyKey` (store received keys in a dedup window, typically 1 hour)

### Ordering Guarantees

- Events for a **single unit** (identified by URI) are ordered sequentially
- Events across **different units** have no guaranteed order
- Subscribers should not assume ordering of gate.fired vs gate.resolved across different chains

---

## Dead-Letter Queue (DLQ)

Events that fail processing go to the **poison queue**:

```
stratt.dlq.{domain}.{event-type}
```

**Example:** `stratt.dlq.dev.unit.published`

**Trigger Conditions:**
1. Subscriber fails to acknowledge within 30s (3 retries, then DLQ)
2. Subscriber explicitly NACKs with `requeue: false`
3. Event payload fails JSON schema validation

**DLQ Retention:** 7 days; requires manual intervention to reprocess or discard.

**Monitoring:** Alert on DLQ messages; include in observability dashboards.

---

## Consumer Group Patterns (Choco Block K Integration)

Choco subscribes via **durable consumer groups** on mirrored subjects:

### Consumer Group: `choco.strat.unit.published`

```yaml
stream: CHOCO_STRATT
durable: choco-unit-published
subjects:
  - "choco.strat.*.unit.published"
deliveryPolicy: new  # start with new messages
maxAckPending: 100
idleHeartbeat: 30s
```

**Handler:** Choco unit publisher service (auto-update docs, trigger dependent chains, etc.)

---

### Consumer Group: `choco.strat.unit.deprecated`

```yaml
stream: CHOCO_STRATT
durable: choco-unit-deprecated
subjects:
  - "choco.strat.*.unit.deprecated"
deliveryPolicy: last
maxAckPending: 50
```

**Handler:** Choco deprecation auditor (flag old chains, notify teams)

---

### Consumer Group: `choco.strat.unit.tampered`

```yaml
stream: CHOCO_STRATT
durable: choco-unit-tampered
subjects:
  - "choco.strat.*.unit.tampered"
deliveryPolicy: last
maxAckPending: 10
```

**Handler:** Choco security response (quarantine, alert ops team)

---

### Consumer Group: `choco.strat.chain.gate`

```yaml
stream: CHOCO_STRATT
durable: choco-chain-gate
subjects:
  - "choco.strat.*.chain.gate.>"
deliveryPolicy: new
maxAckPending: 200
```

**Handler:** Choco human approval router (send to Slack, email, dashboard)

---

## Implementation Roadmap

### Phase 1: STRATT Event Emission (TASKSET 11+)
- Modify `@stratt/graph/ci.ts` to emit events on FM-01 tamper detection
- Modify `@stratt/schema/lifecycle.ts` to emit on status transitions
- Add NATS client library (e.g., `nats ^2.12.0`) to `@stratt/cli`

### Phase 2: Choco Consumer Setup (Choco Block K)
- Create mirror consumer in Choco's NATS to forward `strat.*` → `choco.strat.*`
- Implement durable consumer groups for each event type
- Create handlers for unit.published, unit.deprecated, unit.tampered, chain.gate.*

### Phase 3: Observability
- Log all events to `@stratt/cli` telemetry
- Expose event metrics (events/sec, latency, DLQ count)
- Dashboard: event flow by domain and type

---

## Schema Evolution

Events use semantic versioning in `eventVersion` field:

- **1.0.x**: Current version; backward-compatible payload changes only
- **2.0.0**: Breaking changes; requires new consumer group
- Consumers **must** validate `eventVersion` and error gracefully on unknown versions

---

## Security Considerations

1. **Authentication:** NATS server uses mTLS for Choco consumer connections
2. **Encryption:** All events in transit encrypted via NATS TLS
3. **Authorization:** NATS permission model restricts Choco to read-only on `choco.strat.*` subjects
4. **Event Sanitization:** Payloads omitted of secrets; only URIs, timestamps, and structural metadata included

---

## Observability & Monitoring

### Metrics to Track

- `strat_events_emitted_total` (counter, by event_type and domain)
- `strat_events_latency_ms` (histogram, time from occurrence to emission)
- `nats_consumer_lag` (gauge, Choco consumer lag behind stream)
- `stratt_dlq_messages_total` (counter, dead-letter queue depth)

### Example Alert Rules

- **DLQ Spike**: Alert if `stratt_dlq_messages_total` increases by >10 messages in 5 min
- **Consumer Lag**: Alert if `nats_consumer_lag > 1000` for >10 min
- **Emission Delay**: Alert if p99 latency > 5s

---

## References

- TAD v1.1.0: Unit model and lifecycle
- AC-01/AC-02: 9-state lifecycle model
- Choco Block K Consumer Framework: Durable consumer patterns
- NATS JetStream: `https://docs.nats.io/nats-concepts/jetstream`

---

## Revision History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2026-03-31 | Initial spec; 7 event types, NATS subject mapping, Choco consumer groups |
