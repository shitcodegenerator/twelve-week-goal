# Project Context

## Purpose

**團購主訂單系統（Multi-tenant Group-Buy Order System）**

一個 Multi-tenant SaaS 作品，讓多個團購主共用同一套系統並做到資料隔離。
作為 12 週學習計劃的核心作品集，目標是可部署、可展示、可面試解說的 Next.js 全端應用。

主要功能：
- Public ordering：消費者可下單（無登入或輕量登入）
- Admin console：團購主後台管理商品 / 訂單 / 出貨 / 對帳
- LINE Integration：串接 LINE Messaging API / Webhook，推播訂單狀態
- Multi-tenant：多租戶資料隔離（shared schema + tenant_id）

## Tech Stack

- **Frontend**: Next.js (App Router), React, TypeScript (strict mode)
- **Backend**: Next.js API Routes
- **Database**: PostgreSQL + Prisma ORM
- **Validation**: Zod（request / response schema）
- **Auth**: Session cookie（httpOnly / secure / sameSite）
- **Integration**: LINE Messaging API / Webhook（HMAC 驗簽）
- **Container**: Docker（multi-stage Dockerfile）
- **CI/CD**: GitHub Actions（lint → typecheck → test → build）
- **Testing**: Vitest + React Testing Library（unit），Playwright（E2E smoke）
- **Logging**: 結構化 JSON log，含 request_id / correlation_id

## Project Conventions

### Code Style

- TypeScript `strict` mode 強制開啟，禁止 `any`
- ESLint + Prettier（統一格式）
- 命名：元件用 PascalCase，函式/變數用 camelCase，資料庫欄位用 snake_case
- 小檔案原則：單檔 200–400 行為宜，最多 800 行
- 不可變資料：永遠回傳新物件，不直接 mutate

### Architecture Patterns

路由結構（Next.js App Router）：
- `app/(public)/` — 消費者下單頁 `/t/[tenantSlug]/campaign/[campaignId]`
- `app/(host)/` — 團購主後台 `/host/dashboard`、`/host/orders`
- `app/api/` — API routes
- `lib/db/` — Prisma client
- `lib/auth/` — session / RBAC
- `lib/line/` — LINE SDK wrapper
- `lib/logger/` — structured logging
- `lib/validation/` — Zod schemas

Multi-tenant 資料隔離：
- 策略：shared database + shared schema + `tenant_id`（row-level isolation）
- 所有核心表必帶 `tenant_id`；Service 層強制 scope，禁止跨租戶查詢

核心資料模型：Tenant, Product/Variant, Campaign, Order/OrderItem, Customer, Payment, Shipment, LineChannel

API 錯誤合約：`{ "error_code": "...", "message": "...", "request_id": "uuid" }`

Repository Pattern：資料存取封裝在 repository 層，Service 層呼叫，禁止 Controller 直接操作 DB

RBAC：角色 Customer / Host / Staff / Admin，API + UI 雙層 guard

### Testing Strategy

- **Unit**：Vitest + React Testing Library → 元件 render、props、表單 validation、事件
- **Integration**：API contract test（status code / error format / pagination）+ DB CRUD（migration 後行為）
- **E2E**：Playwright smoke test → 登入 → dashboard → 建立資料 → 列表出現
- **CI**：失敗即擋 PR；目標可產生 coverage report
- **Webhook test**：驗簽 unit test 必做

### Git Workflow

- Conventional commits：`feat / fix / refactor / docs / test / chore / perf / ci`
- 每週 commit / PR 次數作為 lead 指標（KPI）
- ADR（Architecture Decision Records）記錄重大技術決策：
  - ADR-000：Multi-tenant 策略（shared schema + tenant_id）
  - ADR-001：Auth 策略（session vs JWT）
  - ADR-002：通知設計（sync vs async）
  - ADR-003：DB indexing 與查詢策略
  - ADR-004：Webhook 驗簽與事件處理策略
- 禁止 `--no-verify` 跳過 hooks

## Domain Context

作品背景：台灣電商團購場景，團購主（Host）在 LINE 官方帳號經營訂單，消費者透過公開連結下單，系統自動推播訂單狀態。

核心角色：
- **Customer**：外部下單者（無帳號或輕量登入）
- **Host**（團購主）：擁有後台管理功能
- **Staff**：Host 的團隊成員（可選）
- **Admin**：平台管理者（可選）

Order Status Machine：建立 → 待付款 → 已付款 → 出貨中 → 完成 / 取消

LINE 通知觸發點：訂單成立通知 Host、訂單狀態變更通知 Customer（需 LINE userId 綁定）

Tenant 辨識：path-based `/t/[tenantSlug]/...`（最簡單、最好 debug）

## Important Constraints

- **資料隔離**：任何查詢都必須帶 `tenant_id` scope，嚴禁跨租戶資料外洩（IDOR 防護）
- **Secrets 管理**：LINE channel secret/token 不得 hardcode，使用環境變數；不得 bake 進 Docker image
- **Idempotency**：`POST /api/public/orders` 必須支援 idempotency key，防止重複下單
- **Webhook 安全**：LINE webhook 必須 HMAC 驗簽，禁止未驗簽直接處理
- **Rate limiting**：public 下單 endpoint 與 LINE webhook endpoint 必須限流（429）
- **TypeScript strict**：禁止 `any`，所有 API request/response 都需要 Zod schema

## External Dependencies

- **PostgreSQL**：主要資料庫，Prisma ORM，migration 可重現
- **LINE Messaging API**：推播通知，每個 tenant 有獨立 channel secret/token
- **Docker**：容器化部署，multi-stage build，secrets 不進 image
- **GitHub Actions**：CI/CD，lint → typecheck → test → build
- **Vercel / Railway**：部署平台，preview + production 環境分離
