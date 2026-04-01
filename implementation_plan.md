# 🚀 Zenith Chat — FINAL Plan (v2)

## ✅ Stack đã xác nhận

| Layer | Technology | Ghi chú |
|---|---|---|
| **AI** | Gemini Flash + Pro (hybrid) | 95% Flash · 5% Pro · Function Calling |
| **Backend** | Node.js + Fastify + TypeScript | Webhook · AI orchestration · Channel adapters |
| **Database** | Supabase (PostgreSQL) | ACID · Managed · Realtime built-in |
| **Realtime** | Supabase Realtime | Push tới Flutter Dashboard |
| **Auth** | Supabase Auth | Agent login |
| **Storage** | Supabase Storage | Attachments · QR images |
| **Frontend** | Flutter Web | Agent dashboard trên browser |
| **Queue** | BullMQ + Redis | Follow-up delayed jobs · Message buffer |
| **Channels** | FB Messenger · Instagram · Lazada (mock) | Extensible Adapter Pattern |
| **Payment** | VietQR API · Momo API | Tạo QR thanh toán |
| **Bank Hook** | SePay / Casso | Xác nhận chuyển khoản tự động |
| **Ops Notify** | Telegram Bot | Push notification nội bộ cho nhân viên |
| **Shipping** | Ahamove / GHTK | Tạo vận đơn tự động |

---

## 🏗️ System Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                         INPUT CHANNELS                              │
│   📱 FB Messenger      📸 Instagram DM      🛒 Lazada (Mock)        │
└──────────────┬──────────────────┬──────────────────┬───────────────┘
               │                  │                  │
               └──────────────────┴──────────────────┘
                                  │ POST /webhook/:channelId
                                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│                       FASTIFY API LAYER                             │
│  Channel Adapter (normalize) → Message Aggregator (Redis 3-5s)     │
└────────────────────────────────┬────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    ⭐ AI AGENT ENGINE (Hybrid Model)                 │
│            Gemini Flash (95%) · Gemini Pro (5%)            │
│                                                                     │
│  SALES AI ──────────────────────────────────────────▶ CS AI        │
│  (Intent · Inventory · Checkout · Follow-up)    (Post-sale care)   │
│                                                                     │
│  Tools: search_product · check_inventory · confirm_order           │
│         collect_customer_info · create_qr_payment                  │
│         apply_voucher · escalate_to_human                          │
│         schedule_followup · switch_to_cs_mode                      │
│         notify_staff · create_shipping_order                       │
└───────────────────────────────┬─────────────────────────────────────┘
                                │
              ┌─────────────────┼─────────────────┐
              ▼                 ▼                  ▼
    Supabase (PostgreSQL)   BullMQ Worker     Telegram Bot
    + Realtime push         (Follow-up jobs)  (Staff notify)
              │
              ▼
     Flutter Web Dashboard
     (Agent Inbox + Realtime)
```

---

## 🔄 6 Workflows Chi Tiết

### Workflow 1 — Tiếp nhận & Phân tích Intent (Sales AI)

```
Khách nhắn tin (FB/Instagram/Lazada)
        │
        ▼
POST /webhook/:channelId
        │
        ▼
Channel Adapter → NormalizedMessage
        │
        ▼
Redis Buffer (3-5s debounce) ← gom tin nhắn split
        │
        ▼
Load conversation history từ Supabase
        │
        ▼
AI Agent phân loại intent:
   ├── Hỏi thông tin chung → Reply trực tiếp
   ├── Muốn mua hàng       → Kích hoạt Workflow 2
   └── Phàn nàn / tức giận → Kích hoạt Handoff
        │
        ▼
Sinh reply ngắn gọn, chia nhỏ text tự nhiên
        │
        ▼
Gọi Channel API gửi reply về platform
        │
        ▼
INSERT into messages (Supabase) → Realtime push → Dashboard
```

---

### Workflow 2 — Truy vấn Tồn kho & Báo giá

```
Phát hiện intent "muốn mua" từ Workflow 1
        │
        ▼
AI extract entity: Tên sản phẩm
        │
        ▼
Tool: search_product(query)
  └─▶ Supabase: SELECT * FROM products WHERE name ILIKE '%query%'
  └─▶ [Optional] Full-text search description
        │
        ▼
Tool: check_inventory(product_id, variant)
  └─▶ Supabase: SELECT stock FROM products WHERE id = ?
        │
        ▼
IF stock > 0:
   → AI re-confirm: "Dạ còn 2 cái, mình lấy 1 đúng không ạ?"
   → Khách confirmed → Kích hoạt Workflow 3
ELSE:
   → AI xin lỗi + gợi ý sản phẩm tương tự
   → Tool: apply_voucher() nếu cần giữ chân khách
```

> **Note:** Dùng Function Calling (không phải Text-to-SQL trực tiếp) để đảm bảo an toàn. AI gọi tool định nghĩa sẵn, function nội bộ mới chạy SQL parameterized.

---

### Workflow 3 — Thu thập Thông tin & Tạo Hóa đơn (Checkout)

```
Khách confirmed số lượng
        │
        ▼
Tool: collect_customer_info()
AI kiểm tra lịch sử chat, xem đã có biến nào:
   Cần đủ 5 biến: [ Tên · SĐT · Địa chỉ · Ngày giờ giao · Ghi chú ]
        │
        ▼
┌── Còn thiếu biến? ──YES──▶ Hỏi bổ sung → Khách cung cấp ──┐
└──────────────────────────────────────────────────────────────┘
        │ (Đủ 5 biến)
        ▼
Tổng hợp thành JSON:
{
  name, phone, address,
  delivery_at, note,
  items: [{ product_id, qty, price }],
  total
}
        │
        ▼
INSERT into orders (status: 'awaiting_payment')
        │
        ▼
AI gửi hóa đơn tóm tắt cho khách confirm
        │
        ▼
Khách confirmed hóa đơn
        │
        ▼
Tool: create_qr_payment(order_id, amount)
  └─▶ Gọi VietQR API / Momo API
  └─▶ Nhận URL ảnh QR + cú pháp chuyển khoản
        │
        ▼
Gửi text + ảnh QR cho khách
        │
        ▼
UPDATE orders SET status = 'awaiting_payment', qr_code_url = ?
        │
        ▼
Kích hoạt Workflow 4 (lắng nghe bank webhook)
```

---

### Workflow 4 — Xác nhận Thanh toán & Ghi Log

```
SePay / Casso webhook: POST /webhook/payment
  └─▶ { bank_ref, amount, description (cú pháp mã đơn) }
        │
        ▼
Match description với orders.payment_code
        │
        ▼
IF match:
   UPDATE orders SET status = 'paid', paid_at = now()
        │
        ▼
Parallel actions:
   ├── SQL: UPDATE products SET stock = stock - qty (mỗi item)
   ├── Supabase Realtime → Dashboard cập nhật badge "✅ Đã thanh toán"
   ├── Google Sheets API: append row
   │     [ID · Tên · SĐT · Địa chỉ · Ngày giờ · Nội dung · "Đã thanh toán"]
   ├── Inbox API (Pancake): update tag → "Đã mua"
   └── AI reply cho khách: "Em đã nhận được tiền, đang chuẩn bị đơn ạ 😊"
        │
        ▼
Trigger: Switch Sales AI → CS AI (switch_to_cs_mode)
        │
        ▼
Kích hoạt Workflow 5 (Operations)
```

---

### Workflow 5 — Vận hành & Giao hàng (Operations)

```
Order status = 'paid' (trigger từ Workflow 4)
        │
        ▼
Tool: notify_staff(order_id)
  └─▶ Telegram Bot gửi vào group nhân viên:
      "🛒 Đơn mới #ID
       📦 Bánh kem dâu x1
       ⏰ Giao lúc 17:00
       📍 123 Nguyễn Văn A, Q.1
       [✅ Xác nhận hoàn thành]  ← Inline button"
        │
        ▼
Nhân viên làm xong → bấm "Xác nhận hoàn thành"
        │
        ▼
Telegram callback: POST /webhook/telegram/callback
        │
        ▼
Tool: create_shipping_order(order_id)
  └─▶ Gọi Ahamove / GHTK API
  └─▶ Truyền: người gửi, người nhận, địa chỉ, ghi chú
  └─▶ Nhận: tracking_code, tracking_url
        │
        ▼
UPDATE orders SET
  status = 'shipping',
  tracking_code = ?,
  tracking_url = ?
        │
        ▼
CS AI gửi tự động cho khách:
"Dạ bánh của anh/chị đã được gửi đi rồi ạ!
 Theo dõi đơn hàng tại: [link tracking] 🚀"
        │
        ▼
BullMQ: schedule_followup(delay=1440min)
  └─▶ "+24h: Hỏi thăm nhận hàng chưa + xin đánh giá"
```

---

### Workflow 6 — Xử lý Ngoại lệ (Edge Cases)

#### 6a. Follow-up Bám đuổi
```
BullMQ: Cứ 15 phút check tất cả conversations
        │
        ▼
IF (now() - last_msg_at > threshold)
AND conversation.status = 'ai_handling'
AND conversation.tag NOT IN ('Đã mua', 'Chốt'):
        │
        ▼
AI sinh câu follow-up tự nhiên:
  ├── 1h sau: "Dạ không biết anh/chị cần hỗ trợ thêm gì không ạ? 😊"
  ├── 3h sau: "Bên em đang có chương trình freeship hôm nay anh/chị ơi"
  └── 24h sau: Dừng follow-up, mark 'cold_lead'
```

#### 6b. Smart Handoff — Chuyển tay Human
```
Mỗi message đến → AI phân tích sentiment liên tục
        │
        ▼
IF sentiment < -0.7
OR keyword in ['lừa đảo', 'hoàn tiền', 'tệ quá', 'chửi thề', 'tố cáo']
OR unanswered_count > 3:
        │
        ▼
Tool: escalate_to_human(reason, priority)
  ├── AI ngừng tự reply
  ├── conversations.status = 'human_handling'
  ├── Gắn tag "Cần hỗ trợ gấp"
  └── Push notification đến điện thoại nhân viên CS
        │
        ▼
Dashboard: 🔴 Badge đỏ + alert sound
Nhân viên vào chat → tự reply đè lên bot
```

---

## 📁 Project Structure

```
zenith-chat/
├── apps/
│   ├── api/                              # Fastify Backend
│   │   ├── src/
│   │   │   ├── channels/                 # Channel Adapter System
│   │   │   │   ├── types.ts              # NormalizedMessage interface
│   │   │   │   ├── registry.ts           # ChannelRegistry
│   │   │   │   ├── aggregator.ts         # Redis buffer 3-5s
│   │   │   │   └── adapters/
│   │   │   │       ├── facebook.ts
│   │   │   │       ├── instagram.ts
│   │   │   │       └── lazada.ts
│   │   │   ├── ai/
│   │   │   │   ├── agent.ts              # Gemini orchestration + model selector
│   │   │   │   ├── tools.ts              # 10 Function Calling tools
│   │   │   │   ├── handover.ts           # Smart Handover logic
│   │   │   │   ├── checkout.ts           # Collect info state machine
│   │   │   │   └── prompts.ts            # System prompts tiếng Việt
│   │   │   ├── routes/
│   │   │   │   ├── webhook.ts            # POST /webhook/:channelId
│   │   │   │   ├── webhook-payment.ts    # POST /webhook/payment (SePay/Casso)
│   │   │   │   ├── webhook-telegram.ts   # POST /webhook/telegram/callback
│   │   │   │   ├── conversations.ts
│   │   │   │   └── messages.ts
│   │   │   ├── services/
│   │   │   │   ├── vietqr.ts             # VietQR / Momo API
│   │   │   │   ├── shipping.ts           # Ahamove / GHTK API
│   │   │   │   ├── telegram-bot.ts       # Staff notification
│   │   │   │   └── google-sheets.ts      # Log đơn hàng
│   │   │   └── workers/
│   │   │       ├── followup.ts           # BullMQ delayed follow-up
│   │   │       └── payment-check.ts      # Polling fallback nếu webhook fail
│   │   └── supabase/
│   │       ├── migrations/
│   │       └── seed.ts
│   │
│   └── dashboard/                        # Flutter Web
│       ├── lib/
│       │   ├── screens/
│       │   │   ├── login_screen.dart
│       │   │   ├── inbox_screen.dart
│       │   │   ├── conversation_screen.dart
│       │   │   └── order_screen.dart      # [NEW] Quản lý đơn hàng
│       │   ├── widgets/
│       │   │   ├── message_bubble.dart
│       │   │   ├── product_context_panel.dart
│       │   │   ├── channel_badge.dart
│       │   │   └── order_status_badge.dart # [NEW]
│       │   └── services/
│       │       ├── supabase_service.dart
│       │       └── api_service.dart
│       └── pubspec.yaml
│
├── docker-compose.yml                    # Redis only
└── .env.example
```

---

## 🗄️ Supabase Schema (v2)

```sql
-- Customers
create table customers (
  id           uuid primary key default gen_random_uuid(),
  channel_id   text not null,
  platform_id  text not null,
  name         text,
  phone        text,
  avatar_url   text,
  tag          text default 'new',     -- new | interested | purchased | cold_lead
  created_at   timestamptz default now(),
  unique(channel_id, platform_id)
);

-- Conversations
create table conversations (
  id                  uuid primary key default gen_random_uuid(),
  customer_id         uuid references customers(id),
  channel_id          text not null,
  status              text default 'ai_handling',   -- ai_handling | human_handling | closed
  ai_role             text default 'sales',          -- sales | cs
  sentiment           float default 0,
  assigned_to         uuid,
  unanswered_count    int default 0,
  last_msg_at         timestamptz default now(),
  created_at          timestamptz default now()
);

-- Messages
create table messages (
  id              uuid primary key default gen_random_uuid(),
  conversation_id uuid references conversations(id),
  role            text not null,  -- customer | ai | agent
  content         text not null,
  metadata        jsonb,          -- tool_calls, product_context, qr_url
  sent_at         timestamptz default now()
);

-- Products
create table products (
  id          uuid primary key default gen_random_uuid(),
  name        text not null,
  description text,
  price       integer not null,  -- VND
  stock       integer default 0,
  variants    jsonb,
  image_url   text
);

-- Orders (extended v2)
create table orders (
  id                uuid primary key default gen_random_uuid(),
  customer_id       uuid references customers(id),
  conversation_id   uuid references conversations(id),
  status            text default 'pending',
  -- pending | awaiting_payment | paid | shipping | delivered | cancelled
  items             jsonb not null,     -- [{ product_id, name, qty, price }]
  total             integer not null,
  -- Customer info
  recipient_name    text,
  recipient_phone   text,
  recipient_address text,
  delivery_at       timestamptz,
  note              text,
  -- Payment
  payment_code      text unique,        -- Cú pháp chuyển khoản để match
  payment_method    text,               -- vietqr | momo
  qr_code_url       text,
  paid_at           timestamptz,
  -- Shipping
  tracking_code     text,
  tracking_url      text,
  shipped_at        timestamptz,
  created_at        timestamptz default now()
);

-- Enable Realtime
alter publication supabase_realtime add table messages;
alter publication supabase_realtime add table conversations;
alter publication supabase_realtime add table orders;
```

---

## 🤖 AI Function Calling Tools (10 tools)

```typescript
const toolDefinitions = [
  // --- Workflow 1 & 2: Sales ---
  {
    name: 'search_product',
    description: 'Tìm sản phẩm theo tên/mô tả khi khách hỏi.',
    parameters: {
      properties: {
        query: { type: 'string' },
        category: { type: 'string' }
      },
      required: ['query']
    }
  },
  {
    name: 'check_inventory',
    description: 'Kiểm tra tồn kho khi khách hỏi còn hàng không.',
    parameters: {
      properties: {
        product_id: { type: 'string' },
        variant: { type: 'string' }
      },
      required: ['product_id']
    }
  },
  {
    name: 'apply_voucher',
    description: 'Tạo voucher khi khách mặc cả hoặc cần kích cầu.',
    parameters: {
      properties: {
        customer_id: { type: 'string' },
        discount_percent: { type: 'number' }
      },
      required: ['customer_id', 'discount_percent']
    }
  },

  // --- Workflow 3: Checkout ---
  {
    name: 'collect_customer_info',
    description: 'Thu thập đủ 5 biến: Tên, SĐT, Địa chỉ, Ngày giờ, Ghi chú. Scan lịch sử chat trước.',
    parameters: {
      properties: {
        conversation_id: { type: 'string' }
      },
      required: ['conversation_id']
    }
  },
  {
    name: 'confirm_order',
    description: 'Tổng hợp đơn hàng và gửi hóa đơn cho khách confirm.',
    parameters: {
      properties: {
        customer_id: { type: 'string' },
        items: { type: 'array' }
      },
      required: ['customer_id', 'items']
    }
  },
  {
    name: 'create_qr_payment',
    description: 'Tạo mã QR VietQR/Momo sau khi khách confirm hóa đơn.',
    parameters: {
      properties: {
        order_id: { type: 'string' },
        amount: { type: 'number' },
        method: { type: 'string', enum: ['vietqr', 'momo'] }
      },
      required: ['order_id', 'amount']
    }
  },

  // --- Workflow 5: Operations ---
  {
    name: 'notify_staff',
    description: 'Gửi thông báo Telegram cho nhân viên khi có đơn mới đã thanh toán.',
    parameters: {
      properties: {
        order_id: { type: 'string' }
      },
      required: ['order_id']
    }
  },
  {
    name: 'create_shipping_order',
    description: 'Tạo vận đơn Ahamove/GHTK sau khi nhân viên xác nhận hoàn thành.',
    parameters: {
      properties: {
        order_id: { type: 'string' }
      },
      required: ['order_id']
    }
  },

  // --- Workflow 6: Edge Cases ---
  {
    name: 'schedule_followup',
    description: 'Lên lịch nhắn follow-up tự động sau X phút.',
    parameters: {
      properties: {
        conversation_id: { type: 'string' },
        message: { type: 'string' },
        delay_minutes: { type: 'number' }
      },
      required: ['conversation_id', 'message', 'delay_minutes']
    }
  },
  {
    name: 'escalate_to_human',
    description: 'Ngừng AI, gắn tag urgent, ping nhân viên CS khi khách tức giận hoặc vấn đề phức tạp.',
    parameters: {
      properties: {
        reason: { type: 'string' },
        priority: { type: 'string', enum: ['normal', 'urgent'] }
      },
      required: ['reason', 'priority']
    }
  },

  // --- Mode Switch ---
  {
    name: 'switch_to_cs_mode',
    description: 'Chuyển từ Sales AI sang CS AI sau khi đơn đã thanh toán.',
    parameters: {
      properties: {
        order_id: { type: 'string' }
      },
      required: ['order_id']
    }
  }
]
```

---

## 🤖 Hybrid Model Selector

```typescript
// ai/agent.ts
async function selectModel(conversation: Conversation): Promise<string> {
  const needsPro =
    conversation.sentiment < -0.8 ||
    conversation.unanswered_count > 2 ||
    conversation.last_message_length > 500

  return needsPro
    ? 'gemini-2.5-pro-latest'   // ~5% requests — phức tạp, nhạy cảm
    : 'gemini-2.0-flash'        // ~95% requests — nhanh, rẻ
}
```

---

## 📅 3-Week Sprint

### Week 1 — Core Backend + AI

| Day | Task |
|---|---|
| **1-2** | Monorepo setup · Docker Redis · Supabase schema v2 · Fastify boilerplate |
| **3-4** | Channel Adapters (FB · IG · Lazada) · Webhook route · Message Aggregator |
| **5-6** | AI Agent + 10 tools · Hybrid model selector · Conversation history |
| **7** | Supabase Realtime bridge · Seed demo data · Workflow 1 & 2 end-to-end test |

### Week 2 — Checkout + Payment

| Day | Task |
|---|---|
| **8** | Checkout state machine (collect 5 biến) · confirm_order tool |
| **9** | VietQR API integration · Momo API integration · create_qr_payment tool |
| **10** | SePay / Casso webhook · Payment matching logic · Inventory decrement |
| **11** | Google Sheets API logging · Pancake inbox tag update |
| **12** | Telegram Bot notify staff · Inline button callback handler |
| **13** | Ahamove / GHTK shipping API · create_shipping_order tool |
| **14** | CS mode switch · Post-sale follow-up chuỗi 24h/72h |

### Week 3 — Dashboard + Polish

| Day | Task |
|---|---|
| **15-16** | Flutter Web: Login · Inbox · Conversation screen · Realtime |
| **17** | Order management screen · Payment status badge |
| **18** | Smart Handover UI · Sentiment display · Human override reply |
| **19** | Prompt tuning tiếng Việt · Edge case testing |
| **20** | Deploy API (Railway) · Flutter Web build · Demo rehearsal |

---

## 🎬 Demo Flow (đầy đủ 6 workflows)

```
1. [WF1] Khách nhắn FB: "shop ơi" / "bánh kem dâu còn không"
   → AI gom tin 4s → phân loại intent → reply tự nhiên

2. [WF2] AI search + check tồn kho:
   → "Dạ còn 2 cái, mình lấy 1 đúng không ạ?"
   → Khách: "ừ lấy 1"

3. [WF3] AI thu thập thông tin:
   → Hỏi tên, SĐT, địa chỉ, giờ giao
   → Gửi hóa đơn confirm
   → Tạo QR VietQR → gửi ảnh QR cho khách

4. [WF4] Khách chuyển khoản:
   → SePay webhook → match mã đơn → "Em nhận được tiền rồi ạ 😊"
   → Google Sheets tự ghi · Inventory -1 · Tag "Đã mua"

5. [WF5] Telegram nhân viên:
   → "🛒 Đơn mới: Bánh kem dâu · 17:00 · Q.1"
   → Nhân viên bấm ✅ → Ahamove tự tạo đơn vận chuyển
   → CS AI gửi tracking link cho khách

6. [WF6] Demo edge cases:
   → Nhắn câu tức giận → 🔴 Urgent · Nhân viên nhận ping ngay
   → Bỏ chat 1h → AI tự follow-up nhẹ nhàng
```

---

## 🔑 Environment Variables

```bash
# AI
GEMINI_API_KEY=

# Supabase
SUPABASE_URL=
SUPABASE_ANON_KEY=
SUPABASE_SERVICE_ROLE_KEY=

# Meta (Facebook + Instagram)
META_APP_ID=
META_APP_SECRET=
META_VERIFY_TOKEN=
META_PAGE_ACCESS_TOKEN=

# Redis
REDIS_URL=redis://localhost:6379

# Payment
VIETQR_API_KEY=
VIETQR_BANK_ACCOUNT=
MOMO_PARTNER_CODE=
MOMO_ACCESS_KEY=
MOMO_SECRET_KEY=

# Bank Webhook
SEPAY_WEBHOOK_SECRET=
# hoặc
CASSO_API_KEY=

# Google Sheets
GOOGLE_SHEETS_ID=
GOOGLE_SERVICE_ACCOUNT_JSON=

# Telegram (Staff Bot)
TELEGRAM_BOT_TOKEN=
TELEGRAM_STAFF_CHAT_ID=

# Shipping
AHAMOVE_API_KEY=
# hoặc
GHTK_TOKEN=
```

---

## 🚀 Start Commands

```bash
# Setup
git clone ... && cd zenith-chat && pnpm install

# Start Redis
docker-compose up -d

# Start API
cd apps/api && pnpm dev

# Start Flutter Web
cd apps/dashboard && flutter run -d chrome
```
