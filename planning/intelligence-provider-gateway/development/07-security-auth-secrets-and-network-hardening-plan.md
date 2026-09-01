# 07 — Security, Authentication, Secrets, and Network Hardening Plan

## 1. Goal

ทำให้ Cortex ปลอดภัยตั้งแต่ local-first V1 และมีเส้นทางขยายไป LAN/remote โดยไม่ต้องรื้อ security model ใหม่

## 2. Stage 1 — Local secure baseline

- bind loopback only
- management API same-host only initially
- no provider credentials in logs/UI
- prompt/output retention disabled by default
- request size and queue limits enabled

## 3. Stage 2 — Client identity

สร้าง canonical `client_id` และ policy mapping

local mode อาจเริ่มด้วย configured client tokens/credentials ที่เรียบง่าย แต่ต้องแยก identity จาก user-supplied headers

## 4. Stage 3 — Authorization policy

per-client policy fields:

```text
allowed endpoints
allowed aliases/models
max active
max queued
rate limit
budget
cloud routing permission
priority class ceiling
```

policy evaluation อยู่ admission service

## 5. Stage 4 — Management authorization

แยก operator permissions จาก inference client permissions

read-only dashboard account/credential ไม่ควรมี provider.disable/config.reload โดยอัตโนมัติ

## 6. Secrets

สร้าง secret-provider abstraction:

```text
resolve(secret_ref) -> secret value only inside backend process
```

V1 อาจรองรับ environment/OS credential mechanism ที่เหมาะสม แต่ config ใช้ reference

## 7. Redaction tests

ทดสอบ intentional canary secret ผ่าน:

- startup errors
- provider errors
- structured logs
- management API
- Web UI
- exception trace

assert canary ไม่ปรากฏ

## 8. Content-retention tests

ส่ง unique prompt/output marker แล้วตรวจ DB/log directory ว่าไม่ถูก persist เมื่อ default policy

## 9. Network mode

ก่อนรองรับ non-loopback ต้องเพิ่ม:

- explicit enable flag
- authentication mandatory
- TLS/secure tunnel deployment guidance
- restrictive CORS/origin rules
- rate limiting
- management bind policy
- threat review

## 10. Cloud provider guard

routing test ต้องพิสูจน์ว่า client/profile ที่ `cloud_allowed=false` ไม่มีทาง fallback ไป cloud แม้ local provider down

## 11. Input hardening

limits:

- body bytes
- message count
- tool definition count/size
- metadata size
- estimated context
- header size via server config

reject ก่อน expensive normalization/token work เมื่อเป็นไปได้

## 12. Dependency/supply security

- pin/lock dependencies appropriately
- CI dependency review/scanning ตาม tooling ที่เลือก
- minimal dependencies
- provider version recorded in qualification

## 13. Audit

management changes produce structured audit events with actor/action/target/config revision/time ไม่ include secret/content

## 14. Exit criteria

- loopback default verified
- client cannot self-escalate model/priority policy
- management and provider permissions separated
- canary secrets absent from logs/UI/API
- content retention default test passes
- cloud fallback negative test passes
- oversized malformed requests rejected early
- network exposure cannot be enabled accidentally by missing config
