# 04 — Inference Scheduler, Fairness, and Backpressure

## 1. Purpose

Cortex ต้องควบคุมการแย่งใช้ intelligence resource โดยเฉพาะเมื่อมี local model หนึ่งตัวหรือ GPU/memory จำกัด เป้าหมายไม่ใช่แค่ “ทำ queue” แต่ต้องให้ระบบ predictable, fair, inspectable และไม่ปล่อยให้ client ใดผูกขาด model

## 2. Why scheduling belongs in Cortex

หาก X-Agent, Hermes, OpenClaw และ applications เรียก Ollama โดยตรง แต่ละระบบมองเห็นเฉพาะ queue ของตัวเอง จึงไม่สามารถ enforce global resource fairness ได้

```text
X-Agent ─┐
Hermes  ─┼─→ Cortex Scheduler → Ollama
App A   ─┤
App B   ─┘
```

Task scheduling ของ X-Agent ยังคงอยู่ X-Agent; Cortex scheduling จัดเฉพาะ inference requests

## 3. Baseline compute envelope

V1 ต้องรองรับอย่างน้อย:

```yaml
compute_envelope:
  resource_pool: ollama-local
  max_active_inference: 1
  max_loaded_models: 1
  queue_limit: configurable
```

หนึ่ง slot ต้องมี lease/fence ที่ชัดเจน

## 4. Scheduling classes

แนะนำ classes:

```text
INTERACTIVE
FOREGROUND
BACKGROUND
BATCH
```

ไม่ให้ client กำหนด effective class เองโดยตรง client ส่ง hint/identity แล้ว policy map เป็น class

ตัวอย่าง:

```text
X-Agent interactive user turn → INTERACTIVE
X-Agent long autonomous task → FOREGROUND
nightly analysis → BACKGROUND/BATCH
```

## 5. Fairness objectives

1. interactive latency ต่ำเมื่อเป็นไปได้
2. foreground progress ต่อเนื่อง
3. background ไม่ทำให้ foreground ค้าง
4. background ไม่ starvation ตลอดกาล
5. client เดียวไม่ monopolize class เดียวกัน
6. large requests ไม่ทำ queue collapse

## 6. Recommended policy

ใช้ combination:

- strict priority between broad classes พร้อม bounded aging
- weighted fair queueing หรือ deficit round-robin ภายใน class
- per-client concurrency and queue caps
- bounded consecutive grants ต่อ owner

ตัวอย่าง conceptual score:

```text
effective_priority = class_weight + aging_credit + policy_adjustment
```

อย่าให้ model/LLM ตัดสิน scheduling order

## 7. Admission vs queueing

Admission อาจ reject ก่อนเข้าคิวเมื่อ:

- client quota exceeded
- queue full
- deadline impossible
- model unavailable with no allowed route
- request exceeds configured limits

ไม่ควรรับ request ทุกอย่างแล้วปล่อย queue โตไม่จำกัด

## 8. Backpressure

Cortex ต้องมี bounded buffers ทุกชั้น:

- HTTP body
- admitted queue
- per-client queue
- stream buffer
- provider adapter buffer
- telemetry/event queue

เมื่อ limit ถึง ต้องมี explicit behavior เช่น reject, shed background load, pause admission หรือ apply deadline policy

## 9. Queue timeout and deadline

แยก:

- `queue_timeout`
- absolute `deadline_at`
- provider execution timeout

request ที่หมด deadline ระหว่างรอไม่ควรได้ slot แล้วเริ่ม model โดยไม่มีประโยชน์

## 10. Resource pools

scheduler ต้องคิดเป็น pool ไม่ผูกกับ global singleton ตลอดไป

```text
Pool A: local Ollama GPU, slots=1
Pool B: cloud provider, slots=8
```

route decision เลือก candidate pool แล้ว scheduler enforce capacity ของแต่ละ pool

## 11. Model loading and thrash

ถ้า local provider ต้อง load/unload model การสลับ model บ่อยสร้าง latency/memory cost

routing/scheduler MAY ใช้ locality affinity เช่น prefer currently-loaded compatible model แต่ต้องไม่ละเมิด exact model request

future policy อาจมี:

```text
model_switch_cost
minimum_residency
idle_unload
```

V1 ควรวัดก่อน optimize

## 12. Cancellation

Queued request cancel ง่าย: remove/mark cancelled atomically

Running request:

- request cancellation
- adapter forwards if supported
- lease remains owned until provider attempt terminal/abandoned according to contract
- slot ห้ามปล่อยก่อนแน่ใจว่า concurrent execution จะไม่เกิน capacity

นี่สำคัญกับ backend ที่ cancel เป็น best-effort

## 13. Streaming and slot release

slot ปล่อยเมื่อ provider inference จบ/ถูก terminate จริงตาม adapter semantics ไม่ใช่เมื่อ client disconnect

client disconnect อาจทำให้ Cortex request cancel แต่ provider อาจยังใช้ GPU อยู่

## 14. Scheduler persistence

V1 สามารถใช้ in-memory queue สำหรับ ordinary synchronous provider semantics แต่ต้องมี durable operational records เท่าที่จำเป็นต่อ accounting/audit

ถ้าต่อไปมี Durable Inference Job contract จึงค่อยเพิ่ม durable queue semantics

ห้ามเรียก in-memory queue ว่า durable

## 15. Recovery after Cortex restart

สำหรับ ordinary HTTP inference:

- active connections ตาย
- clients เห็น connection failure และอาจ retry ตาม client policy
- Cortex restart reconstructs provider health/config/accounting
- stale leases from previous process epoch invalidated

ต้องไม่ pretend ว่าสามารถ deliver old synchronous response หลัง process restart เว้นแต่ใช้ durable-job extension

## 16. Metrics

ควรมี:

- queue depth by class/client/pool
- queue wait percentiles
- active slots
- grants/client
- starvation age
- cancellations queued/running
- deadline expirations
- model switches
- TTFT and total latency

## 17. Abuse resistance

per-client controls:

- max queued requests
- max active requests
- request body/token estimate limit
- rate limit
- budget
- scheduling weight

client priority header เพียงอย่างเดียวไม่สามารถ bypass ได้

## 18. Acceptance criteria

- slot=1 แล้วไม่เกิด concurrent provider inference
- two equal clients ได้ service อย่างเป็นธรรม
- interactive request ได้ next eligible grant ตาม policy
- background ยังมี bounded progress ภายใต้ sustained higher-class load ตาม aging policy
- queue limit enforce ได้
- queued cancellation ไม่ invoke provider
- running cancellation ไม่ปล่อย slot prematurely
- expired deadline ไม่เริ่ม execution
- restart fences stale lease
- metrics อธิบายได้ว่าทำไม request รอ
