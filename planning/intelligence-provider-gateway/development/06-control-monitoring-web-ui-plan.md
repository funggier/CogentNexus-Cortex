# 06 — Control and Monitoring Web UI Plan

## 1. Goal

สร้าง Local Web Dashboard ที่ช่วย operator เห็นและควบคุม Cortex โดยไม่เข้าไปอยู่ใน inference correctness path

## 2. Delivery stages

### Stage A — Read-only status

หน้าแรกแสดง:

- service state
- provider health
- model registry
- active slot
- queue depth
- request rate
- recent normalized errors

### Stage B — Live events

เพิ่ม SSE/WebSocket feed สำหรับ queue/request/provider changes

reconnect ใช้ snapshot + event cursor ไม่ rely DOM history

### Stage C — Request/queue inspection

views:

- request lifecycle
- route decision
- provider attempts
- timing/usage
- queue age/class/client

prompt/output content hidden by default

### Stage D — Controls

เพิ่ม controls ตามลำดับความเสี่ยง:

1. pause/resume admissions
2. cancel queued request
3. request cancellation active attempt
4. drain provider
5. disable/enable route
6. reload validated config

ทุก action ผ่าน Management API และ audit

## 3. Backend API first

ก่อน UI component ต้องมี management endpoints/services ที่ test ได้เอง

ตัวอย่าง:

```text
GET /cortex/v1/status
GET /cortex/v1/providers
GET /cortex/v1/models
GET /cortex/v1/queue
GET /cortex/v1/requests/{request_id}
GET /cortex/v1/usage
POST /cortex/v1/control/admissions/pause
POST /cortex/v1/providers/{provider_id}/drain
POST /cortex/v1/requests/{request_id}/cancel
```

route names เป็น draft implementation shape ไม่ใช่ invariant

## 4. Frontend architecture

frontend framework เลือกตามความเหมาะสม แต่แนะนำ TypeScript + component-based UI เมื่อเริ่มจริง

UI state แยก:

- snapshot state จาก REST
- live event deltas
- local view/filter state

## 5. UX principles

- state words ตรงกับ canonical ontology
- error มี reason/context ไม่ใช่สีแดงอย่างเดียว
- queue wait บอกเหตุผล
- exact model vs alias แสดงต่างกัน
- resolved provider/model inspect ได้
- partial/interrupted stream ไม่ใช้ icon success

## 6. Sensitive data

default UI ไม่แสดง:

- full prompt
- full output
- provider keys
- raw authentication headers

operator debug detail ถ้าเพิ่มต้อง permission-gated

## 7. Control confirmation

actions ที่กระทบ traffic เช่น disable provider/reload route ควรแสดง impact summary ก่อนส่งคำสั่ง แต่ไม่เพิ่ม confirmation dialog ให้ทุก harmless actionจน UX หนัก

## 8. Accessibility/operability

- keyboard navigable controls
- readable status text ไม่พึ่งสีอย่างเดียว
- timestamps local + raw/UTC inspect option
- filtering by client/model/provider/error

## 9. Performance

UI ไม่ควร subscribe token-by-token ของทุก stream โดย default เพราะ event volume สูง

monitor aggregate/lifecycle events; detailed stream debugging เป็น opt-in

## 10. Packaging

V1 serve frontend static assets จาก Cortex service เพื่อ setup ง่าย

ภายหลังแยก frontend deployment ได้โดย management API unchanged

## 11. Tests

- API contract tests
- reconnect state reconstruction
- event loss/reorder simulation
- unauthorized control rejection
- stale UI snapshot then control conflict handling
- provider disappears while page open

## 12. Exit criteria

- dashboard gives truthful current status
- service works with browser closed
- UI cannot mutate DB/provider directly
- reconnect reconstructs correct state
- controls audited and authorized
- sensitive content hidden by default
- queue/provider/request failures explainable from UI
