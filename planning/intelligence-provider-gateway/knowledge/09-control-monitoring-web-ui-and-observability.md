# 09 — Control, Monitoring Web UI, and Observability

## 1. Purpose

Cortex ควรมี Web UI ที่ทำหน้าที่เป็น **operator control and monitoring surface** สำหรับดู provider/model health, queue, client usage, latency, failures และ resource state โดยไม่ทำให้ UI กลายเป็น authority ที่เขียน state เอง

## 2. UI principle

> **The UI observes and commands; the Cortex service validates and commits.**

ห้าม Web UI:

- เขียน SQLite โดยตรง
- call provider โดยตรง
- inject arbitrary scheduler state
- alter route mapping โดย bypass policy service

## 3. Recommended topology

```text
Browser
  │
  ├── Management REST API
  └── SSE/WebSocket event feed
  │
  ▼
Cortex Service
  │
  ├── Scheduler
  ├── Provider Registry
  ├── Usage
  └── Audit
```

V1 สามารถ serve static UI จาก service เดียวกันเพื่อ packaging ง่าย

## 4. Primary dashboard

ควรเห็นอย่างน้อย:

```text
Service         RUNNING
Providers       1 healthy / 1 total
Models          4 discovered / 2 qualified
Active slots    1 / 1
Queue           3
Oldest wait     1.8 s
Requests/min    ...
TTFT p50/p95    ...
Errors          ...
```

## 5. Queue view

แสดง:

- request ID แบบย่อ
- client
- requested alias/model
- effective class
- queue age
- target pool
- deadline
- state

ไม่แสดง prompt body โดย default

operator ต้องเห็น reason ที่ request ยังไม่ได้รัน เช่น `WAITING_CAPACITY`, `PROVIDER_DRAINING`, `BUDGET_HOLD`

## 6. Provider view

ต่อ provider:

- configured endpoint label
- adapter type
- provider version ถ้ารู้
- health dimensions
- active attempts
- capacity
- discovered models
- qualification state
- recent failures
- drain/disable state

## 7. Model view

- alias mappings
- concrete models
- capabilities
- context limit
- qualification state
- loaded/resident status ถ้ารู้
- recent TTFT/throughput
- current route eligibility

## 8. Client view

- authenticated client identity
- allowed aliases/profile summary
- active/queued count
- usage
- rejection count
- latency
- rate/budget state

ไม่ควร expose credential material

## 9. Request detail

operator inspect lifecycle:

```text
RECEIVED
VALIDATED
ADMITTED
QUEUED
CLAIMED
RUNNING
STREAM OPEN
COMPLETED
```

พร้อม timestamps, route decision, attempts, normalized errors, usage และ correlation metadata

content view ถ้ามีต้องถูก policy-gated และ default off

## 10. Controls

safe/read-mostly controls ก่อน:

```text
pause new admissions
drain provider
disable provider route
cancel queued request
request cancel active attempt
reload validated config
```

ทุก control ต้องแสดง consequence ก่อน execute และสร้าง audit event

## 11. Event model

UI ควร subscribe typed events เช่น:

```text
request.admitted
request.queued
request.started
stream.started
request.completed
request.failed
provider.health_changed
provider.draining
model.qualification_changed
config.activated
```

UI event stream ไม่ใช่ source of truth; reconnect แล้วต้อง GET snapshot ปัจจุบันใหม่

## 12. Reconnect behavior

browser refresh/reconnect:

```text
fetch current snapshot
→ obtain event cursor/high-water mark
→ subscribe new events
```

อย่า assume DOM state เดิมครบ

## 13. Metrics

minimum metrics:

### Traffic
- requests total/rate
- admitted/rejected
- streaming/non-streaming

### Scheduler
- queue depth
- queue wait histogram
- slot utilization
- fairness grants
- deadline expiration

### Provider
- availability
- connect failures
- TTFT
- throughput if measurable
- total latency
- cancellation outcome

### Routing
- alias resolution counts
- fallback counts
- selected providers/models

### Reliability
- stream interruptions
- malformed upstream responses
- internal errors
- config reload failures

## 14. Tracing

one inference should correlate:

```text
public request
→ admission
→ route
→ queue
→ lease
→ provider attempt
→ stream/result
```

trace/span IDs ไม่ต้อง leak internally sensitive topology to ordinary clients

## 15. Logs

structured logs preferred:

```json
{
  "event":"provider_attempt.failed",
  "request_id":"...",
  "provider_id":"...",
  "error_class":"PROVIDER_TIMEOUT"
}
```

prompt/output content excluded by default

## 16. Health endpoints

แยก service liveness จาก provider readiness

```text
/liveness  → Cortex process/event loop alive
/readiness → required components ready under deployment policy
```

Ollama down อาจทำ readiness false หรือ degraded ขึ้นกับ configured routes

## 17. GUI technology

Web UI framework เป็น implementation detail เปลี่ยนได้ แต่ API/event contracts ต้อง stable กว่า frontend framework

แนะนำ TypeScript component-based UI เมื่อ dashboard เริ่มซับซ้อน; prototype สามารถเริ่มเรียบง่ายก่อน

## 18. Acceptance criteria

- UI ปิดแล้ว provider API ยังทำงาน
- browser reconnect reconstruct current state ได้
- queue/provider/request state ตรงกับ service snapshot
- control command ผ่าน management authorization
- no direct DB/provider access from frontend
- prompt/output ไม่โผล่ telemetry โดย default
- operator มองเห็นเหตุผลของ queue/routing/failure ได้
- event loss/reconnect ไม่ทำ UI แสดง completed แบบผิดสถานะ
