# Bug Audit — Proclean Dashboard (2026-05-04)

## Scope
- Static code audit from `index.html`.
- Runtime browser audit automation attempted but blocked by environment (Playwright browser download 403).
- Live site endpoint checked at `https://procleandashboard.vercel.app`.

## Findings (mapped to requested 10 checks)

| # | ประเด็น | สถานะ | อาการ | สาเหตุ | ไฟล์ที่ต้องแก้ | วิธีแก้ |
|---|---|---|---|---|---|---|
| 1 | Login ต้องไม่แสดง Dashboard ก่อนเข้าสู่ระบบ | ✅ ผ่าน | หน้า login overlay แสดงก่อน และ `#app` ถูกซ่อนด้วย opacity 0 จนกว่าจะ login สำเร็จ | logic ใน `doLogin()` เพิ่ม class `show` หลัง auth สำเร็จเท่านั้น | `index.html` | ไม่ต้องแก้ |
| 2 | Role permission | ⚠️ พบจุดเสี่ยง | Read-only ถูกซ่อนเมนูหลักแล้ว แต่ยังสามารถแก้สิทธิ์ราย action ได้ถ้ามีการให้ `perms` ผิดพลาดจาก DB | ระบบยึด `CU.perms` เป็นหลักตอนซ่อน nav (ไม่ normalize ตาม role ทุกครั้ง) | `index.html` | ตอน login/โหลด user ให้ normalize permissions ตาม role baseline แล้วค่อย merge เพิ่มเติมเฉพาะ admin policy ที่อนุญาต |
| 3 | Preview Excel ต้องอ่าน .xlsx/.xls ได้จริง | ✅ ผ่าน (เชิงโค้ด) | มี `accept` รองรับ `.xlsx/.xls` และใช้ `XLSX.read` + `sheet_to_json` | ใช้ FileReader + xlsx library ถูกต้อง | `index.html` | ไม่ต้องแก้ |
| 4 | นำเข้าข้อมูลแล้ว Dashboard ต้องอัปเดต | ✅ ผ่าน | หลัง import มี `refresh()` และมีการอัปเดต `DAILY_EXTRA`, `DATA`, `DAILY_CARS` | import flow ครบ | `index.html` | ไม่ต้องแก้ |
| 5 | ตัวกรองเดือน/ปี/ช่วงวันที่ไม่ทำให้ข้อมูลหายผิดปกติ | ❌ บั๊กจริง | ถ้าเลือกปีอื่นที่ไม่ใช่ 2025/2026 ข้อมูล fallback ไปปี 2026 ทำให้กราฟ/ตัวเลขผิด | หลายจุด hardcode ปี 2025/2026 (`getRev`, import restore, chart compare) | `index.html` | refactor data model เป็น dynamic year key (เช่น `DATA[id].revByYear[yyyy]`) และไม่ fallback เป็น 2026 |
| 6 | คาดการณ์ยอดสิ้นเดือนต้องไม่ค้าง “กำลังคำนวณ...” | ✅ ผ่าน (เชิงโค้ด) | `renderForecast()` เซ็ตเนื้อหาใหม่ทุก `refresh()` ไม่มี async lock | ไม่มี pending state ที่ค้างถาวรในฟังก์ชัน forecast | `index.html` | ไม่ต้องแก้ |
| 7 | เป้าสาขาต้องดึงไปแสดงหน้า Dashboard อัตโนมัติ | ✅ ผ่าน | `renderTarget()` sum จาก `branchTargets`; `loadBranchData()` + `loadTargetForPeriod()` ดึงค่าจาก Firestore และ refresh | flow target auto-pull ครบ | `index.html` | ไม่ต้องแก้ |
| 8 | ประวัติอัปโหลดต้องถูกบันทึกและแสดงย้อนหลัง | ⚠️ พบข้อจำกัด | ดึง log แค่ `limit(500)` เริ่มต้น ทำให้ย้อนหลังเกิน 500 รายการไม่แสดง | query จำกัดเอกสาร 500 รายการ | `index.html` | เพิ่ม pagination/infinite scroll หรือ date range query และบอกผู้ใช้เมื่อถูก truncate |
| 9 | ตรวจ console error/network error | ⚠️ ตรวจได้บางส่วน | พยายามทำ browser automation แล้วติดตั้ง Chromium ไม่ได้ (403 Domain forbidden) | สภาพแวดล้อมบล็อก download browser binary | N/A | รัน E2E จากเครื่องที่ติดตั้ง browser ได้ หรือใช้ GitHub Actions ที่ allow Playwright browsers |
| 10 | สรุปบั๊ก อาการ/สาเหตุ/ไฟล์/วิธีแก้ | ✅ ทำแล้ว | ตารางนี้คือสรุปบั๊ก | - | `BUG_AUDIT.md` | - |

## High-priority bugs to fix first
1. **Year handling hardcode 2025/2026** — ทำให้ filter ปี/เดือนให้ผลผิดทันทีเมื่อ data ปีใหม่เพิ่มเข้ามา.
2. **Upload history capped at 500 docs** — ทำให้ผู้ใช้เข้าใจว่าข้อมูลเก่าหาย ทั้งที่จริงยังอยู่ DB.
3. **Permission normalization** — ลดความเสี่ยง role drift จากข้อมูลผู้ใช้ใน DB.
