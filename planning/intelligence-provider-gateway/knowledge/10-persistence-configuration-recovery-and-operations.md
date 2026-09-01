# 10 — Persistence, Configuration, Recovery, and Operations

## 1. Purpose

Cortex ไม่ใช่ Agent durable-work engine แต่ยังต้องมี persistence สำหรับ configuration authority, policy revisions, provider/model qualification, usage/accounting, audit และ operational recovery โดยไม่แสร้งว่าทุก synchronous inference สามารถ resume ได้หลัง process crash

## 2. Durability boundary

ต้องแยกให้ชัด:

### Durable Cortex facts

- configuration revisions
- client identities/policies
- provider registrations
- model/alias qualification state
- route policy revisions
- usage/accounting summaries
- management audit
- optional request metadata/history ตาม retention policy

### Ephemeral execution state

- active HTTP connection
- in-memory streaming buffers
- current socket state
- ordinary synchronous response delivery

## 3. No false recovery promise

เมื่อ Cortex process ตายกลาง ordinary `/v1/chat/completions`:

- client connection ล้ม
- active provider attempt อาจจบหรือถูกตัดตาม OS/provider behavior
- client เป็นผู้ตัดสิน retry
- Cortex restart ไม่ควร claim ว่าจะ resume stream เดิม

ถ้าอนาคตต้องการ durable inference jobs ให้สร้าง API contract แยกพร้อม durable queue/result retrieval

## 4. SQLite recommendation

V1 เหมาะกับ SQLite เพราะ:

- local-first
- transactional
- operational data ไม่ใหญ่มาก
- deployment ง่าย
- backup/inspection straightforward

settings:

```text
foreign_keys=ON
journal_mode=WAL
busy_timeout configured
short transactions
```

ไม่เปิด transaction ค้างระหว่าง provider/network call

## 5. Suggested tables

```text
schema_migrations
system_meta
clients
client_policies
providers
provider_health_snapshots (optional bounded)
models
model_qualifications
model_aliases
routing_policy_revisions
request_records
provider_attempt_records
usage_records
audit_events
config_revisions
```

`request_records` default metadata-only ไม่เก็บ prompt/output body

## 6. Configuration model

แยก static bootstrap config จาก runtime policy/config

bootstrap ตัวอย่าง:

```yaml
listen:
  host: 127.0.0.1
  port: configurable
storage:
  path: ...
```

runtime config:

```yaml
providers: ...
aliases: ...
clients: ...
budgets: ...
scheduler: ...
```

## 7. Configuration activation

ใช้ lifecycle:

```text
DRAFT/FILE
  ↓ parse
VALIDATED
  ↓ stage
STAGED
  ↓ atomic activation
ACTIVE
```

invalid revision ต้องไม่ทำให้ active config หาย

active revision ID ถูกบันทึกเพื่อ route/audit correlation

## 8. Hot reload safety

reload ต้องจำแนก fields:

- safe for new requests immediately
- requires drain
- requires service restart

ตัวอย่าง provider endpoint change อาจใช้ new requests หลัง activation ขณะที่ in-flight attempt เก็บ old provider binding จนจบ

## 9. Process epoch

ทุก Cortex process startup มี `process_epoch`

leases/active execution จาก epoch เก่าถือ stale หลัง restart

ช่วยป้องกัน stale callback/internal worker commit state ข้าม generation

## 10. Startup recovery scan

เมื่อ start:

1. verify schema version
2. load/validate active config
3. invalidate stale leases/process-owned execution markers
4. inspect requests left non-terminal from prior epoch
5. mark ordinary in-flight requests `INTERRUPTED_BY_RESTART` ตาม operational history
6. refresh provider health/model registry
7. resume accounting/audit queues if durable
8. expose readiness onlyเมื่อ policy requirements satisfied

## 11. Migrations

migrations:

- ordered
- checksummed
- forward-only preferred
- transaction where SQLite permits
- backup before destructive/large migration
- Core defines min/max compatible schema
- future unsupported schema → refuse start rather than mutate blindly

## 12. Backup

backup scope:

- SQLite DB
- active configuration files/revisions
- optional TLS/cert references according to secure backup process

provider model weights ของ Ollama ไม่ใช่ Cortex backup responsibility โดย default

## 13. Request retention

policy examples:

```yaml
retention:
  request_metadata_days: 7
  usage_days: 90
  audit_days: 180
  prompt_content: disabled
  output_content: disabled
```

ต้องมี cleanup job ที่ bounded และไม่ block inference path

## 14. Usage accounting consistency

usage record ควร distinguish:

```text
provider_reported
cortex_estimated
unknown
```

อย่าแต่ง token number หาก provider ไม่รายงานและ tokenizer ไม่ตรง

## 15. Crash boundaries

ควร test crash:

- after admission metadata written but before enqueue
- after lease claim before provider call
- after provider issue before first output
- mid-stream
- after provider completion before accounting write
- during config activation
- during migration

ระบบต้องกลับมาเป็น operational state ที่อธิบายได้

## 16. Operations controls

support:

- graceful drain
- stop accepting new inference
- wait bounded time for active requests
- cancel remaining according to policy
- shutdown

service restart จาก Web UI ต้องระวัง connection ที่ใช้สั่ง restart; command audit ก่อน process exit

## 17. Packaging data paths

แยก:

```text
application binaries
config
state database
logs
cache/temp
```

เพื่อ upgrade binary โดยไม่ลบ state

## 18. Observability of persistence

metrics/status:

- DB writable
- schema revision
- active config revision
- last successful cleanup
- audit queue lag
- accounting write failures
- DB size

## 19. Acceptance criteria

- invalid config reload ไม่แทน active config
- no transaction spans provider call
- restart invalidates old execution leases
- interrupted ordinary request ไม่ถูก report success
- future schema ถูก refuse safely
- backup/restore รักษา config/policy/provider registry
- prompt/output body ไม่ persisted by default
- usage source ระบุ reported/estimated/unknown
- graceful drain ป้องกัน new admission และจัดการ active requests ตาม policy
