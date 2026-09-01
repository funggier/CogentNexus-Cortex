# 10 — Detailed Work Breakdown and Dependency Map

## 1. Purpose

แปลง architecture เป็น work packages ที่สามารถเปิด task/issue และตรวจ dependency ได้โดยไม่ต้องตีความใหม่ทุกครั้ง

## 2. Dependency overview

```text
A Foundation
│
├─→ B Domain Contracts
│    ├─→ C Fake Provider
│    ├─→ D Scheduler
│    └─→ E Persistence/Config
│
├─→ F Inference Service
│      ├─→ G Public API
│      ├─→ H Streaming
│      └─→ I Ollama Adapter
│
├─→ J Routing/Registry
│      └─→ K Aliases
│
├─→ L Security/Policy
│
├─→ M Management API
│      └─→ N Web UI
│
└─→ O Conformance/Chaos/Performance
       └─→ P Packaging/Release Acceptance
```

## 3. Work Package A — Repository Foundation

- A1 create Python package/pyproject
- A2 lint/format/type/test tooling
- A3 CI baseline
- A4 process entry point
- A5 typed bootstrap config
- A6 structured logging

**Exit:** service starts with no provider and reports deterministic state

## 4. Work Package B — Canonical Domain Contracts

- B1 request IDs/entities
- B2 request lifecycle enum/guards
- B3 normalized request/response
- B4 Provider Attempt
- B5 Stream State
- B6 normalized errors
- B7 usage record
- B8 route decision
- B9 resource pool/lease

**Dependency:** A

## 5. Work Package C — Fake Provider

- C1 provider interface
- C2 success response
- C3 deterministic streaming
- C4 latency controls
- C5 pre-output failure
- C6 partial-stream failure
- C7 malformed data
- C8 cancellation variants

**Dependency:** B

## 6. Work Package D — Scheduler

- D1 bounded queue
- D2 one-slot lease
- D3 lifecycle integration
- D4 scheduling classes
- D5 per-client fairness
- D6 aging
- D7 deadlines
- D8 queued cancellation
- D9 running cancellation safety
- D10 process epoch fence
- D11 metrics

**Dependencies:** B, C

## 7. Work Package E — Persistence and Configuration

- E1 SQLite bootstrap
- E2 migrations
- E3 provider/model registry persistence
- E4 client policy persistence
- E5 config revision activation
- E6 audit records
- E7 request metadata retention
- E8 cleanup
- E9 restart recovery

**Dependency:** B

## 8. Work Package F — Inference Service

- F1 admission interface
- F2 route invocation
- F3 scheduler submit/grant
- F4 provider attempt creation
- F5 terminal normalization
- F6 accounting
- F7 cancellation coordination

**Dependencies:** B, C, D

## 9. Work Package G — Provider-compatible API

- G1 `/v1/models`
- G2 `/v1/chat/completions` request DTO
- G3 non-stream response
- G4 error mapping
- G5 client request/correlation metadata
- G6 support matrix tests

**Dependency:** F

## 10. Work Package H — Streaming

- H1 upstream event parser interface
- H2 normalized chunk events
- H3 SSE renderer
- H4 output exposure tracking
- H5 bounded buffer
- H6 client disconnect
- H7 partial interruption
- H8 slow-consumer policy

**Dependencies:** C, F, G

## 11. Work Package I — Ollama Adapter

- I1 live discovery of exact current API behavior
- I2 health
- I3 model discovery
- I4 non-stream invoke
- I5 stream invoke
- I6 error mapping
- I7 usage mapping
- I8 cancellation qualification
- I9 provider restart tests

**Dependencies:** C, F; H for stream acceptance

## 12. Work Package J — Registry and Routing

- J1 normalized provider registry
- J2 model registry
- J3 capability qualification
- J4 exact route resolver
- J5 health-aware eligibility
- J6 route decision audit

**Dependencies:** B, E, I

## 13. Work Package K — Model Aliases

- K1 alias config schema
- K2 `cortex:auto`
- K3 capability requirements
- K4 fallback policy
- K5 loaded-model affinity only after measurement

**Dependency:** J

## 14. Work Package L — Security and Policy

- L1 client identity
- L2 admission authorization
- L3 queue/rate limits
- L4 model/provider permissions
- L5 cloud routing guard
- L6 secret references
- L7 redaction tests
- L8 content-retention negative tests
- L9 management permissions

**Dependencies:** A, E, F

## 15. Work Package M — Management API

- M1 service status
- M2 providers/models
- M3 queue snapshot
- M4 request detail
- M5 usage
- M6 pause admission
- M7 cancel request
- M8 drain provider
- M9 config reload
- M10 management events

**Dependencies:** D, E, J, L

## 16. Work Package N — Web UI

- N1 dashboard shell
- N2 snapshot loading
- N3 live events
- N4 queue view
- N5 provider/model views
- N6 request detail
- N7 usage charts
- N8 control actions
- N9 reconnect/state reconstruction

**Dependency:** M

## 17. Work Package O — Validation

- O1 unit suite
- O2 provider contract suite
- O3 API conformance
- O4 client SDK conformance
- O5 chaos matrix
- O6 security negative suite
- O7 direct-vs-Cortex benchmark
- O8 overload/backpressure benchmark
- O9 soak test
- O10 X-Agent integration acceptance

**Dependencies:** feature packages as relevant

## 18. Work Package P — Packaging/Release

- P1 executable/package strategy
- P2 state/config paths
- P3 fresh install
- P4 install-over/migration
- P5 graceful drain/service lifecycle
- P6 backup/restore
- P7 release evidence report
- P8 frozen candidate acceptance

**Dependency:** O

## 19. Critical path

สำหรับ minimal usable V1:

```text
A → B → C → D → F → G → H → I → O → P
```

E/J/L/M/N สามารถเริ่มขนานบางส่วนหลัง interfaces นิ่ง แต่ไม่ควรปล่อย release โดยไม่มี minimum security/config/operations

## 20. Recommended task granularity

หนึ่ง implementation task ควร:

- มี behavior/invariant เดียวชัด
- มี RED test หรือ acceptance fixture ก่อน fix/implementation เมื่อเหมาะสม
- จบด้วย evidence เช่น tests/benchmark/contract output
- ไม่รวม refactor unrelated จำนวนมาก

## 21. Definition of Done per task

- contract understood
- tests added/updated
- implementation minimal
- relevant tests green
- no direct dependency-rule violation
- docs updated if public/canonical behavior changed
- exact files/commit recorded
- known limitations explicit

## 22. System-level Done

Cortex V1 ไม่ถือว่า complete เพียงเพราะ work packages code complete ต้องผ่าน frozen-candidate acceptance โดย target client สามารถใช้ Cortex แทน direct Ollama endpoint ใน supported scenarios และ chaos/security invariants ผ่านครบ
