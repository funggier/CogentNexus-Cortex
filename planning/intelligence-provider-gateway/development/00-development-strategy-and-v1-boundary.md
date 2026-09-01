# 00 — Development Strategy and V1 Boundary

## 1. Development objective

สร้าง CogentNexus-Cortex จาก contract ที่เล็กแต่ครบก่อน แล้วขยายโดยไม่เปลี่ยน core semantics

V1 success ไม่ใช่มี feature เยอะ แต่คือเส้นทาง inference หลักมี behavior ที่พิสูจน์ได้:

```text
OpenAI-compatible Client
  → Cortex admission
  → normalized request
  → scheduler slot=1
  → Ollama adapter
  → local model
  → response/stream
  → accounting/observability
```

## 2. Engineering principles

- TDD สำหรับ protocol/state/scheduler behavior
- deterministic Fake Provider ก่อน real Ollama integration
- no direct provider calls outside adapter
- no unbounded queue/buffer
- small vertical slices
- failure injection ตั้งแต่ต้น ไม่รอท้าย project
- explicit unsupported behavior
- security/local-only defaults ก่อน convenience
- benchmark ก่อน performance optimization

## 3. V1 functional scope

MUST:

- Python service
- OpenAI-compatible `GET /v1/models`
- supported subset of `POST /v1/chat/completions`
- non-stream + stream
- normalized internal request/response
- client identity baseline
- one resource pool / one inference slot
- bounded queue
- deterministic scheduling
- Ollama Provider Adapter
- provider/model health discovery
- exact model route
- minimal alias support (`cortex:auto` optional after exact path works)
- normalized errors
- basic usage/timing
- structured logs
- management status API
- basic local Web monitoring
- graceful shutdown/drain

## 4. Explicit V1 non-goals

- distributed Cortex cluster
- multi-node consensus
- automatic prompt optimization
- Agent Task semantics
- RAG/memory
- provider tool emulation by hidden prompting
- complex cost optimizer
- multiple cloud providers
- durable inference job resume
- multi-user internet deployment
- production HA

## 5. Milestone gates

### Gate A — Domain/contract foundation

- ontology types
- request lifecycle
- error taxonomy
- provider adapter interface
- Fake Provider

### Gate B — Inference kernel

- scheduler slot=1
- admission
- route exact model
- provider attempt lifecycle
- non-stream result

### Gate C — Streaming

- bounded stream pipeline
- output exposure tracking
- cancel/disconnect semantics
- malformed stream tests

### Gate D — Ollama

- discovery
- model call
- streaming
- error mapping
- health

### Gate E — Compatibility

- target client conformance
- supported/unsupported matrix
- exact field behavior

### Gate F — Operations

- management API
- Web monitor
- usage/metrics
- config reload
- drain/shutdown

### Gate G — Reliability/Security acceptance

- queue overload
- provider crash
- Cortex restart
- partial stream
- invalid config
- access controls

## 6. Release philosophy

ไม่เรียก V1 ready จน:

- happy path ผ่าน
- key negative/chaos paths ผ่าน
- advertised compatibility backed by tests
- default install local-only
- no prompt/output persistence by default
- X-Agent สามารถชี้ Base URL มาที่ Cortex และใช้งานผ่าน supported API ได้

## 7. Incremental implementation sequence

```text
contracts
→ Fake Provider
→ scheduler
→ internal inference service
→ public API non-stream
→ streaming
→ Ollama adapter
→ compatibility tests
→ management API
→ Web monitor
→ persistence/config
→ security hardening
→ chaos/performance
```

## 8. Definition of safe progress

ทุก phase ต้องรักษา:

- tests green
- no bypass path
- bounded resource behavior
- explicit errors
- documentation update หาก contract เปลี่ยน

## 9. V1 acceptance narrative

ผู้ใช้ติดตั้ง Cortex บนเครื่องเดียวกับ Ollama แล้วตั้ง client Base URL มาที่ Cortex จากนั้น:

1. client list model/aliases ได้
2. ส่ง inference ได้
3. Cortex จัดคิวเมื่อ slot busy
4. stream มาถูกต้อง
5. dashboard เห็น queue/provider/model/latency
6. provider ล้มแล้ว client ได้ explicit error
7. request ใหม่กลับมาทำงานหลัง provider recover
8. X-Agent/Hermes/อื่นสามารถเป็น client เพิ่มโดยไม่แก้ Cortex Core semantics
