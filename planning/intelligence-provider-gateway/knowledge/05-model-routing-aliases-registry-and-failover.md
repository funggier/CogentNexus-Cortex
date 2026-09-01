# 05 — Model Routing, Aliases, Registry, and Failover

## 1. Purpose

Routing แยก “สิ่งที่ client ขอ” ออกจาก “provider/model จริงที่ execute” โดยไม่ทำให้ client สูญเสีย control

## 2. Two request modes

### Concrete request

client ระบุ concrete identity ที่ exact

```text
provider/model
```

Cortex pin route และ MUST NOT silently substitute

### Semantic alias

client ขอ capability/policy name เช่น:

```text
cortex:auto
cortex:fast
cortex:coding
cortex:reasoning
cortex:local
```

Cortex resolve ตาม active policy revision

## 3. Why aliases matter

alias ทำให้ client ไม่ต้องแก้ config เมื่อ:

- model ถูกอัปเกรด
- backend ย้ายเครื่อง
- local/cloud routing เปลี่ยน
- memory constraint เปลี่ยน
- provider outage

แต่ alias ต้องมี **declared semantic envelope** ไม่ใช่ชื่อ marketing ที่ route ไปอะไรก็ได้

ตัวอย่าง:

```yaml
alias: cortex:coding
requirements:
  supports_chat: true
  supports_tools: true
  min_context_tokens: 32768
preferences:
  locality: local_first
  cost_class: bounded
```

## 4. Provider Registry

registry ควรแยก:

- configured provider instances
- discovered runtime health
- discovered model inventory
- qualified capabilities
- administrative enable/disable state

หนึ่ง provider instance ตัวอย่าง:

```yaml
provider_id: ollama-local
adapter_type: ollama
endpoint_ref: local-loopback
state: HEALTHY
capacity_pool: local-gpu
```

อย่าเก็บ secret ใน registry record ที่ Web UI อ่านได้

## 5. Model Registry

normalized model record:

```yaml
model_id: model_...
provider_id: ollama-local
native_name: qwen...
public_name: optional
capabilities:
  chat: true
  tools: qualified
  structured_output: qualified
  context_tokens: ...
qualification_state: QUALIFIED
last_seen_at: ...
```

provider-reported capability ไม่จำเป็นต้องเท่ากับ Cortex-qualified capability

## 6. Qualification states

แนะนำ:

```text
DISCOVERED
QUALIFYING
QUALIFIED
DEGRADED
DISABLED
INCOMPATIBLE
```

model ใหม่ที่ provider discover ได้ไม่ควรกลายเป็น public alias target อัตโนมัติสำหรับ critical features

## 7. Route Decision

ทุก alias route สร้าง record แบบ deterministic/inspectable:

```yaml
route_decision:
  requested_model: cortex:coding
  policy_revision: ...
  selected_provider: ollama-local
  selected_model: ...
  reason_codes:
    - CAPABILITY_MATCH
    - LOCAL_PREFERRED
    - HEALTHY
```

ไม่เก็บ chain-of-thought

## 8. Routing inputs

- requested alias/exact identity
- client policy
- required API features
- context requirement
- health
- administrative state
- resource pool status
- budget/cost class
- locality policy
- fallback permission

queue length MAY influence route for aliases แต่ไม่ควรทำให้ semantics หลุด requirement

## 9. Health model

แยก:

```text
process_reachable
api_healthy
model_present
model_callable
capability_qualified
capacity_available
```

HTTP 200 health endpoint ไม่ได้พิสูจน์ model callable

## 10. Failover

Failover มีสองชนิด:

### Pre-execution route failover

provider unavailable ก่อน attempt เริ่ม สามารถเลือก candidate อื่นได้ถ้า alias/policy อนุญาต

### Post-issue failover

หลัง provider request ถูก issue แล้ว ต้องพิจารณา output exposure/retry semantics ไม่ควรเริ่ม backend ใหม่แบบเงียบหลัง stream เริ่ม

## 11. Fallback policy

ควร explicit เช่น:

```yaml
fallback:
  allowed: true
  require_same_capability_profile: true
  allow_local_to_cloud: false
  max_route_attempts: 2
```

exact model request default `allowed=false`

## 12. Cost/budget routing

future cloud routing สามารถใช้ cost signals แต่ budget ต้องเป็น policy input ไม่ใช่ sole criterion

ห้าม route ไป model ต่ำกว่าความสามารถขั้นต่ำเพียงเพราะราคาถูก

## 13. Loaded-model affinity

สำหรับ Ollama/local GPU Cortex อาจ prefer model ที่โหลดอยู่แล้วเพื่อลด switch cost

affinity เป็น preference ไม่ override exact identity หรือ mandatory capabilities

## 14. Configuration revision

alias table และ routing policy ต้อง version/revision แบบ operational เพื่อ audit ว่า request ในอดีตใช้ mapping ไหน

ไม่ต้องใส่ revision ใน semantic alias name

## 15. Manual controls

Management API/UI ควรมี:

- disable provider
- drain provider
- disable model as route target
- pin alias temporarily
- reload route config

control action ทุกอย่างต้อง audit

## 16. Avoid hidden intelligence selection

ถ้า `cortex:auto` เลือก model ต่างกันตาม policy ถือว่าปกติ แต่ client ควร inspect resolved route ผ่าน response metadata/management telemetry ตาม permission

## 17. Acceptance criteria

- exact request ไม่ fallback โดย default
- alias route ผ่าน capability requirements
- unhealthy/disabled provider ไม่รับ route ใหม่
- current loaded model affinity ไม่ละเมิด required capability
- policy revision ถูกบันทึก
- discovered-but-unqualified model ไม่ถูกใช้ใน qualified alias
- pre-execution failover ทำได้โดยไม่มี duplicate stream
- post-output failure ไม่ silently reroute
- operator สามารถอธิบาย selected route จาก reason codes ได้
