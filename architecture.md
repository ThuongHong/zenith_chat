# 🏗️ Zenith Chat — Architecture & Technical Pipeline

## 1. System Overview

Zenith Chat là một **Omnichannel AI Agent** xử lý tin nhắn từ nhiều nền tảng (Facebook, Instagram, Lazada) thông qua một AI Engine thống nhất, kết hợp với dashboard thời gian thực cho agent người dùng.

```
┌─────────────────────────────────────────────────────────────────┐
│                        INPUT CHANNELS                           │
│   📱 FB Messenger     📸 Instagram DM     🛒 Lazada (Mock)      │
└─────────────┬───────────────┬───────────────┬───────────────────┘
              │               │               │
              └───────────────┴───────────────┘
                              │ POST /webhook/:channelId
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                      FASTIFY API LAYER                          │
│  ┌─────────────────┐   ┌──────────────────┐                    │
│  │ Channel Adapter │──▶│ Message Aggregator│ (Redis 3-5s)       │
│  │ (Normalize msg) │   │ (Debounce buffer) │                    │
│  └─────────────────┘   └────────┬─────────┘                    │
└───────────────────────────────── │ ───────────────────────────--┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────┐
│                    ⭐ AI AGENT ENGINE                            │
│                   (Gemini Flash / Pro)                  │
│                                                                 │
│   PERCEIVE ──▶ REASON ──▶ ACT ──▶ OBSERVE ──▶ RESPOND          │
│                                                                 │
│   Tools: search_product · check_inventory · apply_voucher       │
│          get_order_status · escalate_to_human                   │
│          schedule_followup · switch_to_cs_mode                  │
└───────────────────────────────┬─────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                       SUPABASE LAYER                            │
│  PostgreSQL (persistent) ──▶ Realtime Publisher                 │
│  tables: customers · conversations · messages · orders          │
└────────────────────┬────────────────────────────────────────────┘
                     │
        ┌────────────┴────────────┐
        ▼                         ▼
 Flutter Web Dashboard      BullMQ Worker (Redis)
 (Agent Inbox + Realtime)   (Delayed follow-up jobs)
```

---

## 2. Technical Pipeline — Chi tiết từng bước

### Bước 1 — Channel Normalization

Mỗi platform gửi webhook với cấu trúc khác nhau. **Channel Adapter** chuẩn hóa tất cả về một interface duy nhất:

```
FB Messenger  ──┐
Instagram DM  ──┼──▶  NormalizedMessage  ──▶  AI Engine
Lazada Mock   ──┘
```

```typescript
interface NormalizedMessage {
  channelId:  'facebook' | 'instagram' | 'lazada'
  customerId: string      // platform-specific ID đã được map
  content:    string      // text đã gộp
  timestamp:  Date
  metadata:   object      // attachments, product context...
}
```

> **Lợi ích:** Thêm kênh mới (Zalo, Shopee) chỉ cần thêm 1 adapter, không đụng vào AI hay DB.

---

### Bước 2 — Message Aggregation (Redis Buffer)

Khách thường nhắn nhiều tin liên tiếp trong vài giây:

```
t=0.0s  "shop ơi"
t=0.8s  "cái áo đen size M"
t=1.5s  "còn không"

❌ Không buffer → gọi AI 3 lần → tốn tiền + reply lộn xộn
✅ Redis buffer 3-5s → gom thành 1 message → gọi AI 1 lần
```

Cơ chế: **Debounce với sliding window**, key = `buffer:{conversationId}`.

---

### Bước 3 — AI Agent Loop ⭐

Đây là trọng tâm agentic của hệ thống. AI không chỉ trả lời — nó **lên kế hoạch và hành động**:

```
                    ┌─────────────────────────────┐
                    │         PERCEIVE             │
                    │  - Message hiện tại          │
                    │  - Lịch sử conversation      │
                    │  - Sentiment score           │
                    └──────────────┬──────────────┘
                                   │
                    ┌──────────────▼──────────────┐
                    │          REASON              │
                    │  - Intent là gì?             │
                    │  - Cần tool nào?             │
                    │  - Thứ tự gọi tool?          │
                    └──────────────┬──────────────┘
                                   │
                    ┌──────────────▼──────────────┐
                    │            ACT               │
                    │  Gọi 1 hoặc nhiều tools      │
                    │  (có thể chain nhiều bước)   │
                    └──────────────┬──────────────┘
                                   │
                    ┌──────────────▼──────────────┐
                    │          OBSERVE             │
                    │  Nhận kết quả từ tools       │
                    │  Có cần gọi thêm tool không? │
                    └──────────────┬──────────────┘
                                   │
                    ┌──────────────▼──────────────┐
                    │          RESPOND             │
                    │  Sinh reply tiếng Việt       │
                    │  Context-aware, natural      │
                    └─────────────────────────────┘
```

**Ví dụ multi-step tool chaining:**
```
Khách: "còn áo đen size M không, tôi muốn đặt luôn"

→ [ACT 1]  search_product("áo đen")         → product_id: "P001"
→ [ACT 2]  check_inventory("P001", "size M") → stock: 5
→ [ACT 3]  apply_voucher(customer, 10%)      → AI tự quyết kích cầu vì intent mua rõ
→ [RESPOND] "Dạ còn 5 cái ạ! Em hỗ trợ thêm 10% còn 315k, freeship đơn từ 300k 😊"
```

---

### Bước 4 — Storage & Realtime Push

```
AI generates response
        │
        ▼
Supabase INSERT into messages
  { role: 'ai', content, metadata: { tool_calls, sentiment } }
        │
        ▼
PostgreSQL WAL (Write-Ahead Log)
        │
        ▼
Supabase Realtime Publisher
        │
        ▼
Flutter Web Dashboard nhận push < 100ms
```

Không cần Socket.IO riêng — Supabase Realtime xử lý toàn bộ.

---

### Bước 5 — Proactive Follow-up (BullMQ)

Sau khi khách chốt đơn, AI **chủ động** lên lịch follow-up:

```
t=0      Khách: "ok đặt"
          └─▶ AI gọi schedule_followup(delay=120min, msg="Đơn đang giao...")
          └─▶ JOB ENQUEUED vào Redis

t=120min  BullMQ Worker tự kích hoạt
          └─▶ Gửi tin qua Channel Adapter
          └─▶ Không cần agent/admin ra lệnh
```

**Chuỗi follow-up điển hình:**
| Thời gian | Nội dung |
|-----------|---------|
| +2h | Xác nhận đơn hàng + mã vận đơn |
| +24h | Cập nhật trạng thái giao hàng |
| +72h | Yêu cầu đánh giá sản phẩm |

---

## 3. Tính Agentic — Phân tích chi tiết

> **Chatbot** = Nhận input → Trả output (1 bước)
> **AI Agent** = Perception → Reasoning → Action → Observation loop (N bước)

Zenith Chat có **4 đặc tính agentic** cốt lõi:

### 3.1 Multi-step Tool Chaining
AI tự quyết định tool nào cần gọi, theo thứ tự nào, dựa trên kết quả trước đó — không phải if/else cứng.

### 3.2 Goal-Directed Mode Switching
```
SALES MODE (default)     ←── Mục tiêu: Chốt đơn
        │
        │ [order confirmed]
        ▼
  CS MODE                ←── Mục tiêu: Giữ chân khách hàng
        │
        │ [negative sentiment < -0.7]
        ▼
  ESCALATION             ←── Mục tiêu: Bảo vệ brand
```

AI tự chuyển giữa các mode không cần rule cứng.

### 3.3 Proactive Actions
AI **chủ động hành động** mà không cần khách nhắn thêm:
- Tự lên lịch follow-up
- Tự apply voucher khi phát hiện intent mua
- Tự escalate khi sentiment nguy hiểm

### 3.4 Adaptive Sentiment Response
```
Sentiment > 0      → Friendly sales tone
Sentiment -0.3–0   → Empathetic, giải quyết nhanh
Sentiment < -0.7   → escalate_to_human(priority='urgent')
```

---

## 4. Smart Handover — Decision Tree

```
Message đến
      │
      ▼
  sentiment < -0.7? ──YES──▶ 🔴 URGENT: escalate_to_human
      │ NO
      ▼
  complaint/refund keyword? ──YES──▶ 🟡 NORMAL: escalate_to_human
      │ NO
      ▼
  AI confidence > 80%? ──YES──▶ 🟢 AUTO: AI xử lý hoàn toàn
      │ NO
      ▼
                     ──────▶ 🟡 REVIEW: AI trả lời + flag cho agent
```

**Dashboard hiển thị:**
- 🔴 Badge đỏ + push notification → Agent can thiệp ngay
- 🟡 Badge vàng → Agent review khi rảnh
- 🟢 Không badge → AI đang handle tốt

---

## 5. Model Selection Strategy

Dùng **Hybrid model** để tối ưu chi phí và chất lượng:

```
Mọi tin nhắn
      │
      ├── 95% cases ──▶ gemini-2.0-flash
      │   (Function calling, queries đơn giản)
      │   Latency: ~1s · Cost: $0.075/1M tokens
      │
      └── 5% cases ──▶ gemini-2.5-pro-latest
          Trigger khi:
          · sentiment < -0.8
          · content.length > 500 chars
          · escalationCount > 2
          Latency: ~5s · Cost: $1.25/1M tokens
```

**Ước tính chi phí** (1,000 conv/ngày, 10 msg/conv, 500 tokens/req):
| Strategy | Chi phí/tháng |
|----------|--------------|
| Toàn Flash | ~$11 |
| Hybrid 95/5 | ~$20 ✅ |
| Toàn Pro | ~$187 |

---

## 6. Data Architecture

### Tại sao Supabase + Redis, không phải chỉ một?

| Lớp | Công nghệ | Vai trò |
|-----|-----------|---------|
| **Persistent Store** | Supabase (PostgreSQL) | Lưu vĩnh viễn: customers, messages, orders |
| **Realtime Push** | Supabase Realtime | PostgreSQL WAL → Flutter Dashboard |
| **Ephemeral Buffer** | Redis | Message debounce 3-5s (mất khi restart = OK) |
| **Job Queue** | Redis + BullMQ | Delayed follow-up jobs |

### Tại sao messages lưu ở Supabase, không phải NoSQL?

1. **Supabase Realtime** phụ thuộc vào PostgreSQL WAL — tách ra là mất feature
2. Messages cần JOIN với conversations và customers thường xuyên
3. Volume e-commerce chatbot không đủ lớn cần NoSQL (< 1M msg/ngày)
4. Một SDK, một connection, đơn giản hơn nhiều

---

## 7. Full Stack Summary

```
┌─────────────────────────────────────────────────┐
│              ZENITH CHAT — STACK                │
├─────────────────┬───────────────────────────────┤
│ AI Engine       │ Gemini 2.0 Flash + 2.5 Pro    │
│ Backend         │ Node.js + Fastify + TypeScript │
│ Database        │ Supabase (PostgreSQL)          │
│ Realtime        │ Supabase Realtime              │
│ Auth            │ Supabase Auth                  │
│ Queue           │ BullMQ + Redis                 │
│ Frontend        │ Flutter Web                    │
│ Channels        │ FB + Instagram + Lazada        │
│ Pattern         │ Adapter + Agent Loop           │
└─────────────────┴───────────────────────────────┘
```
