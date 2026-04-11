---
date: 2026-04-11
type: resource
tags:
  - permanent
  - software-engineering
  - migration
  - system-design
  - architecture
source: บทความ / การอ่าน
---

# Legacy Migration — เปลี่ยนหัวใจขณะคนไข้ยังวิ่ง

> การยกเครื่องระบบโดยที่ Business Logic ยังเหมือนเดิม ท้าทายกว่าการเริ่มจากศูนย์ เพราะมี "ความคาดหวังเดิม" เป็นตัวตั้ง

---

## Core Philosophy

**อย่า "Big Bang"** (เปลี่ยนทุกอย่างในวันเดียว)
→ ให้ใช้ **Strangler Pattern ค่อยๆ กัดกินระบบเดิมทีละส่วน**

---

## 1. Discovery — รู้ความจริงก่อนทุกอย่าง

ห้ามเชื่อคำบอกเล่า → เชื่อสิ่งที่เกิดขึ้นจริงเท่านั้น

- **Reverse Engineering** — ไล่ดู Code เดิม สรุป Business Rules ทั้งหมด
- **API Inventory** — ทำรายการ Endpoints พร้อม Request/Response Schema
- **Data Mapping** — ตรวจสอบโครงสร้าง Database เดิม vs ระบบใหม่
- **Performance Baseline** — วัด Latency / Throughput เดิมไว้เปรียบเทียบ

---

## 2. Migration Patterns

### A. Strangler Fig Pattern ⭐ (แนะนำ — ปลอดภัยที่สุด)

ค่อยๆ ย้ายทีละส่วน ระบบเดิมยังรันระหว่างนี้:

```
1. วาง Reverse Proxy / Cloudflare Workers ไว้ด้านหน้า
2. Route traffic ส่วนใหญ่ → ระบบเดิม (Go/Nuxt)
3. พัฒนา / ย้าย Feature ทีละชิ้น → ระบบใหม่ (Rust/React)
4. สลับ Traffic เฉพาะ Path นั้น → ระบบใหม่
5. ทำซ้ำจนระบบเดิมไม่มี Traffic เหลือ → ปิดเครื่องเก่า
```

### B. Parallel Run (เหมาะกับ Backend ที่ sensitive เรื่อง Data)

- ส่ง Request ไปทั้งระบบเก่าและใหม่พร้อมกัน (Shadowing)
- เปรียบเทียบผลลัพธ์ว่าตรงกันไหม
- ถ้าตรงกัน 100% ตามระยะที่กำหนด → สลับไปใช้ระบบใหม่ถาวร

---

## 3. Infrastructure Strategy (DigitalOcean → Cloudflare)

เปลี่ยนจาก Server-based → **Serverless / Edge-based**

- **POC ก่อน** — ย้าย Function ที่เล็กที่สุดไปก่อน เพื่อเช็คข้อจำกัดจริง
- **CI/CD Pipeline** — วางตั้งแต่วันแรก เพื่อให้การย้ายโค้ดราบรื่น

---

## 4. Testing Strategy

เมื่อ Business Process เหมือนเดิม → Test Case เดิมต้องผ่านทั้งหมด

| ประเภท | วัตถุประสงค์ |
|---|---|
| **Regression Testing** | รัน Test Suite เดิมกับระบบใหม่ |
| **Integration Testing** | ตรวจสอบการสื่อสารระหว่างระบบเก่าและใหม่ระหว่างช่วงย้าย |
| **E2E Testing** | เน้น Flow สำคัญของผู้ใช้ว่าทำงานได้ปกติ |

---

## 5. Phased Rollout

- **Canary Deployment** — เริ่มที่ 5-10% ของ Traffic ก่อน
- **Observability** — ติดตั้ง Log/Monitoring (Sentry, Cloudflare Analytics) เพื่อดักจับ Error ใน Stack ใหม่

---

## ⚠️ ข้อควรระวังเฉพาะ Stack นี้

**Go → Rust**
Learning curve สูง ระวัง Memory/Concurrency model ที่ต่างกัน ใช้เวลาพัฒนานานกว่าช่วงแรก

**Nuxt → React**
ตรวจสอบ SSR สำหรับ SEO และ Meta tags ว่ายังทำงานได้ดีเหมือนเดิม (Next.js หรืออื่นๆ)

**DigitalOcean → Cloudflare Workers**
เช็ค CPU time limit และ Memory limit หาก Backend Logic หนักเกินไป

---

## Connections
- เกี่ยวข้องกับ: [[BPMN 2.0 - Process-Driven Requirement Engineering]] (ใช้ BPMN แกะ Logic เดิมระหว่าง Discovery Phase)
- เกี่ยวข้องกับ: [[Software Architecture MOC]]
- Concept: Strangler Fig Pattern (Martin Fowler)

---
**Tags:** #permanent #software-engineering #migration #system-design #architecture #cloudflare #rust
