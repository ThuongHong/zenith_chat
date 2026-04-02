# 🚀 Zenith Chat — FINAL Plan (v3)

## ✅ Stack đã xác nhận

| Layer | Technology | Ghi chú |
|---|---|---|
| **AI** | Gemini 2.0 Flash + 2.5 Pro (hybrid) | 95% Flash · 5% Pro · Function Calling |
| **Backend** | Node.js + Fastify + TypeScript | Webhook · AI orchestration · Channel adapters |
| **Database** | Supabase (PostgreSQL) | ACID · Managed · Realtime built-in |
| **Realtime** | Supabase Realtime | Push tới Flutter Dashboard |
| **Auth** | Supabase Auth | Agent login |
| **Storage** | Supabase Storage | Attachments · QR images · Product images |
| **Frontend** | Flutter Web | Agent dashboard trên browser |
| **Queue** | BullMQ + Redis | Follow-up delayed jobs · Message buffer |
| **Channels** | FB Messenger · Instagram · TikTok Shop · Shopee | Extensible Adapter Pattern |
| **Payment** | VietQR API · Momo API | Tạo QR thanh toán chính xác đến từng đồng |
| **Bank Hook** | SePay / Casso | Xác nhận chuyển khoản · idempotency check |
| **Ops Notify** | Telegram Bot / Zalo OA | Push notification nội bộ cho nhân viên |
| **Shipping** | Ahamove · Viettel Post | Tạo vận đơn · tracking tự động |

---

## 🤖 AI Persona

```
System prompt (warm tone):
"Bạn là trợ lý bán hàng thân thiện của shop. 
Luôn xưng 'em', dùng tone ấm áp, tự nhiên như nhân viên thật.
Trả lời ngắn gọn, chia nhỏ tin nhắn. Không viết wall of text."

Few-shot examples:
User: "mẫu này còn không"
AI:   "Dạ còn ạ! Mẫu này em còn 3 cái size M và 2 cái size L nè 😊
       Anh/chị muốn lấy size nào ạ?"

User: "hết rồi à buồn quá"
AI:   "Dạ em xin lỗi anh/chị nha 🙏 
       Mẫu này hết rồi nhưng em có mẫu tương tự nè, 
       anh/chị xem thử không ạ? [gửi ảnh gợi ý]"
```

---

## 🏷️ Hệ thống Tag Hội thoại

Mỗi conversation được gắn tag tự động dựa trên trạng thái:

| Tag | Ý nghĩa | AI Mode |
|-----|---------|---------|
| `SA` | Sales Agent — đang tư vấn, bán hàng | Sales AI active |
| `CA` | Customer Service Agent — sau mua | CS AI active |
| `CSKH` | Cần chăm sóc khách hàng khẩn cấp | Human Handoff |
| `Đã mua` | Đã thanh toán thành công | CS AI + follow-up |
| `Cold lead` | Bỏ chat, chưa mua | Follow-up queue |
| `Invoice expired` | Hết giờ thanh toán | Trả về WF1 |

---

## 🔄 Workflow 1 — Tiếp nhận, Phân tích Intent & Kiểm tra Tồn kho

```
Khách nhắn tin (FB Messenger / TikTok Shop / Shopee)
        │
        ▼
POST /webhook/:channelId
Channel Adapter → NormalizedMessage
        │
        ▼
Redis Buffer (3-5s debounce) ← gom tin nhắn split
        │
        ▼
Load conversation history từ Supabase
        │
        ▼
AI Agent phân loại & gắn tag:
   ├── Tag SA (Sales) ──────────────────▶ Subflow 1A
   └── Tag CA (Post-sale / CS)  ────────▶ Subflow 1B
```

### Subflow 1A — Sales Agent (SA)

```
Nhận input: ảnh sản phẩm hoặc câu hỏi text
        │
        ├── [Có ảnh] Image Recognition
        │         └─▶ Gemini Vision: nhận diện mẫu mã trong database
        │         └─▶ Map tới product_id chính xác
        │
        ├── [Có text] Entity Extraction
        │         └─▶ Tách tên sản phẩm, màu, size từ ngôn ngữ tự nhiên
        │         └─▶ Map tới product_id
        │
        ▼
Tool: search_product(query / product_id)
  └─▶ SELECT * FROM products WHERE ... (parameterized)
        │
        ▼
Tool: check_inventory(product_id, variant)
  └─▶ SELECT stock FROM products WHERE id = ?
        │
        ├── [stock > 0]
        │     AI báo giá + xác nhận size/màu/số lượng
        │     Tone: "Dạ còn ạ! ..." → thuyết phục mua
        │     │
        │     ├── [Khách xác nhận mua] ──▶ Phát event → Workflow 3
        │     └── [Khách im lặng]      ──▶ BullMQ: follow-up sau 2h
        │
        └── [stock = 0]
              AI xin lỗi + báo hết hàng
              Tool: search_similar_products(product_id)
              Gợi ý 2-3 mẫu tương tự
              └─▶ Khách quan tâm mẫu mới → lặp lại Subflow 1A

--- Liên tục trong quá trình chat ---
Sentiment Analysis (mỗi message):
  └── score < -0.5 ──▶ escalate_to_human() → Tag CSKH
```

### Subflow 1B — Customer Service Agent (CA)

```
Phát hiện context: đơn đã mua / câu hỏi sau mua
        │
        ├── [Khách hài lòng]
        │     AI cảm ơn tự nhiên
        │     Tool: request_review(conversation_id)
        │     → "Anh/chị có thể để lại đánh giá giúp em được không ạ? 🙏"
        │
        └── [Khách không hài lòng]
              Sentiment scoring
              └── score thấp ──▶ escalate_to_human() → Tag CSKH
              └── score trung bình ──▶ AI xử lý, đề xuất hỗ trợ
```

### Lưu trữ & Học theo thời gian

```
Tất cả hội thoại → INSERT into messages (Supabase)
        │
        ▼
Sau mỗi conversation kết thúc (closed):
  IF conversation.status = 'closed'
  AND order.status = 'paid':
      → Đánh dấu là "success_conversation"
      → Lưu vào bảng success_patterns:
        { conversation_id, key_phrases, tool_sequence, sentiment_curve }
        
  → Dùng để:
    1. Fine-tune few-shot examples trong system prompt
    2. Phân tích pattern bán hàng thành công
    3. Cải thiện AI theo thời gian (không cần retrain)
```

---

## 🔄 Workflow 3 — Thu thập Thông tin & Chốt Hóa đơn

```
Event: Khách xác nhận mua hàng
        │
        ▼
Tool: lookup_customer(platform_id, channel_id)
  └─▶ SELECT * FROM customers WHERE platform_id = ? AND channel_id = ?
        │
        ├── [Khách cũ] Load profile: tên, SĐT, địa chỉ lần trước
        │     AI hỏi confirm: "Giao về [địa chỉ cũ] như lần trước đúng không ạ?"
        │
        └── [Khách mới] Cần thu thập đủ 5 biến
        │
        ▼
Tool: collect_customer_info(conversation_id)
  Scan lịch sử chat → điền sẵn biến nào đã có
  Hỏi bổ sung từng biến còn thiếu (loop):

  REQUIRED_FIELDS = [
    { key: 'name',        label: 'Tên người nhận' },
    { key: 'phone',       label: 'Số điện thoại'  },
    { key: 'address',     label: 'Địa chỉ giao hàng' },
    { key: 'delivery_at', label: 'Thời gian giao (ngày/giờ)' },
    { key: 'note',        label: 'Ghi chú thêm (màu, size...)' }
  ]

  └── Đủ 5 biến? ──NO──▶ Hỏi tiếp (max 3 lượt/biến)
  └── Đủ 5 biến? ──YES──▶ Tiếp tục
        │
        ▼
AI xác nhận lại địa chỉ giao hàng một lần nữa
→ "Giao đến [địa chỉ], [giờ], đúng không ạ?"
        │
        ▼
INSERT into orders (status: 'awaiting_payment')
  payment_code = generateCode(order_id)  ← cú pháp chuyển khoản duy nhất
  expires_at   = now() + 2 hours
        │
        ▼
Tool: create_qr_payment(order_id, amount, method)
  └─▶ VietQR API / Momo API
  └─▶ Nhận QR image URL + payment_code
        │
        ▼
AI gửi Phiếu xác nhận:
  "📋 ĐƠN HÀNG #[ID]
   Sản phẩm: ...
   Tổng tiền: ...VND
   Giao đến: [địa chỉ] lúc [giờ]
   
   Chuyển khoản theo QR bên dưới 👇
   Cú pháp: [payment_code]
   
   ⏰ QR có hiệu lực trong 2 giờ"
  + [Ảnh QR]
        │
        ▼
BullMQ: Job expires_at = now() + 2h
  └── Nếu order.status vẫn = 'awaiting_payment' sau 2h:
        UPDATE orders SET status = 'expired'
        AI nhắn: "Đơn hàng của anh/chị đã hết hạn thanh toán.
                  Anh/chị có muốn đặt lại không ạ?"
        → Trả về Workflow 1 nếu khách muốn tiếp tục
```

---

## 🔄 Workflow 4 — Xác nhận Thanh toán & Đổi Trạng thái

```
SePay / Casso webhook:
POST /webhook/payment
{
  bank_transaction_id: "TXN_123456",  ← ID duy nhất từ ngân hàng
  amount: 315000,
  description: "ZENITH ORDER-789",   ← chứa payment_code
  timestamp: "2025-04-01T10:30:00Z"
}
```

```
        │
        ▼
BƯỚC 1 — Idempotency Check (tránh xử lý trùng)
  SELECT id FROM payment_logs 
  WHERE bank_transaction_id = 'TXN_123456'
        │
        ├── [Đã tồn tại] ──▶ 200 OK, bỏ qua (skip silently)
        │                    Tránh double-process cùng 1 giao dịch
        │
        └── [Chưa tồn tại] ──▶ Tiếp tục xử lý
        │
        ▼
BƯỚC 2 — Match payment_code
  SELECT * FROM orders 
  WHERE payment_code = extract_code(description)
  AND status = 'awaiting_payment'
        │
        ├── [Không match] ──▶ Log warning, notify admin, skip
        │
        └── [Match] ──▶ Tiếp tục
        │
        ▼
BƯỚC 3 — Verify amount
  IF payment.amount < order.total:
      → Báo thiếu tiền, yêu cầu chuyển bổ sung
  IF payment.amount >= order.total:
      → Xác nhận hợp lệ (chênh lệch dư sẽ note)
        │
        ▼
BƯỚC 4 — Atomic transaction (Supabase):
  BEGIN;
    INSERT INTO payment_logs (bank_transaction_id, order_id, amount, ...)
    UPDATE orders SET status = 'paid', paid_at = now()
    UPDATE products SET stock = stock - qty  ← Trừ tồn kho
    UPDATE customers SET tag = 'Đã mua'
    UPDATE conversations SET ai_role = 'cs', status = 'ai_handling'
  COMMIT;
        │
        ▼
BƯỚC 5 — AI nhắn tin xác nhận ngay lập tức:
  "✅ Em đã nhận được [số tiền]đ rồi ạ!
   Đơn hàng của anh/chị đang được chuẩn bị.
   Em sẽ báo lại khi hàng được giao nha 😊"
        │
        ▼
BƯỚC 6 — Cập nhật hệ thống (song song):
  ├── Google Sheets: append row
  │     [ID · Tên · SĐT · Địa chỉ · Ngày giờ · SP · Tiền · Trạng thái]
  │     → Phục vụ báo cáo doanh thu + kê khai thuế
  │
  ├── Hóa đơn điện tử:
  │     → Gọi API VNPT / Viettel để xuất e-invoice
  │     → Lưu URL hóa đơn vào orders.invoice_url
  │
  └── Inbox tag update:
        → Gọi Pancake/Chatbot API: set tag "Khách đã mua"
        │
        ▼
BƯỚC 7 — Switch AI mode:
  Sales AI ──▶ CS AI (switch_to_cs_mode)
        │
        ▼
Kích hoạt Workflow 5 (Operations)
```

### Bảng `payment_logs` (idempotency table)

```sql
create table payment_logs (
  id                   uuid primary key default gen_random_uuid(),
  bank_transaction_id  text unique not null,  -- KEY: tránh duplicate
  order_id             uuid references orders(id),
  amount               integer not null,
  bank_description     text,
  processed_at         timestamptz default now()
);
```

---

## 🔄 Workflow 5 — Vận hành & Giao hàng

```
Trigger: orders.status = 'paid'
        │
        ▼
Tool: notify_staff(order_id)
  └─▶ Telegram Bot / Zalo OA gửi vào group nội bộ:
      "🛒 ĐƠN MỚI #[ID]
       📦 [Tên SP · Số lượng · Màu · Size]
       👤 [Tên khách · SĐT]
       📍 [Địa chỉ giao]
       ⏰ Giao lúc: [giờ]
       📝 Ghi chú: [note]
       
       [✅ Đã đóng gói xong]"  ← Inline button
        │
        ▼
Nhân viên kho nhặt hàng, đóng gói
        │
        ▼
Nhân viên bấm "✅ Đã đóng gói xong"
        │
        ▼
Telegram callback: POST /webhook/telegram/callback
  └─▶ UPDATE orders SET status = 'packed'
        │
        ▼
Tool: create_shipping_order(order_id)
  └─▶ Ahamove / Viettel Post API:
      { sender, recipient, address, items, note }
        │
        ├── [Thành công]
        │     Nhận: tracking_code, tracking_url
        │     UPDATE orders SET status = 'shipping', tracking_code = ?
        │     │
        │     ▼
        │     CS AI nhắn khách:
        │     "🚀 Hàng của anh/chị đã được gửi đi rồi ạ!
        │      Mã vận đơn: [code]
        │      Theo dõi tại: [link] 🗺️"
        │     │
        │     ▼
        │     BullMQ: schedule_followup(delay=1440min)
        │     → "+24h: Hỏi thăm nhận hàng chưa + xin đánh giá"
        │
        └── [API lỗi / địa chỉ sai]
              Notify nhân viên kho:
              "⚠️ Không tạo được vận đơn: [lý do]
               [🔄 Chọn hãng khác] [📞 Liên hệ khách xác minh địa chỉ]"
              → Nhân viên tự xử lý thủ công
```

---

## 🔄 Workflow 6 — Xử lý Ngoại lệ

### 6A. Follow-up Bám đuổi

```
BullMQ: Job scheduled sau mỗi conversation SA kết thúc không thành công
        │
        ▼
IF (now() - last_msg_at > 2h)
AND conversation.tag = 'SA'
AND customer.tag NOT IN ('Đã mua', 'Invoice expired'):
        │
        ▼
Lần 1 (+2h): AI hỏi thăm nhẹ nhàng
  "Dạ anh/chị cần em hỗ trợ thêm gì không ạ? 😊"

Lần 2 (+24h): AI gửi kích thích
  "Hôm nay bên em có freeship đặc biệt anh/chị ơi!
   Mẫu hôm qua anh/chị hỏi vẫn còn, anh/chị muốn lấy không ạ?"
  + Tool: apply_voucher(freeship/discount)

Lần 3 (+72h): Mark 'cold_lead', dừng follow-up
```

### 6B. Human Handoff — Cứu hộ khẩn cấp

```
Continuous sentiment analysis mỗi message
        │
        ▼
IF sentiment < -0.7
OR keyword_match(['lừa đảo', 'hoàn tiền', 'tệ quá', 'báo công an',
                  'đánh giá 1 sao', 'bảo hành phức tạp'])
OR unanswered_count > 3:
        │
        ▼
Tool: escalate_to_human(reason, priority='urgent')
  ├── AI NGỪNG phản hồi ngay lập tức
  ├── UPDATE conversations SET status = 'human_handling'
  ├── UPDATE conversations SET tag = 'CSKH'
  └── Push notification khẩn đến điện thoại nhân viên CS:
      "🔴 KHÁCH CẦN HỖ TRỢ GẤP
       Khách: [tên] · [kênh]
       Lý do: [reason]
       [👉 Vào xử lý ngay]"
        │
        ▼
Dashboard Flutter: Badge 🔴 + alert sound
Nhân viên vào đọc lịch sử → tự reply đè bot
```

---

## 📁 Project Structure

```
zenith-chat/
├── apps/
│   ├── api/
│   │   ├── src/
│   │   │   ├── channels/
│   │   │   │   ├── types.ts              # NormalizedMessage
│   │   │   │   ├── registry.ts
│   │   │   │   ├── aggregator.ts         # Redis buffer 3-5s
│   │   │   │   └── adapters/
│   │   │   │       ├── facebook.ts
│   │   │   │       ├── instagram.ts
│   │   │   │       ├── tiktok.ts
│   │   │   │       └── shopee.ts
│   │   │   ├── ai/
│   │   │   │   ├── agent.ts              # Orchestration + model selector
│   │   │   │   ├── tagger.ts             # SA/CA/CSKH tag logic
│   │   │   │   ├── tools.ts              # 12 Function Calling tools
│   │   │   │   ├── sentiment.ts          # Sentiment scoring
│   │   │   │   ├── vision.ts             # Image → product_id (Gemini Vision)
│   │   │   │   ├── checkout.ts           # Collect 5 fields state machine
│   │   │   │   ├── handover.ts           # Escalation logic
│   │   │   │   └── prompts.ts            # System prompt + few-shot
│   │   │   ├── routes/
│   │   │   │   ├── webhook.ts            # POST /webhook/:channelId
│   │   │   │   ├── webhook-payment.ts    # POST /webhook/payment
│   │   │   │   └── webhook-telegram.ts   # POST /webhook/telegram/callback
│   │   │   ├── services/
│   │   │   │   ├── vietqr.ts
│   │   │   │   ├── shipping.ts           # Ahamove + Viettel Post
│   │   │   │   ├── telegram-bot.ts
│   │   │   │   ├── e-invoice.ts          # VNPT / Viettel e-invoice
│   │   │   │   └── google-sheets.ts
│   │   │   └── workers/
│   │   │       ├── followup.ts           # BullMQ follow-up
│   │   │       └── invoice-expiry.ts     # 2h timeout checker
│   │   └── supabase/
│   │       └── migrations/
│   │
│   └── dashboard/                        # Flutter Web
│       └── lib/
│           ├── screens/
│           │   ├── inbox_screen.dart
│           │   ├── conversation_screen.dart
│           │   └── order_screen.dart
│           └── widgets/
│               ├── message_bubble.dart
│               ├── sentiment_badge.dart
│               ├── tag_chip.dart          # SA / CA / CSKH badge
│               └── order_status_card.dart
│
├── docker-compose.yml
└── .env.example
```

---

## 🗄️ Supabase Schema (v3)

```sql
-- Customers
create table customers (
  id           uuid primary key default gen_random_uuid(),
  channel_id   text not null,
  platform_id  text not null,
  name         text,
  phone        text,
  tag          text default 'new',  -- new | interested | purchased | cold_lead
  created_at   timestamptz default now(),
  unique(channel_id, platform_id)
);

-- Conversations
create table conversations (
  id                uuid primary key default gen_random_uuid(),
  customer_id       uuid references customers(id),
  channel_id        text not null,
  status            text default 'ai_handling',  -- ai_handling | human_handling | closed
  ai_role           text default 'sales',         -- sales (SA) | cs (CA)
  tag               text default 'SA',            -- SA | CA | CSKH | Đã mua | Cold lead
  sentiment         float default 0,
  unanswered_count  int default 0,
  assigned_to       uuid,
  last_msg_at       timestamptz default now(),
  created_at        timestamptz default now()
);

-- Messages
create table messages (
  id              uuid primary key default gen_random_uuid(),
  conversation_id uuid references conversations(id),
  role            text not null,  -- customer | ai | agent
  content         text not null,
  metadata        jsonb,          -- tool_calls, product_context, image_url
  sent_at         timestamptz default now()
);

-- Products
create table products (
  id          uuid primary key default gen_random_uuid(),
  name        text not null,
  description text,
  price       integer not null,
  stock       integer default 0,
  variants    jsonb,              -- { sizes: [], colors: [] }
  image_url   text,
  image_embedding vector(768)    -- [Optional] cho image search
);

-- Orders (v3)
create table orders (
  id                uuid primary key default gen_random_uuid(),
  customer_id       uuid references customers(id),
  conversation_id   uuid references conversations(id),
  status            text default 'pending',
  -- pending | awaiting_payment | paid | packed | shipping | delivered | expired | cancelled
  items             jsonb not null,
  total             integer not null,
  recipient_name    text,
  recipient_phone   text,
  recipient_address text,
  delivery_at       timestamptz,
  note              text,
  payment_code      text unique,
  payment_method    text,
  qr_code_url       text,
  expires_at        timestamptz,   -- QR hết hạn sau 2h
  paid_at           timestamptz,
  invoice_url       text,          -- Hóa đơn điện tử URL
  tracking_code     text,
  tracking_url      text,
  shipped_at        timestamptz,
  created_at        timestamptz default now()
);

-- Idempotency: tránh xử lý trùng bank transaction
create table payment_logs (
  id                   uuid primary key default gen_random_uuid(),
  bank_transaction_id  text unique not null,  -- KEY constraint
  order_id             uuid references orders(id),
  amount               integer not null,
  bank_description     text,
  processed_at         timestamptz default now()
);

-- Success patterns (AI tự học)
create table success_patterns (
  id              uuid primary key default gen_random_uuid(),
  conversation_id uuid references conversations(id),
  key_phrases     text[],         -- Câu nói khách dùng khi sắp mua
  tool_sequence   text[],         -- Thứ tự tools AI đã dùng
  sentiment_curve float[],        -- Biến thiên cảm xúc theo thời gian
  duration_mins   int,            -- Thời gian từ lúc hỏi đến chốt
  created_at      timestamptz default now()
);

-- Enable Realtime
alter publication supabase_realtime add table messages;
alter publication supabase_realtime add table conversations;
alter publication supabase_realtime add table orders;
```

---

## 🤖 AI Function Calling Tools (12 tools)

```typescript
const tools = [
  // WF1 — Sales & Inventory
  'search_product',         // Tìm SP theo text
  'match_product_image',    // Gemini Vision: ảnh → product_id
  'check_inventory',        // Kiểm tra tồn kho
  'search_similar',         // Gợi ý SP tương tự khi hết hàng
  'apply_voucher',          // Tạo voucher kích cầu

  // WF3 — Checkout
  'lookup_customer',        // Tìm KH cũ trong DB
  'collect_customer_info',  // Thu thập 5 biến bắt buộc
  'confirm_order',          // Tổng hợp + gửi hóa đơn
  'create_qr_payment',      // Tạo QR VietQR / Momo

  // WF5 — Operations
  'notify_staff',           // Ping Telegram/Zalo nhân viên
  'create_shipping_order',  // Ahamove / Viettel Post

  // WF6 — Edge Cases
  'schedule_followup',      // BullMQ delayed follow-up
  'escalate_to_human',      // Human handoff + ping CSKH
]
```

---

## 📅 3-Week Sprint

### Week 1 — Core Backend + AI + WF1

| Day | Task |
|---|---|
| **1-2** | Monorepo · Docker Redis · Supabase schema v3 · Fastify boilerplate |
| **3** | Channel Adapters (FB · IG · TikTok · Shopee) · Webhook route |
| **4** | Message Aggregator (Redis 3-5s) · SA/CA tagger |
| **5** | Gemini Vision (match_product_image) · search_product · check_inventory |
| **6** | Sentiment analysis · escalate_to_human · search_similar |
| **7** | System prompt + few-shot · WF1 end-to-end test |

### Week 2 — Checkout + Payment + Operations

| Day | Task |
|---|---|
| **8** | lookup_customer · collect_customer_info (5-field state machine) |
| **9** | VietQR / Momo API · create_qr_payment · Invoice expiry BullMQ job |
| **10** | SePay/Casso webhook · Idempotency check (payment_logs) · verify amount |
| **11** | Atomic DB transaction WF4 · Google Sheets API · e-invoice API |
| **12** | Telegram Bot notify_staff · Inline button callback |
| **13** | Ahamove / Viettel Post API · Error handling khi shipping fail |
| **14** | Success pattern logging · WF3→4→5 end-to-end test |

### Week 3 — Dashboard + Polish

| Day | Task |
|---|---|
| **15-16** | Flutter Web: Login · Inbox · Conversation (Realtime) |
| **17** | SA/CA/CSKH tag badges · Order status screen |
| **18** | Human override reply · Sentiment display · Alert sound |
| **19** | Follow-up worker WF6 · Prompt tuning tiếng Việt |
| **20** | Deploy Railway · Flutter Web build · Demo rehearsal |

---

## 🎬 Demo Flow (đầy đủ)

```
1. [WF1-SA] Khách gửi ảnh áo lên Messenger
   → AI nhận diện: "Đây là mẫu Áo Polo Trắng" → check stock
   → "Dạ còn 3 cái size M ạ! Giá 350k, anh/chị lấy không ạ?"

2. [WF1-SA] Khách: "bớt không"
   → AI apply_voucher 10% → "Dạ em hỗ trợ còn 315k + freeship nha 😊"
   → Khách: "ok lấy"

3. [WF3] AI kiểm tra DB → khách mới → hỏi 5 thông tin
   → Tổng hợp hóa đơn → Tạo QR VietQR → gửi kèm "⏰ QR hiệu lực 2h"

4. [WF4] Khách chuyển khoản
   → SePay webhook → idempotency check → verify amount
   → Atomic: paid + stock-1 + tag "Đã mua" + CS AI bật
   → Google Sheets ghi tự động · e-invoice xuất
   → AI: "✅ Em nhận được tiền rồi ạ!"

5. [WF5] Telegram nhân viên kho
   → Nhân viên bấm ✅ → Ahamove tạo vận đơn
   → CS AI gửi tracking link cho khách

6. [WF6] Demo edge cases:
   → Nhắn câu tức giận → 🔴 Urgent ping CSKH ngay
   → Khách bỏ chat 2h → AI follow-up freeship
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
CASSO_API_KEY=

# Google Sheets
GOOGLE_SHEETS_ID=
GOOGLE_SERVICE_ACCOUNT_JSON=

# E-invoice
VNPT_USERNAME=
VNPT_PASSWORD=

# Ops
TELEGRAM_BOT_TOKEN=
TELEGRAM_STAFF_CHAT_ID=

# Shipping
AHAMOVE_API_KEY=
VIETTEL_POST_TOKEN=
```
