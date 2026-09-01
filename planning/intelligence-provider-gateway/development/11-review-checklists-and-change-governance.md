# 11 — Review Checklists and Change Governance

## 1. Purpose

Cortex มีความเสี่ยงที่จะค่อย ๆ กลายเป็น proxy monolith หาก feature ใหม่เพิ่มโดยไม่ตรวจ responsibility และ compatibility เอกสารนี้เป็น checklist สำหรับ design/code/release review

## 2. Architecture review checklist

ก่อนรับ feature ใหม่ถาม:

- เป็นเรื่อง allocation/access to intelligence จริงหรือไม่?
- กำลังนำ Agent Task/Intent semantics เข้ามาใน Cortex หรือไม่?
- สามารถวางหลัง existing service/adapter boundary ได้หรือไม่?
- provider-specific behavior รั่วเข้า domain/public API หรือไม่?
- มี bypass scheduler/capacity path หรือไม่?
- behavior ยังถูกต้องเมื่อ slot=1 หรือไม่?
- queue/buffer ใหม่ bounded หรือไม่?

## 3. Public API change checklist

- endpoint/field เป็น standard-compatible หรือ Cortex extension?
- omitted/null/default semantics ระบุหรือไม่?
- unsupported providers/models ถูก gate อย่างไร?
- error mapping มี conformance tests หรือไม่?
- streaming behavior เปลี่ยนหรือไม่?
- existing client compatibility แตกหรือไม่?
- ต้อง bump compatibility version หรือไม่?

## 4. Provider Adapter review

- Core imports native DTO หรือไม่?
- adapter declares capability honestly?
- error normalized แต่ native diagnostics retained หรือไม่?
- cancellation level tested หรือ assumed?
- stream parser handles arbitrary fragmentation?
- provider update requires requalification อะไร?

## 5. Scheduler review

- request can bypass queue/lease หรือไม่?
- priority from untrusted client ถูก trusted หรือไม่?
- fairness/starvation impact?
- new resource pool capacity definition clear?
- cancel releases slot at safe boundary?
- deadline behavior deterministic?
- metrics explain wait reason?

## 6. Routing review

- exact model pinning preserved?
- alias semantic envelope changed?
- fallback explicit?
- cloud/local data movement policy respected?
- route decision audit records reason/policy revision?
- unqualified model can become candidate accidentally?

## 7. Streaming/retry review

- output exposure tracked?
- retry after first chunk possible accidentally?
- buffer bounded?
- client disconnect behavior defined?
- partial stream rendered terminal correctly?
- provider timeout category specific?

## 8. Security review

- new listener/network exposure?
- new credential/secret path?
- new content retention?
- management permission required?
- client can self-escalate field?
- Web UI receives sensitive material unnecessarily?
- cloud routing can occur unexpectedly?

## 9. Persistence/migration review

- data truly needs durability?
- prompt/output body being stored? why?
- migration forward/rollback/backup considered?
- transaction spans external I/O?
- retention/cleanup defined?
- active config remains safe on failed update?

## 10. Web UI review

- UI reads from management API?
- control goes through application policy?
- snapshot/reconnect works?
- UI state terminology matches canonical ontology?
- redaction/default privacy preserved?
- event feed treated as cache/delta rather than truth?

## 11. Performance review

ก่อนเพิ่ม optimization:

- measured bottleneck evidence?
- benchmark reproducible?
- optimization changes semantics/fairness?
- memory bounds still known?
- complexity justified by material gain?

## 12. Documentation governance

Canonical changes must update relevant:

- knowledge document
- development/acceptance plan
- API matrix
- tests
- changelog/ADR when implementation exists

ห้ามมีสองคำสำหรับ entity เดียวโดยไม่มีเหตุผล

## 13. ADR triggers

ควรสร้าง Architecture Decision Record เมื่อ:

- เปลี่ยน public compatibility surface
- เปลี่ยน scheduler fairness algorithm materially
- เพิ่ม durable inference-job semantics
- เพิ่ม remote/network deployment model
- เปลี่ยน persistence engine
- เพิ่ม provider family ที่ทำให้ normalized contractเปลี่ยน
- เพิ่ม prompt transformation layer
- รวม/split process boundaries

## 14. Change classification

### Internal compatible

refactor ไม่มี semantic/public contract change

### Operational compatible

config/metrics enhancement ที่ default behaviorเดิม

### API minor compatible

optional additive behavior ที่ old clients ไม่เสีย

### Semantic breaking

เปลี่ยน field meaning, alias guarantee, retry behavior หรือ security default

breaking change ต้อง migration/compatibility plan

## 15. Release review checklist

- exact HEAD frozen
- working tree/package provenance clean
- supported API matrix current
- unit/contract/integration/conformance green
- chaos critical scenarios green
- security negative tests green
- benchmark regression acceptable
- Ollama/provider qualification current
- default loopback verified
- retention default verified
- known limitations documented
- migration/fresh install tested

## 16. Post-release learning

incident/bug ต้องถาม:

1. contract ไม่ชัดหรือ implementation ผิด?
2. test layer ไหนควรจับได้?
3. monitoring มองเห็นอาการก่อน failure หรือไม่?
4. invariant/checklist ต้องเพิ่มหรือไม่?
5. bug provider-specific หรือ core semantic?

ทุก reliability bug สำคัญควรกลายเป็น regression/chaos case

## 17. Governance principle

> **Cortex should become more capable by adding qualified providers, policies, and capacity—not by absorbing the semantic responsibilities of every client.**

หาก feature ต้องรู้ว่าผู้ใช้ “กำลังทำงานอะไร” เพื่อทำหน้าที่หลักของมัน มีโอกาสสูงว่า feature นั้นควรอยู่ X-Agent/client layer มากกว่า Cortex
