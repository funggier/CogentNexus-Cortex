# 05 — Streaming Conformance and Client Compatibility Plan

## 1. Goal

พิสูจน์ว่า Cortex ใช้งานเป็น provider endpoint ได้จริงกับ clients ที่ตั้ง Base URL มาที่ Cortex โดยเฉพาะ streaming ซึ่งมักเป็นจุดที่ proxy-compatible implementations พัง

## 2. Test layers

### Protocol fixtures

ทดสอบ HTTP/SSE framing โดยไม่ต้องมี model จริง

### Fake Provider integration

ทดสอบ upstream chunk patterns/failure timing แบบ deterministic

### Ollama integration

ทดสอบ native stream จริง

### Client-library conformance

ทดสอบ client ที่ Cortex ประกาศรองรับ

## 3. Chunk-boundary matrix

Fake Provider ต้อง generate:

- one event per network chunk
- one event splitหลาย chunks
- multiple events in one chunk
- UTF-8 character split boundary
- terminal event separated/coalesced
- malformed/truncated final event

Cortex output ต้อง remain valid

## 4. Exposure/failure tests

- fail before first output → eligible retry according policy
- fail exactly after first exposed chunk → no transparent regenerate
- fail mid-stream → explicit interrupted terminal behavior
- client disconnect → upstream cancel policy
- client reconnect does not resume ordinary stream

## 5. Slow consumer tests

simulate client reading slowly until bounded buffer limit

assert:

- memory bounded
- policy-triggered termination occurs
- provider cancellation attempted
- slot released only at safe point

## 6. Client SDK suite

สำหรับแต่ละ target client:

1. configure Base URL
2. list models
3. non-stream call
4. stream call
5. invalid model
6. unsupported feature
7. timeout
8. provider unavailable

record compatibility evidence and exact client version

## 7. Golden semantics

ไม่ compare model text exact เพราะ nondeterministic

compare:

- request translation
- event structure/order
- IDs
- finish/terminal semantics
- usage mapping
- error class

## 8. Tool/structured output qualification

เพิ่ม test เฉพาะเมื่อ provider/model capability ถูก qualified

ถ้า unsupported, test ต้องพิสูจน์ rejection ก่อน provider invocation

## 9. Regression corpus

เก็บ fixtures ของ bugs เช่น:

- broken SSE framing
- duplicate terminal chunk
- missing model field
- disconnect after header
- provider sends unknown field

ทุก bug เพิ่ม regression test

## 10. Compatibility report

ก่อน release generate matrix:

```text
Endpoint / Feature / Supported / Qualified Providers / Known Limits / Test Evidence
```

## 11. Exit criteria

- arbitrary upstream fragmentation ผ่าน
- output-exposed retry guard ผ่าน
- client disconnect/backpressure tests ผ่าน
- at least target X-Agent provider client works
- advertised SDK/client versions pass suite
- unsupported semantics fail deterministically
- no hidden prompt rewriting used to fake compatibility
