---
created: 2026-04-15
tags: [cloudflare, edge-dns, ssl-tls, proxy, level-1]
status: completed
---

# Edge DNS & Proxy & SSL/TLS

## 1. Edge DNS & Proxy Logic

### Edge DNS
คือการตอบคำถามเรื่องที่อยู่เว็บไซต์จาก Server ที่ใกล้ผู้ใช้ที่สุด เพื่อความรวดเร็ว

### Proxy (CDN/WAF)
เปรียบเสมือน "ด่านหน้า"

| Component | หน้าที่ |
|-----------|---------|
| **CDN** | เก็บสำเนาไฟล์ (Cache) ไว้ใกล้ผู้ใช้ ลดภาระ Server หลัก |
| **WAF** | รปภ. คัดกรอง Request อันตรายทิ้งตั้งแต่ต้นทาง |

---

## 2. SSL/TLS & HTTPS

### SSL Termination
คือการแกะรหัสข้อมูลที่ Edge เพื่อตรวจสอบ ก่อนจะส่งต่อเข้าสู่ระบบภายใน

### Full (Strict) Mode
มาตรฐานสูงสุด — ข้อมูลจะถูกล็อคกุญแจ (Encrypt) ตั้งแต่ User → Cloudflare → Origin Server โดยมีการตรวจสอบใบเซอร์ (Certificate) อย่างเข้มงวดตลอดสาย

**ความง่าย:** Cloudflare ช่วยให้การทำ HTTPS เป็นเรื่องง่ายเพียงแค่คลิกเปิดโหมดที่ต้องการ และมีใบเซอร์ฟรีให้โหลดไปติดตั้งที่ Server

---

## 3. การเชื่อมโยงความรู้เดิม (Nginx/Kong vs. Cloudflare)

| Layer | ตัวอย่าง | ระดับ |
|-------|----------|-------|
| **Cloudflare (Edge Proxy)** | จัดการ Traffic ปริมาณมหาศาลจากภายนอก | Global Level |
| **Nginx/Kong (Internal Proxy/Gateway)** | จัดการ Business Logic ภายใน | Data Center Level |

### Shift Left Concept
ย้ายภาระบางอย่าง (Cache, Security, Rate Limit) จากที่เคยรันบนเครื่องตัวเอง ขึ้นไปไว้บน Cloudflare เพื่อประหยัดทรัพยากรของ Server เราเอง

---

## 4. การ Migration (DigitalOcean + Google Workspace)

### ย้ายได้ 100%
โดยการเปลี่ยน Nameservers จาก DigitalOcean มาเป็น Cloudflare

### ข้อควรระวังเรื่องเมฆ (Cloud Color)

| Mode | ใช้กับ | เหตุผล |
|------|--------|--------|
| 🟠 **เมฆสีส้ม (Proxied)** | Web/API (A Record) | เอา Feature ความปลอดภัยและ SSL |
| ⚪ **เมฆสีเทา (DNS Only)** | Email (MX Record) | ให้อีเมล Google ทำงานได้ปกติไม่ดับ |

---

## Summary

```
User Request
     ↓
[Edge DNS] → ชี้ไปที่ Edge ที่ใกล้ที่สุด
     ↓
[Proxy Layer] → CDN (Cache) + WAF (Security)
     ↓
[SSL/TLS] → Decrypt ที่ Edge (Full Strict = Encrypt ตลอดสาย)
     ↓
[Origin Server] → ได้ Request ที่ปลอดภัยแล้ว
```