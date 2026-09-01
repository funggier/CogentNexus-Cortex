# 11 — Threat Model, Reliability, and Failure Taxonomy

## 1. Purpose

Cortex อยู่ระหว่าง clients กับ intelligence providers จึงต้องรับมือทั้ง reliability failures และ security abuse โดย failure ต้องถูกจำแนกให้ operator และ client เข้าใจได้ ไม่ถูกซ่อนด้วย generic `500` หรือ blind retry

## 2. Trust boundaries

```text
Untrusted/partially trusted Client
        │
        ▼
Cortex Public API
        │
        ▼
Cortex Core Services
        │
        ▼
Provider Adapter
        │
        ▼
External/Local Provider
```

Management API เป็น trust boundary แยกที่มีอำนาจมากกว่า ordinary inference client

## 3. Threat classes

### T1 — Request flooding

client ส่ง requests จำนวนมากจน queue/memory โต

Mitigations:
- admission limits
- per-client rate/queue caps
- bounded request size
- backpressure

### T2 — Priority abuse

client อ้าง interactive/high priority เพื่อผูกขาด slot

Mitigation: server-side scheduling policy

### T3 — Oversized context/tool schema

request ขนาดใหญ่ทำ parsing/memory/tokenization แพงก่อน admission

Mitigations: body/count/estimated-token limits ก่อน heavy work

### T4 — Provider credential exposure

ผ่าน logs/UI/config dump/error response

Mitigations: opaque references, redaction, secret storage, negative tests

### T5 — Hidden cloud exfiltration

local client คิดว่าใช้ local แต่ alias fallback ไป cloud

Mitigation: explicit routing/data-residency policy; cloud fallback off unless allowed

### T6 — Malformed provider stream

upstream ส่ง truncated/invalid frames ทำ parser crash หรือ stream corruption

Mitigation: defensive parser, bounded buffers, provider protocol error

### T7 — Slow consumer

client ไม่อ่าน stream ทำ memory growth

Mitigation: bounded per-stream buffer + timeout/termination policy

### T8 — Stale lease completion

old worker/attempt complete หลัง slot/request ถูก supersede/restart

Mitigation: process epoch + lease generation/fencing

### T9 — Configuration poisoning/error

invalid route/policy reload ทำ outage

Mitigation: validate-stage-activate, keep prior active revision

### T10 — Management surface misuse

browser/client สั่ง disable/drain/config change โดยไม่มี authority

Mitigation: separate management auth + audit + least privilege

## 4. Reliability failure taxonomy

### Client-origin failures

```text
INVALID_REQUEST
UNSUPPORTED_FEATURE
AUTHENTICATION_FAILED
AUTHORIZATION_FAILED
QUOTA_EXCEEDED
DEADLINE_EXPIRED
```

### Cortex admission/scheduler failures

```text
QUEUE_FULL
QUEUE_TIMEOUT
RESOURCE_UNAVAILABLE
POLICY_REJECTED
CONFIGURATION_UNAVAILABLE
```

### Provider failures

```text
PROVIDER_UNAVAILABLE
PROVIDER_RATE_LIMITED
PROVIDER_TIMEOUT
MODEL_NOT_FOUND
MODEL_LOAD_FAILED
PROVIDER_PROTOCOL_ERROR
PROVIDER_CANCEL_UNCERTAIN
```

### Response transport failures

```text
CLIENT_DISCONNECTED
STREAM_INTERRUPTED
SLOW_CONSUMER_TERMINATED
```

### Internal failures

```text
PERSISTENCE_ERROR
INTERNAL_INVARIANT_VIOLATION
ADAPTER_FAILURE
UNEXPECTED_ERROR
```

## 5. Failure principles

1. timeout ไม่เท่ากับ “provider ไม่ได้ทำงาน”
2. client disconnect ไม่เท่ากับ provider failure
3. provider success ไม่เท่ากับ client received full response
4. partial stream ไม่เท่ากับ success
5. cancellation request ไม่เท่ากับ cancellation confirmed
6. health endpoint success ไม่เท่ากับ target model callable
7. retry ต้องมี reason + bounded attempt

## 6. Fail-open vs fail-closed

### Fail closed

- authentication/authorization
- forbidden provider/cloud route
- exact model mismatch
- invalid config activation
- unsupported capability that changes semantics

### May degrade/fail open carefully

- optional telemetry sink
- noncritical historical metrics
- UI event feed

Cortex ต้องไม่หยุด inference เพียงเพราะ dashboard disconnected

## 7. Overload behavior

ต้องเลือก degrade อย่าง intentional:

1. reject new batch/background first ตาม policy
2. cap queue
3. preserve management/control responsiveness
4. prevent telemetry from consuming unbounded memory
5. expose `DEGRADED/OVERLOADED` status

อย่าปล่อย OS OOM เป็น backpressure mechanism

## 8. Provider outage

เมื่อ provider unhealthy:

- stop new route ไป provider ตาม health policy
- requests exact-pinned อาจ fail/queue ตาม explicit config
- alias requests may route alternative if allowed
- active attempts resolve independently
- health recovery requires probe/qualification threshold เพื่อลด flap

## 9. Flapping protection

health state SHOULD use debounce/hysteresis เช่น require N successes before HEALTHY และ N failures before UNHEALTHY ตาม provider type

## 10. Circuit breaker

future/provider cloud สามารถใช้ circuit breaker แต่ต้องไม่ซ่อน root cause

states conceptual:

```text
CLOSED → OPEN → HALF_OPEN → CLOSED
```

route decision ต้องรู้ breaker state

## 11. Data integrity threats

SQLite/config:

- partial writes
- disk full
- corrupt DB
- incompatible schema

startup/readiness ต้องตรวจและ fail explicitly

critical config/audit persistence failure ไม่ควรถูก ignore ถ้าทำให้ policy authorityไม่แน่นอน

## 12. Privacy threats

- prompt accidentally logged
- output shownใน admin UI ที่ผู้ใช้ไม่ตั้งใจ
- request metadata contains sensitive path/project names

ใช้ data minimization และ permissioned detail views

## 13. Supply/upstream changes

Provider update อาจเปลี่ยน:

- stream shape
- field semantics
- cancellation
- model naming

Adapter qualification suite คือ defense; advanced capability can degrade independently

## 14. Chaos scenarios

ต้องพิสูจน์:

- kill Cortex mid-request
- kill provider before first token
- kill provider mid-stream
- restart provider while queued
- delay provider 60s
- corrupt one stream frame
- client disconnect rapidly
- flood queue
- config reload invalid
- rotate provider state during traffic
- disk becomes unavailable for telemetry

## 15. Reliability target hierarchy

ก่อน optimize latency ให้เรียง:

1. semantic correctness
2. bounded resource use
3. failure transparency
4. isolation
5. recovery
6. latency
7. throughput

## 16. Security/reliability acceptance

- no unbounded queues/buffers in designed paths
- stale lease cannot finalize current request incorrectly
- partial stream cannot become completed
- provider outage cannot crash service loop
- one abusive client cannot starve all others indefinitely
- invalid config cannot replace good active config
- cloud route cannot occur contrary to policy
- secrets/content retention negative tests pass
- management API remains isolated from ordinary client permission
- chaos outcomes map to documented error/state taxonomy
