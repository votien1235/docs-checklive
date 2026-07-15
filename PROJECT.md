# Checklive Bot - Tài liệu dự án

> File này là bản tổng hợp bối cảnh để các phiên làm việc / chat sau hiểu dự án đang làm gì, đã có gì, và còn gì phải làm. Cập nhật khi có thay đổi lớn.

Cập nhật lần cuối: 2026-07-15

---

## 1. Dự án là gì

Hệ thống **bot Telegram check live/die UID Facebook** cho khách hàng, gồm:

1. **Bot Telegram**: thêm vào nhóm Telegram, quản lý danh sách UID Facebook, tự động check trạng thái live/die theo chu kỳ, thông báo khi có UID DIE.
2. **Hệ thống Admin (web + backend)**: kiểm soát nhóm/khách nào được dùng bot (chống nhóm lạ tự thêm bot dùng free gây quá tải), cấp phép bằng **duyệt nhóm + license key + gói dịch vụ (quota)**.

Tham chiếu yêu cầu gốc của khách: `docs/CODE TOOL.xlsx` (có ảnh mô tả giao diện mong muốn).
Báo giá đã gửi khách: `docs/BAO GIA - BOT TELEGRAM CHECKLIVE.xlsx`.
**Hướng dẫn sử dụng hệ thống (Admin + người dùng bot): `docs/HUONG-DAN-SU-DUNG.md`.**

### Yêu cầu chức năng bot (từ khách)
- Lệnh: `/add UID|Tên`, `/addlist` (nhiều dòng `UID|Tên`), `/delete UID`, `/checkall`.
- Bot thêm được vào bất kỳ nhóm nào; check 1 phút/lần; **chỉ báo khi UID DIE** (còn sống thì im lặng).
- Thông báo dạng: `UID - Tên` / `Trạng thái: Die` / `ngày giờ`.
- Ảnh 1 trong xlsx: danh sách UID đang theo dõi (kết quả `/checkall`).
- Ảnh 2 trong xlsx: thẻ chi tiết 1 UID + nút `Cập nhật` / `Ẩn thông tin` / `Huỷ kèo` / `Done kèo` (workflow "kèo" nâng cao - CHƯA làm).

---

## 2. Quyết định quan trọng (đã chốt với khách)

- FB check: **tự build** (không thuê API bên thứ 3).
- Phạm vi: **full** (cả thẻ chi tiết + nút quản lý kèo như ảnh 2) - phần thẻ kèo hiện chưa implement.
- Quy mô: **lớn** (nhiều nhóm, hàng nghìn UID) -> cần **proxy pool** (Railway IP cố định, không tự xoay).
- Hạ tầng: deploy **Railway** (project riêng), Postgres + Redis add-on.
- Hệ thống Admin: cả **duyệt nhóm + license key** + **web dashboard đầy đủ (BE+FE)**.
- Đây là **dự án độc lập**: repo/DB/deploy riêng. `super-ad-card` chỉ dùng làm **nguồn tham khảo copy pattern**, KHÔNG code chung.

### Giá (tham khảo nội bộ)
- Bot core (ưu đãi khách quen): ~10 triệu VND.
- Hệ thống Admin (BE+FE): ~15-25 triệu VND. Trọn gói ~25-30 triệu.
- Vận hành/tháng (khách chịu): Railway ~5-25 USD + proxy ~1-5 triệu.

---

## 3. Repo & Git

- Backend: `/Users/mscmini/Documents/Tool/checklive-be` -> `git@github.com:votien1235/checklive-be.git` (branch `main`).
- Frontend: `/Users/mscmini/Documents/Tool/checklive-fe` -> `git@github.com:votien1235/checklive-fe.git` (branch `main`).
- Lưu ý git: SSH key push dưới tài khoản **votien1235** (key `~/.ssh/id_ed25519_votien`). `.env` KHÔNG commit (chỉ `.env.example`).

---

## 4. Backend (`checklive-be`)

**Stack:** NestJS 11 + Prisma 6 + PostgreSQL. Auth JWT (cookie + bearer), RBAC, Swagger tại `/api/v1/docs`. Prefix API: `/api/v1`.

> Lưu ý: đã cố tình DÙNG Prisma 6 (không phải 7) vì Prisma 7 bắt buộc driver-adapter, phức tạp khi tích hợp NestJS.

### Cấu trúc module (`src/`)
- `prisma/` - PrismaService/Module (global).
- `common/` - decorators (`roles`, `public`, `current-user`), `guards/roles.guard`, `filters/all-exceptions.filter`.
- `auth/` - login/register/logout/me, JWT strategy + guard.
- `users/` - CRUD user, approve/ban/role.
- `plans/` - CRUD gói dịch vụ.
- `customers/` - CRUD khách.
- `licenses/` - phát/thu hồi key + logic `activate(key, chatId)`.
- `bot-groups/` - list/approve/block/update nhóm + `getAccess(chatId)` (cổng kiểm soát truy cập).
- `bot/` - `telegram.service` (gửi tin), `telegram.controller` (webhook), `bot-update.service` (xử lý lệnh), `monitored-uids.service` (quản lý UID + quota), `fb-checker.service` (engine check FB), `checker.scheduler` (cron mỗi phút).

### Model dữ liệu (Prisma, `prisma/schema.prisma`)
- `User` (role ADMIN/SUB_ADMIN/USER, canUseApp, isSuperAdmin, parentUser hierarchy).
- `SiteSettings` (registrationEnabled), `ActivityLog`.
- `Customer`, `Plan` (maxUids, minCheckIntervalSec, maxGroups, durationDays, priceVnd).
- `License` (key, status ACTIVE/USED/REVOKED/EXPIRED, planId, activatedChatId, expiresAt).
- `BotGroup` (telegramChatId unique, status PENDING/APPROVED/BLOCKED, planId, customerId, checkIntervalSec override, expiresAt).
- `MonitoredUid` (uid, name, note, serviceFee, status LIVE/DIE/UNKNOWN, dealState WATCHING/DONE/CANCELLED, lastCheckedAt, lastDieNotifiedAt).
- `UsageStat` (theo ngày: checksCount, dieCount).

### Luồng kiểm soát truy cập
1. Bot được add vào nhóm -> webhook tạo `BotGroup` trạng thái **PENDING**, bot báo "chờ duyệt".
2. Kích hoạt bằng 1 trong 2 cách:
   - Admin duyệt nhóm trên web (`PATCH /bot-groups/:id/approve` + chọn gói) -> APPROVED + expiresAt.
   - User gõ `/activate KEY` trong nhóm -> `licenses.activate` set nhóm APPROVED theo gói của key.
3. Mọi lệnh check (`/add`, `/checkall`...) đều qua `getAccess(chatId)`: chỉ chạy khi APPROVED + chưa hết hạn. Thêm UID bị chặn theo `plan.maxUids` (quota).
4. Scheduler mỗi phút: quét nhóm APPROVED còn hạn, check UID tới kỳ (theo `checkIntervalSec` hoặc `plan.minCheckIntervalSec`), **chỉ gửi thông báo khi UID mới chuyển sang DIE**, ghi `UsageStat`.

### FB checker (`src/bot/fb-checker.service.ts`)
- Heuristic: gọi `graph.facebook.com/{uid}/picture?redirect=false`, `is_silhouette=true` => DIE.
- Có sẵn điểm cắm proxy (`PROXY_LIST`) + `CHECK_CONCURRENCY`. **Đấu nối proxy agent thực tế CHƯA làm** (cần cho quy mô lớn/anti-block).

### Biến môi trường (xem `.env.example`)
`DATABASE_URL`, `JWT_SECRET`, `JWT_EXPIRES_IN`, `TELEGRAM_BOT_TOKEN`, `TELEGRAM_WEBHOOK_SECRET`, `PROXY_LIST`, `CHECK_CONCURRENCY`, `SCHEDULER_ENABLED`, `FRONTEND_URL`, `PORT`, `SEED_ADMIN_EMAIL/PASSWORD`.

### Webhook Telegram
`POST /api/v1/telegram/webhook/<TELEGRAM_WEBHOOK_SECRET>`

### Chạy local
```bash
cd checklive-be
cp .env.example .env    # sửa DATABASE_URL, JWT_SECRET, TELEGRAM_BOT_TOKEN
npx prisma migrate dev
npm run prisma:seed     # tạo admin + 3 gói (Trial/Basic/Pro)
npm run start:dev       # http://localhost:3300, docs /api/v1/docs
```
Admin seed mặc định: `admin@checklive.local` / `admin12345`.

### Trạng thái DB local (máy dev hiện tại)
- Postgres brew `postgresql@14`, superuser `mscmini` (không mật khẩu).
- DB `checklive` đã tạo; `.env` local dùng `postgresql://mscmini@localhost:5432/checklive?schema=public`.
- Đã migrate + seed thành công; health + login đã test OK.

---

## 5. Frontend (`checklive-fe`)

**Stack:** Next.js 16 (App Router) + React 19 + Tailwind v4 + shadcn/ui + TanStack Query + Axios. Auth qua cookie JWT (`withCredentials`).

### Trang
- `src/app/login` - đăng nhập.
- `src/app/(admin)/` (có role guard trong `layout.tsx`): `dashboard`, `groups` (duyệt/khóa + gán gói), `licenses` (phát/thu hồi key), `plans` (tạo gói + quota), `users` (duyệt/ban).
- `src/hooks/use-auth.ts`, `src/hooks/use-resources.ts`.
- `src/lib/api.ts` (axios instance), `src/components/layout/sidebar.tsx`, `src/components/providers.tsx`.

### Env
`NEXT_PUBLIC_API_URL` (mặc định `http://localhost:3300/api/v1`). File `.env.local`.

### Chạy local
```bash
cd checklive-fe && npm run dev   # http://localhost:3000
```

---

## 6. Việc đang làm / còn lại (TODO)

- [ ] **Theme + font giống super-ad-card-fe** (Inter, shadcn baseColor `neutral`, tokens màu trong globals.css) - ĐANG YÊU CẦU, chưa làm.
  - super-ad-card-fe dùng Tailwind **v3** + shadcn `new-york`/`neutral`; checklive-fe đang Tailwind **v4** -> cần map tokens sang cú pháp v4 (`@theme`/oklch hoặc giữ hsl var).
- [ ] **i18n giống super-ad-card-fe**: custom context (`language-context.tsx` dùng `useSyncExternalStore`), `use-translation.ts`, `translations/index.ts` (vi/en), `language-selector` (cờ), persist `localStorage` key `app_language`, mặc định `vi`.
- [ ] Đấu nối **proxy agent thực tế** cho FB checker (undici ProxyAgent) + anti-block ở quy mô lớn.
- [ ] Trang **chi tiết nhóm + quản lý UID** và **thẻ kèo** (nút Cập nhật/Ẩn thông tin/Huỷ kèo/Done kèo - ảnh 2).
- [ ] Redis + tách worker khi scale (hiện scheduler chạy in-process).
- [ ] Deploy Railway (project riêng: admin-be, admin-fe, bot/worker + Postgres/Redis).
- [ ] Bổ sung endpoint usage/stats chi tiết cho dashboard.

---

## 7. Tham khảo super-ad-card (chỉ copy pattern, KHÔNG code chung)
- Backend `/Users/mscmini/Documents/super_ad_card/super-ad-card-be`: pattern auth JWT, RBAC, user CRUD, ActivityLog, Telegram service, Railway config.
- Frontend `/Users/mscmini/Documents/super_ad_card/super-ad-card-fe`: admin shell/sidebar, theme (Inter + shadcn neutral, Tailwind v3), i18n custom (vi/en), language selector.
- Chi tiết theme/i18n để nhân bản: xem mục 6.

---

## 8. Kế hoạch (plan files)
- `.cursor/plans/telegram_fb_checklive_bot_*.plan.md` - plan bot core.
- `.cursor/plans/bot_admin_management_system_*.plan.md` - plan hệ thống Admin.
