# 📋 Requirements Specification — Backend Code MVP (กลุ่ม 1)

**ระบบ:** Line Siam U Student Registration API (ฝั่ง Backend)
**ขอบเขต:** โฟกัสเฉพาะฝั่ง API — อุดช่องโหว่ความปลอดภัย 5 รายการ
**เวอร์ชัน:** 1.0
**วันที่:** 16 สิงหาคม 2569

---

## 1. บทนำ

### 1.1 วัตถุประสงค์
เอกสารนี้ระบุความต้องการของระบบ (Requirements) สำหรับการปรับปรุงความปลอดภัยฝั่ง API
ของระบบลงทะเบียนนักศึกษา โดยอ้างอิงช่องโหว่จริงที่พบจากการทดสอบความปลอดภัย (Pentest)

### 1.2 ภาพรวมช่องโหว่ที่ต้องแก้

| # | ช่องโหว่ | ความรุนแรง | ฟีเจอร์ที่แก้ |
|---|---|---|---|
| 1 | ข้อมูลรั่ว + Export โดยไม่ต้องล็อคอิน | 🔴 Critical | JWT Auth & Role Middleware |
| 2 | Origin เปิดกว้าง `Access-Control-Allow-Origin: *` | 🔴 Critical | CORS Configuration |
| 3 | SQL Error รั่วไหล (error disclosure) | 🟠 High | Global Error Handler |
| 4 | OTP brute-force + SMS Bombing | 🟡 Medium | OTP Rate Limiter |
| 5 | API Spec เปิดสาธารณะ | 🟡 Medium | Swagger Docs Protection |

---

## 2. ความต้องการเชิงฟังก์ชัน (Functional Requirements)

### FR-1: JWT Authentication & Role Middleware

| ID | ความต้องการ | เกณฑ์ยอมรับ |
|---|---|---|
| FR-1.1 | ระบบต้องตรวจสอบ JWT token ก่อนเข้าถึง API ที่ต้องการสิทธิ์ | ไม่มี token → `401` |
| FR-1.2 | ระบบต้องปฏิเสธ token ที่ไม่ถูกต้อง/หมดอายุ | token ปลอม → `401` |
| FR-1.3 | ระบบต้องบังคับ role ตาม endpoint (`student`, `employee`, `admin`) | role ไม่ตรง → `403` |
| FR-1.4 | ระบบต้องกันนักศึกษาดูข้อมูลคนอื่น (Ownership / กัน IDOR) | ดูคนอื่น → `403` |
| FR-1.5 | ระบบต้องบังคับ role `employee`/`admin` สำหรับ endpoint export ข้อมูลทั้งหมด | นักศึกษา → `403` |
| FR-1.6 | ระบบต้องบังคับ role `employee`/`admin` สำหรับ endpoint ข้อมูลอ่อนไหว (`/api/young`) | นักศึกษา/ไม่ล็อคอิน → `401`/`403` |
| FR-1.7 | ระบบต้องรองรับการออก token ผ่าน login (จำลอง LINE OAuth) | login สำเร็จ → ได้ token |

### FR-2: CORS Configuration

| ID | ความต้องการ | เกณฑ์ยอมรับ |
|---|---|---|
| FR-2.1 | ระบบต้องอนุญาตเฉพาะ origin ใน allowlist | origin ใน list → ได้ CORS header |
| FR-2.2 | ระบบต้อง**ไม่**คืน `Access-Control-Allow-Origin: *` | origin นอก list → ไม่มี CORS header |
| FR-2.3 | ระบบต้องส่ง header `Vary: Origin` เพื่อให้ cache แยกตาม origin | ทุก response ที่มี origin → มี `Vary: Origin` |
| FR-2.4 | ระบบต้องรองรับ preflight request (`OPTIONS`) | `OPTIONS` → `204` |

### FR-3: Global Error Handler

| ID | ความต้องการ | เกณฑ์ยอมรับ |
|---|---|---|
| FR-3.1 | error ทั้งหมดต้องผ่าน middleware เดียว (catch-all ตัวสุดท้าย) | ทุก route ครอบคลุม |
| FR-3.2 | response ต้อง**ไม่**เปิดเผยข้อความ error ภายใน (SQL, stack trace, path) | 500 → `Internal server error` |
| FR-3.3 | SQL error (MySQL code) ต้องถูกแปลงเป็นข้อความทั่วไป | `ER_BAD_FIELD_ERROR` → `500` ทั่วไป |
| FR-3.4 | ระบบต้อง log รายละเอียด error ไว้ฝั่ง server เท่านั้น | log มี path/message/stack |
| FR-3.5 | Validation error ต้องคืน `400` พร้อมข้อความภาษาไทยที่เข้าใจได้ | → `400 ข้อมูลไม่ถูกต้อง` |

### FR-4: OTP Rate Limiter

| ID | ความต้องการ | เกณฑ์ยอมรับ |
|---|---|---|
| FR-4.1 | จำกัดการ verify OTP: 5 ครั้ง / 10 นาที ต่อเบอร์โทร | ครั้งที่ 6 → `429` |
| FR-4.2 | จำกัดการส่ง SMS: 3 ครั้ง / ชั่วโมง ต่อเบอร์โทร | ครั้งที่ 4 → `429` |
| FR-4.3 | ระบบต้องส่ง `RateLimit-*` headers มาตรฐาน | response มี header |
| FR-4.4 | key ของ rate limit ต้องอิงเบอร์โทร (fallback เป็น IP) | ต่อเบอร์, ไม่ใช่ต่อเครื่อง |

### FR-5: Swagger Docs Protection

| ID | ความต้องการ | เกณฑ์ยอมรับ |
|---|---|---|
| FR-5.1 | ใน environment dev เปิด `/api-docs/` ให้ดูได้ | dev → `200` |
| FR-5.2 | ใน environment production ต้องล็อคอินก่อนดู spec | ไม่ auth → `401` |
| FR-5.3 | ใช้ได้ทั้ง basic auth และ JWT role (`employee`/`admin`) | ตาม config |

---

## 3. ความต้องการเชิงไม่ฟังก์ชัน (Non-Functional Requirements)

| หมวด | ID | ความต้องการ |
|---|---|---|
| **ความปลอดภัย** | NFR-1 | secret ต้องเก็บใน environment variables ไม่ hardcode (`JWT_SECRET` ≥ 32 ตัวอักษร) |
| **ความปลอดภัย** | NFR-2 | response ไม่เปิดเผยข้อมูลภายใน (internal detail) ในทุกสถานะ error |
| **ความปลอดภัย** | NFR-3 | รองรับ HTTPS + ไม่ส่งข้อมูลอ่อนไหวผ่าน URL query |
| **ประสิทธิภาพ** | NFR-4 | middleware ทั้ง 5 ตัวต้องเพิ่ม latency < 10ms ต่อ request |
| **ความพร้อมใช้งาน** | NFR-5 | rate limiter ต้องไม่ทำให้ผู้ใช้ปกติใช้งานไม่ได้ (window ใหญ่พอ) |
| **การบำรุงรักษา** | NFR-6 | โค้ดแยกเป็นไฟล์ middleware เดี่ยว ๆ ในโฟลเดอร์ `middleware/` |
| **ความเข้ากันได้** | NFR-7 | รองรับ Node.js ≥ 18 และ Express 4.x |
| **การตรวจสอบย้อนกลับ** | NFR-8 | ทุก export ข้อมูลต้องมี audit log (ผู้ใช้, เวลา, จำนวนแถว) |

---

## 4. ข้อจำกัดและสมมติฐาน

### ข้อจำกัด
- ระบบปัจจุบันเป็น **Node.js + Express** (ไม่เปลี่ยน stack)
- ไม่มีการแก้ไขโครงสร้างฐานข้อมูลเดิมในเฟสนี้ (เพิ่มตารางใหม่ได้)
- การใช้งานจริงต้องเชื่อม LINE OAuth — เฟสนี้ใช้ login จำลอง

### สมมติฐาน
- มี `JWT_SECRET` ที่ปลอดภัยใน environment
- ผู้ดูแลระบบมีบทบาท `employee` / `admin` แยกจากนักศึกษา
- Rate limit state เก็บใน memory ได้สำหรับเฟส MVP (production แนะนำ Redis)

---

## 5. ลำดับความสำคัญ (Priority)

| ลำดับ | ฟีเจอร์ | เหตุผล |
|---|---|---|
| P0 | JWT Auth & Role | ปิดรูรั่วข้อมูลร้ายแรงที่สุด (ข้อมูล 16k คน) |
| P0 | CORS Configuration | ป้องกันเว็บมุ่งร้ายอ่านข้อมูลผ่าน browser |
| P1 | Global Error Handler | ลดข้อมูลที่ใช้โจมตีต่อ (SQL structure) |
| P1 | Swagger Docs Protection | ลด attack surface |
| P2 | OTP Rate Limiter | ป้องกันการโจมตีแบบ brute-force |

---

*Requirements Specification — Backend Code MVP กลุ่ม 1*
