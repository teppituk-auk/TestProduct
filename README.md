# SlipFolio — Crypto Slip Tracker

Web application สำหรับอัพโหลดสลิปการซื้อขาย Bitcoin และเหรียญ Crypto จาก Bitkub, Binance TH และ Binance พร้อมระบบ Portfolio Tracking และ Subscription ผ่าน Lightning Network

---

## Features

### Feature 0 — Admin Panel
- แยก User ระหว่าง Admin และ Customer ใช้หน้า Login เดียวกัน
- Admin ดู Dashboard ภาพรวมทั้งหมดได้
- Admin Activate / Inactive ลูกค้าได้
- **User Online/Offline Status** — Admin เห็น real-time status ของทุก User (heartbeat ทุก 30 วินาที)
- **Role Management** — สร้าง Role ใหม่ กำหนดได้ว่า Role นั้นเข้าถึงเมนูไหนได้บ้าง
- **User Management** — กำหนด Role ให้ User แต่ละคนได้ รองรับทั้ง System Role และ Custom Role

### Feature 1 — Login
- สมัครสมาชิกและเข้าสู่ระบบด้วย Email / Password
- รองรับ OAuth (Google) ผ่าน NextAuth.js
- JWT-based session management พร้อม Role และ Permission ใน token

### Feature 2 — Reset Password
- รีเซ็ตรหัสผ่านผ่าน Email Link (Magic Link)
- Token หมดอายุใน 15 นาที

### Feature 3 — Dashboard
- แสดง Portfolio แยกรายเหรียญ และรวมทุกเหรียญ
- สกุลเงินหลัก: THB / USD (เลือกได้)
- ราคาปัจจุบันดึงจาก Bitkub API / Binance API / CoinGecko API
- แสดง Unrealized P&L, Cost Basis, Portfolio Allocation

### Feature 4 — Buy/Sell Chart
- กราฟแสดงจุดซื้อ/ขายบน Timeline (แกน X = วันที่)
- กราฟราคา Candlestick / Line เป็น background
- ใช้ TradingView Lightweight Charts
- กรองตามเหรียญ, Exchange, ช่วงวันที่

### Feature 5 — Upload Slip
- รองรับไฟล์ JPG, PNG, WebP
- **AI OCR อัตโนมัติ**: ใช้ Gemini 2.0 Flash เป็นหลัก → fallback ไป Claude Sonnet 4.6 อัตโนมัติเมื่อ quota หมด
- รองรับ format ของ Bitkub, Binance TH, Binance และหุ้นไทย (SET/MAI)
- ผู้ใช้แก้ไขข้อมูลได้ก่อน Confirm (Human-in-the-loop)
- 1 สลิป = 1 Transaction

### Feature 7 — Share Portfolio
- สร้าง Share Link แบบ Private (มี token)
- เลือกแสดง/ซ่อนข้อมูลทุน และกำไรได้
- แชร์แบบ Real-time หรือ Snapshot
- ตั้ง Expire Date ของ Link ได้

### Feature 8 — Subscription
- Free Tier: จำกัดจำนวน Transaction / เดือน
- Paid Tier: Unlimited Transactions + ฟีเจอร์ครบ
- รับชำระด้วย Lightning Network ผ่าน BTCPay Server (self-hosted)
- ยืนยันการชำระเงินอัตโนมัติผ่าน Webhook
- เมื่อหมด Subscription: ข้อมูลยังคงอยู่ แต่ Lock ฟีเจอร์ Paid

### Feature 9 — High Concurrency
- รองรับผู้ใช้งาน **100,000 Concurrent Users**
- Horizontal Scaling ด้วย Kubernetes / AWS ECS
- CDN + Edge Caching สำหรับ Static Assets
- Database Connection Pooling (PgBouncer)
- Queue-based OCR Processing (Redis + BullMQ)

### Feature 10 — Multi-language
- รองรับภาษาไทย และ English สามารถเปลี่ยนภาษาได้ตลอด

---

## Tech Stack

| Layer | Technology |
|---|---|
| Frontend | Next.js 15 (App Router), Tailwind CSS, shadcn/ui |
| Backend | Next.js API Routes |
| Database | PostgreSQL + Prisma ORM |
| AI/OCR | Google Gemini 2.0 Flash (primary) + Claude Sonnet 4.6 (fallback) |
| Auth | NextAuth.js (JWT strategy) |
| Charts | TradingView Lightweight Charts |
| Lightning | BTCPay Server (self-hosted) หรือ OpenNode (managed) |
| Storage | AWS S3 / Cloudflare R2 (เก็บรูปสลิป) |
| Queue | Redis + BullMQ (OCR job queue) |
| Deploy | Vercel (Frontend) + Railway / Render (Backend) |

---

## Data Model

```
User
  id, email, password, name, role, isActive, plan, planExpiresAt,
  lastSeenAt,   ← online/offline tracking
  createdAt

Role
  id, name (unique), label, permissions (JSON), isSystem, createdAt
  permissions: { dashboard, upload, chart, share, subscription, settings }

Coin
  id, symbol (BTC, ETH, PTT, ...), name, assetType (CRYPTO|STOCK), coingeckoId

Transaction
  id, user_id, coin_id, type (BUY/SELL),
  amount, price, total_value, currency (THB/USDT/USD),
  exchange (bitkub/binanceth/binance/set/nyse/nasdaq/...),
  tx_date, slip_image_url, created_at

Portfolio
  คำนวณ real-time จาก transactions (ไม่ store แยก)

ShareLink
  id, user_id, token, config (JSON), expires_at, created_at

Subscription
  id, user_id, invoice_id, amount_sats, status, paid_at
```

---

## Deployment

### AWS Cloud Architecture
```
Route 53 → CloudFront → ALB
                         ├── ECS (Next.js App)
                         └── RDS PostgreSQL (Multi-AZ)

S3 / Cloudflare R2    → Slip Image Storage
ElastiCache Redis      → Job Queue + Session Cache
BTCPay Server (EC2)   → Lightning Network Payments
```

### Environment Variables
```env
# Database
DATABASE_URL=

# NextAuth
NEXTAUTH_SECRET=
NEXTAUTH_URL=

# Google OAuth (optional)
GOOGLE_CLIENT_ID=
GOOGLE_CLIENT_SECRET=

# AI / OCR
GEMINI_API_KEY=          # primary OCR (aistudio.google.com)
ANTHROPIC_API_KEY=       # fallback OCR เมื่อ Gemini quota หมด (console.anthropic.com)

# Storage
AWS_ACCESS_KEY_ID=
AWS_SECRET_ACCESS_KEY=
AWS_REGION=
S3_BUCKET_NAME=

# Queue
REDIS_URL=

# Lightning Payment
BTCPAY_URL=
BTCPAY_API_KEY=
BTCPAY_STORE_ID=
BTCPAY_WEBHOOK_SECRET=

# Email (Reset Password)
SMTP_HOST=
SMTP_PORT=
SMTP_USER=
SMTP_PASSWORD=
EMAIL_FROM=
```

---

## Getting Started

```bash
# Install dependencies
npm install

# Setup database
npx prisma migrate dev

# Seed system roles (ADMIN, CUSTOMER)
psql $DATABASE_URL -c "
INSERT INTO \"Role\" (id, name, label, permissions, \"isSystem\", \"createdAt\")
VALUES
  (gen_random_uuid()::text, 'ADMIN', 'Administrator', '{\"dashboard\":true,\"upload\":true,\"chart\":true,\"share\":true,\"subscription\":true,\"settings\":true}', true, now()),
  (gen_random_uuid()::text, 'CUSTOMER', 'Customer', '{\"dashboard\":true,\"upload\":true,\"chart\":true,\"share\":true,\"subscription\":true,\"settings\":true}', true, now())
ON CONFLICT (name) DO NOTHING;
"

# Run development server
npm run dev
```

---

## Project Structure

```
/
├── app/
│   ├── (auth)/                   # Login, Register, Reset Password
│   ├── admin/
│   │   ├── page.tsx              # Admin Dashboard Overview
│   │   ├── users/page.tsx        # User Management + Online Status
│   │   └── roles/page.tsx        # Role Management + Permission Editor
│   ├── dashboard/                # Portfolio Dashboard
│   ├── upload/                   # Slip Upload + OCR
│   ├── chart/                    # Buy/Sell Chart
│   ├── share/[token]/            # Public Share Portfolio
│   └── settings/                 # Profile, Share, Subscription
├── components/
│   ├── nav.tsx                   # Sidebar (กรองเมนูตาม role permissions)
│   ├── admin-nav.tsx             # Admin Sidebar
│   ├── heartbeat-provider.tsx    # Online/Offline tracking
│   └── slip-upload-form.tsx      # Upload + OCR form
├── lib/
│   ├── auth.ts                   # NextAuth config + JWT (role + permissions)
│   ├── gemini.ts                 # Gemini Vision OCR
│   └── anthropic.ts              # Claude Vision OCR (fallback)
├── prisma/
│   └── schema.prisma             # DB schema
└── types/
    └── next-auth.d.ts            # Session type extensions
```

---

## License

Private — All rights reserved.
