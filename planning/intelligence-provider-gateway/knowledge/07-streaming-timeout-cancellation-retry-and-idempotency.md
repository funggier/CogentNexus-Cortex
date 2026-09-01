# 07 — Streaming, Timeout, Cancellation, Retry, and Idempotency

## 1. Purpose

Inference เป็นงานที่ไม่มี external side effect แบบ `repo.push` แต่ failure semantics ยังซับซ้อน โดยเฉพาะเมื่อ response ถูก stream ออกไปแล้ว การ retry generation แบบไม่ระวังสามารถทำให้ client ได้ข้อความซ้ำ ขาดช่วง หรือ semantic discontinuity

## 2. Key distinction: compute outcome vs response exposure

Cortex ต้องแยกสองเรื่อง:

```text
provider attempt outcome
```

กับ:

```text
what the client has already observed
```

ตัวอย่าง provider อาจ generate สำเร็จ แต่ client disconnect ก่อนรับครบ หรือ provider อาจ fail หลัง client เห็น 30 chunks แล้ว

## 3. Exposure boundary

เก็บ explicit flags/counters:

```yaml
stream:
  opened: true
  output_exposed: true
  chunks_exposed: 30
  terminal_exposed: false
```

`output_exposed` เป็น guard สำคัญของ retry

## 4. Retry zones

### Zone A — before provider issue

safe ที่จะ requeue/reroute ตาม policy เพราะยังไม่มี provider execution

### Zone B — provider issued, no output exposed

อาจ retry/fallback ได้ในบางกรณี แต่ต้องพิจารณา:

- provider request อาจกำลัง compute อยู่
- cancellation support
- budget duplication
- nondeterministic output

Cortex ควรมี bounded attempt count

### Zone C — output exposed

default: **no transparent regeneration retry**

ถ้า upstream fail ให้ terminate stream ด้วย explicit interruption/error behavior ตาม public API

client/Agent ระดับบนเป็นผู้ตัดสินว่าจะเริ่ม inference ใหม่หรือไม่

## 5. Why inference request IDs still matter

แม้ model call ไม่มี mutation semantic แต่ IDs จำเป็นเพื่อ:

- correlate retries
- detect duplicate client requests if extension supports it
- accounting
- debugging
- route audit

ห้ามใช้ request ID เป็น guarantee ว่า provider จะ return output เดิม

## 6. Idempotency

Standard LLM generation โดยทั่วไป **ไม่ deterministic-idempotent** แม้ input เดิม เพราะ sampling/provider state อาจต่างกัน

ดังนั้น `same request_id` หมายถึง identity/correlation ไม่ได้หมายถึง result reuse

ถ้าจะมี cache/replay output ต้องเป็น explicit feature พร้อม contract แยก

## 7. Timeouts

### Queue timeout

request รอนานเกิน policy ก่อน acquire slot

### Connect timeout

เชื่อม provider ไม่สำเร็จ

### TTFT timeout

provider accepted แต่ไม่มี first token ภายใน limit

### Stream idle timeout

stream เปิดแล้วแต่ไม่มี activity นานเกินไป

### Total deadline

upper bound ของ request lifecycle

ทุก timeout ต้องมี metric/error category แยก

## 8. Timeout does not prove non-execution

provider timeout อาจหมายถึง:

- request ไม่ถึง provider
- provider รับแล้วแต่ช้า
- network response หาย

สำหรับ inference ผลหลักคือ compute duplication/response ambiguity ไม่ใช่ external mutation แต่ยังต้อง report ตรง ๆ

## 9. Client disconnect

client disconnect policy options:

```text
CANCEL_UPSTREAM
CONTINUE_FOR_ACCOUNTING
CONTINUE_IF_DURABLE_JOB
```

V1 ordinary synchronous API ควร default cancel upstream best-effort เพื่อประหยัด resource แต่ slot release ต้องรอ adapter semantics

## 10. Cancellation contract

cancellation lifecycle:

```text
REQUESTED
→ FORWARDED
→ CONFIRMED | BEST_EFFORT | TOO_LATE
```

ถ้า provider ไม่มี confirm Cortex ต้องถือว่า compute อาจยัง active

## 11. Backpressure on streams

ถ้า client อ่านช้า ห้าม buffer unlimited

options:

- bounded per-stream queue
- upstream read throttling ถ้า protocol รองรับ
- terminate slow consumer ตาม configured limit

ต้อง metric `slow_consumer_disconnects`

## 12. Partial stream error

Cortex ต้อง distinguish:

```text
FAILED_BEFORE_OUTPUT
FAILED_AFTER_PARTIAL_OUTPUT
```

สำหรับ latter client/UI ต้องเห็นว่า response incomplete

Web UI ห้าม render partial text เป็น completed response

## 13. Non-stream response

แม้ non-stream จะง่ายกว่า แต่ response body อาจใหญ่ ต้องมี limits และ provider cancellation on client disconnect ตาม policy

## 14. Retry budget

กำหนด separate:

```yaml
retry_policy:
  max_provider_attempts: 2
  pre_output_only: true
  retryable_errors:
    - PROVIDER_UNAVAILABLE
    - TRANSIENT_CONNECT_FAILURE
```

อย่า retry `INVALID_REQUEST`, `MODEL_NOT_FOUND` แบบไร้เหตุผล

## 15. Fallback and retry interaction

Retry same provider กับ fallback new provider ต้องถูกนับใน attempt budget เดียวหรือ policy ที่อธิบายได้

Route Decision history ต้องบันทึกทุก attempt

## 16. Response ordering

แต่ละ stream ต้อง monotonic event sequence ภายใน Cortex เพื่อ debug missing/duplicate internal events

ไม่ต้อง global ordering ข้าม requests

## 17. Testing matrix

ต้อง inject failure อย่างน้อย:

- fail before connect
- fail after issue before output
- fail immediately after first chunk
- fail mid-stream
- client disconnect before first chunk
- client disconnect mid-stream
- cancellation ignored
- cancellation confirmed
- slow client backpressure
- provider sends malformed final event

## 18. Acceptance criteria

- Cortex ไม่ transparently regenerate หลัง output exposed
- partial stream ไม่ถูก report success
- slot ไม่ release prematurely on best-effort cancel
- timeout categories แยก metric/error ได้
- retry bounded และ reason-coded
- stream buffer bounded
- client disconnect behavior deterministic ตาม policy
- attempt history แสดง route/retry sequence ได้
