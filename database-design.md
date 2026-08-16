# 🗄️ Database Design — Backend Code MVP (กลุ่ม 1)

**ระบบ:** Line Siam U Student Registration API (ฝั่ง Backend)
**ขอบเขต:** ตารางที่เกี่ยวข้องกับ 5 ฟีเจอร์ความปลอดภัย (Auth, CORS, Error, OTP Rate Limit, Swagger)
**ฐานข้อมูล:** MySQL — โครงสร้างตารางอ้างอิงจาก **schema จริงใน API docs**
**เวอร์ชัน:** 2.0 (ปรับให้ตรงกับ schema จริง)

---

## 1. ERD (Entity Relationship Diagram)

```
┌──────────────────┐     ┌──────────────────────┐     ┌─────────────────────┐
│     students     │     │     employees        │     │       young         │
├──────────────────┤     ├──────────────────────┤     ├─────────────────────┤
│ id (PK)          │     │ id (PK)              │     │ id (PK)             │
│ student_code     │     │ role  ← ใช้สิทธิ์     │     │ name                │
│ line_user_id     │     │ first_name           │     │ email               │
│ first_name       │     │ last_name            │     │ phone               │
│ last_name        │     │ faculty              │     │ line_id             │
│ phone_number     │     │ phone                │     │ gender              │
│ email            │     │ line_id              │     │ school_name         │
│ status           │     │ email                │     │ otp_verified        │
│ created_at       │     │ created_at           │     │ pdpa_consent        │
└────────┬─────────┘     └──────────────────────┘     └─────────────────────┘
         │
         │ 1
         │ N
┌────────▼─────────┐     ┌──────────────────────┐     ┌─────────────────────┐
│   game_scores    │     │    otp_codes         │     │   sms_transactions  │
├──────────────────┤     ├──────────────────────┤     ├─────────────────────┤
│ id (PK)          │     │ id (PK)              │     │ id (PK)             │
│ student_code (FK)│     │ student_id (FK)      │     │ phone_number        │
│ game_type        │     │ otp_code             │     │ message             │
│ score            │     │ phone_number         │     │ status              │
│ created_at       │     │ expires_at           │     │ created_at          │
└──────────────────┘     │ is_used              │     └─────────┬───────────┘
                         └──────────────────────┘               │
                                                                 │ 1
                         ┌──────────────────────┐                │ N
                         │  line_registrations  │    ┌───────────▼───────────┐
                         ├──────────────────────┤    │   delivery_reports   │
                         │ id (PK)              │    ├───────────────────────┤
                         │ line_user_id         │    │ id (PK)               │
                         │ student_code         │    │ transaction_id (FK)   │
                         │ phone_number         │    │ status                │
                         │ status               │    │ timestamp             │
                         │ verified_at          │    └───────────────────────┘
                         └──────────────────────┘

   ┌── ตารางใหม่ที่เพิ่ม (เฟสนี้) ──────────────────────────────┐
   │  rate_limit_logs      audit_logs                          │
   │  ├ key (เบอร์/IP)     ├ user_id → students/employees.id   │
   │  ├ endpoint           ├ action (EXPORT_USERS, VIEW_YOUNG) │
   │  ├ ip_address         ├ detail (JSON)                     │
   │  ├ blocked (0/1)      ├ row_count                         │
   │  └ created_at         └ created_at                        │
   └───────────────────────────────────────────────────────────┘
```

---

## 2. ตารางในระบบเดิม (schema จริง — ใช้ต่อ ไม่แก้)

### 2.1 students — นักศึกษา
| Column | Type | คำอธิบาย |
|---|---|---|
| `id` | INT PK | รหัสผู้ใช้ (อ้างอิงใน `student_id` ของตารางอื่น) |
| `student_code` | VARCHAR(20) UNIQUE | รหัสนักศึกษา — **key หลักของระบบ** |
| `line_user_id` | VARCHAR(64) | LINE User ID (ข้อมูลอ่อนไหว — ต้อง auth ก่อนเห็น) |
| `first_name` / `last_name` | VARCHAR(100) | ชื่อ-นามสกุล |
| `phone_number` | VARCHAR(20) | เบอร์โทร (ข้อมูลอ่อนไหว) |
| `email` | VARCHAR(100) | อีเมล (ข้อมูลอ่อนไหว) |
| `status` | ENUM | เช่น verified / unverified |
| `created_at` / `updated_at` | TIMESTAMP | เวลา |

*หมายเหตุ: ข้อมูลเพิ่มเติม (faculty_name, department_name, cumulative_gpa, id_card_passport, current_phone_mobile) อยู่ใน schema `StudentImport` — ใช้ร่วมกับตารางนี้*

### 2.2 employees — พนักงาน/ผู้ดูแล
| Column | Type | คำอธิบาย |
|---|---|---|
| `id` | INT PK | รหัสพนักงาน |
| `role` | VARCHAR/ENUM | **บทบาท** — employee / admin (ใช้ใน `requireRole`) |
| `first_name` / `last_name` | VARCHAR | ชื่อ-นามสกุล |
| `faculty` | VARCHAR | คณะ |
| `phone` / `line_id` / `email` | VARCHAR | ข้อมูลติดต่อ (อ่อนไหว) |
| `department` | VARCHAR | แผนก |
| `created_at` / `updated_at` | TIMESTAMP | เวลา |

### 2.3 young — ผู้สมัคร/เยาวชน
| Column | Type | คำอธิบาย |
|---|---|---|
| `id` | INT PK | รหัส |
| `name` | VARCHAR | ชื่อ (อ่อนไหว) |
| `email` / `phone` / `line_id` | VARCHAR | ข้อมูลติดต่อ (อ่อนไหว — ต้อง role employee/admin) |
| `gender` / `school_name` | VARCHAR | ข้อมูลส่วนตัว |
| `education_level` | VARCHAR | ระดับการศึกษา |
| `otp_verified` | BOOLEAN | สถานะ verify OTP |
| `pdpa_consent` | BOOLEAN | ยินยอม PDPA |
| `created_at` / `updated_at` | TIMESTAMP | เวลา |

### 2.4 otp_codes — รหัส OTP
| Column | Type | คำอธิบาย |
|---|---|---|
| `id` | INT PK | รหัส |
| `student_id` | INT FK → students.id | ผู้ใช้ |
| `otp_code` | VARCHAR(6) | รหัส OTP (**ควร hash** — กันรั่วจาก DB leak) |
| `phone_number` | VARCHAR(20) | เบอร์ที่ส่ง |
| `expires_at` | TIMESTAMP | วันหมดอายุ (ตรวจก่อน verify) |
| `is_used` | BOOLEAN | ใช้ไปแล้วหรือยัง (กัน replay) |

### 2.5 sms_transactions + delivery_reports — การส่ง SMS
| Column | Type | คำอธิบาย |
|---|---|---|
| `sms_transactions.id` | INT PK | รายการส่ง SMS |
| `phone_number` / `message` / `status` | VARCHAR | ข้อมูลการส่ง |
| `created_at` | TIMESTAMP | เวลา |
| `delivery_reports.id` | INT PK | รายงานการส่งถึง |
| `transaction_id` | INT FK → sms_transactions.id | อ้างอิงรายการ |
| `status` / `timestamp` | VARCHAR/TIMESTAMP | สถานะ/เวลา |

### 2.6 game_scores — คะแนนเกม
| Column | Type | คำอธิบาย |
|---|---|---|
| `id` | INT PK | รหัส |
| `student_code` | VARCHAR(20) FK → students.student_code | รหัสนักศึกษา |
| `game_type` | VARCHAR(20) | quiz / trash-catch / trash-sorting / matching |
| `score` | INT | คะแนน |
| `created_at` / `updated_at` | TIMESTAMP | เวลา |

### 2.7 line_registrations — การลงทะเบียนผ่าน LINE
| Column | Type | คำอธิบาย |
|---|---|---|
| `id` | INT PK | รหัส |
| `line_user_id` | VARCHAR(64) | LINE ID |
| `student_code` / `phone_number` | VARCHAR | ข้อมูล |
| `status` | ENUM | pending / verified |
| `created_at` / `verified_at` | TIMESTAMP | เวลา |

---

## 3. ตารางใหม่ที่เพิ่ม (สำหรับฟีเจอร์ MVP)

### 3.1 rate_limit_logs — log การถูกบล็อก (FR-4)

```sql
CREATE TABLE rate_limit_logs (
  id         BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  rate_key   VARCHAR(64)  NOT NULL,   -- เบอร์โทร หรือ IP
  endpoint   VARCHAR(128) NOT NULL,   -- /api/otp/verify
  ip_address VARCHAR(45)  NOT NULL,
  blocked    BOOLEAN      DEFAULT 0,
  created_at TIMESTAMP    DEFAULT CURRENT_TIMESTAMP,
  INDEX idx_key (rate_key, created_at)
) ENGINE=InnoDB;
```

**ใช้ทำอะไร:** ตรวจสอบการโจมตี (brute-force/SMS bombing) ย้อนหลัง + ใช้แจ้งเตือน

### 3.2 audit_logs — ตรวจสอบย้อนกลับ export/ข้อมูลอ่อนไหว (NFR-8)

```sql
CREATE TABLE audit_logs (
  id         BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  user_id    INT UNSIGNED NOT NULL,   -- FK → students.id / employees.id
  user_type  ENUM('student','employee','admin') NOT NULL,
  action     VARCHAR(50)  NOT NULL,   -- EXPORT_USERS, VIEW_YOUNG, ...
  detail     JSON         NULL,       -- params, filters
  row_count  INT UNSIGNED DEFAULT 0,  -- จำนวนแถวที่ export
  created_at TIMESTAMP    DEFAULT CURRENT_TIMESTAMP,
  INDEX idx_user (user_id, user_type),
  INDEX idx_action (action, created_at)
) ENGINE=InnoDB;
```

**ใช้ทำอะไร:** ทุกครั้งที่ export (`/api/export/users`) หรือดูข้อมูลอ่อนไหว (`/api/young`) ต้องเขียน log — ตรวจพบการรั่วไหลได้

---

## 4. ความสัมพันธ์ (Relationships)

| จาก | ไป | ประเภท | คำอธิบาย |
|---|---|---|---|
| `students` | `game_scores` | 1 : N | นักศึกษา 1 คน มีคะแนนหลายรายการ (ผ่าน student_code) |
| `students` | `otp_codes` | 1 : N | นักศึกษา 1 คน ขอ OTP หลายครั้ง |
| `sms_transactions` | `delivery_reports` | 1 : N | SMS 1 รายการ มี delivery report หลายอัน |
| `line_registrations` | `students` | N : 1 | ลงทะเบียน LINE แล้วผูกกับนักศึกษา |
| `students`/`employees` | `audit_logs` | 1 : N | ผู้ใช้ 1 คน มี audit log หลายรายการ |

---

## 5. ดัชนี (Indexes) และเหตุผล

| ตาราง | Index | เหตุผล |
|---|---|---|
| `students.student_code` | UNIQUE | lookup เร็วตอนตรวจ ownership (IDOR) |
| `otp_codes(phone_number, is_used)` | INDEX | ค้นหา OTP ล่าสุดต่อเบอร์ + กัน replay |
| `sms_transactions(phone_number, created_at)` | INDEX | query SMS bombing ตามเวลา |
| `game_scores(student_code)` | INDEX | ดึงคะแนนของนักศึกษาเร็ว |
| `rate_limit_logs(rate_key, created_at)` | INDEX | query การโจมตีตามเวลา |
| `audit_logs(action, created_at)` | INDEX | ค้นหา export ที่น่าสงสัย |

---

## 6. ข้อควรระวังด้านความปลอดภัย

1. **OTP code:** เก็บแบบ hash (bcrypt/argon2) ไม่เก็บ plaintext — กันรั่วจาก DB leak
2. **`line_user_id`, `phone_number`, `email`:** ข้อมูลอ่อนไหว — ต้องผ่าน `authenticate` + `requireRole('employee','admin')` ทุกครั้ง (เช่น `/api/young`)
3. **`audit_logs.detail`:** ห้ามเก็บ token/รหัสผ่าน — เก็บเฉพาะพารามิเตอร์ที่จำเป็น
4. **Rate limit state:** สำหรับ production หลาย instance ควรย้าย state ไป **Redis** แทน in-memory
5. **`JWT_SECRET`:** ไม่เก็บใน DB/code — ใช้ environment variable

---

## 7. ตัวอย่าง Query ที่ใช้กับฟีเจอร์

```sql
-- ตรวจ OTP (FR-4/AC-4.5): นับครั้ง + เช็คสถานะ
SELECT is_used, expires_at FROM otp_codes
WHERE phone_number = '0812345678' AND is_used = 0
ORDER BY created_at DESC LIMIT 1;

-- กัน replay OTP
UPDATE otp_codes SET is_used = 1 WHERE id = ? AND is_used = 0;

-- เขียน audit log เมื่อ export (NFR-8)
INSERT INTO audit_logs (user_id, user_type, action, detail, row_count)
VALUES (?, 'employee', 'EXPORT_USERS', JSON_OBJECT('format','xlsx'), 16191);

-- หาการโจมตี SMS bombing (จาก sms_transactions)
SELECT phone_number, COUNT(*) AS sends
FROM sms_transactions
WHERE created_at > NOW() - INTERVAL 1 HOUR
GROUP BY phone_number HAVING sends >= 3;

-- ตรวจผู้ใช้ที่ export เยอะผิดปกติ (จาก audit_logs)
SELECT user_id, user_type, COUNT(*) AS exports
FROM audit_logs WHERE action = 'EXPORT_USERS'
GROUP BY user_id, user_type
ORDER BY exports DESC LIMIT 10;
```

---

*Database Design — Backend Code MVP กลุ่ม 1 · อ้างอิง schema จริง (students, employees, young, otp_codes, sms_transactions, game_scores)*
