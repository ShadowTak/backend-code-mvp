# ✅ Acceptance Criteria — Backend Code MVP (กลุ่ม 1)

**ระบบ:** Line Siam U Student Registration API (ฝั่ง Backend)
**ขอบเขต:** 5 ฟีเจอร์อุดช่องโหว่ — JWT Auth & Role, CORS, Global Error, OTP Rate Limit, Swagger Protection
**อ้างอิง endpoint จริง:** `/api/students`, `/api/young`, `/api/export/users`, `/api/otp/verify`, `/api/game-scores/stats` ฯลฯ
**รูปแบบ:** Given / When / Then

---

## 1. JWT Authentication & Role Middleware

### AC-1.1: ไม่มี token → 401
```
GIVEN  ยังไม่ล็อคอิน (ไม่มี Authorization header)
WHEN   ส่ง GET /api/young หรือ GET /api/export/users หรือ GET /api/students
THEN   ระบบตอบ 401 Unauthorized พร้อม message "Unauthorized: ต้องล็อคอินก่อน"
```

### AC-1.2: token ปลอม/หมดอายุ → 401
```
GIVEN  มี JWT token ที่ปลอมขึ้นมา หรือหมดอายุแล้ว
WHEN   ส่ง token นี้ใน Authorization header ไปที่ API ที่ต้องสิทธิ์
THEN   ระบบตอบ 401 Unauthorized พร้อม message "token ไม่ถูกต้องหรือหมดอายุ"
```

### AC-1.3: role ไม่ตรง → 403
```
GIVEN  ล็อคอินเป็นนักศึกษา (role=student) ด้วย token ที่ถูกต้อง
WHEN   ส่ง GET /api/young (ต้องเป็น employee/admin เท่านั้น)
THEN   ระบบตอบ 403 Forbidden พร้อม message "Forbidden: ต้องเป็น employee/admin เท่านั้น"
```

### AC-1.4: นักศึกษาดูข้อมูลคนอื่น → 403 (กัน IDOR)
```
GIVEN  ล็อคอินเป็นนักศึกษา 6705100003 ด้วย token ที่ถูกต้อง
WHEN   ส่ง GET /api/students/6705100042 หรือ GET /api/game-scores/student/6705100042 (ของคนอื่น)
THEN   ระบบตอบ 403 Forbidden พร้อม message "ไม่มีสิทธิ์ดูข้อมูลนี้"
```

### AC-1.5: นักศึกษาดูข้อมูลตัวเอง → 200
```
GIVEN  ล็อคอินเป็นนักศึกษา 6705100003 ด้วย token ที่ถูกต้อง
WHEN   ส่ง GET /api/students/6705100003/with-english หรือ GET /api/game-scores/student/6705100003
THEN   ระบบตอบ 200 OK พร้อมข้อมูลของตัวเอง
```

### AC-1.6: Export ต้องเป็น employee/admin
```
GIVEN  ล็อคอินเป็นนักศึกษา (role=student)
WHEN   ส่ง GET /api/export/users หรือ GET /api/export/game-scores
THEN   ระบบตอบ 403 Forbidden — ห้าม export

GIVEN  ล็อคอินเป็นพนักงาน (role=employee จาก employees.role)
WHEN   ส่ง GET /api/export/users
THEN   ระบบตอบ 200 OK พร้อมไฟล์ Excel และมี audit log (NFR-8)
```

### AC-1.7: รับ token ผ่าน flow LINE จริง
```
GIVEN  ผู้ใช้ยืนยันตัวตนผ่าน LINE (GET /api/line/profile/{userId}) และ verify OTP สำเร็จ
WHEN   ระบบออก JWT token (payload: id, role, student_code)
THEN   token นี้ใช้เข้าถึง API ที่มีสิทธิ์ได้ และหมดอายุตาม config (เช่น 2 ชั่วโมง)
```

---

## 2. CORS Configuration

### AC-2.1: Origin ใน allowlist → ได้ CORS header
```
GIVEN  Origin = https://siam-frontend-production.up.railway.app (อยู่ใน allowlist)
WHEN   ส่ง request พร้อม Origin header
THEN   response มี Access-Control-Allow-Origin = origin นั้น และมี Vary: Origin
```

### AC-2.2: Origin นอก allowlist → ไม่มี CORS header
```
GIVEN  Origin = https://evil.com (ไม่อยู่ใน allowlist)
WHEN   ส่ง request พร้อม Origin header
THEN   response ไม่มี Access-Control-Allow-Origin → browser บล็อกการอ่านข้อมูล
```

### AC-2.3: Preflight OPTIONS → 204
```
GIVEN  Origin อยู่ใน allowlist
WHEN   ส่ง OPTIONS request (preflight)
THEN   ระบบตอบ 204 No Content
```

---

## 3. Global Error Handler

### AC-3.1: SQL error ถูกซ่อน (กรณีจริงจาก /api/game-scores/stats)
```
GIVEN  เกิด SQL error — Unknown column 'student_code' in 'field list' (พบจริงจาก GET /api/game-scores/stats)
WHEN   request ถึง route ที่ error
THEN   response 500 พร้อม body {"success":false,"message":"Internal server error"}
       และ response ไม่มีคำว่า student_code / SQL / column / stack trace
```

### AC-3.2: Error ถูก log ฝั่ง server
```
GIVEN  เกิด error ใด ๆ
WHEN   request ถึง route ที่ error
THEN   console ของ server แสดง log: path, method, message, stack, user
```

### AC-3.3: Validation error → 400 (OTP จริง)
```
GIVEN  ส่งข้อมูลไม่ถูกต้อง (เช่น ไม่มี otp_code)
WHEN   POST /api/otp/verify โดยไม่ส่ง otp_code
THEN   ระบบตอบ 400 พร้อม message ที่เข้าใจได้ (เช่น "กรุณากรอกรหัส OTP")
```

---

## 4. OTP Rate Limiter

### AC-4.1: Verify OTP เกิน 5 ครั้ง → 429
```
GIVEN  เบอร์โทร 0812345678
WHEN   ส่ง POST /api/otp/verify (หรือ /api/students/verify-otp) 6 ครั้งภายใน 10 นาที
THEN   ครั้งที่ 6 ตอบ 429 "Too many attempts — ลองใหม่ใน 10 นาที"
```

### AC-4.2: ส่ง OTP/SMS เกิน 3 ครั้ง/ชม. → 429
```
GIVEN  เบอร์โทร 0812345678
WHEN   ส่ง POST /api/otp/send หรือ POST /api/otp/resend 4 ครั้งภายใน 1 ชั่วโมง
THEN   ครั้งที่ 4 ตอบ 429 "ส่ง OTP เกินกำหนด — ลองใหม่ใน 1 ชั่วโมง"
```

### AC-4.3: มี RateLimit headers
```
GIVEN  ถูก rate limit
WHEN   ตรวจสอบ response headers
THEN   มี RateLimit-Policy / RateLimit (standardHeaders: true)
```

### AC-4.4: Rate limit แยกตามเบอร์
```
GIVEN  เบอร์ A ยิงครบ 5 ครั้ง (ถูก 429)
WHEN   เบอร์ B ยิง verify ครั้งแรก
THEN   เบอร์ B ยังยิงได้ปกติ (ไม่ถูกบล็อกด้วยเบอร์ A)
```

### AC-4.5: OTP ใช้แล้ว/หมดอายุ → ปฏิเสธ (ตรวจตาราง otp_codes)
```
GIVEN  otp_code ในตาราง otp_codes มี is_used = true หรือ expires_at ผ่านไปแล้ว
WHEN   ส่ง POST /api/otp/verify ด้วย otp_code นั้น
THEN   ระบบตอบ 400 "OTP หมดอายุหรือใช้ไปแล้ว"
```

---

## 5. Swagger Docs Protection

### AC-5.1: Dev mode เปิดดูได้
```
GIVEN  NODE_ENV != production
WHEN   เปิด GET /api-docs/
THEN   ระบบตอบ 200 — ดู spec ได้ (143 endpoints)
```

### AC-5.2: Production ต้อง auth
```
GIVEN  NODE_ENV = production
WHEN   เปิด GET /api-docs/ โดยไม่ส่ง basic auth
THEN   ระบบตอบ 401 (challenge: true)

GIVEN  NODE_ENV = production และส่ง basic auth (admin + SWAGGER_PASSWORD)
WHEN   เปิด GET /api-docs/
THEN   ระบบตอบ 200 — ดู spec ได้
```

### AC-5.3: ใช้ JWT role ได้ด้วย
```
GIVEN  ใช้ JWT variant (authenticate + requireRole('employee','admin'))
WHEN   ล็อคอินเป็น employee แล้วเปิด /api-docs/
THEN   ระบบตอบ 200
```

---

## สรุปการทดสอบ (ผลจริงจาก pentest + demo)

| AC | ทดสอบ (endpoint จริง) | ผล |
|---|---|---|
| AC-1.1 | `GET /api/young` โดยไม่ล็อคอิน | ✅ 401 |
| AC-1.6 | `GET /api/export/users` โดยไม่ล็อคอิน (ก่อนอุดได้ Excel 1.3MB) | ✅ 401 หลังอุด |
| AC-2.2 | `Origin: https://evil.com` | ✅ ไม่มี CORS header |
| AC-3.1 | `GET /api/game-scores/stats` (ก่อนอุดคืน SQL error) | ✅ `Internal server error` |
| AC-4.1 | ยิง OTP verify 6 ครั้ง | ✅ 400,400,400,400,400,429 |
| AC-5.2 | เปิด `/api-docs/` ใน fixed mode | ✅ 401 ต้องล็อคอิน |

---

*Acceptance Criteria — Backend Code MVP กลุ่ม 1 · อ้างอิง endpoint และ schema จริงจาก API docs*
