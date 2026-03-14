# Telegram Support Ẩn Danh (Khách – NV – Leader) – Khả thi & Cách làm

## Câu hỏi của khách

Làm support trong nhóm Telegram nhưng **nhân viên support không được biết username/contact của khách**. Mô hình tham khảo:

- Nhóm 1: **Leader + Khách + Bot** — khách chỉ có trong nhóm này (chỉ leader thấy được khách).
- Nhóm 2: **Nhân viên + Bot** (không thêm khách vào; đặt tên/ký hiệu riêng cho mỗi nhóm, ví dụ `[Case #001] Support`). Nếu thêm khách vào nhóm NV thì nhân viên sẽ thấy được khách → mất ẩn danh.
- Tin nhắn khách (trong nhóm Leader) → bot sao chép sang **nhóm NV** với format kiểu “Khách: …” (ưu tiên **sao chép**).
- Tin nhắn nhân viên (trong nhóm NV) → bot sao chép sang **nhóm Leader** (khách đọc tại đó).
- Tin nhắn có **@username, link Tele, SĐT, hoặc từ khóa** (ib, inbox, whatsapp, tele, telegram, dm, pm, contact...) → gửi thêm vào **nhóm riêng có Leader** để leader kiểm tra
- **Chặn** tin nhắn dạng @username (không cho NV biết contact khách)
- **Cho phép** gửi file media (ảnh, video)

**Trả lời ngắn: Làm được.** Điều quan trọng là **nhóm NV chỉ có Nhân viên + Bot**, không add khách vào — bot làm cầu nối relay tin hai chiều để NV không bao giờ thấy profile/username của khách.

---

## 1. Tại sao làm được?

| Yêu cầu | Telegram Bot API | Ghi chú |
|--------|-------------------|--------|
| Bot trong nhiều nhóm | ✅ | Thêm bot vào từng nhóm, nhận `message` từ mọi nhóm |
| Sao chép tin nhắn sang nhóm khác | ✅ | `copyMessage` (không hiện “forwarded from”) |
| Chuyển tiếp tin nhắn | ✅ | `forwardMessage` |
| Gửi ảnh/video | ✅ | `sendPhoto`, `sendVideo`, `sendDocument` |
| Nhận tin nhắn real-time | ✅ | Webhook hoặc long polling `getUpdates` |
| Gửi tin vào nhóm “chỉ Leader” | ✅ | Bot tạo/tham gia nhóm alert, gửi bằng `sendMessage`/`copyMessage` |

Điều cần làm: **một con bot riêng** (hoặc module mới) xử lý **incoming updates** (tin nhắn vào các nhóm), sau đó quyết định copy/forward/block và gửi sang nhóm tương ứng.

---

## 2. Kiến trúc gợi ý

```
[Khách] ←→ Nhóm A (Leader + Khách + Bot)  ←→ [Bot]  ←→ Nhóm B (chỉ NV + Bot)
         khách chỉ ở đây, leader thấy              relay tin hai chiều
         khách                                        NV không thấy khách
                                                              ↓
                                              Nhóm Alert (Leader + Bot)
                                              (tin có @username, link, SĐT, từ khóa)
```

- Mỗi “case” support = 2 nhóm (khách **không** có trong nhóm NV):
  - **Nhóm Leader**: Leader + Khách + Bot — khách chat tại đây, leader thấy được khách.
  - **Nhóm NV**: **Chỉ Nhân viên + Bot** (đặt tên/ký hiệu riêng, ví dụ: `[Case #001] Support`). NV không bao giờ thấy khách; mọi trao đổi với “khách” đều qua bot relay.
- Bot lưu **mapping**: `nhóm Leader ↔ nhóm NV` (1 case = 1 cặp nhóm).
- **Một nhóm Alert** cố định: Leader + Bot; mọi tin “nhạy cảm” đều được gửi thêm vào đây.

Luồng:

1. **Khách gửi tin trong nhóm Leader**  
   Bot nhận → copy sang **nhóm NV** (format kiểu “Khách: &lt;nội dung&gt;” hoặc “\[Case #001\] &lt;nội dung&gt;”). Nếu tin có “nhạy cảm” → gửi thêm bản copy/trích dẫn vào **Nhóm Alert**.

2. **Nhân viên gửi tin trong nhóm NV**  
   Bot nhận → kiểm tra:
   - Có @username → **chặn** (không chuyển sang khách), đồng thời gửi tin đó (hoặc thông báo) vào **Nhóm Alert**.
   - Có link Tele, SĐT, từ khóa (ib, inbox, whatsapp, tele, telegram, dm, pm, contact...) → **chặn** hoặc **vẫn chuyển nhưng gửi thêm bản copy vào Nhóm Alert** (tùy chính sách).
   - Tin bình thường / chỉ media → **copy** sang **nhóm Leader** (khách đọc tại đó).

3. **Media (ảnh, video, file)**  
   Luôn được phép: bot dùng `copyMessage` (nếu Telegram hỗ trợ copy loại message đó) hoặc tải file lên rồi gửi lại bằng `sendPhoto`/`sendVideo`/`sendDocument`.

---

## 3. Kỹ thuật cần có

### 3.1. Nhận tin nhắn (incoming updates)

- **Webhook** (production): endpoint HTTPS nhận POST từ Telegram khi có tin nhắn.
- **Polling** (dev/test): gọi `getUpdates` định kỳ.

Bot cần có **quyền đọc tin nhắn** trong nhóm (default khi add bot vào group).

### 3.2. Phân biệt nguồn tin và đích gửi

- Mỗi nhóm có `chat.id` (số, âm với supergroup).
- Lưu DB hoặc config:
  - Nhóm nào là “Leader+Khách”, nhóm nào là “NV+Bot” (không có khách), nhóm nào là “Alert”.
  - Cặp nhóm Leader ↔ NV (mapping 1–1).
- Khi nhận `message.chat.id`, tra bảng → biết tin này từ nhóm nào, cần copy/forward sang nhóm nào và có cần gửi Alert không.

### 3.3. Sao chép / chuyển tiếp

- **Sao chép**: `copyMessage(chat_id_from, chat_id_to, message_id)` – tin ở nhóm đích không hiện “forwarded”, phù hợp yêu cầu “ưu tiên sao chép”.
- **Chuyển tiếp**: `forwardMessage` – giữ nguyên định dạng, có “forwarded from”.
- Với **media**: copy nếu API hỗ trợ, không thì download → `sendPhoto`/`sendVideo`/`sendDocument` vào chat đích.

### 3.4. Nhận diện tin “nhạy cảm” (gửi Nhóm Alert)

- **@username**: regex kiểu `@[\w]+` (hoặc dùng `entities` trong message nếu Telegram đánh dấu mention).
- **Link Telegram**: `t\.me/`, `telegram\.me/`, `telegram\.dog/`…
- **SĐT**: dãy số 10–11 số (có thể giới hạn prefix 0, 84…).
- **Từ khóa**: ib, inbox, whatsapp, tele, telegram, dm, pm, contact (so khớp không dấu, có thể thêm biến thể).

Tin nào match → gửi thêm vào **Nhóm Alert** (Leader). Có thể gửi kèm thông tin: nhóm nguồn, thời gian, nội dung (đã mask @ nếu cần).

### 3.5. Chặn tin @username từ NV → Khách

- Khi tin từ **nhóm NV** (xác định bằng `chat.id`) và **gửi bởi nhân viên** (không phải khách, không phải bot):
  - Nếu có @username → **không** copy/forward sang nhóm khách; chỉ gửi vào Nhóm Alert (và có thể thông báo trong nhóm NV: “Tin chứa @username không được chuyển tới khách”).

---

## 4. Hạn chế / Lưu ý

- **Quyền bot**: Bot phải là member (ít nhất quyền gửi tin) ở tất cả nhóm tham gia; muốn copy/forward thì cần đọc được tin trong nhóm.
- **Tạo nhóm**: Nhóm “Leader+Khách+Bot” (leader tạo, add khách + bot). Nhóm “NV+Bot” (leader hoặc NV tạo, **chỉ add nhân viên + bot**, không add khách). Bot chỉ relay tin nên khách không cần có trong nhóm NV.
- **Định danh khách**: Trong nhóm NV, khách không xuất hiện; bot gửi tin với prefix “Khách:” hoặc “\[Case #xxx\]” để NV biết đang support case nào.
- **Spam / tốc độ**: Telegram có giới hạn gửi tin (ví dụ ~30 tin/giây cho bot); nếu nhiều tin cùng lúc cần queue.
- **Lưu trữ**: Nếu cần audit, nên log (không cần lưu nội dung tin nhắn dài hạn nếu không cần thiết).

---

## 5. So với hệ thống hiện tại (Super AD Card)

- **Super AD Card** hiện dùng Telegram chỉ để **gửi ra** (thông báo code, báo cáo số dư, giao dịch từ chối…): một chiều, không xử lý tin nhắn vào.
- Mô hình support ẩn danh cần **nhận tin nhắn** và **điều hướng giữa các nhóm** → cần thêm:
  - **Bot riêng** (hoặc bot token riêng), và
  - **Service/worker** nhận webhook (hoặc polling), có logic map nhóm + detection + copy/forward/block.

Có thể đặt trong repo hiện tại dưới dạng module mới (ví dụ `support-telegram-bot`) hoặc repo riêng, tùy cách bạn tách service.

---

## 6. Ước lượng thời gian triển khai

Ước tính cho **1 dev fullstack** quen NestJS/Node + Telegram Bot API, từ spec đến chạy thử với vài nhóm thật.

| Hạng mục | Thời gian | Ghi chú |
|----------|-----------|--------|
| **Bot + webhook/polling** | 1–2 ngày | Tạo bot với BotFather, endpoint nhận updates, parse message/chat_id/user |
| **Mapping nhóm (DB + CRUD)** | 1–2 ngày | Bảng Leader group ↔ NV group, nhóm Alert; API hoặc admin đơn giản để gán cặp nhóm |
| **Relay tin cơ bản** | 2–3 ngày | Tin từ nhóm Leader → copy sang nhóm NV (prefix "Khách: …"); tin từ nhóm NV → copy sang nhóm Leader; xử lý text + copyMessage |
| **Media (ảnh, video, file)** | 1–2 ngày | copyMessage cho media hoặc download → sendPhoto/sendVideo/sendDocument; xử lý lỗi/giới hạn Telegram |
| **Nhận diện tin nhạy cảm** | 1–2 ngày | Regex/entities: @username, link t.me, SĐT, từ khóa (ib, inbox, contact, …); gửi bản copy vào Nhóm Alert |
| **Chặn @username NV→Khách** | 0.5–1 ngày | Không relay tin có @; có thể gửi cảnh báo trong nhóm NV + bản tin vào Alert |
| **Test + xử lý lỗi biên** | 2–3 ngày | Nhiều case đồng thời, rate limit, tin xóa/sửa (nếu cần), log/debug |

**Tổng ước lượng:**

- **POC tối thiểu** (1 Leader group, 1 NV group, relay text + chặn @ + Alert đơn giản): **~1–1,5 tuần**.
- **Bản đủ dùng** (nhiều case, media, danh sách từ khóa chỉnh được, mapping rõ ràng): **~2–3 tuần**.
- **Bản ổn định production** (monitoring, retry, quản lý nhóm/role, tài liệu vận hành): **+1–2 tuần** nữa.

Thời gian có thể rút ngắn nếu đã có sẵn server/webhook và người vận hành biết tạo nhóm Telegram; có thể kéo dài nếu cần tích hợp SSO, dashboard quản lý case, hoặc yêu cầu audit/backup chi tiết.

---

## 7. Kết luận

- **Có làm được**: Đúng với mô hình **Leader + Khách + Bot** (khách chỉ ở đây) và **Nhân viên + Bot** (không có khách → NV không thấy contact khách), bot relay tin hai chiều, nhóm alert cho leader, chặn @username, cho phép media.
- **Cần**: Một bot Telegram (token riêng), backend nhận updates (webhook/polling), lưu mapping nhóm, logic phát hiện từ khóa/link/@/SĐT và quy tắc chặn/gửi Alert.
- **Không nằm trong** chức năng Telegram hiện có của Super AD Card (chỉ gửi thông báo), nên cần phát triển thêm module/con bot riêng cho use case support ẩn danh này.

Nếu khách muốn bước tiếp, có thể đề xuất: (1) tạo spec chi tiết (bảng mapping, danh sách từ khóa, quy tắc chặn), (2) thiết kế API/webhook và (3) POC bot đơn giản (1 nhóm Leader+Khách+Bot, 1 nhóm NV+Bot, 1 nhóm Alert) để demo luồng copy + chặn @ + gửi Alert. Ước lượng thời gian: xem **§6. Ước lượng thời gian triển khai**.
