# JustRead 開發日誌 (Diary)

> 格式：最新在上（倒敘），每次任務完成後記錄。

---

## 2026-05-13 Flutter Android App + OTA 自動更新服務

### 目標
建立 JustRead Flutter 跨平台 App（Android），整合自動更新 OTA 服務並部署至 k8s。

### 完成項目

#### Flutter App (`flutter/`)
- Flutter 3.22，dio + flutter_riverpod + go_router + flutter_secure_storage
- 登入/註冊（email+password）、書庫、我的書籍 CRUD、書籍編輯器（AI 續寫）、閱讀器
- 啟動後 2 秒自動檢查更新（`/app/version`），有新版顯示 Dialog + 下載連結
- Android: minSdk 21, targetSdk 34, packageId: cc.justit.justread

#### Rust OTA AppServer (`backend/appserver/`)
- `GET /app/version` — 最新版本 JSON（version, build_number, download_url, sha256）
- `GET /app/releases` — 版本歷史（保留最近 10 筆）
- `GET /app/download/:filename` — APK 下載
- `POST /app/upload?token=...` — GitHub Actions 上傳 APK
- 版本資料持久化到 `/data/apk/releases.json`（PVC）

#### K8s (`k8s/appserver.yaml`)
- Deployment `justread-appserver`，NodePort 30086，PVC 2Gi
- APP_UPLOAD_TOKEN 已加入 justread-secrets（JustReadApp2026）

#### GitHub Actions (`.github/workflows/flutter-android.yml`)
- trigger: push `flutter/**` 到 develop/main 或 workflow_dispatch
- build APK（ubuntu-latest, Flutter 3.22, Java 17）→ build-appserver → deploy-and-publish
- Discord 通知含 APK 下載連結

#### Nginx
- upstream `justread_appserver` → 103.251.113.34:30086
- `location /app/` proxy + client_max_body_size 200m

### 驗證
- appserver pod Running，`GET /app/releases` → `{"releases":[]}`
- nginx reload 成功
- `GET /app/` 下載頁面 HTTP 200，動態 SSR（版本更新後即時反映）

---

## 2026-05-13 — 前端用戶書籍 CRUD + AI 協助撰寫

### 目標
已登入的前端用戶可以新增電子書、管理章節，並使用 AI 協助撰寫摘要及內容。

### 變更摘要

**DB**
- `books` 表加 `owner_id UUID REFERENCES users(id) ON DELETE SET NULL`（nullable，管理後台書籍為 NULL）
- 建立 `idx_books_owner_id` 索引

**backend/auth**
- 新增 `routes/user_login.rs`：`POST /api/auth/user-login`（email + password 登入 users 表，驗證 is_verified）
- 回傳 `access_token`、`refresh_token`、`user_id`、`username`（role = "user"，JWT RS256）

**backend/book**
- 新增 `jsonwebtoken = "9"` 依賴
- 新增 `routes/user_books.rs`：用戶書籍完整 CRUD + 章節 CRUD + AI 協助
  - `GET/POST /api/user/books` — 列出/建立自己的書
  - `GET/PUT/DELETE /api/user/books/:id` — 書籍詳情/更新/刪除
  - `GET/POST /api/user/books/:book_id/chapters` — 章節列表/建立
  - `GET/PUT/DELETE /api/user/books/:book_id/chapters/:ch_id` — 章節操作
  - `POST /api/user/books/:book_id/chapters/ai-assist` — AI 協助（summary/content/continue 三種模式）
  - 所有路由從 `Authorization: Bearer` 驗證 JWT，確認 role = "user" 及 owner_id 所有權
- `justread-jwt-keys` secret 掛載到 book deployment

**nginx（跳板機）**
- 在 `/api/user/` 之前插入 `/api/user/books` + `/api/user/books/` → 導向 `justread_book`

**front**
- `LoginPage.tsx`：全改為真實 API（email + password 登入、email 驗證流程、完整錯誤提示）
- `Library.tsx`：auth 改用 `access_token`、加「我的書籍」按鈕
- 新增 `api/user_book.ts`：完整 user book API client
- 新增 `MyBooks.tsx`：書庫管理頁（列表、新增、刪除）
- 新增 `UserBookEditor.tsx`：三欄式編輯器（章節列表 | 內容編輯器 | AI 面板）
  - AI 三種模式：生成摘要 / 生成內容 / 續寫
  - 字數統計、儲存狀態顯示
- `routes.tsx`：加 `/my-books`、`/my-books/:bookId/edit`

### 流程
`/` (LoginPage) → 登入/註冊 → `/library` (書庫) → 「我的書籍」→ `/my-books` (MyBooks) → 「編輯」→ `/my-books/:id/edit` (UserBookEditor)

---

## 2026-05-12 — 電子書多語系支援

### 需求
- 電子書內容加語系標示（預設正體中文）
- 不同語系各自獨立 JSON 保存
- 語系可隨時增減
- 獨立語系管理 CRUD 頁面

### DB 變更
- `books` 加 `default_language VARCHAR(16) DEFAULT 'zh-TW'`
- 新建 `book_languages` 表：`(book_id, lang, lang_name)` UNIQUE
- 新建 `chapter_contents` 表：`(chapter_id, lang, content, content_json)` UNIQUE
- 既有章節內容遷移至 `chapter_contents` 的 `zh-TW` 記錄

### 後端 API（book service）
| Method | Path | 說明 |
|--------|------|------|
| GET | `/api/book/:id/languages` | 列出書籍語系 + default_language |
| POST | `/api/book/:id/languages` | 新增語系 (lang, lang_name) |
| DELETE | `/api/book/:id/languages/:lang` | 刪除語系（不可刪預設） |
| PUT | `/api/book/:id/default-language` | 設定預設語系 |
| GET | `/api/book/:id/chapters/:ch_id/content?lang=` | 取指定語系內容 |
| PUT | `/api/book/:id/chapters/:ch_id/content` | 儲存指定語系內容 |
| GET | `/api/book/:id/chapters/:ch_id/content/all` | 列出所有語系 |

### 前端
- 新增 `BookLanguages` 頁面（`/books/:id/languages`）
  - 語系清單表格：顯示名稱、代碼、預設狀態、新增時間
  - 新增語系：從常用語系選擇（zh-TW/zh-CN/en/ja/ko/fr/de/es）或自訂
  - 設為預設、刪除語系（預設語系不可刪）
- `BookEditor` 改版
  - Header 加「語系管理」快捷按鈕
  - 章節內容區改為語系 Tab：點 tab 切換語系並載入對應內容
  - 每個語系內容獨立儲存至 `chapter_contents`
  - AI 協助結果套用至當前語系 tab

---

## 2026-05-12 — AI 協助改接 OpenRouter

### 變更
- **後端 book service**：`ai_assist` handler 改用 OpenRouter API
  - endpoint: `https://openrouter.ai/api/v1/chat/completions`
  - model: `openrouter/free`（固定）
  - 讀取 k8s secret `OPENROUTER_API_KEY`
  - HTTP 錯誤或 model 異常時，回傳 `model_error: true` + 明確錯誤訊息
- **前端 manage BookEditor**：加入底部狀態欄
  - AI 生成中 → 藍底「AI 生成中…」
  - 成功 → 綠底「AI 生成完成，內容已套用」
  - model 異常 → 紅底「model 異常：{錯誤訊息}」
  - 可點 × 關閉
- **k8s secret** `justread-secrets` 新增 `OPENROUTER_API_KEY`

### 影響服務
- `justread-book:latest` ✅ rebuilt + pushed
- `justread-manage:latest` ✅ rebuilt + pushed

---

## 2026-05-12 — 管理後台登入修復

### 問題
- 登入後出現 `404 Not Found` — `window.location.href = '/login'` 缺少 `/manage` 前綴
- `admin_users` 在 PostgreSQL 中為空（MySQL 資料未遷移）
- login.rs 的 `SELECT id` 未加 `::text`（UUID 型別不相容）
- list.rs 的 `SELECT id` 未加 `::text`
- admin 帳號 hash 插入時被 shell `$` 展開截斷

### 修復
- `client.ts` + `App.tsx`：`/login` → `/manage/login`
- `login.rs`：`SELECT id::text`
- `list.rs`：`SELECT id::text`（兩條查詢）
- admin 帳號建立：htpasswd bcrypt hash，透過 SQL 檔案安全寫入（避免 shell 展開）
- admin 帳號密碼：`Admin@JustRead2026`

### 驗證
- `POST /api/auth/login` → access_token ✅
- `GET /api/book/list` → 書籍列表 ✅
- 管理後台 `https://justread.justit.cc/manage/` 登入後正常進入 Dashboard ✅

---

## 2026-05-12 — MySQL → PostgreSQL 遷移 + Chapter JSONB 儲存

### 完成項目
- **PostgreSQL DB 建立**：在 `postgre` namespace 建立 `justread` database，包含 `users`、`email_verifications`、`books`、`chapters` 完整 schema
- **auth 服務遷移**：`sqlx` features `mysql` → `postgres`，`MySqlPool` → `PgPool`，所有 SQL `?` → `$N`，UUID 欄位 `SELECT id::text`
- **book 服務遷移**：同上，chapters 新增 `content_json JSONB`（段落結構）及 `full_text TEXT`（完整文字原文）
- **k8s 更新**：`justread-config` DATABASE_HOST/PORT 改為 PostgreSQL，`justread-secrets` DATABASE_USER/PASSWORD 改為 postgres 帳號
- **驗證**：以 `lucasfom@gmail.com` 註冊 → 驗證信發送成功 → 點擊 token 驗證成功 (`is_verified = t`)
- **chapter content_json**：章節 CRUD 正常，JSONB 段落結構儲存於 `content_json`，原文保存於 `full_text`

### 技術重點
- PostgreSQL UUID 欄位對應 Rust `String` 需在 SQL 使用 `id::text` 轉型
- `email_verifications` 使用獨立資料表而非 users 欄位
- `TIMESTAMPTZ` 對應 `chrono::DateTime<Utc>`

---

## 2026-05-12 — 升級 Node.js 24 + pnpm 11.1.0

- `front/Dockerfile` & `manage/Dockerfile`: node:20-alpine → node:24-alpine
- pnpm 版本統一固定為 pnpm@11.1.0（本機 + Docker 一致）
- 根 `package.json` 加 `pnpm.onlyBuiltDependencies`（`@tailwindcss/oxide`, `esbuild`）
- `pnpm-lock.yaml` settings 更新 `onlyBuiltDependencies`
- Dockerfile install 改為 `pnpm install --frozen-lockfile || (pnpm approve-builds --all && pnpm install --frozen-lockfile)` 解決 pnpm v11 的 `ERR_PNPM_IGNORED_BUILDS`
- 新增根目錄 `.npmrc`（`allow-scripts=true`）
- build / push / rollout 兩個前端 deployment ✅

---

## 2026-05-11 (3) — 修正：新增電子書 405 Not Allowed

**問題描述：**
管理後台「新增書籍」保存失敗，瀏覽器收到 HTTP 405 Not Allowed。

**根本原因：**
跳板機 nginx 只有 `location /api/book/`（有尾斜線），但前端 axios `baseURL=/api`，create 打出的是 `POST /api/book`（無尾斜線），nginx 無匹配 → 落到 `location /` → 前端服務 → 405。

**異動：**
- 跳板機 `/etc/nginx/sites-enabled/justread` — 新增 `location = /api/book` 精確匹配
- `manage/nginx.conf` — 同步新增 `location = /api/book`
- rebuild/push `justread-manage`，reload 跳板機 nginx

**驗證：** `POST https://justread.justit.cc/api/book` → 201 ✅

---

## 2026-05-11 (2) — 電子書 CRUD + 章節管理 + Auth 驗證流程

### 異動內容
1. **book service 完整實作**
   - `POST /api/book` — 建立書籍
   - `GET /api/book/list` — 分頁列表
   - `GET /api/book/:id` — 書籍詳情（含章節列表）
   - `PUT /api/book/:id` — 更新書籍
   - `DELETE /api/book/:id` — 刪除書籍
   - `POST /api/book/upload` — .txt 上傳並拆分章節
   - `GET /api/book/:id/chapters` — 章節列表
   - `POST /api/book/:id/chapters` — 建立章節
   - `PUT /api/book/:id/chapters/:ch_id` — 更新章節
   - `DELETE /api/book/:id/chapters/:ch_id` — 刪除章節
   - `POST /api/book/:id/chapters/ai-assist` — AI 輔助寫作

2. **Bug 修正**
   - Axum 0.7 路由語法：`{id}` 改為 `:id`（0.7 不支援 `{param}` 語法）
   - 同一路徑多 method 需在同一 `.route()` 中定義，否則 merge 會覆蓋
   - `DATE_FORMAT(...)` alias 導致 sqlx `FromRow` 反序列化靜默失敗，改用直接 SELECT
   - `connect()` 改 `connect_lazy()` 避免啟動時強制 DB 連線

3. **Auth 驗證流程驗證**
   - `lucasfom@gmail.com` 完整測試：register → email 發送（Resend）→ verify-email → is_verified=1

### 影響範圍
- `backend/book/` — 全新服務
- `backend/book/src/routes/mod.rs` — 統一路由定義
- `backend/book/src/routes/chapters.rs` — DATE_FORMAT 移除

### 驗證結果
- ✅ 書籍 CRUD 全部端點正常
- ✅ 章節 CRUD 全部端點正常
- ✅ Auth 註冊驗證流程正常

---

## 2026-05-11 (1) — 修正：管理後台統計資料無法載入

**問題描述：**
Dashboard 顯示「無法載入統計資料」，Users/Books 頁面無法列表。

**根本原因（三個問題）：**
1. user/book service `list` handler 使用 `Json(_req)` 接收 GET 請求 → GET 無 body → HTTP 415
2. manage nginx.conf proxy 路徑錯誤：`/api/books/`、`/api/users/` → 應為 `/api/book/`、`/api/user/`
3. 前端讀取路徑錯誤：`u.data.total` → `u.total`、`res.data.data` → `res.users`

**異動檔案：**
- `backend/user/src/routes/list.rs` — `Json` 改 `Query`
- `backend/book/src/routes/list.rs` — 同上
- `manage/nginx.conf` — proxy 路徑修正
- `manage/src/pages/Dashboard.tsx` / `Users.tsx` / `Books.tsx` — 回應欄位修正

**驗證：** Dashboard 正常顯示，無 console 錯誤 ✅

---

## 2026-05-11 (0) — 修正：管理後台空白頁面

**問題描述：**
`https://justread.justit.cc/manage/` 頁面空白，assets 路徑錯誤（/assets/ 而非 /manage/assets/）。

**根本原因（三層）：**
1. Vite base path 未設定
2. Dockerfile COPY 到根目錄而非 `/manage/` 子目錄
3. 跳板機 nginx `proxy_pass` 尾部多 `/` 截掉了路徑前綴

**異動：**
- `manage/vite.config.ts` — `base: '/manage/'`
- `manage/Dockerfile` — `COPY dist /usr/share/nginx/html/manage`
- `manage/nginx.conf` — SPA fallback 修正
- `manage/src/App.tsx` — `basename="/manage"`
- 跳板機 nginx — 移除 `proxy_pass` 尾部 `/`

**驗證：** 登入頁正常，admin 登入成功 ✅

---

## 2026-05-10 — 文件建立

**新增檔案：**
- `README.md` — 專案總覽
- `docs/architecture/overview.md` — 三層架構說明
- `docs/architecture/tech-spec.md` — 技術棧版本表、設計決策
- `docs/api/README.md` — API 端點總覽
- `docs/deployment/README.md` — k8s 操作、部署流程
- `docs/development/README.md` — 本地開發指南
- `docs/development/TASKS.md` — 任務清單

---

## 2026-05-09 — 管理後台前端實作

**異動：**
- `manage/src/pages/Login.tsx` — 登入頁，呼叫 `/api/auth/login`
- `manage/src/pages/Dashboard.tsx` — 統計卡片
- `manage/src/pages/Users.tsx` — 使用者列表
- `manage/src/pages/Books.tsx` — 書籍列表
- `manage/src/components/Layout.tsx` — Sidebar
- `manage/src/api/auth.ts` — loginAdmin、refreshToken
- `manage/src/api/client.ts` — axios + Bearer token
- `manage/src/App.tsx` — TokenRefresher（每 30 分鐘）+ PrivateRoute

---

## 2026-05-08 — Auth Service：JWT RS256 + 管理員登入

**異動：**
- `backend/auth/src/routes/login.rs` — 管理員登入，RS256 JWT，access 1h + refresh 7d
- `backend/auth/src/routes/refresh.rs` — refresh token 驗證，簽發新 access token
- k8s Secret `justread-jwt-keys` — RSA 2048 金鑰對
- MySQL `admin_users` 資料表 + 預設帳號（admin / aZ123456）

**設計決策：**
- RS256：公鑰可分發，不需共享私鑰
- refresh jti 存 Redis：支援伺服器端 invalidation

---

## 2026-05-07 — Auth Service：Email 驗證流程

**異動：**
- `backend/auth/src/email.rs` — ResendClient（reqwest + rustls）
- `backend/auth/src/routes/register.rs` — 註冊 + 寄驗證信
- `backend/auth/src/routes/verify_email.rs` — token 驗證，回傳 HTML
- `backend/auth/Dockerfile` — Alpine → debian:bookworm-slim

**關鍵修正：** Alpine musl libc 在 k8s CoreDNS 環境 DNS 解析失敗，改用 debian glibc 解決。

---

## 2026-05-06 — 專案初始化

- 建立 k8s namespace `justread`
- 7 個微服務骨架：auth / book / user / reader / notification / front / manage
- ConfigMap、Secret、Deployment、Service 配置
- GHCR registry 設定
- 跳板機 nginx 反向代理設定
