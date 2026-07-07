Dưới đây là 3 cách chạy repo này.

## 0) Chuẩn bị chung

```bash
npm install
cp .env.example .env
cp apps/admin/.env.example apps/admin/.env
cp apps/demo/.env.example apps/demo/.env
```

Nên sửa 3 token demo trong `.env` và copy đúng giá trị sang `apps/admin/.env`:

```env
DEMO_ADMIN_TOKEN=...
DEMO_DEVELOPER_TOKEN=...
DEMO_VIEWER_TOKEN=...
```

Tương ứng:

```env
VITE_DEMO_ADMIN_TOKEN=...
VITE_DEMO_DEVELOPER_TOKEN=...
VITE_DEMO_VIEWER_TOKEN=...
```

---

# Cách 1 — Không dùng Docker Compose

Dùng npm workspace để chạy backend, admin, demo. PostgreSQL có thể là local PostgreSQL hoặc container đơn bằng `docker run`.

## 1.1. Nếu dùng PostgreSQL local

Đảm bảo `.env` có:

```env
DATABASE_URL=postgresql://ffp:ffp_dev_password@localhost:5432/ffp_dev?schema=public
```

Tạo database/user tương ứng nếu chưa có.

## 1.2. Nếu không có PostgreSQL local, chạy container đơn

```bash
docker run --name ffp-postgres \
  -e POSTGRES_USER=ffp \
  -e POSTGRES_PASSWORD=ffp_dev_password \
  -e POSTGRES_DB=ffp_dev \
  -p 5432:5432 \
  -d postgres:16
```

Nếu container đã tồn tại:

```bash
docker start ffp-postgres
```

## 1.3. Migrate, generate, seed

```bash
npm run prisma:migrate --workspace=@ffp/backend
npm run prisma:generate --workspace=@ffp/backend
npm run db:seed --workspace=@ffp/backend
```

## 1.4. Chạy từng app trong 3 terminal

Terminal 1:

```bash
npm run dev:backend
```

Terminal 2:

```bash
npm run dev:admin
```

Terminal 3:

```bash
npm run dev:demo
```

Optional staging demo:

```bash
npm run dev:demo:staging
```

URLs:

```text
Backend API: http://localhost:3000/v1
Swagger:     http://localhost:3000/docs
Admin:       http://localhost:5173
Demo prod:   http://localhost:5174
Demo stage:  http://localhost:5175
```

---

# Cách 2 — Dùng Docker Compose toàn bộ

Cách này phù hợp để demo/presentation vì Compose tự chạy PostgreSQL, migrate, seed, backend, admin, demo production và demo staging.

```bash
cp .env.example .env
docker compose up --build
```

Hoặc chạy nền:

```bash
docker compose up --build -d
docker compose ps -a
```

Thứ tự service kỳ vọng:

```text
postgres healthy
-> migrate completed
-> demo-seed completed
-> backend healthy
-> admin/demo/demo-staging healthy
```

URLs:

```text
Backend API: http://localhost:3000/v1
Swagger:     http://localhost:3000/docs
Admin:       http://localhost:5173
Demo prod:   http://localhost:5174
Demo stage:  http://localhost:5175
```

Kiểm tra nhanh:

```bash
curl http://localhost:3000/v1/health
curl --head http://localhost:5173
curl --head http://localhost:5174
curl --head http://localhost:5175
```

Nếu port `5432` bị chiếm:

```bash
POSTGRES_HOST_PORT=55432 docker compose up --build
```

Dừng nhưng giữ data:

```bash
docker compose down
```

Xóa luôn volume database — cẩn thận vì mất data Compose:

```bash
docker compose down --volumes
```

---

# Cách 3 — Kết hợp Docker Compose + npm để dễ debug

Đây là cách mình khuyên dùng khi code/debug.

## Option A: Compose chỉ chạy PostgreSQL, code chạy local

Chạy database bằng Compose:

```bash
docker compose up -d postgres
```

Sau đó migrate/seed bằng npm local:

```bash
npm run prisma:migrate --workspace=@ffp/backend
npm run prisma:generate --workspace=@ffp/backend
npm run db:seed --workspace=@ffp/backend
```

Rồi chạy app local:

```bash
npm run dev:backend
npm run dev:admin
npm run dev:demo
```

Ưu điểm: backend hot reload, dễ đặt breakpoint/log, không phải rebuild image.

Nếu đổi port PostgreSQL Compose:

```bash
POSTGRES_HOST_PORT=55432 docker compose up -d postgres
```

Thì sửa `.env`:

```env
DATABASE_URL=postgresql://ffp:ffp_dev_password@localhost:55432/ffp_dev?schema=public
```

## Option B: Compose chạy PostgreSQL + backend, frontend chạy local

Chạy backend stack bằng Compose:

```bash
docker compose up --build -d postgres migrate demo-seed backend
```

Chạy admin/demo bằng npm:

```bash
npm run dev:admin
npm run dev:demo
npm run dev:demo:staging
```

Ưu điểm: backend giống môi trường demo hơn, frontend vẫn hot reload.

---

## Lệnh debug hữu ích

```bash
docker compose ps -a
docker compose logs postgres
docker compose logs migrate
docker compose logs demo-seed
docker compose logs backend
```

Mở Prisma Studio cho npm-local:

```bash
npm run prisma:studio --workspace=@ffp/backend
```

Mở Prisma Studio cho Compose DB:

```bash
docker compose --profile tools up --build prisma-studio
```

Rồi mở:

```text
http://localhost:5555
```