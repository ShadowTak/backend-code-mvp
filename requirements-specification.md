# 📋 Requirements Specification — Backend Code MVP (กลุ่ม 1)

**ระบบ:** Line Siam U Student Registration API (ฝั่ง Backend)
**ขอบเขต:** โฟกัสเฉพาะฝั่ง API — อุดช่องโหว่ความปลอดภัย 5 รายการ
**อ้างอิง API จริง:** 143 endpoints — `GET /api/students`, `GET /api/young`, `GET /api/export/users`, `POST /api/otp/verify` ฯลฯ
**เวอร์ชัน:** 2.0 (ปรับให้ตรงกับ API จริง)
**วันที่:** 16 สิงหาคม 2569

---

## 1. บทนำ

### 1.1 วัตถุประสงค์
เอกสารนี้ระบุความต้องการของระบบ (Requirements) สำหรับการปรับปรุงความปลอดภัยฝั่ง API
ของระบบลงทะเบียนนักศึกษา (`siam-u-line-welcome-production.up.railway.app`) โดยอ้างอิง
ช่องโหว่จริงที่พบจากการทดสอบความปลอดภัย (Pentest)

### 1.2 ภาพรวมช่องโหว่ที่ต้องแก้ (จาก pentest จริง)

| # | ช่องโหว่ | หลักฐานจริง | ความรุนแรง | ฟีเจอร์ที่แก้ |
|---|---|---|---|---|
| 1 | ข้อมูลรั่ว + Export โดยไม่ต้องล็อคอิน | `GET /api/young` คืนชื่อ/อีเมล/เบอร์/LINE ID · `GET /api/export/users` คืน Excel 1.3MB (16,191 คน) | 🔴 Critical | JWT Auth & Role Middleware |
| 2 | Origin เปิดกว้าง | `Access-Control-Allow-Origin: *` | 🔴 Critical | CORS Configuration |
| 3 | SQL Error รั่วไหล | `GET /api/game-scores/stats` คืน `Unknown column 'student_code' in 'field list'` | 🟠 High | Global Error Handler |
| 4 | OTP brute-force + SMS Bombing | `POST /api/otp/verify`, `/api/otp/send`, `/api/sms/*` ยิงซ้ำได้ไม่จำกัด | 🟡 Medium | OTP Rate Limiter |
| 5 | API Spec เปิดสาธารณะ | `/api-docs/` แสดง spec 143 endpoints ฟรี | 🟡 Medium | Swagger Docs Protection |

---

## 2. ความต้องการเชิงฟังก์ชัน (Functional Requirements)

### FR-1: JWT Authentication & Role Middleware

ระบบเดิมมี 3 กลุ่มผู้ใช้ (ดูจาก schema จริง): **Student** (`/api/students`), **Employee** (`/api/employees` — มี field `role`), **Young** (`/api/young`)

| ID | ความต้องการ | เกณฑ์ยอมรับ |
|---|---|---|
| FR-1.1 | ระบบต้องตรวจสอบ JWT token ก่อนเข้าถึง API ที่ต้องการสิทธิ์ | ไม่มี token → `401` |
| FR-1.2 | ระบบต้องปฏิเสธ token ที่ไม่ถูกต้อง/หมดอายุ | token ปลอม → `401` |
| FR-1.3 | ระบบต้องบังคับ role ตาม endpoint (`student` / `employee` / `admin` — อ่านจาก `employees.role`) | role ไม่ตรง → `403` |
| FR-1.4 | ระบบต้องกันนักศึกษาดูข้อมูลคนอื่น (Ownership / กัน IDOR) — เช่น `GET /api/students/{studentId}`, `GET /api/students/{studentId}/id-card`, `GET /api/students/{studentId}/with-english`, `GET /api/game-scores/student/{student_code}` | ดูคนอื่น → `403` |
| FR-1.5 | ระบบต้องบังคับ role `employee`/`admin` สำหรับ export — `GET /api/export/users`, `GET /api/export/game-scores` | นักศึกษา → `403` |
| FR-1.6 | ระบบต้องบังคับ role `employee`/`admin` สำหรับข้อมูลอ่อนไหว — `GET /api/young`, `GET /api/young/card`, `GET /api/sms/transactions` | ไม่ล็อคอิน → `401` / role ไม่ตรง → `403` |
| FR-1.7 | ระบบต้องออก token หลังยืนยันตัวตนผ่าน LINE (flow จริง: `GET /api/line/profile/{userId}` → verify OTP → ออก JWT) | login สำเร็จ → ได้ token |

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
| FR-3.2 | response ต้อง**ไม่**เปิดเผยข้อความ error ภายใน (SQL, stack trace, path) — schema จริง `Error` มี field `details` ที่ต้องไม่รั่ว | 500 → `{"success":false,"message":"Internal server error"}` |
| FR-3.3 | SQL error (MySQL code) ต้องถูกแปลงเป็นข้อความทั่วไป — เช่น `ER_BAD_FIELD_ERROR` จาก `/api/game-scores/stats` | `500` ทั่วไป ไม่มีคำว่า SQL/column |
| FR-3.4 | ระบบต้อง log รายละเอียด error ไว้ฝั่ง server เท่านั้น | log มี path/message/stack |
| FR-3.5 | Validation error ต้องคืน `400` พร้อมข้อความที่เข้าใจได้ | → `400` |

### FR-4: OTP Rate Limiter

| ID | ความต้องการ | เกณฑ์ยอมรับ |
|---|---|---|
| FR-4.1 | จำกัดการ verify OTP — `POST /api/otp/verify`, `/api/students/verify-otp`, `/api/employees/verify-otp`, `/api/young/verify-otp`: 5 ครั้ง / 10 นาที ต่อเบอร์ | ครั้งที่ 6 → `429` |
| FR-4.2 | จำกัดการส่ง OTP/SMS — `POST /api/otp/send`, `/api/otp/resend`, `/api/students/resend-otp`: 3 ครั้ง / ชั่วโมง ต่อเบอร์ | ครั้งที่ 4 → `429` |
| FR-4.3 | ระบบต้องส่ง `RateLimit-*` headers มาตรฐาน | response มี header |
| FR-4.4 | key ของ rate limit ต้องอิง `phone_number` (fallback เป็น IP) | ต่อเบอร์, ไม่ใช่ต่อเครื่อง |
| FR-4.5 | ตาราง `otp_codes` มี field `is_used`, `expires_at` — ระบบต้องตรวจ `attempts` จาก DB ร่วมด้วย (กันข้าม instance) | OTP ใช้แล้ว/หมดอายุ → ปฏิเสธ |

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
| **ความปลอดภัย** | NFR-2 | response ไม่เปิดเผยข้อมูลภายใน (internal detail) ในทุกสถานะ error — โดยเฉพาะ `Error.details` |
| **ความปลอดภัย** | NFR-3 | ข้อมูลอ่อนไหว (`line_user_id`, `phone_number`, `email`) ต้องผ่าน auth + role ทุกครั้ง |
| **ประสิทธิภาพ** | NFR-4 | middleware ทั้ง 5 ตัวต้องเพิ่ม latency < 10ms ต่อ request |
| **ความพร้อมใช้งาน** | NFR-5 | rate limiter ต้องไม่ทำให้ผู้ใช้ปกติใช้งานไม่ได้ (window ใหญ่พอ) |
| **การบำรุงรักษา** | NFR-6 | โค้ดแยกเป็นไฟล์ middleware เดี่ยว ๆ ในโฟลเดอร์ `middleware/` |
| **ความเข้ากันได้** | NFR-7 | รองรับ Node.js ≥ 18 และ Express 4.x (stack เดิมของระบบ) |
| **การตรวจสอบย้อนกลับ** | NFR-8 | ทุก export ข้อมูลต้องมี audit log (ผู้ใช้, เวลา, จำนวนแถว) |

---

## 4. ข้อจำกัดและสมมติฐาน

### ข้อจำกัด
- ระบบปัจจุบันเป็น **Node.js + Express + MySQL** (ไม่เปลี่ยน stack)
- schema จริงมี 3 กลุ่มผู้ใช้แยกตาราง: `students`, `employees` (มี field `role`), `young`
- ไม่มีการแก้ไขโครงสร้างตารางเดิม — เพิ่มตารางใหม่ได้

### สมมติฐาน
- มี `JWT_SECRET` ที่ปลอดภัยใน environment
- `employees.role` ใช้ระบุสิทธิ์ (employee/admin) ส่วน student/young เป็น role อัตโนมัติจากตาราง
- Rate limit state เก็บใน memory ได้สำหรับเฟส MVP (production แนะนำ Redis)

---

## 5. ลำดับความสำคัญ (Priority)

| ลำดับ | ฟีเจอร์ | เหตุผล |
|---|---|---|
| P0 | JWT Auth & Role | ปิดรูรั่วข้อมูลร้ายแรงที่สุด (ข้อมูล 16k คน จาก `/api/young`, `/api/export/users`) |
| P0 | CORS Configuration | ป้องกันเว็บมุ่งร้ายอ่านข้อมูลผ่าน browser |
| P1 | Global Error Handler | ลดข้อมูลที่ใช้โจมตีต่อ (SQL structure จาก `/api/game-scores/stats`) |
| P1 | Swagger Docs Protection | ลด attack surface (143 endpoints) |
| P2 | OTP Rate Limiter | ป้องกัน brute-force OTP + SMS bombing |

---

## 6. Endpoint จริงที่เกี่ยวข้อง (จาก API docs)

| กลุ่ม | Endpoint | มาตรการ |
|---|---|---|
| ข้อมูลอ่อนไหว | `GET /api/young`, `GET /api/young/card`, `GET /api/young/export.csv` | authenticate + requireRole(employee, admin) |
| Export | `GET /api/export/users`, `GET /api/export/game-scores`, `GET /api/export/users/count` | authenticate + requireRole(employee, admin) |
| นักศึกษา | `GET /api/students`, `GET /api/students/{studentId}`, `GET /api/students/{studentId}/id-card`, `GET /api/students/{studentId}/with-english` | authenticate + ownership |
| คะแนน | `GET /api/game-scores/student/{student_code}`, `GET /api/game-scores/stats` | authenticate + ownership (stats → employee) |
| OTP | `POST /api/otp/send`, `POST /api/otp/verify`, `POST /api/otp/resend`, `*/verify-otp` | rate limiter |
| SMS | `GET /api/sms/transactions`, `GET /api/sms/statistics` | authenticate + requireRole(employee, admin) |
| เอกสาร | `/api-docs/` | basic auth / JWT (production) |

---

*Requirements Specification — Backend Code MVP กลุ่ม 1 · อ้างอิง API จริง 143 endpoints*
