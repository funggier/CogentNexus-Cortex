# 09 — Release, Packaging, Migration, and Operations Plan

## 1. Goal

ทำให้ Cortex ติดตั้ง อัปเกรด สำรอง กู้คืน และหยุด/เริ่ม service ได้โดยไม่ทำให้ provider configuration, policy, usage state หรือ operator diagnostics สูญหาย

## 2. Packaging target

V1 ควรให้ผู้ใช้รันเป็น local application/service โดยไม่ต้องเข้าใจ Python environment ภายใน

แนวทาง packaging อาจใช้ executable bundler ภายหลัง แต่ architecture ต้องไม่ผูกกับ bundler ใด

แยก directories:

```text
bin/
config/
state/
logs/
cache/
web-assets/
```

upgrade binaries ไม่ควรลบ `state/` หรือ `config/`

## 3. Runtime commands

ต้องมี CLI/management equivalents สำหรับ:

```text
start
stop
status
drain
validate-config
show-providers
show-models
```

service lifecycle ต้องมี exit codes ที่ machine-readable

## 4. Startup sequence

```text
load bootstrap config
→ open persistence
→ validate schema
→ load active config
→ increment process epoch
→ recover stale records
→ initialize scheduler
→ initialize provider registry
→ health/discovery
→ bind API
→ readiness true
```

หาก critical step fail ต้องไม่เปิด readiness หลอก

## 5. Graceful shutdown

```text
pause new admissions
→ mark DRAINING
→ wait configured grace period
→ request cancel remaining attempts if policy
→ flush critical audit/accounting
→ close providers/DB
→ exit
```

## 6. Release candidate discipline

freeze exact candidate SHA ก่อน final acceptance

report:

- exact SHA
- package hash
- schema revision
- config format revision
- supported API matrix
- provider qualification versions
- tests/benchmarks

อย่า patch installed candidate แล้วเรียก SHA เดิม

## 7. Migration policy

DB migrations:

- ordered
- checksummed
- forward-only default
- backup before high-risk migration
- verified after apply

configuration migration ต้อง explicit ไม่ silently drop fields

## 8. Compatibility windows

กำหนด:

```text
minimum supported schema
maximum supported schema
minimum config revision
provider adapter qualification ranges
```

future unsupported state → fail with actionable message

## 9. Backup and restore

backup includes:

- Cortex DB
- config/policy files or revisions
- optional operator certificates/refs according deployment process

ไม่ต้อง backup Ollama model weights โดย default

restore test ต้องรันจริง ไม่ถือว่า file copy สำเร็จ = restore usable

## 10. Log rotation and retention

logs ต้อง bounded:

- rotate by size/time
- retention configurable
- structured format
- no prompt/output by default

DB history cleanup มี separate retention job

## 11. Observability during operations

`status` ต้องรายงาน:

- service state
- process epoch
- schema/config revision
- provider readiness
- active/queued count
- admission paused/draining state
- last critical persistence error

## 12. Installer defaults

secure defaults:

- loopback binding
- no cloud provider configured implicitly
- no content logging
- one-slot scheduler unless detected/configured otherwise
- Web UI local only

## 13. Upgrade test matrix

- fresh install
- install-over previous supported release
- config retained
- DB migration
- rollback plan when migration not backward-compatible
- provider unavailable during upgrade
- interrupted upgrade recovery

## 14. Operational runbooks

create docs later for:

- Ollama unavailable
- model missing
- queue stuck
- DB locked/corrupt
- config invalid
- high latency
- provider flapping
- excessive memory

runbook ต้องอ้าง telemetry/status ที่ implementation มีจริง

## 15. Exit criteria

- fresh install starts local service
- upgrade preserves state/config
- migrations deterministic
- graceful drain tested
- status usable when provider down
- logs/storage bounded
- restore test passes
- release artifact traceable to exact source SHA
