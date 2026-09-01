# 08 — Test, Chaos, Performance, and Acceptance Plan

## 1. Goal

พิสูจน์ Cortex จาก contracts และ failure behavior ไม่ใช่จาก happy path เท่านั้น โดยแยก correctness, compatibility, chaos และ performance เพื่อให้ปัญหาแต่ละชนิดวินิจฉัยได้ง่าย

## 2. Test pyramid

### Unit

- ontology/state transitions
- routing rules
- error normalization
- policy evaluation
- fairness algorithm
- config validation

### Contract

- Provider Adapter interface
- Public API DTO/response/error
- Management API
- persistence repositories

### Integration

- API → Inference Service → Scheduler → Fake Provider
- SQLite/config
- Web event stream

### Conformance

- OpenAI-compatible client behavior
- streaming framing
- exact/alias semantics

### Real-provider

- Ollama local qualification

### Chaos

- process/provider/network failures

### Performance

- latency/throughput/memory/backpressure

## 3. Core invariant tests

Must test:

- public API cannot bypass scheduler
- slot=1 never overlaps provider attempts
- exact model cannot silently fallback
- unqualified capability rejected
- client priority hint cannot self-escalate
- partial stream not success
- stale lease ignored
- invalid config not activated
- prompt/output not retained by default

## 4. Chaos matrix

Inject:

1. provider down before request
2. provider dies before first token
3. provider dies after first token
4. provider sends malformed stream
5. client disconnects before output
6. client disconnects mid-stream
7. cancel ignored by provider
8. Cortex killed mid-request
9. Cortex restarted with stale execution records
10. queue flooded
11. slow consumer
12. invalid hot reload
13. DB unavailable during noncritical telemetry
14. provider health flaps

ทุก scenario ต้องมี expected state/error/slot outcome

## 5. Performance baseline

วัดแยก overhead ของ Cortex จาก model latency

metrics:

- API admission overhead
- queue scheduling overhead
- streaming proxy overhead
- TTFT delta vs direct provider
- throughput delta
- memory per queued request
- memory per active stream
- CPU idle/load

## 6. Comparison benchmark

สำหรับ local Ollama:

```text
Direct Client → Ollama
vs
Client → Cortex → Ollama
```

ใช้ request set เดียวกันหลายรอบและรายงาน distribution ไม่ใช้ sample เดียว

## 7. Backpressure benchmark

สร้าง clients มากกว่า capacity และพิสูจน์:

- memory bounded
- queue rejection predictable
- service management API responsive
- no starvation violation

## 8. Long-running soak

อย่างน้อยเมื่อ V1 ใกล้ release:

- repeated stream/non-stream requests
- provider restart cycles
- config reload cycles
- client connect/disconnect

ตรวจ memory growth, orphan leases, DB growth, stuck queue

## 9. Fuzz/property testing candidates

- stream frame fragmentation
- request field combinations
- routing candidate ordering
- scheduler fairness invariants
- state transition legality

## 10. Acceptance suite groups

### A — API
Supported API behavior matches matrix

### B — Scheduler
Capacity/fairness/backpressure correct

### C — Provider
Ollama contract qualified

### D — Streaming
No corruption/false completion

### E — Security
local default/authorization/retention/redaction

### F — Recovery
restart/config/provider failure outcomes explicit

### G — Operations
monitor/control/drain work

### H — Integration
X-Agent can point provider config to Cortex and complete representative model calls

## 11. Release evidence

release candidate report SHOULD include:

- exact commit SHA
- Python/dependency versions
- Ollama version
- tested model identity
- compatibility matrix
- test counts/results
- chaos results
- benchmark summary
- known limits

## 12. Exit criteria

V1 acceptance is PASS only when all MUST invariants pass and no unresolved critical failure allows:

- capacity bypass
- hidden reroute
- false stream success
- secret/content leak under default policy
- unauthorized management control
- unbounded resource growth in tested paths
