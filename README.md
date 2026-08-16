# 💻 Backend Code MVP — กลุ่ม 1 (เอกสาร)

**โฟกัสเฉพาะฝั่ง API** — ปิดรูรั่วข้อมูลและ Export ที่เจอจาก pentest จริง บนระบบ
`Line Siam U Student Registration API` (Express.js)

> Repository นี้เป็น **ชุดเอกสาร** ของ Backend Code MVP กลุ่ม 1
> (โค้ดโปรเจกต์หลักอยู่ในโปรเจกต์แยกต่างหาก)

## 📄 เอกสารใน repo

| ไฟล์ | เนื้อหา |
|---|---|
| [`project.md`](./project.md) | **เอกสารหลัก** — โค้ด 5 ฟีเจอร์อุดช่องโหว่ + ผลลัพธ์ก่อน/หลัง |
| [`requirements-specification.md`](./requirements-specification.md) | ความต้องการของระบบ (FR / NFR / ลำดับความสำคัญ) |
| [`acceptance-criteria.md`](./acceptance-criteria.md) | เกณฑ์ยอมรับแบบ Given/When/Then + ผลทดสอบจริง |
| [`database-design.md`](./database-design.md) | การออกแบบฐานข้อมูล (ERD, ตารางใหม่, index, query) |

## 🎯 5 ฟีเจอร์

| # | ฟีเจอร์ | ช่องโหว่ที่อุด | ความรุนแรง |
|---|---|---|---|
| 1 | **JWT Authentication & Role Middleware** | ข้อมูลรั่ว + Export โดยไม่ต้องล็อคอิน | 🔴 Critical |
| 2 | **CORS Configuration** | Origin เปิดกว้าง `*` | 🔴 Critical |
| 3 | **Global Error Handler** | SQL Error รั่วไหล | 🟠 High |
| 4 | **OTP Rate Limiter** | Brute-force & SMS Bombing | 🟡 Medium |
| 5 | **Swagger Docs Protection** | API Spec เปิดสาธารณะ | 🟡 Medium |

---

*Backend Code MVP — กลุ่ม 1 · อุดช่องโหว่ฝั่ง API จาก pentest จริง*
