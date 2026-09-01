# CogentNexus-Cortex — Intelligence Provider Gateway Design Pack

> **Document status:** Design Baseline / Living Specification  
> **Primary language:** ไทย โดยใช้คำศัพท์เชิงสถาปัตยกรรมภาษาอังกฤษเป็น canonical terms  
> **Normative language:** MUST / MUST NOT / SHOULD / MAY ใช้ในความหมายเชิงข้อกำหนด  
> **Scope:** CogentNexus-Cortex เป็น provider-facing intelligence infrastructure ที่รับ inference API จาก clients แล้วควบคุม admission, scheduling, routing, provider execution, streaming, accounting และ observability ก่อนส่งไปยัง LLM providers เช่น Ollama

## 1. Why Cortex exists

CogentNexus-Cortex ถูกแยกออกจาก CogentNexus-X-Agent เพื่อรักษา **separation of responsibilities** ระหว่าง “ระบบที่เข้าใจและขับเคลื่อนงาน” กับ “ระบบที่จัดสรรทรัพยากร intelligence”

- **X-Agent** เป็นเจ้าของ Intent, Ticket, Task, Context, Agent Runtime, Evidence, Side Effects, Recovery และ Delivery
- **Cortex** เป็นเจ้าของ Inference Request, Provider Routing, Model Scheduling, Queue, Capacity, Streaming, Provider Health, Usage/Budget และ API Compatibility
- **LLM Provider** เช่น Ollama/OpenAI เป็น execution backend ที่ Cortex เรียกใช้

ดังนั้นเส้นทางพื้นฐานคือ:

```text
External Client / X-Agent / Hermes / OpenClaw
                  │
                  │ Provider API
                  ▼
        CogentNexus-Cortex
                  │
          ┌───────┼────────┐
          ▼       ▼        ▼
       Ollama   OpenAI   Future Provider
```

Cortex MUST NOT กลายเป็น Agent โดยเผลอรับผิดชอบ Ticket/Task หรือ domain reasoning ของ client

> **Cortex governs access to intelligence; clients govern the meaning of their work.**

## 2. Primary objectives

Cortex มีวัตถุประสงค์หลักดังนี้:

1. ทำให้ clients ใช้ provider endpoint เดียวโดยไม่ต้องผูกกับ backend vendor
2. รองรับ OpenAI-compatible surface เพื่อให้เปลี่ยน Base URL ได้ง่าย
3. ทำให้ model aliases เช่น `cortex:auto`, `cortex:fast`, `cortex:coding` เป็น stable semantic interface
4. จัดคิวและ fairness เมื่อหลาย clients ใช้ model/resource เดียวกัน
5. รองรับ single-model / single-slot baseline อย่างถูกต้อง ไม่ starvation และไม่ hidden concurrency
6. แยก provider-specific behavior ไว้ใน replaceable Provider Adapters
7. จัดการ streaming, timeout, cancellation และ retry โดยไม่บิดเบือน semantics
8. มี observability และ usage accounting ที่มองเห็นทั้งระบบ
9. มี Control/Monitoring Web UI โดย UI ไม่เป็น authority path ที่ bypass Core service
10. default local-first, secure-by-default และไม่เปิด network exposure โดยไม่ตั้งใจ
11. ทำให้ X-Agent, Hermes, OpenClaw และโปรแกรมอื่นใช้ intelligence infrastructure เดียวกันได้
12. สามารถย้าย provider ไปอีกเครื่องหรือเพิ่ม cloud provider ได้โดย clients ไม่ต้องเปลี่ยน architecture

## 3. Architectural boundary

Cortex **owns**:

- API admission and request identity
- client authentication and provider-access policy
- request class / scheduling policy
- model alias resolution
- model/provider routing
- provider health and capability discovery
- inference queue and resource slots
- provider invocation lifecycle
- stream forwarding and stream state
- provider error normalization
- usage accounting and budget enforcement
- management/control API
- Cortex-local configuration, persistence and audit

Cortex **does not own**:

- user Intent
- Ticket / Task semantics
- agent planning
- prompt meaning beyond protocol validation/policy
- tool authorization or external side effects of an Agent
- domain Evidence/Claim semantics
- agent completion/delivery semantics
- application conversation truth

Metadata such as `correlation_id`, `trace_id`, `client_request_id`, `workload_class` MAY pass through Cortex but remain opaque coordination metadata unless explicitly defined by the Cortex protocol.

## 4. Canonical request path

```text
Client
  │
  │ 1. Provider-compatible request
  ▼
API Compatibility Layer
  │
  │ 2. Parse + validate + authenticate
  ▼
Admission & Policy
  │
  │ 3. Resolve alias/provider requirements
  ▼
Model Router
  │
  │ 4. Create schedulable inference request
  ▼
Inference Scheduler
  │
  │ 5. Acquire resource/model slot
  ▼
Provider Adapter
  │
  │ 6. Translate request
  ▼
Actual Provider / Model
  │
  │ 7. stream/result/error
  ▼
Provider Adapter
  │
  │ 8. normalize
  ▼
Cortex API
  │
  ▼
Client
```

No public API handler SHOULD call Ollama/provider directly. Provider calls MUST pass through the common inference service and scheduler so observability, capacity and policy remain coherent.

## 5. Three API behavior classes

### Transparent Compatibility

Preserve client request semantics as closely as possible. Cortex validates, schedules, routes and observes but MUST NOT silently rewrite prompts/messages/tool schemas.

### Managed Routing

Client intentionally requests Cortex semantics through aliases/policy options. Cortex may choose a backend/model within an explicitly defined compatibility envelope.

### Cortex Management API

Separate authenticated surface for status, provider health, model registry, queue inspection, budgets, drain/disable/reload/cancel controls and Web UI. Management operations MUST NOT be mixed into provider-compatible endpoints.

## 6. Core invariants

1. **Provider compatibility must not imply semantic rewriting.**
2. **Exact model requests are pinned unless the client explicitly allows fallback.**
3. **Aliases may route; concrete provider/model identities do not silently drift.**
4. **One model slot means one active model lease.**
5. **All clients share resource policy through the same scheduler when they target the same constrained resource.**
6. **A client priority hint is not scheduling authority.**
7. **Streaming output already exposed to a client changes retry semantics.**
8. **Timeout is not proof that the provider never processed the request.**
9. **Cortex must not invent Agent-level durability guarantees for ordinary synchronous provider APIs.**
10. **Provider-specific behavior is isolated behind Provider Adapters.**
11. **Cortex management UI/API cannot bypass policy or mutate persistence tables directly.**
12. **Secrets never enter model prompts, normal telemetry, or browser bundles by default.**
13. **Prompt/output retention is explicit policy, not an accidental logging side effect.**
14. **Local-only binding is the secure default.**
15. **X-Agent correctness must not require Cortex to understand Ticket/Task semantics.**
16. **Cortex must remain useful without X-Agent.**

## 7. Design-pack structure

```text
planning/intelligence-provider-gateway/
├── README.md
├── DOCUMENT-MANIFEST.md
├── knowledge/
│   ├── 00-vision-principles-and-invariants.md
│   ├── 01-canonical-ontology-and-request-state-model.md
│   ├── 02-system-boundary-and-service-architecture.md
│   ├── 03-provider-api-compatibility-and-contracts.md
│   ├── 04-inference-scheduler-fairness-and-backpressure.md
│   ├── 05-model-routing-aliases-registry-and-failover.md
│   ├── 06-provider-adapter-contract-and-ollama-mapping.md
│   ├── 07-streaming-timeout-cancellation-retry-and-idempotency.md
│   ├── 08-security-auth-policy-tenancy-and-secret-management.md
│   ├── 09-control-monitoring-web-ui-and-observability.md
│   ├── 10-persistence-configuration-recovery-and-operations.md
│   └── 11-threat-model-reliability-and-failure-taxonomy.md
└── development/
    ├── 00-development-strategy-and-v1-boundary.md
    ├── 01-foundation-bootstrap-and-project-layout.md
    ├── 02-provider-api-and-compatibility-implementation-plan.md
    ├── 03-inference-scheduler-and-resource-control-plan.md
    ├── 04-ollama-adapter-routing-and-model-registry-plan.md
    ├── 05-streaming-conformance-and-client-compatibility-plan.md
    ├── 06-control-monitoring-web-ui-plan.md
    ├── 07-security-auth-secrets-and-network-hardening-plan.md
    ├── 08-test-chaos-performance-and-acceptance-plan.md
    ├── 09-release-packaging-migration-and-operations-plan.md
    ├── 10-detailed-work-breakdown-and-dependency-map.md
    └── 11-review-checklists-and-change-governance.md
```

## 8. Recommended reading order

อ่าน `knowledge/00 → 11` เพื่อเข้าใจ architecture และ invariants ก่อน จากนั้นใช้ `development/00 → 11` เป็น implementation sequence และ acceptance roadmap

ถ้าจะเริ่มเขียน V1 อย่างเร็วที่สุด ต้องอ่านอย่างน้อย:

- `knowledge/00`
- `knowledge/01`
- `knowledge/02`
- `knowledge/03`
- `knowledge/04`
- `knowledge/06`
- `knowledge/07`
- `knowledge/08`
- `development/00`
- `development/01`

## 9. V1 direction

V1 ควรพิสูจน์เส้นทางเล็กแต่ครบ:

```text
OpenAI-compatible client
  → Cortex
  → admission
  → one-slot scheduler
  → Ollama adapter
  → one local model
  → streamed/non-streamed result
  → accounting + monitoring
```

จากนั้นจึงเพิ่ม aliases, multi-client fairness, budgets, cancellation, Web UI controls, provider qualification และ cloud/future backends

## 10. Relationship to CogentNexus-X-Agent

X-Agent SHOULD configure Cortex as a normal provider endpoint เช่น:

```yaml
provider:
  type: openai_compatible
  base_url: http://127.0.0.1:<configured-port>/v1
  model: cortex:auto
```

X-Agent retains its own Task Scheduler; Cortex owns only Inference Scheduling. ดังนั้น:

```text
X-Agent Task Scheduler
        │
        │ requests intelligence
        ▼
Cortex Inference Scheduler
        │
        ▼
Provider / Model
```

สอง scheduler มีคนละ responsibility และไม่ควรถูกรวมเป็น state machine เดียวกัน

---

ชุดเอกสารนี้เป็น architecture source of truth ก่อนเริ่ม production implementation ของ CogentNexus-Cortex การเบี่ยงจาก shared invariants ควรถูกบันทึกเป็น explicit design decision/ADR พร้อม rationale และ acceptance impact
