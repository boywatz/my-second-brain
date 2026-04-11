---
tags:
  - guide
  - permanent
---

# Obsidian Community Plugins แนะนำสำหรับ Second Brain

## Essential (ต้องมี)

### 1. Templater
- **ทำอะไร:** ใช้แทน Core Templates plugin — ทรงพลังกว่ามาก
- **ทำไมต้องมี:** รองรับ dynamic dates, prompts, และ JavaScript ใน template
- **ตั้งค่า:** Settings → Templater → Template Folder Location → `Templates`
- **วิธีใช้:** Alt + N สร้างโน้ตจาก template ได้เลย

### 2. Calendar
- **ทำอะไร:** แสดงปฏิทินใน sidebar
- **ทำไมต้องมี:** คลิกวันที่เพื่อสร้าง/เปิด Daily Note ได้ทันที
- **ตั้งค่า:** เปิด Weekly Notes ด้วยถ้าต้องการ review รายสัปดาห์

### 3. Dataview
- **ทำอะไร:** Query โน้ตเหมือน database
- **ทำไมต้องมี:** สร้าง dashboard ที่อัปเดตอัตโนมัติ เช่น แสดงโน้ตที่มี tag `#to-process` ทั้งหมด
- **ตัวอย่างใน Daily Note:**
  ```dataview
  TABLE status, deadline FROM "01_Projects"
  WHERE status = "active"
  SORT deadline ASC
  ```

### 4. Quick Add
- **ทำอะไร:** สร้างโน้ตใหม่จาก template ด้วย hotkey เดียว
- **ทำไมต้องมี:** ลด friction ในการจดโน้ต — ยิ่งง่ายยิ่งจดบ่อย

---

## Recommended (แนะนำอย่างยิ่ง)

### 5. Periodic Notes
- **ทำอะไร:** จัดการ Daily, Weekly, Monthly, Yearly Notes
- **ทำไมต้องมี:** ทำให้ routine review เป็นระบบ
- **ตั้งค่า:** กำหนด folder สำหรับ daily notes เป็น `00_Inbox`

### 6. Excalidraw
- **ทำอะไร:** วาด diagram, mind map, sketch ใน Obsidian
- **ทำไมต้องมี:** บางไอเดียอธิบายด้วยภาพได้ดีกว่าข้อความ

### 7. Kanban
- **ทำอะไร:** สร้าง Kanban board จาก Markdown
- **ทำไมต้องมี:** จัดการ Projects แบบ visual — To Do / Doing / Done

### 8. Tag Wrangler
- **ทำอะไร:** Rename, merge, delete tags ทั้ง vault
- **ทำไมต้องมี:** เมื่อ tags เริ่มเยอะ จะช่วยจัดระเบียบได้ง่าย

### 9. Obsidian Git
- **ทำอะไร:** Auto backup vault ไปยัง Git repository
- **ทำไมต้องมี:** คุณมี Git repo อยู่แล้ว! ตั้งให้ auto commit ทุก 30 นาที
- **ตั้งค่า:**
  - Auto backup interval: 30 minutes
  - Auto pull on startup: ON
  - Commit message format: `vault backup: {{date}}`

---

## Nice to Have (เสริม)

### 10. Natural Language Dates
- **ทำอะไร:** พิมพ์ "tomorrow" หรือ "next friday" แล้วแปลงเป็น date link อัตโนมัติ

### 11. Admonition / Callouts
- **ทำอะไร:** สร้างกล่องข้อความสวยๆ (tip, warning, note)
- **ตัวอย่าง:**
  ```
  > [!tip] เคล็ดลับ
  > ใช้ callouts เพื่อเน้นจุดสำคัญในโน้ต
  ```

### 12. Omnisearch
- **ทำอะไร:** Search ที่ฉลาดกว่า built-in — รองรับ fuzzy search
- **ทำไมต้องมี:** เมื่อโน้ตเยอะ การค้นหาที่ดีคือสิ่งจำเป็น

---

## วิธีติดตั้ง Plugin
1. เปิด Settings → Community Plugins
2. ปิด Restricted Mode (ถ้ายังเปิดอยู่)
3. กด Browse แล้วค้นหาชื่อ plugin
4. กด Install → Enable

## ลำดับที่แนะนำให้ติดตั้ง
1. **Templater** (ใช้กับ Templates ที่สร้างไว้)
2. **Calendar** (เริ่มทำ Daily Notes)
3. **Obsidian Git** (auto backup)
4. **Dataview** (dashboard)
5. ที่เหลือค่อยๆ เพิ่มเมื่อรู้สึกว่าต้องการ

---
> *อย่าติดตั้งทุกอันพร้อมกัน — เริ่มจาก 3-4 ตัวแรก แล้วค่อยเพิ่มเมื่อคุ้นเคย*
