# 02 — Provider API and Compatibility Implementation Plan

## 1. Goal

สร้าง public provider surface ที่ clients ใช้งานได้จริง โดยทุก endpoint แปลงเข้า normalized inference contract แล้วผ่าน common application path

## 2. Phase 1 — API skeleton

สร้าง:

```text
GET /v1/models
POST /v1/chat/completions
```

พร้อม:

- request IDs
- typed request validation
- normalized error renderer
- non-stream response path
- loopback-only default

ตอนแรกใช้ Fake Provider เพื่อแยก API correctness จาก Ollama behavior

## 3. Phase 2 — Field-by-field support matrix

กำหนด field inventory ของ target API แล้ว classify:

```text
SUPPORTED
SUPPORTED_WITH_PROVIDER_CAPABILITY
UNSUPPORTED_REJECT
RESERVED/FUTURE
```

ทุก field ที่ advertises supported ต้องมี positive/negative tests

## 4. Phase 3 — Normalization

เขียน mapping:

```text
OpenAI DTO
→ NormalizedInferenceRequest
→ Inference Service
```

และ reverse:

```text
NormalizedResponse/Error
→ OpenAI-compatible response
```

preserve omitted/null semantics เมื่อ relevant

## 5. Phase 4 — Streaming API

เพิ่ม SSE framing หลัง scheduler/provider lifecycle stable

ต้อง test:

- ordered chunks
- final event
- mid-stream provider failure
- client disconnect
- arbitrary upstream chunk boundaries
- slow consumer

## 6. Phase 5 — Models endpoint

`/v1/models` อ่านจาก public Model Registry view ไม่ query provider สดทุก request

registry refresh asynchronous/policy-controlled

## 7. Phase 6 — Cortex extensions

เพิ่ม optional headers/metadata สำหรับ:

- client request correlation
- trace correlation
- optional workload hint

extensions ไม่เป็น requirement สำหรับ standard client

## 8. Compatibility tests

สร้าง golden/contract fixtures สำหรับ:

- minimal chat
- multi-turn messages
- sampling options ที่รองรับ
- invalid model
- unsupported feature
- stream true/false
- provider unavailable
- queue timeout

## 9. SDK tests

ทดสอบ target client library โดยตั้ง Base URL มาที่ local Cortex instance

assert:

- model listing
- non-stream completion
- stream iteration
- expected exception shape on errors

## 10. Unsupported behavior

อย่า emulate feature แค่เพื่อให้ test ผ่าน ถ้า semantics ไม่เหมือน

ตัวอย่าง unsupported tool mode ควร fail ก่อน provider call พร้อม clear error

## 11. API performance guardrails

- request body limit
- JSON parsing limit
- no heavy provider/model discovery in request path
- no sync blocking I/O in event loop
- response streaming bounded

## 12. Observability hooks

แต่ละ request emit milestones:

```text
api.received
request.validated
request.admitted
request.queued
request.started
stream.started
request.completed/failed
```

ไม่ log message body

## 13. Security checks

API tests ต้องพิสูจน์:

- anonymous/local policy ตาม config
- authenticated network mode when enabled later
- client cannot set effective policy fields directly
- management routes ไม่อยู่ใต้ `/v1`

## 14. Exit criteria

- target SDK/client tests pass
- support matrix อยู่ใน docs/tests
- unsupported fields deterministic
- API path uses Inference Service, no direct Provider Adapter call
- streaming framing tests pass
- errors normalized
- exact model request preserved
