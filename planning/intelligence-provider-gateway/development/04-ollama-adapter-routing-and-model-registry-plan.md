# 04 — Ollama Adapter, Routing, and Model Registry Plan

## 1. Goal

เชื่อม Ollama เป็น provider แรกโดยไม่ให้ Ollama-specific behavior รั่วเข้า Core และสร้าง model registry/routing ที่พร้อมรองรับ provider อื่นในอนาคต

## 2. Discovery before implementation assumptions

ก่อน map exact API ต้อง capture Ollama version และ public behavior ที่ใช้จริง:

- model listing
- chat/generation endpoints
- streaming format
- usage/timing fields
- model load behavior
- cancellation behavior
- errors/status codes

สร้าง qualification notes/fixtures จาก behavior จริง ไม่ยึด memory หรือ undocumented assumption

## 3. Adapter skeleton

implement interface:

```text
health
discover_models
invoke
stream
cancel
normalize_error
normalize_usage
```

unsupported capability ต้อง return declared unsupported state

## 4. Fake-to-real parity

ทุก scenario ที่ Fake Provider ใช้ทดสอบ Core ควรมี mapping ว่า Ollama สามารถ reproduce/observe อย่างไร หรืออะไรเป็น adapter-only simulation

## 5. Model registry refresh

startup:

```text
load configured providers
→ health probe
→ discover models
→ normalize inventory
→ compare qualification records
→ publish registry snapshot
```

refresh periodic/manual ตาม config แต่ไม่ query provider live จาก `/v1/models` request ทุกครั้ง

## 6. Qualification

ต่อ model/provider capability:

```text
chat
streaming
tools
structured_output
context limit
usage quality
cancellation
```

status:

```text
DISCOVERED → QUALIFYING → QUALIFIED
                         ↘ DEGRADED
```

## 7. Exact route first

ก่อน aliases ให้ exact selected model ทำงานครบ:

```text
API → normalized → scheduler → ollama adapter → model
```

จากนั้นเพิ่ม `cortex:auto`

## 8. Alias routing V1

เริ่ม deterministic static rules:

```yaml
aliases:
  cortex:auto:
    candidates:
      - provider: ollama-local
        model: <qualified-local-model>
```

ยังไม่ต้อง LLM-based router

## 9. Health probes

แยก cheap periodic health จาก inference qualification probe

- reachability frequent
- inventory moderate
- actual probe inference only when needed/qualified to avoid unnecessary compute

## 10. Provider drain

implement state:

```text
ACTIVE
DRAINING
DISABLED
```

DRAINING: no new route, allow current attempts to finish

## 11. Model switching measurements

เก็บ baseline:

- cold load latency
- warm TTFT
- memory behavior
- switch latency

ใช้ข้อมูลจริงก่อนเพิ่ม scheduler affinity

## 12. Error tests

- Ollama not running
- model missing
- provider restart mid-call
- malformed/unexpected response fixture
- timeout
- slow stream

## 13. Version/update qualification

record provider version/model digest if available

เมื่อ version/digest changes:

- mark capabilities requiring requalification according to policy
- basic reachability may remain
- advanced alias should not rely on stale qualification blindly

## 14. Exit criteria

- Ollama adapter passes provider contract suite
- Core has no native Ollama request classes
- inventory and health visible
- exact route works stream/non-stream
- provider restart produces normalized failure/recovery
- static alias resolves with auditable decision
- unqualified feature rejected before invocation
