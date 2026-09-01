# 02 — System Boundary and Service Architecture

## 1. Purpose

เอกสารนี้กำหนด boundary ภายใน CogentNexus-Cortex เพื่อป้องกันไม่ให้ API layer, scheduler, provider-specific code, persistence และ Web UI ผูกกันแน่นจนเปลี่ยน provider หรือ scale ระบบยาก

เป้าหมายคือให้ Cortex เป็น **small, explicit, inspectable intelligence control service**

## 2. Logical architecture

```text
                 Provider-compatible Clients
                           │
                           ▼
                 ┌───────────────────┐
                 │ Public API Layer  │
                 └─────────┬─────────┘
                           ▼
                 ┌───────────────────┐
                 │ Admission & Auth  │
                 └─────────┬─────────┘
                           ▼
                 ┌───────────────────┐
                 │ Inference Service │
                 └──────┬─────┬──────┘
                        │     │
             ┌──────────┘     └───────────┐
             ▼                            ▼
      Model Router                  Scheduler
             │                            │
             └────────────┬───────────────┘
                          ▼
                  Provider Registry
                          │
                          ▼
                  Provider Adapter
                          │
                          ▼
                  Actual LLM Provider
```

Management path แยกจาก provider API:

```text
Web UI / Admin CLI
        │
        ▼
Management API
        │
        ├── status/read models/queue/usage
        ├── drain/disable/reload/cancel
        └── policy/config operations
```

## 3. Modules and responsibilities

### 3.1 Public API Compatibility Layer

รับ protocol request เช่น OpenAI-compatible endpoint

หน้าที่:

- HTTP parsing
- protocol validation
- request size limits
- canonical field mapping
- streaming protocol framing
- response/error compatibility mapping

ไม่ทำ:

- direct provider calls
- scheduling decisions
- secret lookup business logic
- provider-specific transformation

### 3.2 Admission Service

ตัดสินว่า request เข้า system ได้หรือไม่

ตรวจ:

- authentication
- client status
- endpoint permission
- model/alias permission
- request limits
- queue/capacity admission policy
- deadlines
- budget/quota

Admission กับ scheduling ต้องแยก เพราะ admitted request อาจรอ queue ได้

### 3.3 Inference Service

orchestration boundary หลักของ Cortex

รับ normalized request และ coordinate:

```text
route → enqueue → lease → invoke → stream/result → normalize → account
```

ทุก request path ควรผ่าน service นี้

### 3.4 Model Router

resolve `requested_model` ไป route candidate(s)

inputs:

- exact model vs alias
- provider health
- model capabilities
- client policy
- locality
- context length
- tool/structured-output requirement
- cost/budget
- load/capacity signals

outputs เป็น explicit Route Decision

### 3.5 Inference Scheduler

จัด fairness/resource capacity ไม่ทำ semantic routing

Router ตอบ “สามารถไปที่ไหน”
Scheduler ตอบ “เมื่อไร request ไหนได้ slot”

### 3.6 Provider Registry

เก็บ normalized provider/model inventory และ health/capabilities

ไม่ควรให้ code ส่วนอื่น query Ollama `/api/tags` โดยตรง

### 3.7 Provider Adapter

provider-specific boundary

เช่น `OllamaAdapter` รับ normalized Cortex request แล้วแปลง protocol

Adapter ต้อง implement contract ที่ทดสอบได้โดย provider fixture/fake server

### 3.8 Usage & Accounting Service

record:

- queue wait
- provider latency
- time-to-first-token
- total duration
- token usage if known
- estimated usage if exact unavailable
- route/model/provider
- termination/error class

Accounting ต้องไม่เป็นเหตุให้ response path ล้มเหลวถ้าสามารถ degrade safely ได้ แต่ durable budget enforcement ที่จำเป็นต้อง fail closed ต้องอยู่ admission path

### 3.9 Configuration & Policy Service

config โหลดจาก canonical source และ validate ก่อน activate

ควรรองรับ:

```text
load → validate → stage → activate
```

เพื่อไม่ให้ invalid hot reload ทำ service พัง

### 3.10 Persistence

เก็บ operational metadata/config revisions/audit/usage ตาม policy

Cortex ไม่ควรเก็บ conversation history โดย default

### 3.11 Event/Telemetry Bus

internal typed events สำหรับ Web UI/metrics/audit

ไม่ควรใช้ WebSocket UI เป็น internal event bus

### 3.12 Management API

read/control surface สำหรับ operator

ต้องผ่าน authorization เช่นเดียวกับ client API แต่เป็น permission set คนละชุด

## 4. Process architecture

### V1 recommendation

```text
cortex-service.exe / python process
  ├── FastAPI/ASGI server
  ├── asyncio inference scheduler
  ├── provider adapters
  ├── SQLite operational store
  └── static Web UI / event stream
```

เริ่ม one-process เพื่อความง่าย แต่แยก modules ตาม logical boundary เพื่อ split process ได้ภายหลัง

### Future split

```text
Cortex API/Control
      │
      ├── Scheduler service
      └── Remote Provider Workers
```

ไม่ควรทำ distributed ก่อนมีความต้องการจริง

## 5. Call direction rules

ควร enforce dependency direction:

```text
API → Application Services → Domain Contracts
                            ↓
                    Infrastructure Adapters
```

ตัวอย่าง:

```text
openai_route.py
   ↓
inference_service.py
   ↓
scheduler/router interfaces
   ↓
ollama_adapter.py
```

ห้าม:

```text
openai_route.py → requests.post(ollama_url)
```

## 6. Shared vs provider-specific state

### Shared state

- request identity
- client identity
- scheduling class
- route decision
- lifecycle
- normalized usage
- normalized error
- lease

### Provider-specific diagnostics

- native request ID
- native error code
- native headers
- native model digest
- provider version

provider-specific fields ควรอยู่ namespaced metadata ไม่ทำให้ Core schema แตกทุกครั้งที่ provider เปลี่ยน

## 7. X-Agent integration boundary

X-Agent เป็น client ธรรมดาใน provider plane

```text
X-Agent Runtime
   ↓
OpenAI-compatible Provider Client
   ↓
Cortex
```

Cortex MAY receive correlation metadata เช่น:

```text
x-cnx-correlation-id
x-cnx-client-request-id
```

แต่ไม่ควร import X-Agent packages หรือ query X-Agent persistence

## 8. Hermes/OpenClaw integration boundary

หาก Hermes/OpenClaw รองรับ custom base URL/model provider ก็ชี้มาที่ Cortex โดยไม่ต้องมี Agent integration plugin เพื่อใช้ provider gateway

ถ้าจำเป็นต้องมี compatibility shim ให้ shim อยู่ client edge ไม่ขยาย Core semantics

## 9. Failure containment

### Provider down
Cortex service ยังอยู่และรายงาน provider unavailable

### Web UI down
Provider API ยังทำงาน

### Persistence telemetry failure
ขึ้นกับประเภทข้อมูล: telemetry ที่ไม่ critical MAY degrade; auth/budget/config authority MUST fail according to explicit policy

### One client floods requests
Scheduler/admission isolates via per-client limits/fairness

### One adapter crashes logically
ต้องไม่ทำ provider registry/scheduler state corrupt

## 10. Threading and concurrency model

Python `asyncio` เหมาะสำหรับ V1 เพราะงานส่วนใหญ่เป็น network I/O/streaming/queueing

หลัก:

- event loop ไม่ block ด้วย subprocess/file CPU-heavy operations
- provider streaming ใช้ bounded buffers
- expensive hashing/parsing offload เมื่อจำเป็น
- capacity controlled โดย explicit scheduler ไม่อาศัยจำนวน asyncio tasks เป็น resource policy

## 11. API surfaces

แนะนำแบ่ง namespace:

```text
/v1/...                    provider compatibility
/cortex/v1/status          management/read
/cortex/v1/providers
/cortex/v1/models
/cortex/v1/requests
/cortex/v1/queue
/cortex/v1/usage
/cortex/v1/control/...
/cortex/v1/events          SSE/WebSocket management events
```

version ใน URL เป็น API compatibility boundary ไม่ใช่ conceptual name

## 12. What not to put in Cortex

- Agent planner
- Task decomposition
- Git/file tools
- external action authorization
- long-term project memory
- RAG knowledge base by default
- prompt rewriting hidden from clients
- application-specific tool schemas
- business workflow engine

สิ่งเหล่านี้อาจเป็น clients/adjacent services

## 13. Acceptance criteria

Architecture review ผ่านเมื่อ:

- public API cannot bypass scheduler to provider
- Provider Adapter can be replaced with FakeProvider in tests
- Web UI can be removed without changing inference behavior
- X-Agent packages are not required by Cortex
- Ollama-specific types do not leak into Core interfaces
- one model slot can be enforced centrally
- exact/alias routing decisions are inspectable
- provider failure cannot corrupt unrelated client/request state
- management controls use API/service boundaries, not direct persistence writes
