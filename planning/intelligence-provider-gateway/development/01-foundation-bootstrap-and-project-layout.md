# 01 — Foundation Bootstrap and Project Layout

## 1. Purpose

วาง repository/package/process structure ให้ contract boundaries ถูก enforce ตั้งแต่ commit แรก แทนที่จะ refactor หลัง provider-specific code กระจาย

## 2. Recommended stack

```text
Python 3.x
FastAPI / ASGI
Pydantic or equivalent typed validation
asyncio
SQLite
pytest
```

เลือก dependency เพิ่มเท่าที่จำเป็น

## 3. Proposed repository layout

```text
src/cogentnexus_cortex/
├── api/
│   ├── provider/
│   └── management/
├── application/
│   ├── admission.py
│   ├── inference.py
│   ├── routing.py
│   └── control.py
├── domain/
│   ├── requests.py
│   ├── models.py
│   ├── providers.py
│   ├── scheduling.py
│   ├── errors.py
│   └── events.py
├── scheduler/
├── providers/
│   ├── base.py
│   ├── fake.py
│   └── ollama.py
├── persistence/
│   ├── sqlite/
│   └── migrations/
├── security/
├── telemetry/
├── config/
├── web/
└── main.py

tests/
├── unit/
├── contract/
├── integration/
├── conformance/
├── chaos/
└── performance/
```

## 4. Dependency rules

- `domain/` ไม่ import FastAPI/Ollama
- `api/` เรียก application services
- provider-specific HTTP client อยู่ `providers/`
- persistence implements interfaces ไม่ถูกเรียกแบบ SQL จาก UI/API route
- Web frontend ใช้ management API เท่านั้น

ควรมี architecture tests/import checks หากทำได้

## 5. First domain types

สร้างก่อน external integration:

```text
InferenceRequest
RequestState
ProviderAttempt
RouteDecision
ModelReference
ModelAlias
ProviderReference
ResourcePool
ResourceLease
NormalizedError
UsageRecord
StreamState
```

## 6. Configuration bootstrap

เริ่ม config ขนาดเล็ก:

```yaml
listen:
  host: 127.0.0.1
  port: 18800
storage:
  path: ./state/cortex.db
scheduler:
  default_slots: 1
providers:
  - id: ollama-local
    type: ollama
    endpoint: http://127.0.0.1:11434
```

เลข port เป็น default candidate ไม่ใช่ semantic invariant และต้อง configurable

## 7. Development environment

ต้องมี commands มาตรฐาน:

```text
install dev dependencies
format/lint
type check
test unit
test integration
run local service
```

ลงใน README ของ repository เมื่อ implementation เริ่ม

## 8. Logging bootstrap

structured log fields minimum:

```text
timestamp
level
event
request_id
client_id
provider_id
attempt_id
trace_id
error_class
```

ห้าม include request body โดย default

## 9. Test fixtures first

ก่อน Ollama Adapter สร้าง Fake Provider ที่ programmable behavior เพื่อให้ core tests ไม่ขึ้นกับ model speed/network

## 10. CI baseline

ตั้ง workflow:

- install
- lint/type
- unit
- contract
- selected integration with fake provider

real Ollama CI อาจเป็น optional/self-hosted stage ภายหลัง

## 11. Coding discipline

- no catch-all exception without normalized handling/log
- no background task without lifecycle owner
- no unbounded asyncio.Queue
- no network call inside DB transaction
- no mutable global config without revision/activation path
- no direct `httpx`/provider call from API routes

## 12. Initial bootstrap tasks

1. create `pyproject.toml`
2. package skeleton
3. typed config loader
4. domain enums/data models
5. normalized errors
6. Fake Provider interface
7. minimal health service
8. unit test harness
9. CI
10. local service entry point

## 13. Exit criteria

Foundation phase จบเมื่อ:

- package imports clean
- one command starts service
- domain unit tests pass
- Fake Provider can be injected
- config invalidity produces deterministic startup error
- loopback default verified
- no Ollama-specific type leaks into domain
- CI reproducible
