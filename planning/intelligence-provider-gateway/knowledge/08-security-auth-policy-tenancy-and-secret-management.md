# 08 — Security, Authentication, Policy, Tenancy, and Secret Management

## 1. Purpose

Cortex อยู่หน้าทรัพยากร LLM ของทั้งเครื่องหรือทั้งเครือข่าย จึงเป็น security boundary โดยธรรมชาติ หากเปิดผิด scope client ที่ไม่ควรมีสิทธิ์อาจใช้ compute, ส่งข้อมูลออก cloud provider หรืออ่าน operational metadata ได้

## 2. Secure default

V1 SHOULD bind ที่ loopback เท่านั้น:

```text
127.0.0.1
```

network/LAN exposure ต้องเป็น explicit configuration พร้อม authentication และ origin/network controls

## 3. Identity model

แยก:

- **Client Identity** — application/service ใดเรียก
- **Operator Identity** — ผู้ควบคุม Cortex ผ่าน management API/UI
- **Provider Credential** — credential ที่ Cortex ใช้เรียก backend

สามอย่างนี้ห้าม collapse เป็น token เดียว

## 4. Authentication vs authorization

Authentication ตอบว่า “ใคร”
Authorization ตอบว่า “ทำอะไรได้”

policy อาจกำหนดต่อ client:

```yaml
client_policy:
  allowed_aliases: [cortex:fast, cortex:coding]
  allow_exact_models: false
  max_queue_depth: 4
  max_active_requests: 1
  cloud_routing: false
  daily_budget: optional
```

## 5. Management permissions

Management API ต้องมี permissions แยก เช่น:

```text
status.read
providers.read
requests.read
usage.read
queue.control
provider.drain
provider.disable
config.reload
policy.modify
```

Web UI ไม่ควรได้ high-risk control permission เพียงเพราะ browser เปิดจาก localhost

## 6. Provider credentials

แนวทาง:

- local Ollama อาจไม่ต้องมี provider credential
- cloud keys ใช้ OS credential store/secret manager หรือ protected environment/config reference
- config เก็บ opaque secret reference ไม่ใช่ค่าจริงถ้าเป็นไปได้
- logs/UI/API ต้อง redact

## 7. Prompt and output privacy

Cortex เห็น payload ที่อาจมี source code, personal data หรือ secrets

Default retention policy ควรเป็น:

```text
metadata: retained according to operational policy
prompt body: not persisted by default
output body: not persisted by default
```

debug content logging ต้องเปิด explicit, มี retention limit และ warning

## 8. Telemetry minimization

เก็บได้โดยไม่เก็บ content:

- request ID
- client ID
- model alias/concrete model
- sizes/token counts
- timings
- error category
- route reason codes
- status

ถ้าต้อง hash content ต้องพิจารณาว่า hash ของข้อมูลสั้นอาจเดาได้ จึงไม่ใช้ hash เป็น privacy mechanism โดยลำพัง

## 9. Network exposure

เมื่อเปิด LAN/remote:

- authenticate every request
- TLS หรือ trusted secure tunnel ตาม deployment
- management surface อาจแยก bind/address จาก provider API
- validate WebSocket/SSE origins ตาม use case
- deny broad CORS defaults
- rate limit per client

## 10. Browser security

Web UI:

- ไม่ฝัง provider credentials
- management session short-lived ตาม policy
- state-changing operations require authenticated control request
- protect against cross-origin command triggering
- render model/request metadata safely ไม่ trust raw provider strings as HTML

## 11. Client impersonation

client-supplied headers เช่น `X-Cortex-Client-Id` ไม่ใช่ identity ถ้าไม่มี trusted authentication binding

server map authenticated credential → canonical client_id

## 12. Priority abuse

client ส่ง priority hint ได้ แต่ effective class มาจาก policy

prevent:

```text
untrusted client → INTERACTIVE highest priority forever
```

## 13. Model access policy

บาง models/providers อาจมีข้อจำกัดด้าน cost/data residency

policy อาจระบุ:

```yaml
routing_policy:
  cloud_allowed: false
  provider_allowlist: [ollama-local]
  max_context_tokens: ...
```

Cortex MUST NOT send request ไป cloud โดย surprise fallback

## 14. Denial-of-service considerations

controls:

- max body size
- max messages/tools count
- max estimated tokens
- max queued/client
- rate limit
- max stream duration
- idle timeout
- bounded telemetry queues

## 15. Configuration safety

config change ต้อง:

```text
parse → validate → authorization → stage → activate → audit
```

invalid config ไม่ควร replace active working config

## 16. Secret rotation

provider credential rotation ควรทำได้โดยไม่ restart architecture ทั้งระบบ

new requests ใช้ active credential generation ใหม่; in-flight attempts ไม่ควรถูก mutate กลาง request

## 17. Tenant isolation

V1 อาจเป็น single-user local service แต่ data model ไม่ควร assume anonymous global client

อย่างน้อยต้องมี `client_id` เพื่อ accounting/fairness/policy

multi-user tenancy ที่แท้จริงเป็น future scope และต้องเพิ่ม isolation review ก่อนเปิด

## 18. Audit

ควร audit management events เช่น:

- client policy changed
- provider enabled/disabled
- route config activated
- budget changed
- management authentication failure summary

Audit ไม่ควร include prompt content โดย default

## 19. Security acceptance

- default install ไม่ listen external interfaces
- unauthenticated remote request ถูก reject เมื่อ network mode เปิด
- client ไม่สามารถ self-upgrade priority/permissions
- exact provider credentials ไม่ปรากฏใน UI/log/API
- prompt/output retention default ปิดหรือ minimized
- cloud fallback ต้อง explicit policy
- management control ผ่าน authorization
- malformed/oversized request ถูก reject ก่อน resource-heavy processing
- config reload fail closed ต่อ invalid revision โดย active config เดิมยังใช้ได้
