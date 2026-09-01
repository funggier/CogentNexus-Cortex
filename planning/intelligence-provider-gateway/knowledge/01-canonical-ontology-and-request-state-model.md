# 01 — Canonical Ontology and Request State Model

## 1. Purpose

Cortex ต้องมี vocabulary ที่ไม่ปะปนกับ Agent semantics เพื่อให้ API, scheduler, persistence, Web UI, provider adapters และ tests พูดภาษาเดียวกัน

หลักสำคัญคือ:

> **Cortex schedules inference, not tasks.**

คำว่า Task, Ticket, Evidence หรือ Effect ของ X-Agent จึงไม่ควรถูกนำมาใช้แทน entity ของ Cortex

## 2. Canonical entities

### 2.1 Client

ตัวตนที่เรียก Cortex API เช่น:

- CogentNexus-X-Agent
- Hermes
- OpenClaw
- CLI
- IDE plugin
- custom application

Client มี identity, authentication method, policy profile และ optional quota/budget

### 2.2 Client Request

request ที่เข้ามาทาง public provider API มี client-supplied identity/metadata ตาม protocol

### 2.3 Inference Request

canonical Cortex record หลังผ่าน parsing/admission

ควรมีอย่างน้อย:

```yaml
inference_request:
  request_id: req_...
  client_id: client_...
  client_request_id: optional
  api_surface: openai.chat_completions
  requested_model: cortex:auto
  stream: true
  request_hash: sha256:...
  scheduling_class: INTERACTIVE
  deadline_at: optional
  admitted_at: ...
```

Inference Request ไม่ได้หมายถึง Agent Task

### 2.4 Model Alias

stable semantic name ที่ resolve ไป concrete model/provider ผ่าน policy เช่น:

```text
cortex:auto
cortex:fast
cortex:coding
cortex:reasoning
cortex:local
```

### 2.5 Concrete Model

physical model identity เช่น:

```text
provider=ollama
model=qwen3:14b
```

Concrete identity ต้อง inspect ได้เสมอหลัง routing

### 2.6 Provider

execution backend instance เช่น Ollama daemon หรือ cloud API account/endpoint

### 2.7 Provider Adapter

anti-corruption layer ที่ translate Cortex normalized request ไป provider-native protocol และ normalize response/error กลับ

### 2.8 Resource Pool

กลุ่ม capacity ที่แข่งขันร่วมกัน เช่น:

```text
ollama-local-gpu
openai-account-a
remote-gpu-node-1
```

### 2.9 Resource Slot

leaseable unit ของ concurrent inference capacity

V1:

```text
pool=ollama-local
slot_count=1
```

### 2.10 Inference Lease

fenced ownership ของ slot สำหรับ request/run หนึ่ง

ต้องมี generation/token เพื่อป้องกัน stale completion

### 2.11 Provider Attempt

การเรียก concrete provider หนึ่งครั้ง

request เดียวอาจมีมากกว่า 1 attempt เฉพาะเมื่อ retry/fallback policy อนุญาต

### 2.12 Response Stream

state ของ output exposure ต่อ client ไม่ใช่เพียง network socket

ต้องรู้ว่า:

- ยังไม่ส่ง header/body
- stream opened
- chunk exposed
- terminal event exposed
- client disconnected

### 2.13 Usage Record

normalized accounting เช่น input/output tokens, elapsed time, queue time, provider latency, model identity และ cost estimate ถ้ามี

### 2.14 Route Decision

record ที่อธิบาย alias/policy → concrete provider/model

ไม่ต้องเก็บ private reasoning; เก็บ rule/policy facts ที่ audit ได้

## 3. Request state machine

แนะนำ lifecycle:

```text
RECEIVED
   ↓
VALIDATED
   ↓
ADMITTED
   ↓
QUEUED
   ↓
CLAIMED
   ↓
RUNNING
   ↓
┌───────────────┬─────────────────┬──────────────────┐
▼               ▼                 ▼
SUCCEEDED       FAILED            CANCELLED
                  │
                  ├─ retryable
                  ├─ permanent
                  └─ partial_stream
```

อาจมี terminal/disposition เพิ่ม เช่น `REJECTED` ก่อน admission

## 4. Separate dimensions instead of state explosion

อย่ายัดทุกอย่างเป็น enum เดียว

แนะนำแยก:

```yaml
request_state:
  lifecycle: RUNNING
  stream_state: OPEN
  cancellation: NONE
  route_state: RESOLVED
  accounting_state: IN_PROGRESS
```

เหตุผลคือ request อาจ RUNNING พร้อม stream OPEN และ cancellation REQUESTED ได้

## 5. Admission result

Admission อาจจบเป็น:

```text
ADMITTED
REJECTED_AUTH
REJECTED_POLICY
REJECTED_INVALID_REQUEST
REJECTED_UNSUPPORTED
REJECTED_OVER_CAPACITY
REJECTED_DEADLINE
```

`429`/`503` ที่ส่ง client เป็น protocol mapping ไม่ใช่ canonical semantic state

## 6. Provider Attempt state

```text
PREPARED
  ↓
ISSUED
  ↓
RUNNING
  ↓
SUCCEEDED | FAILED | CANCELLED | OUTCOME_UNKNOWN
```

สำหรับ pure inference `OUTCOME_UNKNOWN` ไม่ได้มี side-effect danger แบบ external mutation แต่มีผลต่อ retry/output consistency โดยเฉพาะ streaming

## 7. Stream state

```text
NOT_STARTED
OPENING
OPEN
PARTIAL
COMPLETED
ABORTED_CLIENT
ABORTED_PROVIDER
ABORTED_CORTEX
```

Core ต้อง track `output_exposed=true/false` แยกด้วย เพราะเป็น guard ของ retry

## 8. Cancellation state

```text
NONE
REQUESTED
FORWARDED
CONFIRMED
BEST_EFFORT_ONLY
TOO_LATE
```

API client disconnect อาจ trigger cancellation policy แต่ไม่ควร assume provider หยุดจริงจน adapter รายงาน

## 9. Identity domains

ห้ามใช้ ID เดียวครอบทุกอย่าง

- `request_id` — Cortex inference request
- `client_request_id` — client-supplied dedupe/correlation
- `attempt_id` — provider attempt
- `lease_id` — resource ownership
- `stream_id` — response stream/session
- `trace_id` — observability correlation

การ conflation ทำให้ retry/debugging ยาก

## 10. Commands and events

Commands คือความต้องการ:

```text
admit_request
cancel_request
drain_provider
disable_route
reload_configuration
```

Events คือสิ่งที่เกิดแล้ว:

```text
request.admitted
request.queued
lease.claimed
provider_attempt.started
stream.chunk_exposed
provider_attempt.failed
request.completed
provider.drained
```

อย่าใช้ event ชื่อ imperative

## 11. Error taxonomy

Canonical error families:

```text
CLIENT_REQUEST_ERROR
AUTHENTICATION_ERROR
AUTHORIZATION_ERROR
POLICY_REJECTION
MODEL_NOT_FOUND
MODEL_UNAVAILABLE
PROVIDER_UNAVAILABLE
PROVIDER_RATE_LIMITED
PROVIDER_TIMEOUT
PROVIDER_PROTOCOL_ERROR
CORTEX_QUEUE_TIMEOUT
CORTEX_OVER_CAPACITY
CORTEX_INTERNAL_ERROR
STREAM_INTERRUPTED
CANCELLED_BY_CLIENT
CANCELLED_BY_POLICY
```

เก็บ native provider code/message แยกเป็น diagnostics แต่ client contract ใช้ normalized category

## 12. Terminal semantics

### SUCCEEDED

response ตาม supported API contract จบครบ และ Cortex ส่ง terminal response/stream event ได้สำเร็จตาม network semantics

### FAILED

request ไม่สามารถ produce valid completion ภายใต้ policy

### CANCELLED

Cortex ยุติการประมวลผลตาม cancel policy ไม่ได้แปลว่า upstream compute หยุดทันทีเสมอไป

### CLIENT_DISCONNECTED

ควรเป็น transport disposition แยกจาก provider outcome เพราะ provider อาจทำงานต่อจนสำเร็จหลัง client หาย

## 13. Relationship to X-Agent correlation

X-Agent อาจส่ง:

```yaml
metadata:
  correlation_id: ticket_or_model_run_reference
```

Cortex เก็บเป็น opaque correlation metadata และส่งกลับใน telemetry แต่ MUST NOT query X-Agent Task state หรือเปลี่ยน scheduling semantics จากค่า opaque นี้โดยอัตโนมัติ

## 14. Naming rules

- semantic names ไม่ใส่ version suffix
- protocol/schema compatibility version อยู่ metadata
- provider-specific terms อยู่ adapter namespace
- อย่าใช้ `job`, `task`, `workflow` สลับกับ `Inference Request`
- อย่าใช้ `result` แบบกว้างเกินไป; แยก `Provider Response`, `Normalized Response`, `Usage Record`, `Route Decision`

## 15. Acceptance questions

ก่อน implementation ผ่าน ontology review ต้องตอบได้ว่า:

1. request หนึ่งมี provider attempts กี่ครั้งได้?
2. retry ใช้ request ID เดิมหรือใหม่?
3. alias resolution ถูกบันทึกที่ไหน?
4. output chunk แรกเปลี่ยน retry guard อย่างไร?
5. client disconnect ต่างจาก provider failure อย่างไร?
6. scheduler lease ถูก fence อย่างไร?
7. exact model กับ alias model ต่างกันอย่างไร?
8. Cortex state ใดเป็น semantic ของ provider infrastructure และอะไรเป็นของ Agent client?
