# 💻 Backend Code MVP — กลุ่ม 1

**โฟกัสเฉพาะฝั่ง API** — ปิดรูรั่วข้อมูลและ Export ที่เจอจาก pentest จริง บนระบบ
`Line Siam U Student Registration API` (Express.js)

> ใช้ได้จริง — โค้ดทั้งหมดเป็น Express middleware พร้อมนำไปใส่ใน `app.js` ได้ทันที
> ทดลองกดปุ่ม ▶ ทดสอบ ได้ที่หน้า demo: `http://localhost:4173/mvp` (หรือ `/demo`)

---

## 🎯 ภาพรวม 5 ฟีเจอร์

| # | ฟีเจอร์ | ช่องโหว่ที่อุด | ความรุนแรง |
|---|---|---|---|
| 1 | **JWT Authentication & Role Middleware** | ข้อมูลรั่ว + Export โดยไม่ต้องล็อคอิน | 🔴 Critical |
| 2 | **CORS Configuration** | Origin เปิดกว้าง `*` | 🔴 Critical |
| 3 | **Global Error Handler** | SQL Error รั่วไหล | 🟠 High |
| 4 | **OTP Rate Limiter** | Brute-force & SMS Bombing | 🟡 Medium |
| 5 | **Swagger Docs Protection** | API Spec เปิดสาธารณะ | 🟡 Medium |

---

## 1️⃣ JWT Authentication & Role Middleware

### ปัญหา (จาก pentest จริง)
- `GET /api/young` เปิดเผย **ชื่อ + อีเมล + เบอร์โทร + LINE User ID** ของทุกคน โดยไม่ต้องล็อคอิน
- `GET /api/export/users` ดาวน์โหลด **Excel 1.3 MB ข้อมูล 16,191 คน** ได้ฟรี
- `GET /api/students` เปิดเผยรายชื่อนักศึกษา 12,722 คน

### โค้ด

**`middleware/auth.js`**

```js
// middleware/auth.js — JWT Authentication + Role
const jwt = require('jsonwebtoken');

function authenticate(req, res, next) {
  const header = req.headers.authorization || '';
  const token = header.startsWith('Bearer ') ? header.slice(7) : null;
  if (!token) {
    return res.status(401).json({ success: false, message: 'Unauthorized: ต้องล็อคอินก่อน' });
  }
  try {
    req.user = jwt.verify(token, process.env.JWT_SECRET);
    next();
  } catch (err) {
    return res.status(401).json({ success: false, message: 'Unauthorized: token ไม่ถูกต้องหรือหมดอายุ' });
  }
}

function requireRole(...roles) {
  return (req, res, next) => {
    if (!req.user) return res.status(401).json({ success: false, message: 'Unauthorized' });
    if (!roles.includes(req.user.role)) {
      return res.status(403).json({ success: false, message: 'Forbidden: ต้องเป็น ' + roles.join('/') + ' เท่านั้น' });
    }
    next();
  };
}

// Ownership: นักศึกษาดูได้เฉพาะข้อมูลตัวเอง (กัน IDOR)
function ownership(getTargetCode) {
  return (req, res, next) => {
    if (req.user.role === 'employee' || req.user.role === 'admin') return next();
    const target = getTargetCode(req);
    if (req.user.student_code === target) return next();
    return res.status(403).json({ success: false, message: 'Forbidden: ไม่มีสิทธิ์ดูข้อมูลนี้' });
  };
}

module.exports = { authenticate, requireRole, ownership };
```

**`routes/students.js`** — วิธีใช้กับ endpoint

```js
// routes/students.js — ใช้ middleware กับทุก endpoint
const { authenticate, requireRole, ownership } = require('../middleware/auth');

// ข้อมูลส่วนตัว — ต้องล็อคอิน + เป็นเจ้าของ
router.get('/api/students/:id',
  authenticate,
  ownership((req) => req.params.id),
  handler
);

// ข้อมูล young (ชื่อ, อีเมล, เบอร์โทร, LINE ID) — employee เท่านั้น
router.get('/api/young',
  authenticate,
  requireRole('employee', 'admin'),
  handler
);

// Export ข้อมูลทั้งหมด — employee เท่านั้น (สำคัญมาก!)
router.get('/api/export/users',
  authenticate,
  requireRole('employee', 'admin'),
  (req, res) => { /* สร้าง Excel + log audit */ }
);
```

### ผลลัพธ์
| สถานะ | ก่อนอุด | หลังอุด |
|---|---|---|
| ไม่ล็อคอิน → `/api/young` | `200` ข้อมูลรั่ว | `401 Unauthorized` |
| นักศึกษา → `/api/students/:คนอื่น` | `200` (IDOR) | `403 Forbidden` |
| นักศึกษา → `/api/export/users` | `200` ไฟล์ Excel | `403 Forbidden` |

---

## 2️⃣ CORS Configuration

### ปัญหา
- `Access-Control-Allow-Origin: *` — **เว็บมุ่งร้าย** เปิดใน browser ของเหยื่อ แล้วอ่านข้อมูล API ได้ทั้งหมด (ข้อมูลส่วนตัวรั่วผ่าน browser)

### โค้ด

**`middleware/cors.js`**

```js
// middleware/cors.js — CORS allowlist (ไม่ใช้ *)
const ALLOWED_ORIGINS = [
  'https://siam-frontend-production.up.railway.app',
  'http://localhost:3000',
  'http://localhost:4173',
];

function cors(req, res, next) {
  const origin = req.headers.origin;
  if (origin && ALLOWED_ORIGINS.includes(origin)) {
    res.setHeader('Access-Control-Allow-Origin', origin);
    res.setHeader('Vary', 'Origin'); // สำคัญ: ให้ cache แยกตาม origin
    res.setHeader('Access-Control-Allow-Methods', 'GET,POST,PUT,DELETE,OPTIONS');
    res.setHeader('Access-Control-Allow-Headers', 'Content-Type, Authorization');
  }
  // origin นอก allowlist → ไม่ได้ CORS header → browser บล็อกการอ่าน
  if (req.method === 'OPTIONS') return res.sendStatus(204);
  next();
}

// app.js
app.use(cors);
```

### ผลลัพธ์
- `Origin: https://siam-frontend-production.up.railway.app` → ได้ CORS header ✅
- `Origin: https://evil.com` → **ไม่มี** CORS header → browser บล็อกการอ่านข้อมูล ❌

---

## 3️⃣ Global Error Handler

### ปัญหา
- `GET /api/game-scores/stats` คืน `Unknown column 'student_code' in 'field list'` — **เปิดเผยโครงสร้างฐานข้อมูล** (table, column, SQL dialect) ให้ผู้โจมตีใช้วางแผน attack ต่อ

### โค้ด

**`middleware/error.js`**

```js
// middleware/error.js — Global Error Handler
// ใช้ catch-all กับทุก route: app.use(errorHandler) (ต้องเป็นตัวสุดท้าย)

function errorHandler(err, req, res, next) {
  // log รายละเอียดไว้ฝั่ง server เท่านั้น (console / sentry)
  console.error('[ERROR]', {
    path: req.originalUrl,
    method: req.method,
    message: err.message,
    stack: err.stack,
    user: req.user?.id,
  });

  // ตรวจ error ประเภทที่รู้จัก
  if (err.name === 'ValidationError') {
    return res.status(400).json({ success: false, message: 'ข้อมูลไม่ถูกต้อง' });
  }
  if (err.code === 'ER_BAD_FIELD_ERROR' || err.code === 'ER_NO_SUCH_TABLE') {
    // SQL error → ไม่เปิดเผย message ภายใน!
    return res.status(500).json({ success: false, message: 'Internal server error' });
  }

  // fallback ทั่วไป — ไม่มี stack trace, ไม่มี SQL, ไม่มี path
  res.status(500).json({ success: false, message: 'Internal server error' });
}

// ตัวอย่าง async wrapper (กัน error ตกค้าง)
const asyncHandler = (fn) => (req, res, next) =>
  Promise.resolve(fn(req, res, next)).catch(next);
```

**`app.js`**

```js
// app.js — ต่อท้ายสุด (หลัง routes ทั้งหมด)
app.use(errorHandler);
```

### ผลลัพธ์
| สถานะ | ก่อนอุด | หลังอุด |
|---|---|---|
| `/api/game-scores/stats` พัง | `Unknown column 'student_code'...` | `Internal server error` |

---

## 4️⃣ OTP Rate Limiter

### ปัญหา
- ยิง `POST /api/otp/verify` ซ้ำได้ไม่จำกัด — **brute-force รหัส OTP 4 หลัก** (มีโอกาสถูก 1/10,000 ต่อเบอร์)
- ยิง `POST /api/otp/send` ซ้ำได้ไม่จำกัด — **SMS Bombing** (ค่าใช้จ่าย + รบกวนผู้ใช้)

### โค้ด

**`middleware/rateLimit.js`**

```js
// middleware/rateLimit.js — OTP Rate Limiter
// production แนะนำเก็บ state ใน Redis/DB — ตัวอย่างนี้เป็น in-memory

const rateLimiter = require('express-rate-limit');

// จำกัดต่อเบอร์โทร: 5 ครั้ง / 10 นาที (กัน brute-force OTP)
const otpVerifyLimiter = rateLimiter({
  windowMs: 10 * 60 * 1000,   // 10 นาที
  max: 5,                      // 5 ครั้ง
  keyGenerator: (req) => req.body?.phone_number || req.ip,
  message: { success: false, message: 'Too many attempts — ลองใหม่ใน 10 นาที' },
  standardHeaders: true,       // ส่ง RateLimit-* headers
});

// จำกัดการส่ง SMS ต่อเบอร์: 3 ครั้ง / ชั่วโมง (กัน SMS bombing)
const smsSendLimiter = rateLimiter({
  windowMs: 60 * 60 * 1000,
  max: 3,
  keyGenerator: (req) => req.body?.phone_number || req.ip,
  message: { success: false, message: 'ส่ง OTP เกินกำหนด — ลองใหม่ใน 1 ชั่วโมง' },
});

module.exports = { otpVerifyLimiter, smsSendLimiter };
```

**`routes/otp.js`**

```js
// routes/otp.js — ใช้กับ route
const { otpVerifyLimiter, smsSendLimiter } = require('../middleware/rateLimit');

router.post('/api/otp/send', smsSendLimiter, sendOtpHandler);
router.post('/api/otp/verify', otpVerifyLimiter, verifyOtpHandler);
```

### ผลลัพธ์
```
ยิง /api/otp/verify 6 ครั้ง:  400, 400, 400, 400, 400, 429  ← ครั้งที่ 6 โดน rate limit
```

---

## 5️⃣ Swagger Docs Protection

### ปัญหา
- `/api-docs/` เปิดสาธารณะ — ใครก็ดู spec **ทั้ง 112 endpoints** ได้ → ผู้โจมตีรู้ attack surface ครบ (path, parameter, รูปแบบข้อมูล)

### โค้ด

**`app.js`**

```js
// app.js — Swagger Docs Protection
const basicAuth = require('express-basic-auth');

// เปิดเฉพาะตอน dev
if (process.env.NODE_ENV === 'production') {
  // production: ต้อง basic auth ก่อนดู spec
  app.use('/api-docs',
    basicAuth({
      users: { admin: process.env.SWAGGER_PASSWORD }, // ตั้งผ่าน env
      challenge: true,
    }),
    swaggerUi.serve,
    swaggerUi.setup(spec)
  );
} else {
  // dev: เปิดให้ดูได้
  app.use('/api-docs', swaggerUi.serve, swaggerUi.setup(spec));
}

// หรือจะใช้ JWT แทนก็ได้ — requireRole('employee')
app.use('/api-docs',
  authenticate,
  requireRole('employee', 'admin'),
  swaggerUi.serve,
  swaggerUi.setup(spec)
);
```

### ผลลัพธ์
| สถานะ | ก่อนอุด | หลังอุด |
|---|---|---|
| เปิด `/api-docs/` (production) | `200` ดู spec ได้ | `401` ต้องล็อคอิน |

---

## 📦 สรุปไฟล์ที่ต้องเพิ่ม

```
backend/
├── middleware/
│   ├── auth.js        # JWT + Role + Ownership
│   ├── cors.js        # CORS allowlist
│   ├── error.js       # Global error handler
│   └── rateLimit.js   # OTP rate limiter
├── routes/
│   ├── students.js    # ใช้ authenticate + ownership
│   ├── otp.js         # ใช้ rate limiter
│   └── ...
└── app.js             # ต่อ middleware + error handler + swagger protection
```

**Dependencies ที่ต้องติดตั้งเพิ่ม**

```bash
npm install jsonwebtoken express-rate-limit express-basic-auth
```

**Environment variables ที่ต้องตั้ง**

```env
JWT_SECRET=<random-string-ยาว-อย่างน้อย-32-ตัว>
SWAGGER_PASSWORD=<รหัสสำหรับดู api-docs>
NODE_ENV=production   # ใน production
```

---

## ✅ สรุปการทดสอบจริง (จากหน้า demo)

| # | ทดสอบ | ผล |
|---|---|---|
| 1 | `GET /api/young` โดยไม่ล็อคอิน | ✅ `401 Unauthorized` |
| 2 | `Origin: https://evil.com` | ✅ ไม่มี CORS header → browser บล็อก |
| 3 | `GET /api/game-scores/stats` | ✅ `Internal server error` (ไม่มี SQL) |
| 4 | ยิง OTP verify 6 ครั้ง | ✅ `400,400,400,400,400,429` |
| 5 | เปิด `/api-docs/` (fixed) | ✅ `401 ต้องล็อคอิน` |

---

*เอกสารประกอบการนำเสนอ Backend Code MVP — กลุ่ม 1 · โค้ดทดสอบจริงได้ที่หน้า `/mvp` และ `/demo` ของโปรเจกต์ siam-vuln-demo*
