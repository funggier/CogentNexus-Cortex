# 03 — Provider API Compatibility and Contracts

## Purpose

Cortex ต้องเป็น provider endpoint ที่ client ใช้งานได้จริง โดยเฉพาะ OpenAI-compatible clients คำว่า compatibility จึงต้องอ้างถึง behavior ที่ทดสอบได้ ไม่ใช่เพียงใช้ path และ JSON คล้ายกัน

## Compatibility levels

1. **Shape** — endpoint และ field structure
2. **Transport** — HTTP, SSE, ordering, termination
3. **Semantic** — field เดียวกันมีความหมายสอดคล้องกัน
4. **Behavioral** — client libraries ที่ระบุไว้ทำงานผ่าน conformance suite

Cortex ควรประกาศเฉพาะสิ่งที่มี test evidence

## V1 surface

เริ่มจาก subset ขนาดเล็ก:

```text
GET  /v1/models
POST /v1/chat/completions
```

เพิ่ม `/v1/responses` หลัง contract ของ endpoint แรกนิ่งแล้ว

## Request processing

```text
parse
→ structural validation
→ compatibility validation
→ authenticate
→ policy/admission
→ normalize
→ schedule
```

Field ต้องถูกจำแนกเป็น:

- supported-required
- supported-optional
- recognized-but-unsupported
- unknown

recognized-but-unsupported ต้องตอบ error ชัดเจน ไม่ ignore เงียบ

## Normalized inference contract

API DTO ไม่ควรไหลเข้า Provider Adapter ตรง ๆ

```yaml
normalized_inference:
  requested_model: cortex:auto
  messages: [...]
  tools: [...]
  response_format: ...
  temperature: ...
  max_output_tokens: ...
  stop: ...
  stream: true
  client_metadata: {...}
```

บาง field ต้อง preserve ความต่างระหว่าง omitted, null และ explicit value

## Exact model and alias

Exact concrete model ต้อง pin route เว้นแต่ request explicitly อนุญาต fallback

Alias เช่น:

```text
cortex:auto
cortex:fast
cortex:coding
```

มี routing policy ได้ แต่ Route Decision ต้อง inspect ได้

## Model listing

`GET /v1/models` แสดงเฉพาะ model/aliases ที่ client นั้นได้รับอนุญาต ไม่จำเป็นต้องเปิดเผย backend inventory หรือ internal endpoint names ทั้งหมด

## Tool calling

ความสามารถด้าน tools ต้องมี capability matrix:

```yaml
tools: native | explicit_emulation | unsupported
parallel_tool_calls: true | false
structured_output: native | validate_only | unsupported
```

Transparent mode ห้ามใช้ hidden prompt rewriting เพื่อทำให้ unsupported feature ดูเหมือน native

## Structured output

ต้องแยกสามกรณี:

1. provider บังคับ schema natively
2. Cortex ตรวจ schema หลัง generation
3. prompt ขอ JSON แต่ไม่มี enforcement

สามกรณีนี้มี reliability ต่างกันและต้อง report ต่างกัน

## Streaming

Cortex ต้อง parse provider-native stream แล้ว reframe เป็น public protocol อย่างถูกต้อง

ข้อกำหนด:

- ordered chunks
- stable request identity
- valid framing แม้ upstream แบ่ง byte/chunk ไม่ตรง JSON boundary
- bounded buffering
- explicit terminal state
- stream interruption ไม่ถูก report เป็น success

## Error normalization

Cortex มี canonical categories เช่น:

```text
INVALID_REQUEST
AUTHENTICATION_FAILED
AUTHORIZATION_FAILED
MODEL_NOT_FOUND
MODEL_UNAVAILABLE
QUOTA_EXCEEDED
QUEUE_LIMIT
PROVIDER_UNAVAILABLE
PROVIDER_TIMEOUT
PROVIDER_PROTOCOL_ERROR
STREAM_INTERRUPTED
CORTEX_INTERNAL_ERROR
```

จากนั้น API layer map เป็น HTTP/status/error shape ที่ surface รองรับ

native diagnostics เก็บเพื่อ operator แต่ไม่ใช้ native provider error เป็น Core ontology

## Timeout dimensions

แยกอย่างน้อย:

- queue deadline
- provider connect timeout
- time-to-first-token timeout
- stream idle timeout
- total execution deadline

ไม่ใช้ timeout เดียวแบบกำกวม

## Cortex extensions

Cortex-specific metadata ต้อง namespaced และ optional เช่น correlation/client request IDs เพื่อไม่ทำลาย client compatibility

## Version separation

ต้องแยก:

- public API compatibility version
- internal normalized schema revision
- provider adapter compatibility revision
- software release

semantic names ไม่พ่วง release version

## Conformance matrix

ทุก endpoint ต้องมี matrix ตัวอย่าง:

| Feature | V1 status | Behavior |
|---|---|---|
| messages | supported | preserved |
| non-stream response | supported | normalized |
| SSE streaming | supported | normalized |
| tools | capability-dependent | explicit |
| structured output | capability-dependent | explicit |
| unsupported field | rejected | deterministic |

## Client test targets

V1 ควรพิสูจน์กับ:

- raw HTTP fixture
- common OpenAI-compatible Python client configuration
- X-Agent provider client
- representative third-party client อย่างน้อยหนึ่งตัวเมื่อพร้อม

## Non-goals

- รองรับทุก field ตั้งแต่แรก
- silent fallback ของ exact model
- hidden semantic transformation
- claim compatibility จาก curl happy path เพียงอย่างเดียว

## Acceptance criteria

- supported fields ถูกระบุและทดสอบ
- unsupported fields ไม่ถูก ignore แบบเงียบ
- exact model pinning ผ่าน tests
- alias route inspect ได้
- arbitrary streaming chunk boundaries ไม่ทำ framing เสีย
- interruption ไม่กลายเป็น success
- capability-dependent features ถูก gate ก่อน provider invocation
- client conformance suite ผ่านสำหรับ target clients ที่ประกาศรองรับ
