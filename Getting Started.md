---
tags:
  - guide
  - permanent
---

# Getting Started — วิธีใช้ Second Brain ของคุณ

## โครงสร้าง Vault นี้

```
my-second-brain/
├── 00_Inbox/          ← จุดรับไอเดียใหม่ทุกอย่าง
├── 01_Projects/       ← สิ่งที่กำลังทำ มี deadline
├── 02_Areas/          ← สิ่งที่ดูแลต่อเนื่อง (ชีวิต, งาน, การเงิน)
├── 03_Resources/      ← คลังความรู้ที่สนใจ
├── 04_Archive/        ← สิ่งที่จบแล้ว
├── 05_Assets/         ← รูปภาพ, ไฟล์แนบ
├── Templates/         ← แม่แบบโน้ตต่างๆ
└── Home.md            ← Dashboard หลัก
```

## หลักการ PARA

| Folder | คืออะไร | ตัวอย่าง |
|--------|---------|----------|
| **Projects** | มี deadline, มีเป้าหมาย | จัดพอร์ตลงทุน 2026, เรียนคอร์ส AI |
| **Areas** | ดูแลต่อเนื่อง ไม่มีวันจบ | การเงิน, สุขภาพ, ความสัมพันธ์ |
| **Resources** | สนใจ แต่ยังไม่ active | สูตรอาหาร, โน้ตหนังสือ |
| **Archives** | จบแล้ว ไม่ active | โปรเจกต์เก่า, โน้ตที่ไม่ใช้แล้ว |

## Templates ที่มีให้

| Template | ใช้ตอนไหน |
|----------|----------|
| **Daily Note** | จดทุกวัน — ไอเดีย, tasks, journal |
| **Fleeting Note** | จดไอเดียสั้นๆ ที่ผุดขึ้นมา |
| **Meeting Note** | บันทึกการประชุม |
| **Book Note** | สรุปหนังสือ |
| **Project Note** | เริ่มโปรเจกต์ใหม่ |
| **MOC Template** | สร้าง Map of Content สำหรับหัวข้อที่มีโน้ตเยอะ |

### วิธีใช้ Template
1. กด Ctrl/Cmd + N สร้างโน้ตใหม่
2. เปิด Command Palette (Ctrl/Cmd + P)
3. พิมพ์ "Template" แล้วเลือก "Insert template"
4. เลือก template ที่ต้องการ

## Workflow แนะนำ

### Daily (ทุกวัน)
- สร้าง Daily Note ตอนเช้า
- จดทุกอย่างที่นึกออก ไม่ต้องกรอง
- ตรวจ Inbox ย้ายโน้ตไปที่ที่เหมาะสม

### Weekly (ทุกสัปดาห์)
- Review โน้ตที่มี tag `#to-process`
- ย้ายโปรเจกต์ที่จบแล้วเข้า Archive
- สร้าง links ระหว่างโน้ตที่เกี่ยวข้อง

### Monthly (ทุกเดือน)
- Review Areas — มีอะไรที่ควรเป็น Project ไหม?
- สร้าง MOC ใหม่สำหรับหัวข้อที่มีโน้ตเยอะ
- ดู Graph View เพื่อหา patterns

## Tags ที่แนะนำ

| Tag | ความหมาย |
|-----|----------|
| `#to-process` | ยังไม่ได้สรุป/จัดระเบียบ |
| `#permanent` | สรุปเสร็จแล้ว เป็นความรู้ถาวร |
| `#fleeting` | ไอเดียชั่วขณะ รอ process |
| `#moc` | Map of Content |
| `#active` | กำลัง active อยู่ |
| `#review` | ต้องกลับมาทบทวน |

## Keyboard Shortcuts ที่ควรรู้

| Shortcut | ทำอะไร |
|----------|--------|
| `Ctrl/Cmd + O` | Quick Open — เปิดโน้ตด้วยชื่อ |
| `Ctrl/Cmd + N` | สร้างโน้ตใหม่ |
| `Ctrl/Cmd + P` | Command Palette |
| `Ctrl/Cmd + E` | Toggle Edit/Preview |
| `[[` | สร้าง Link ไปหาโน้ตอื่น |
| `#` | ใส่ Tag |
| `Ctrl/Cmd + Shift + F` | Search ทั้ง Vault |

---
> *"สมองที่สองที่ดีที่สุดคือสมองที่คุณใช้งานจริงทุกวัน"*
