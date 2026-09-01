# CogentNexus-Cortex Planning Pack — Document Manifest

## Scope

Manifest นี้เป็นสารบัญของ design baseline สำหรับ `planning/intelligence-provider-gateway/`

ชุดเอกสารแบ่งเป็นสองหมวด:

- `knowledge/` — architecture, semantics, contracts, risks และ principles
- `development/` — implementation sequence, testing, release และ governance

## Root documents

1. `README.md` — overview, responsibility boundary, invariants, structure และ reading order
2. `DOCUMENT-MANIFEST.md` — manifest นี้

## Knowledge documents

1. `knowledge/00-vision-principles-and-invariants.md` — เหตุผลที่ Cortex แยกเป็น intelligence infrastructure และ invariants
2. `knowledge/01-canonical-ontology-and-request-state-model.md` — Client, Inference Request, Provider Attempt, Lease, Stream และ state model
3. `knowledge/02-system-boundary-and-service-architecture.md` — API/Application/Scheduler/Router/Registry/Adapter/Management boundaries
4. `knowledge/03-provider-api-compatibility-and-contracts.md` — OpenAI-compatible contract, supported fields, errors, streaming และ conformance
5. `knowledge/04-inference-scheduler-fairness-and-backpressure.md` — one-slot baseline, fairness, bounded queue, deadlines, cancellation
6. `knowledge/05-model-routing-aliases-registry-and-failover.md` — exact models, semantic aliases, qualification, route decisions, failover
7. `knowledge/06-provider-adapter-contract-and-ollama-mapping.md` — provider anti-corruption layer และ Ollama V1 mapping principles
8. `knowledge/07-streaming-timeout-cancellation-retry-and-idempotency.md` — output exposure boundary, retry zones, timeout/cancel semantics
9. `knowledge/08-security-auth-policy-tenancy-and-secret-management.md` — identity, authorization, local secure default, privacy, secrets
10. `knowledge/09-control-monitoring-web-ui-and-observability.md` — Web dashboard, management API, metrics, tracing, reconnect semantics
11. `knowledge/10-persistence-configuration-recovery-and-operations.md` — SQLite/config revisions, restart semantics, migrations, retention
12. `knowledge/11-threat-model-reliability-and-failure-taxonomy.md` — trust boundaries, threats, failure taxonomy, chaos scenarios

## Development documents

1. `development/00-development-strategy-and-v1-boundary.md` — V1 scope/non-goals, gates และ implementation sequence
2. `development/01-foundation-bootstrap-and-project-layout.md` — Python package layout, dependency rules, CI และ bootstrap
3. `development/02-provider-api-and-compatibility-implementation-plan.md` — API implementation, normalization, SDK compatibility
4. `development/03-inference-scheduler-and-resource-control-plan.md` — queue, leases, fairness, deadlines, cancellation
5. `development/04-ollama-adapter-routing-and-model-registry-plan.md` — Ollama discovery/qualification, registry และ alias rollout
6. `development/05-streaming-conformance-and-client-compatibility-plan.md` — SSE fragmentation, client conformance, regression corpus
7. `development/06-control-monitoring-web-ui-plan.md` — management API first, live dashboard, controls และ reconnect
8. `development/07-security-auth-secrets-and-network-hardening-plan.md` — local baseline, client policy, redaction, network mode
9. `development/08-test-chaos-performance-and-acceptance-plan.md` — test pyramid, chaos matrix, direct-vs-Cortex benchmark, acceptance
10. `development/09-release-packaging-migration-and-operations-plan.md` — packaging, lifecycle, migrations, backup/restore, release evidence
11. `development/10-detailed-work-breakdown-and-dependency-map.md` — work packages A–P, critical path และ per-task DoD
12. `development/11-review-checklists-and-change-governance.md` — architecture/API/security/release reviews และ ADR triggers

## Canonical implementation direction

```text
Clients
  ↓
Provider-compatible API
  ↓
Admission / Policy
  ↓
Inference Service
  ├── Router / Model Registry
  └── Scheduler / Resource Pools
  ↓
Provider Adapter
  ↓
Ollama first; future providers later
```

CogentNexus-X-Agent, Hermes, OpenClaw และ applications อื่นควรสามารถเป็น clients ของ Cortex โดย Cortex ไม่ต้องรู้ semantic Task/Intent ของระบบเหล่านั้น

## V1 critical path

```text
Foundation
→ Domain Contracts
→ Fake Provider
→ One-slot Scheduler
→ Inference Service
→ Provider-compatible API
→ Streaming
→ Ollama Adapter
→ Conformance/Chaos/Security
→ Packaging/Release Acceptance
```

## Document maintenance rule

เมื่อ implementation เปลี่ยน canonical behavior ต้องอัปเดตเอกสารที่เกี่ยวข้องและ tests ใน change เดียวกันเมื่อทำได้ การเพิ่มคำศัพท์ใหม่ต้องตรวจว่าเป็น semantic responsibility ใหม่จริง ไม่ใช่ synonym ของ entity เดิม
