# 00 — Vision, Principles, and Invariants

## 1. Purpose

เอกสารนี้อธิบายว่า **CogentNexus-Cortex คืออะไร ทำไมต้องแยกออกมาเป็นระบบอิสระ และอะไรคือสิ่งที่ห้ามเปลี่ยนแม้ implementation จะเปลี่ยนไป**

Cortex ไม่ได้เกิดขึ้นเพียงเพื่อทำ reverse proxy หน้า Ollama แต่เกิดจากปัญหาที่ใหญ่กว่า: เมื่อมีหลาย applications, agents และ runtimes ใช้ intelligence resources ชุดเดียวกัน การปล่อยให้ทุกระบบเชื่อม provider โดยตรงทำให้การจัดสรร resource, policy, budget, observability และ provider portability กระจัดกระจาย

## 2. Problem statement

สภาพทั่วไปเมื่อไม่มี Cortex:

```text
X-Agent ───────→ Ollama
Hermes ────────→ Ollama
OpenClaw ──────→ Ollama
Other App ─────→ Ollama
```

แต่ละ client มี logic ของตัวเองเกี่ยวกับ:

- endpoint
- model name
- retry
- timeout
- concurrency
- provider health
- usage
- fallback
- authentication
- logs

หากเครื่องมี model ที่รองรับได้เพียงหนึ่ง active inference หรือมี memory จำกัด clients ต่างคนต่างไม่รู้ contention ของกันและกัน ผลที่ตามมาคือ latency spikes, out-of-memory, model thrashing, starvation และ debugging ยาก

Cortex เปลี่ยน topology เป็น:

```text
Clients
  │
  ▼
CogentNexus-Cortex
  │
  ├── admission
  ├── routing
  ├── scheduling
  ├── policy
  ├── accounting
  └── observability
  │
  ▼
LLM Providers
```

## 3. Vision

> **Make intelligence a governed, observable, replaceable infrastructure resource rather than a hidden dependency embedded independently in every application.**

Cortex ควรทำให้ client มอง model/provider เป็น semantic capability ที่เสถียร ขณะที่ backend สามารถเปลี่ยนได้

ตัวอย่าง:

```text
client asks for: cortex:coding
        │
        ▼
Cortex policy
        │
        ├── today  → Ollama / Model A
        └── future → Remote GPU / Model B
```

client ไม่ต้องรู้ topology จริง เว้นแต่ client ขอ exact provider/model identity

## 4. Why it must be separate from X-Agent

X-Agent และ Cortex มี lifecycle และ correctness contract คนละชนิด

### X-Agent asks

- งานนี้คืออะไร?
- ต้องทำขั้นตอนไหน?
- อะไรคือ evidence?
- side effect นี้ได้รับอนุญาตหรือไม่?
- task complete หรือยัง?

### Cortex asks

- request นี้ผ่าน API contract หรือไม่?
- client นี้ได้รับสิทธิ์ใช้ model class นี้หรือไม่?
- request ควรอยู่ queue ไหน?
- resource slot ไหนว่าง?
- alias นี้ resolve ไป model ไหน?
- provider healthy หรือไม่?
- stream จบหรือขาดกลางทาง?
- ใช้ token/เวลา/resource เท่าไร?

หากรวมสองระบบเข้าด้วยกัน จะเกิด authority ambiguity และทำให้ provider infrastructure reuse ยาก

## 5. Core principles

### 5.1 Intelligence is a resource; meaning remains with the client

Cortex SHOULD inspect request structure enough to validate, route and enforce policy แต่ MUST NOT ตีความ domain intent เพื่อสร้าง Task/Plan โดยอัตโนมัติ

### 5.2 Transparent by default

Provider-compatible request SHOULD preserve messages, roles, tool schemas, sampling parameters และ stop semantics เท่าที่ backend capability รองรับ

ถ้าต้อง transform ต้องเป็น explicit mode/option ที่ inspect ได้

### 5.3 Stable semantic interface, replaceable physical provider

model aliases มีหน้าที่แยก client contract จาก deployment reality

### 5.4 Single-model baseline is first-class

ระบบต้องทำงานได้ดีเมื่อ:

```text
max_loaded_models = 1
max_active_inference = 1
```

multi-provider/multi-GPU เป็น capacity optimization ไม่ใช่ correctness requirement

### 5.5 Admission is distinct from execution

รับ HTTP request แล้วไม่ได้หมายความว่า model เริ่มรันแล้ว

canonical lifecycle ต้องแยก:

```text
RECEIVED → ADMITTED → QUEUED → CLAIMED → RUNNING → TERMINAL
```

### 5.6 Streaming changes failure semantics

ก่อน expose output chunk แรก Cortex อาจมีพื้นที่ retry transport บางชนิด แต่หลัง client เห็น output แล้ว Cortex ต้องไม่ restart generation แบบเงียบ ๆ เพราะอาจสร้างข้อความซ้ำหรือ semantic discontinuity

### 5.7 Exact request means exact request

ถ้า client ขอ exact model identity เช่น provider/model ที่เฉพาะเจาะจง Cortex MUST NOT fallback ไป model อื่นโดยไม่ explicit opt-in

aliases เช่น `cortex:auto` สามารถมี routing policy ได้

### 5.8 Policy authority is server-side

client อาจส่ง `priority_hint=interactive` แต่ Cortex policy เป็นผู้กำหนด effective scheduling class ป้องกันทุก client ประกาศตัวเองเป็น highest priority

### 5.9 Observability without accidental surveillance

การเห็น latency, queue, model, token usage และ error ไม่จำเป็นต้องเก็บ prompt/output content

default telemetry SHOULD be metadata-oriented

### 5.10 Provider adapter is anti-corruption layer

Ollama/OpenAI/อนาคตจะมี error model, stream format, cancellation และ usage semantics ต่างกัน Adapter ต้อง normalize ความต่างนั้นก่อนเข้า Core

## 6. Normative invariants

### INV-CORTEX-001 — No Agent semantic ownership
Cortex MUST NOT create or own Agent Task state as a side effect of ordinary inference.

### INV-CORTEX-002 — Common scheduling path
All constrained inference MUST acquire capacity through the common scheduler.

### INV-CORTEX-003 — No direct API-to-provider bypass
Public request handlers MUST NOT directly invoke provider drivers outside the inference service.

### INV-CORTEX-004 — Exact model pinning
Concrete model requests MUST remain pinned unless explicit fallback semantics are present.

### INV-CORTEX-005 — Alias resolution auditability
Every alias-routed run MUST record which concrete provider/model was selected and why at least at policy/rule level.

### INV-CORTEX-006 — Output exposure boundary
Cortex MUST track whether response output has been exposed to the client before deciding retry behavior.

### INV-CORTEX-007 — Resource lease uniqueness
A capacity slot MUST have at most one valid current lease generation at a time.

### INV-CORTEX-008 — Client hint is not authority
Client-provided priority/budget metadata MUST pass server-side policy validation.

### INV-CORTEX-009 — Provider failure normalization
Provider-specific errors MUST be mapped to stable Cortex error categories while preserving native diagnostics for operators.

### INV-CORTEX-010 — No secret leakage
Provider keys/tokens MUST NOT be returned by API, exposed in browser bundles, normal logs, or prompt content.

### INV-CORTEX-011 — Local secure default
Default listener SHOULD bind only to loopback until network exposure is explicitly configured.

### INV-CORTEX-012 — UI is a client
Web UI MUST issue management commands through authenticated Core APIs and MUST NOT write persistent state directly.

### INV-CORTEX-013 — Retention is explicit
Prompt/output content retention MUST be configurable and disabled/minimized by default unless needed.

### INV-CORTEX-014 — Compatibility is tested, not claimed
OpenAI-compatible endpoints MUST have conformance tests for supported fields and explicit documentation for unsupported semantics.

### INV-CORTEX-015 — No false durability
Standard synchronous/streaming provider API MUST NOT claim agent-style guaranteed completion after client disconnect/process crash unless a separate durable-job contract is explicitly used.

### INV-CORTEX-016 — Cortex independence
Cortex MUST remain operational and useful without X-Agent, Hermes or OpenClaw.

## 7. What should remain intentionally simple in V1

- one process service
- one SQLite operational database if persistence is needed
- one Ollama Provider Adapter
- one active inference slot baseline
- OpenAI-compatible chat/responses subset selected explicitly
- local loopback binding
- metadata telemetry
- simple model aliases
- deterministic fairness policy
- basic Web monitoring/control

ไม่ควรเริ่มด้วย distributed consensus, Kubernetes, vector databases, complex prompt transformation หรือ multi-region routing

## 8. Long-term scalability without premature complexity

Cortex ควร scale by replacing capacity components while preserving contracts:

```text
V1
1 Cortex → 1 Ollama → 1 model slot

Later
1 Cortex → N Providers → N resource pools

Further
Cortex control service → remote provider workers / GPU hosts
```

สิ่งที่ควรคงเดิมคือ request identity, admission semantics, routing contract, scheduler fairness model, provider adapter boundary และ observability vocabulary

## 9. Design compass

เมื่อมี feature ใหม่ ให้ถาม:

1. feature นี้เกี่ยวกับ **meaning of work** หรือ **allocation of intelligence**?
2. ถ้าเป็น meaning of work ควรอยู่ client/X-Agent หรือไม่?
3. feature นี้ทำให้ provider backend เปลี่ยนยากขึ้นหรือไม่?
4. feature นี้สามารถทำงานเมื่อมี model slot เดียวหรือไม่?
5. failure ของ feature นี้ถูกแสดงอย่าง explicit หรือถูกซ่อนด้วย retry?
6. operator สามารถ inspect decision ได้หรือไม่?
7. security boundary กว้างขึ้นโดยจำเป็นจริงหรือไม่?

ถ้าคำตอบทำให้ Cortex เริ่มกลายเป็น Agent หรือ provider-specific monolith ควรหยุดและปรับ boundary ใหม่
