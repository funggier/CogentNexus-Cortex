# 03 — Inference Scheduler and Resource Control Plan

## 1. Goal

สร้าง scheduler ที่ enforce capacity จริง โดย V1 เริ่ม `slots=1` และรองรับ fairness/backpressure ก่อนเพิ่ม concurrency

## 2. Phase 1 — Resource Pool and Lease

สร้าง types/storage runtime:

```text
ResourcePool
ResourceSlot
ResourceLease
LeaseGeneration
```

assert invariant: slot หนึ่งมี valid owner เดียว

## 3. Phase 2 — Bounded queue

สร้าง queue ที่มี:

- global limit
- per-client limit
- queue timestamps
- cancel/remove
- deadline checks

ห้าม `asyncio.Queue()` แบบ unbounded

## 4. Phase 3 — Scheduling classes

implement:

```text
INTERACTIVE
FOREGROUND
BACKGROUND
BATCH
```

server policy map client/request hint → effective class

## 5. Phase 4 — Fairness

เริ่ม algorithm ที่อธิบายได้ เช่น per-class queues + round-robin by client และ aging

ต้องมี deterministic unit tests:

- equal clients alternate/progress
- one client flood does not monopolize
- high class gets precedence
- lower class eventually progresses ตาม policy

## 6. Phase 5 — Lifecycle integration

```text
ADMITTED
→ QUEUED
→ CLAIMED
→ RUNNING
→ terminal
→ release lease
```

lease release อยู่ finally/recovery path แต่ไม่ release ก่อน provider execution known terminal/safe

## 7. Phase 6 — Deadlines

check deadline:

- before enqueue
- before grant
- while waiting

expired queued request terminal โดยไม่ invoke provider

## 8. Phase 7 — Cancellation

Queued: cancel immediately

Running:

- mark request cancellation requested
- ask adapter
- maintain slot ownership until provider terminal/known safe

## 9. Phase 8 — Backpressure and overload

implement metrics/behavior:

- queue full rejection
- per-client queue rejection
- slow consumer termination
- admission pause
- optional background shedding

## 10. Scheduler API

internal interface concept:

```text
submit(request) -> await Grant
cancel(request_id)
release(lease, outcome)
pause_admission()
resume_admission()
snapshot()
```

ไม่ expose mutable internal queues ให้ Web UI

## 11. Process epoch fencing

startup creates epoch; lease includes epoch/generation

completion from stale epoch cannot mutate current running record/slot state

## 12. Metrics

- queue depth per class/client
- oldest queue age
- grant counts
- wait histogram
- slot utilization
- rejected over capacity
- deadline expired
- cancellation latency

## 13. Fake Provider scenarios

scheduler integration ใช้ provider ที่:

- blocks until event
- ignores cancellation
- confirms cancellation
- completes instantly
- fails

เพื่อทดสอบ lease release timing

## 14. Performance

scheduler operations ควรไม่ scale แบบ O(total history); queue structures ทำงานกับ active set

อย่า optimize microseconds ก่อน correctness

## 15. Exit criteria

- slot=1 proves no overlapping provider execution
- bounded queue proven under flood
- fairness tests deterministic
- cancellation safe
- deadline enforcement works
- stale lease rejected
- management snapshot read-only
- scheduler metrics available
