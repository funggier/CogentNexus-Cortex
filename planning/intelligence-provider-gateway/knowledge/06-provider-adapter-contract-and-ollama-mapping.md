# 06 — Provider Adapter Contract and Ollama Mapping

## 1. Purpose

Provider Adapter ทำหน้าที่เป็น **anti-corruption layer** ระหว่าง Cortex semantics กับ provider-native API เพื่อให้ Core ไม่ผูกกับ Ollama/OpenAI/ผู้ให้บริการใดโดยตรง

## 2. Adapter responsibilities

Adapter SHOULD provide interfaces for:

```text
discover_models()
health()
capabilities()
invoke(request)
stream(request)
cancel(attempt)
normalize_usage(native_response)
normalize_error(native_error)
```

บาง method อาจ unsupported แต่ต้องประกาศ capability ชัดเจน

## 3. Adapter must not own

- client admission policy
- queue fairness
- alias policy
- client authentication
- global budget
- Agent Task semantics

Adapter MAY enforce local safety limits แต่ห้าม broaden authority

## 4. Provider capability manifest

ตัวอย่าง:

```yaml
provider_capabilities:
  streaming: true
  cancellation: best_effort
  model_discovery: true
  usage_reporting: partial
  native_tools: qualified
  structured_output: qualified
  max_parallel_calls: 1
  health_probe: true
```

capability ต้องมาจาก qualification ไม่ใช่ assumption

## 5. Normalized request boundary

Adapter รับ canonical request ไม่รับ raw OpenAI HTTP object โดยตรง

ข้อดี:

- provider API เปลี่ยนไม่กระทบ public surface
- test adapter ง่าย
- route/scheduler ไม่ต้องเข้าใจ provider details

## 6. Attempt identity

ทุก call สร้าง `attempt_id`

Adapter เก็บ provider-native request identity ถ้ามี และ report lifecycle:

```text
PREPARED → ISSUED → RUNNING → terminal
```

## 7. Ollama V1 mapping

Ollama เป็น provider แรกเพราะเหมาะกับ local-first และ single-model baseline

Adapter ต้องครอบ:

- endpoint discovery/configuration
- model listing
- chat/generation mapping
- streaming parsing
- timeout
- cancellation behavior ที่พิสูจน์ได้
- model load/unload observation เท่าที่ public API สนับสนุน
- native usage/timing fields

รายละเอียด exact API ต้องตรวจตาม Ollama version ที่ qualification ใช้ก่อน implementation

## 8. Model identity

Cortex ควรเก็บทั้ง:

```text
provider_id
native_model_name
optional digest/version metadata
```

อย่า assume name เดียวหมายถึง binary weights เดิมตลอดเวลา ถ้า provider มี digest ให้ใช้เพื่อ diagnostics/qualification

## 9. Streaming parser

Adapter ต้องรับ arbitrary native chunks และประกอบ event อย่างปลอดภัย

ห้าม assume:

- one socket chunk = one JSON object
- final event มี usage เสมอ
- provider closes cleanly เสมอ

ต้อง test fragmented/coalesced chunks

## 10. Error normalization

ตัวอย่าง mapping:

```text
connection refused → PROVIDER_UNAVAILABLE
model missing       → MODEL_NOT_FOUND
request timeout     → PROVIDER_TIMEOUT
bad native payload  → PROVIDER_PROTOCOL_ERROR
```

เก็บ native diagnostics แต่ Core ใช้ canonical error

## 11. Cancellation

Adapter ต้องประกาศระดับ:

```text
NOT_SUPPORTED
BEST_EFFORT
CONFIRMED
```

ถ้า best-effort Cortex scheduler ห้าม release slot เพียงเพราะส่ง cancel แล้ว ต้องรอ terminal/known-safe boundary

## 12. Health qualification

health ไม่ใช่ boolean เดียว

แนะนำ:

```yaml
health:
  transport_reachable: true
  api_responsive: true
  inventory_readable: true
  target_model_present: true
  probe_inference_passed: optional
```

## 13. Adapter isolation

ถ้า adapter parse error ต้อง fail request อย่าง explicit แต่ไม่ทำให้ scheduler loop ล้ม

provider-native exceptions ต้องไม่ escape ขึ้น public API โดยตรง

## 14. Provider updates

เมื่อ Ollama update:

1. detect version/change
2. run adapter qualification suite
3. update capability state
4. only then re-enable affected advanced features

Core semantics ไม่ควรเปลี่ยนตาม provider release

## 15. Fake Provider Adapter

ต้องมี fake/deterministic adapter ตั้งแต่ต้น รองรับ scenarios:

- instant success
- delayed first token
- slow stream
- malformed chunk
- disconnect before output
- disconnect after output
- cancel ignored
- timeout
- model not found
- usage missing

Fake Adapter เป็นหัวใจของ scheduler/stream/retry tests

## 16. Acceptance criteria

- Core imports no Ollama-specific DTOs
- Fake Adapter ใช้แทน Ollama ได้โดยไม่แก้ scheduler
- model discovery normalize ได้
- fragmented stream parse ถูก
- cancellation semantics declared accurately
- provider-native failure mapped stable
- update qualification สามารถ mark feature DEGRADED โดยไม่ปิด provider ทั้งหมดถ้าไม่จำเป็น
- provider diagnostics inspect ได้โดยไม่ leak secret
