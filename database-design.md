# 🗄️ Database Design — Backend Code MVP (กลุ่ม 1)

**ระบบ:** Line Siam U Student Registration API (ฝั่ง Backend)
**ขอบเขต:** ตารางที่เกี่ยวข้องกับ 5 ฟีเจอร์ความปลอดภัย (Auth, CORS, Error, OTP Rate Limit, Swagger)
**ฐานข้อมูล:** MySQL (ระบบเดิม) — เพิ่มตารางใหม่โดยไม่กระทบตารางเดิม

---

## 1. ERD (Entity Relationship Diagram)

```
┌──────────────────┐     ┌──────────────────────┐
│     users        │     │   otp_requests       │
├──────────────────┤     ├──────────────────────┤
│ id (PK)          │     │ id (PK)              │
│ student_code     │     │ phone_number         │
│ role             │     │ otp_code             │
│ name             │     │ attempts             │
│ line_user_id     │     │ last_attempt_at      │
│ is_active        │     │ status               │
│ created_at       │     │ created_at           │
└──────┬───────────┘     └──────────────────────┘
       │ 1
       │
       │ N
┌──────▼───────────┐     ┌──────────────────────┐
│  audit_logs      │     │   rate_limit_logs    │
├──────────────────┤     ├──────────────────────┤
│ id (PK)          │     │ id (PK)              │
│ user_id (FK)     │     │ key (เบอร์/IP)       │
│ action           │     │ endpoint             │
│ detail           │     │ ip_address           │
│ row_count        │     │ blocked (0/1)        │
│ created_at       │     │ created_at           │
└──────────────────┘     └──────────────────────┘
```

---

## 2. ตารางในระบบเดิม (มีอยู่แล้ว — ใช้ต่อ)

### users
| Column | Type | คำอธิบาย |
|---|---|---|
| `id` | INT PK AUTO_INCREMENT | รหัสผู้ใช้ |
| `student_code` | VARCHAR(20) UNIQUE | รหัสนักศึกษา (ใช้เป็น ownership key) |
| `role` | ENUM('student','employee','admin') | บทบาท — **สำคัญสำหรับ middleware** |
| `name` | VARCHAR(100) | ชื่อ-นามสกุล |
| `line_user_id` | VARCHAR(64) | LINE User ID (ข้อมูลอ่อนไหว — ต้อง auth ก่อนเห็น) |
| `is_active` | BOOLEAN | สถานะผู้ใช้ |
| `created_at` | TIMESTAMP | วันที่สมัคร |

### students / game_scores / id_cards
ตารางข้อมูลหลักเดิม (ไม่แก้ไขในเฟสนี้)

---

## 3. ตารางใหม่ที่เพิ่ม

### 3.1 otp_requests — เก็บ OTP + นับจำนวนครั้ง

```sql
CREATE TABLE otp_requests (
  id              BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  phone_number    VARCHAR(20)  NOT NULL,
  otp_code        VARCHAR(6)   NOT NULL,
  attempts        TINYINT UNSIGNED DEFAULT 0,
  last_attempt_at TIMESTAMP    NULL,
  status          ENUM('pending','verified','expired') DEFAULT 'pending',
  created_at      TIMESTAMP    DEFAULT CURRENT_TIMESTAMP,
  expires_at      TIMESTAMP    NOT NULL,
  INDEX idx_phone (phone_number),
  INDEX idx_status (status)
) ENGINE=InnoDB;
```

**ใช้ทำอะไร:** ตรวจ OTP + นับความพยายาม (backend อ่าน `attempts` ก่อน verify → เกิน 5 ครั้งปฏิเสธ) — เสริม rate limiter ใน app layer ให้ตรวจได้ข้าม instance

### 3.2 rate_limit_logs — log การถูกบล็อก

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

### 3.3 audit_logs — ตรวจสอบย้อนกลับ (export/ข้อมูลอ่อนไหว)

```sql
CREATE TABLE audit_logs (
  id         BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  user_id    INT UNSIGNED NOT NULL,   -- FK → users.id
  action     VARCHAR(50)  NOT NULL,   -- EXPORT_USERS, VIEW_YOUNG, ...
  detail     JSON         NULL,       -- params, filters
  row_count  INT UNSIGNED DEFAULT 0,  -- จำนวนแถวที่ export
  created_at TIMESTAMP    DEFAULT CURRENT_TIMESTAMP,
  INDEX idx_user (user_id),
  INDEX idx_action (action, created_at)
) ENGINE=InnoDB;
```

**ใช้ทำอะไร:** ทุกครั้งที่ export ข้อมูล / ดูข้อมูลอ่อนไหว ต้องเขียน log (NFR-8) — ตรวจพบการรั่วไหลได้

---

## 4. ความสัมพันธ์ (Relationships)

| จาก | ไป | ประเภท | คำอธิบาย |
|---|---|---|---|
| `users` | `audit_logs` | 1 : N | ผู้ใช้ 1 คน มี audit log หลายรายการ |
| `users` | `otp_requests` | 1 : N | ผู้ใช้ 1 คน ขอ OTP หลายครั้ง |
| `users` | `students` | 1 : 1 | ผู้ใช้ 1 คน มีข้อมูลนักศึกษา 1 ชุด (ผ่าน student_code) |

---

## 5. ดัชนี (Indexes) และเหตุผล

| ตาราง | Index | เหตุผล |
|---|---|---|
| `users.student_code` | UNIQUE | lookup เร็วตอนตรวจ ownership |
| `otp_requests(phone_number)` | INDEX | ค้นหา OTP ล่าสุดต่อเบอร์ |
| `rate_limit_logs(rate_key, created_at)` | INDEX | query การโจมตีตามเวลา |
| `audit_logs(action, created_at)` | INDEX | ค้นหา export ที่น่าสงสัย |

---

## 6. ข้อควรระวังด้านความปลอดภัย

1. **OTP code:** เก็บแบบ hash (เช่น bcrypt/argon2) ไม่เก็บ plaintext — กันรั่วจาก DB leak
2. **line_user_id:** ถือเป็นข้อมูลอ่อนไหว — ต้องผ่าน `requireRole('employee','admin')` ทุกครั้ง
3. **audit_logs.detail:** ห้ามเก็บ token/รหัสผ่าน — เก็บเฉพาะพารามิเตอร์ที่จำเป็น
4. **Rate limit state:** สำหรับ production หลาย instance ควรย้าย state ไป **Redis** แทน in-memory
5. **JWT_SECRET:** ไม่เก็บใน DB/code — ใช้ environment variable

---

## 7. ตัวอย่าง Query ที่ใช้กับฟีเจอร์

```sql
-- ตรวจ OTP (FR-4): นับครั้งที่ verify ผิด
SELECT attempts FROM otp_requests
WHERE phone_number = '0812345678' AND status = 'pending'
ORDER BY created_at DESC LIMIT 1;

-- อัปเดตเมื่อ verify ผิด
UPDATE otp_requests SET attempts = attempts + 1, last_attempt_at = NOW()
WHERE id = ?;

-- เขียน audit log เมื่อ export (NFR-8)
INSERT INTO audit_logs (user_id, action, detail, row_count)
VALUES (?, 'EXPORT_USERS', JSON_OBJECT('format','xlsx'), 16191);

-- หาการโจมตี SMS bombing
SELECT phone_number, COUNT(*) AS sends
FROM rate_limit_logs
WHERE endpoint = '/api/otp/send' AND created_at > NOW() - INTERVAL 1 HOUR
GROUP BY phone_number HAVING sends >= 3;
```

---

*Database Design — Backend Code MVP กลุ่ม 1 · MySQL*
