# Hướng dẫn sử dụng - Checklive Bot

> Hướng dẫn vận hành hệ thống bot Telegram check live/die UID Facebook.
> Gồm 2 phần: (A) dành cho Admin dùng web quản trị, (B) dành cho người dùng bot trong nhóm Telegram.

Cập nhật lần cuối: 2026-07-15

---

## 0. Các thành phần & địa chỉ

| Thành phần | Địa chỉ (local) | Vai trò |
|---|---|---|
| Web Admin (frontend) | http://localhost:3000 | Trang quản trị |
| API Backend | http://localhost:3300/api/v1 | Xử lý logic, DB, bot |
| API Docs (Swagger) | http://localhost:3300/api/v1/docs | Thử API |
| Telegram Bot | trong nhóm Telegram | Nhận lệnh, check UID |

Tài khoản admin mặc định (đổi trong `.env` / sau khi seed): `admin@checklive.local` / `admin12345`.

---

## A. HƯỚNG DẪN CHO ADMIN (Web quản trị)

### A1. Đăng nhập
1. Mở http://localhost:3000 -> tự chuyển tới trang đăng nhập.
2. Nhập email + mật khẩu admin -> vào **Tổng quan (Dashboard)**.
3. Menu trái gồm: Tổng quan, Nhóm bot, License, Gói dịch vụ, **Quản lý quản trị viên** (mục cuối chỉ hiện với tài khoản ADMIN).

### A2. Bước 1 - Tạo Gói dịch vụ (Plans)
Gói định nghĩa **quota** áp cho nhóm dùng bot.
1. Vào **Gói dịch vụ** -> **Tạo gói**.
2. Điền:
   - **Tên gói** (vd: Basic).
   - **Toi da UID**: số UID tối đa mỗi nhóm được theo dõi.
   - **Chu ky check (giay)**: khoảng cách tối thiểu giữa 2 lần check (vd 60).
   - **Toi da nhom**: số nhóm tối đa (dùng cho license nhiều nhóm).
   - **Thoi han (ngay)**: số ngày hiệu lực khi kích hoạt.
   - **Gia (VND)**: để tham khảo.
3. Lưu. (Seed sẵn 3 gói: Trial / Basic / Pro.)

### A3. Bước 2 - Cấp quyền cho nhóm
Có **2 cách** (dùng cách nào cũng được):

**Cách 1 - Duyệt nhóm trực tiếp trên web:**
1. Người dùng thêm bot vào nhóm Telegram trước (xem phần B1). Nhóm sẽ xuất hiện ở **Nhóm bot** với trạng thái **PENDING**.
2. Vào **Nhóm bot** -> bấm **Duyệt** ở nhóm đó -> chọn **gói dịch vụ** -> **Xác nhận duyệt**.
3. Nhóm chuyển sang **APPROVED** + có hạn dùng. Bot bắt đầu hoạt động.
4. Muốn ngừng: bấm **Khóa** (BLOCKED) -> bot ngừng check nhóm đó.

**Cách 2 - Phát License key cho khách tự kích hoạt:**
1. Vào **License** -> **Tạo license** -> chọn **gói** + **số lượng** -> **Tạo**.
2. Copy key (dạng `CL-XXXX-XXXX-XXXX-XXXX`) gửi khách.
3. Khách gõ `/activate KEY` trong nhóm (xem B2) -> nhóm tự thành APPROVED theo gói.
4. Thu hồi: bấm **Thu hồi** (REVOKED) trên key còn `ACTIVE` -> key hết hiệu lực (chưa kích hoạt được nữa).
5. Hoàn tác: bấm **Hoàn tác** trên key đã `USED` -> key về `ACTIVE` để phát lại; nhóm Telegram đã gắn key sẽ bị **Khóa (BLOCKED)**.

### A4. Quản lý Nhóm bot
Cột hiển thị: tên nhóm, Chat ID, trạng thái (PENDING/APPROVED/BLOCKED), gói, số UID, hạn dùng.
- **Duyệt**: cấp phép + gán gói.
- **Khóa**: chặn nhóm.

### A5. Quản lý quản trị viên (chỉ ADMIN)
Trang **Quản lý quản trị viên** chỉ ADMIN thấy; chứa tài khoản `ADMIN` / `SUB_ADMIN`.
Không có đăng ký web cho khách — khách dùng bot qua Telegram (duyệt nhóm / license).
- **Tạo quản trị viên**: email, mật khẩu, tên, chọn role `ADMIN` hoặc `SUB_ADMIN`.
- **Đổi role**: chuyển giữa ADMIN và SUB_ADMIN.
- **Ban / Bỏ ban**: khóa tài khoản quản trị (không ban chính mình; không thao tác trên OWNER / Super Admin).
- **Xóa**: xóa tài khoản quản trị (không xóa chính mình / Super Admin).

### A6. Dashboard
Xem nhanh: tổng số nhóm (chờ duyệt / đã duyệt), số license (còn hiệu lực), số gói, và (nếu là ADMIN) số **quản trị viên**.

---

## B. HƯỚNG DẪN CHO NGƯỜI DÙNG BOT (trong nhóm Telegram)

### B1. Thêm bot vào nhóm
1. Thêm bot (theo username bot của bạn) vào nhóm Telegram.
2. Bot tự ghi nhận nhóm ở trạng thái **chờ duyệt** và nhắn hướng dẫn.
3. Nhóm CHƯA hoạt động cho tới khi được Admin duyệt hoặc kích hoạt license.

### B2. Kích hoạt bằng license (nếu có key)
```
/activate CL-XXXX-XXXX-XXXX-XXXX
```
Thành công -> bot báo tên gói + hạn dùng, nhóm bắt đầu hoạt động.

### B3. Các lệnh quản lý UID (chỉ dùng được khi nhóm đã được cấp phép)
| Lệnh | Ý nghĩa |
|---|---|
| `/add UID|Tên` | Thêm 1 UID để theo dõi |
| `/addlist` | Thêm nhiều UID (mỗi dòng `UID|Tên`, xuống dòng sau lệnh) |
| `/delete UID` | Xóa 1 UID khỏi danh sách |
| `/checkall` | Kiểm tra toàn bộ UID ngay và trả danh sách trạng thái |
| `/start` hoặc `/help` | Xem danh sách lệnh |

Ví dụ `/addlist`:
```
/addlist
100000123|Nguyen Van A
100000456|Tran Thi B
100000789|Le Van C
```

### B4. Cơ chế thông báo
- Bot tự động check theo chu kỳ của gói (vd 1 phút/lần).
- **Chỉ báo khi có UID chuyển sang DIE**; UID còn sống thì không nhắn (tránh spam).
- Nội dung báo: `UID - Tên` / `Trạng thái: Die` / `ngày giờ`.
- Vượt quota UID của gói -> bot báo đã đạt giới hạn, không thêm được nữa.

---

## C. QUY TRÌNH VẬN HÀNH ĐIỂN HÌNH (end-to-end)

1. Admin tạo Gói (A2) -> phát License cho khách (A3 cách 2) HOẶC chờ duyệt nhóm (A3 cách 1).
2. Khách thêm bot vào nhóm (B1).
3. Khách `/activate KEY` (B2) hoặc Admin bấm Duyệt (A3).
4. Khách `/add` / `/addlist` các UID (B3).
5. Bot tự check + báo DIE (B4). Khách gõ `/checkall` khi muốn xem tức thời.
6. Admin theo dõi trên Dashboard, khóa nhóm/thu hồi key khi cần.

```mermaid
flowchart TD
  A["Admin tao Goi + License"] --> B["Khach them bot vao nhom"]
  B --> C{"Cap phep"}
  C -->|"/activate KEY"| D["Nhom APPROVED"]
  C -->|"Admin bam Duyet"| D
  D --> E["Khach /add /addlist UID"]
  E --> F["Bot check dinh ky"]
  F --> G{"Co UID DIE?"}
  G -->|"Co"| H["Bot bao ve nhom"]
  G -->|"Khong"| F
```

---

## D. TẠO BOT TELEGRAM & KẾT NỐI HỆ THỐNG (người vận hành kỹ thuật)

### D1. Tạo bot bằng BotFather
1. Mở Telegram, tìm **@BotFather** (tích xanh) -> bấm **Start**.
2. Gõ `/newbot`.
3. Nhập **tên hiển thị** của bot (vd: `Checklive Bot`).
4. Nhập **username** kết thúc bằng `bot` (vd: `checklive_uid_bot`) - phải là duy nhất.
5. BotFather trả về **TOKEN** dạng:
   ```
   123456789:AAExxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
   ```
   -> Lưu lại token này (không chia sẻ công khai).

### D2. Cấu hình bot để hoạt động trong nhóm
Vẫn trong BotFather:
1. `/setprivacy` -> chọn bot -> **Disable**.
   (Tắt privacy để bot đọc được tin nhắn/lệnh trong nhóm. Nếu để Enable, bot chỉ thấy lệnh có `@tenbot`.)
2. `/setjoingroups` -> chọn bot -> **Enable** (cho phép thêm bot vào nhóm).
3. (Tùy chọn) `/setcommands` -> chọn bot -> dán danh sách lệnh để hiện gợi ý:
   ```
   activate - Kich hoat license: /activate KEY
   add - Them 1 UID: /add UID|Ten
   addlist - Them nhieu UID
   delete - Xoa UID: /delete UID
   checkall - Kiem tra toan bo UID
   help - Xem huong dan
   ```

### D3. Nạp token vào hệ thống (backend)
Mở file `.env` của `checklive-be` và điền:
```
TELEGRAM_BOT_TOKEN=123456789:AAExxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
TELEGRAM_WEBHOOK_SECRET=mot-chuoi-bi-mat-tu-dat
```
- `TELEGRAM_WEBHOOK_SECRET`: tự đặt 1 chuỗi ngẫu nhiên, dùng để bảo vệ đường dẫn webhook.
- Khởi động lại backend sau khi sửa `.env`.

### D4. Kết nối bot với hệ thống (đăng ký Webhook)
Bot gửi mọi cập nhật (lệnh, sự kiện được add vào nhóm) về backend qua **webhook**. Backend nhận tại:
```
https://<domain>/api/v1/telegram/webhook/<TELEGRAM_WEBHOOK_SECRET>
```

**Bước đăng ký webhook với Telegram** (chạy 1 lần, thay TOKEN / domain / secret):
```bash
curl -X POST "https://api.telegram.org/bot<TOKEN>/setWebhook" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://<domain>/api/v1/telegram/webhook/<TELEGRAM_WEBHOOK_SECRET>",
    "allowed_updates": ["message", "my_chat_member"]
  }'
```
Trả về `{"ok":true,...}` là thành công.

Kiểm tra trạng thái webhook:
```bash
curl "https://api.telegram.org/bot<TOKEN>/getWebhookInfo"
```

Gỡ webhook (nếu cần):
```bash
curl "https://api.telegram.org/bot<TOKEN>/deleteWebhook"
```

### D5. Chạy thử ở local (chưa có domain)
Backend chạy `localhost:3300` không nhận webhook từ Telegram được (cần HTTPS công khai). Dùng **ngrok**:
```bash
ngrok http 3300
```
Lấy URL HTTPS ngrok trả về (vd `https://abc123.ngrok-free.app`) rồi đăng ký webhook như D4 với domain đó:
```
https://abc123.ngrok-free.app/api/v1/telegram/webhook/<TELEGRAM_WEBHOOK_SECRET>
```

### D6. Thêm bot vào nhóm & kiểm tra
1. Trong nhóm Telegram -> Thêm thành viên -> tìm **username bot** -> thêm vào.
2. Nên đặt bot làm **quản trị viên nhóm** (giúp nhận đầy đủ sự kiện, gửi tin ổn định).
3. Bot sẽ nhắn "nhóm đang chờ duyệt" -> nhóm xuất hiện ở web Admin (mục Nhóm bot, trạng thái PENDING).
4. Duyệt nhóm hoặc `/activate KEY`, rồi gõ `/checkall` để kiểm tra kết nối.

> Tóm tắt luồng kết nối: BotFather (tạo bot + token) -> điền token vào `.env` backend -> đăng ký webhook trỏ về backend -> thêm bot vào nhóm -> cấp phép -> dùng.

### D7. Cấu hình sau khi deploy Railway
- Đặt `TELEGRAM_BOT_TOKEN`, `TELEGRAM_WEBHOOK_SECRET` trong biến môi trường của service backend trên Railway.
- Lấy domain công khai Railway cấp cho backend, rồi chạy lại lệnh `setWebhook` (D4) với domain đó.

---

## E. XỬ LÝ SỰ CỐ THƯỜNG GẶP

| Hiện tượng | Nguyên nhân | Cách xử lý |
|---|---|---|
| Bot không phản hồi lệnh | Nhóm chưa APPROVED / hết hạn | Admin duyệt lại hoặc gia hạn/kích hoạt key mới |
| "Nhom chua duoc cap phep" | Chưa duyệt/license | Duyệt nhóm hoặc `/activate KEY` |
| Không thêm được UID | Đã đạt `maxUids` của gói | Nâng gói hoặc xóa bớt UID |
| Đăng nhập web lỗi 401 | Sai tài khoản / cookie | Kiểm tra email/mật khẩu, thử lại |
| Backend lỗi `P1010` | Sai `DATABASE_URL` / chưa tạo DB | Sửa `.env`, tạo DB, chạy `prisma migrate` |
| Bot không tự check | `SCHEDULER_ENABLED=false` hoặc nhóm hết hạn | Bật scheduler, kiểm tra hạn nhóm |
| Check FB sai/nhiều UNKNOWN | Bị Facebook chặn IP | Cấu hình `PROXY_LIST` (proxy pool) |

---

## F. Lưu ý quan trọng
- Độ chính xác live/die phụ thuộc Facebook, không đạt 100%.
- Quy mô lớn (nhiều UID) BẮT BUỘC dùng proxy pool, nếu không sẽ bị FB chặn.
- Facebook hay thay đổi -> cần bảo trì engine check định kỳ.
- Xem thêm bối cảnh kỹ thuật: `docs/PROJECT.md`.
