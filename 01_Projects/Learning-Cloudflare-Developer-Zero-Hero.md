---
created: 2026-04-11
tags: [cloudflare, learning, project, zero-to-hero]
status: in-progress
---

# Learning Cloudflare Developer Zero-Hero

> เปลี่ยนจาก VPC/Container mindset มาเป็น Global Edge Network

## Learning Path Overview

```
Level 1: Foundations (Edge Infrastructure)
Level 2: Intermediate (Serverless & Data at the Edge)  
Level 3: Advanced (Production-Grade & Scalability)
```

---

## 🟢 Level 1: Foundations (Edge Infrastructure)

### Topics

- [x] **การเปลี่ยน Mindset** จาก "Private Network (VPC)" มาเป็น "Global Network"
- [x] **Cloudflare Global Network Architecture** — ทำความเข้าใจว่า Cloudflare ไม่ได้ทำงานเป็น Region เหมือน Cloud ทั่วไป แต่รันอยู่บน 330+ เมืองทั่วโลก (Anycast Network)
- [x] **Edge DNS & Proxy Logic** — การจัดการ DNS records และการทำงานของ Proxy (CDN/WAF)
- [x] **SSL/TLS Termination** — การทำ End-to-End Encryption ในระดับโปรดักชัน (Full Strict)
- [ ] **Cloudflare Tunnel (Zero Trust Connectivity)** — การเชื่อมต่อ Docker Containers หรือ On-premise Server เข้ากับ Cloudflare โดยไม่ต้องเปิด Public IP (Inbound ports)
- [ ] **การสร้าง "Virtual Network"** ผ่าน Tunnel เพื่อทดแทน VPC เดิม
- [ ] **Static Assets Deployment** — การใช้ Cloudflare Pages สำหรับ Frontend ที่เป็น React/Vue/Next.js โดยไม่ต้องรัน Docker Container

---

## 🟡 Level 2: Intermediate (Serverless & Data at the Edge)

### Topics

- [ ] **Cloudflare Workers Fundamentals**
  - [ ] Isolate Model (V8 Engine) ต่างจาก Container/VM อย่างไร (Cold start 0ms)
  - [ ] การเขียนโปรแกรมด้วย Wrangler CLI
  - [ ] การใช้ Environments (Dev/Staging/Prod)
- [ ] **Stateful Storage at the Edge**
  - [ ] **KV (Key-Value)** — สำหรับ Global Configuration หรือ User Session
  - [ ] **R2 Storage** — การทำ Image/Asset storage แบบไม่มีค่า Egress (Compatible กับ S3 API)
  - [ ] **D1 (SQL Database)** — การใช้ Relational Database ที่รันอยู่บนขอบ Network
- [ ] **Request & Response Manipulation** — การทำ Middleware เช่น Redirects, Header Injection, และ Geo-location based content

---

## 🔴 Level 3: Advanced (Production-Grade & Scalability)

### Topics

- [ ] **Scalable Architecture**
  - [ ] **Durable Objects** — หัวใจสำคัญของ Production App ที่ต้องการ "Consistency" (เช่น Inventory management, Real-time collaboration)
  - [ ] **Queues** — การจัดการ Async Task และ Retries เพื่อลดภาระของ Worker หลัก
  - [ ] **Hyperdrive** — การเชื่อมต่อ Worker เข้ากับฐานข้อมูลเดิม (เช่น Postgres/MySQL) ให้เร็วเหมือนรันอยู่ข้างๆ กัน
- [ ] **Enterprise Security & Compliance**
  - [ ] **Advanced WAF & Bot Management** — การตั้งค่า Custom Rules และป้องกันการทำ Scraping
  - [ ] **Zero Trust Access** — การใช้ Identity Providers (Google/Okta) ควบคุมการเข้าถึง Dashboard หรือ Internal API
- [ ] **Observability & CI/CD**
  - [ ] **Logpush** — การส่ง Log ไปยัง External Analytics (เช่น Datadog/S3)
  - [ ] **Terraform/Pulumi integration** — การทำ GitOps เพื่อคุม Infrastructure ทั้งหมดผ่าน Code
  - [ ] **A/B Testing** — การใช้ Cloudflare Workers เพื่อทำ Canary Deployment หรือ Blue-Green Deployment ในระดับ Edge

---

## Resources

- Cloudflare Docs: https://developers.cloudflare.com
- Wrangler CLI: https://developers.cloudflare.com/workers/wrangler
- Cloudflare Dashboard: https://dash.cloudflare.com

---

## Progress

```
🟢 Level 1: 4 / 7
🟡 Level 2: __ / 6  
🔴 Level 3: __ / 8
Total: 4 / 21 topics
```