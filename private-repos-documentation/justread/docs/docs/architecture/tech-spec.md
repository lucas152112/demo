# 技術規格

## 技術棧

### 後端

| 項目 | 技術 | 版本 | 說明 |
|------|------|------|------|
| 語言 | Rust | 1.88 | 系統級效能、記憶體安全 |
| Web 框架 | Axum | 0.8 | 非同步 HTTP，Tower 生態 |
| 資料庫 ORM | sqlx | 0.8 | 非同步 MySQL，編譯期 SQL 驗證 |
| 密碼雜湊 | bcrypt | 0.15 | cost=12 |
| JWT | jsonwebtoken | 9 | RS256 演算法 |
| HTTP 客戶端 | reqwest | 0.12 | rustls-tls（無 openssl-sys 依賴） |
| Redis Pool | deadpool-redis | 0.18 | 非同步連線池 |
| 序列化 | serde / serde_json | 1 | |
| 時間 | chrono | 0.4 | UTC 時間處理 |
| UUID | uuid | 1 | v4 |
| API 文件 | utoipa + swagger-ui | 5 | 自動生成 OpenAPI 3.0 |
| Tracing | tracing / tracing-subscriber | 0.1 | 結構化日誌 |

### 前端

| 項目 | 技術 | 版本 | 說明 |
|------|------|------|------|
| 語言 | TypeScript | 5 | |
| 框架 | React | 18 | |
| 打包工具 | Vite | 5 | |
| 套件管理 | pnpm | 11 | workspace monorepo |
| 執行環境 | Node.js | 24 | Docker build image: node:24-alpine |
| HTTP 客戶端 | axios | 1 | |
| 路由 | react-router-dom | 6 | |

### 基礎設施

| 項目 | 技術 | 說明 |
|------|------|------|
| 容器 | Docker | multi-stage build |
| 容器映像 | debian:bookworm-slim | glibc，避免 musl DNS 問題 |
| 建構映像 | rust:1.88-slim-bookworm | |
| Orchestration | Kubernetes | k8s cluster |
| Container Registry | ghcr.io/lucas152112/ | GitHub Container Registry |
| 資料庫 | MySQL 8 | Namespace: database |
| 快取 / Session | Redis | Namespace: database |
| Email | Resend API | 驗證信寄送 |
| CI/CD | GitHub Actions（self-hosted runner） | 自動 test → build → push → deploy |
| Self-hosted Runner | k8s Node `103.251.113.34` | label: `[self-hosted, justread]` |

---

## 設計決策

### 1. debian:bookworm-slim runtime（非 Alpine）
**原因**：Alpine 使用 musl libc，k8s 叢集 DNS 解析（CoreDNS）在 musl 環境下失敗（`Name or service not known`）。改用 debian glibc 後解決。

### 2. rustls-tls（非 openssl-sys）
**原因**：openssl-sys 在 Alpine/musl 交叉編譯環境下 build 複雜。改用 rustls（純 Rust TLS）避免系統依賴。

### 3. RS256 JWT（非 HS256）
**原因**：公鑰可分發給其他服務做驗證，不需共享私鑰，符合微服務安全原則。

### 4. Refresh Token jti 存 Redis
**原因**：支援伺服器端 invalidation（登出、強制下線）。jti 格式：`refresh_jti:{jti}`，TTL = 7天。

### 5. admin_users 獨立資料表
**原因**：管理員與一般使用者職責分離，避免混淆權限邏輯。

### 6. Email Token 存 MySQL（非 Redis）
**原因**：驗證 token 需要關聯 user_id，且需要 expires_at 精確控制，MySQL 提供更好的查詢能力。

### 7. 前端 30 分鐘 Token 刷新
**原因**：access_token TTL 1小時，每 30 分鐘自動刷新確保持續使用者不被登出。由 `TokenRefresher` 元件（App.tsx）管理。

---

## 環境變數

### Auth Service

| 變數 | 來源 | 說明 |
|------|------|------|
| `DATABASE_URL` | ConfigMap `justread-config` | MySQL 連線字串 |
| `REDIS_URL` | ConfigMap `justread-config` | Redis 連線字串 |
| `RESEND_API_KEY` | Secret `justread-secrets` | Resend Email API |
| `JWT_PRIVATE_KEY` | Secret `justread-jwt-keys` | RSA 2048 私鑰 (PEM) |
| `JWT_PUBLIC_KEY` | Secret `justread-jwt-keys` | RSA 2048 公鑰 (PEM) |
| `PORT` | ConfigMap | 預設 8081 |

### 前端（管理後台）

| 儲存位置 | Key | 說明 |
|---------|-----|------|
| localStorage | `manage_token` | JWT access_token |
| localStorage | `manage_refresh_token` | JWT refresh_token |

---

## JWT Payload 規格

### Access Token
```json
{
  "sub": "admin_user_id",
  "username": "admin",
  "role": "admin",
  "iat": 1234567890,
  "exp": 1234567890
}
```

### Refresh Token
```json
{
  "sub": "admin_user_id",
  "jti": "uuid-v4",
  "type": "refresh",
  "iat": 1234567890,
  "exp": 1234567890
}
```

---

## CI/CD — GitHub Actions

### 架構概覽

```
push to develop  →  Test only (不部署)
push to main     →  Test → Build & Push → Deploy → 通知 Discord
```

### 檔案位置

| 路徑 | 說明 |
|------|------|
| `.github/workflows/ci.yml` | 主 workflow |
| `.github/actions/notify-discord/` | 自訂 composite action：失敗時發 Discord DM |

### Self-hosted Runner

| 項目 | 說明 |
|------|------|
| Runner 主機 | k8s Node `103.251.113.34` |
| Label | `[self-hosted, justread]` |
| 優點 | 直接存取 k8s cluster（不需 kubeconfig 傳遞）、GHCR docker cache 保留、無分鐘限額 |

### 執行流程（3 個 Stage）

#### Stage 1：Test（develop & main 都跑）

| Job | 內容 |
|-----|------|
| `test-frontend` | pnpm install → typecheck → build（front + manage） |
| `test-backend` | matrix: [auth, book, user, reader, notification] → `cargo check` |

- 使用 **matrix strategy**，5 個後端服務並行測試
- `fail-fast: true`：任一失敗立即停止其餘
- 每個 step 有 retry 2x 機制

#### Stage 2：Docker Build & Push（僅 main）

| Job | 內容 |
|-----|------|
| `docker-push-frontend` | build manage + front → push 到 GHCR |
| `docker-push` | matrix: 5 個後端服務 → build → push |

- Image 標籤策略：
  - `ghcr.io/lucas152112/justread-{service}:sha-{git_sha}` — 不可變版本標籤
  - `ghcr.io/lucas152112/justread-{service}:latest` — 便於手動 rollout

- Login 使用 `docker/login-action@v3`，credentials 來自 Secret `GHCR_TOKEN`

#### Stage 3：Deploy（僅 main，需 Stage 2 完成）

| Job | 內容 |
|-----|------|
| `deploy` | matrix: 5 個後端服務 → `kubectl set image` → `kubectl rollout status` |
| `deploy-frontend` | manage + front 的 rollout |
| `notify-success` | 全部成功後發 Discord DM |

- 使用 `sha-{git_sha}` tag 進行 rolling update，確保每次部署使用對應 commit 的 image
- `kubectl rollout status --timeout=120s` 確認部署完成
- `fail-fast: false`：各服務部署失敗不影響其他服務

#### 失敗通知

每個 Job 都有：
```yaml
- name: Notify on failure
  if: failure()
  uses: ./.github/actions/notify-discord
```

透過 Discord Bot 發送 DM，包含：
- 失敗的 Job 名稱
- Commit SHA
- Actions Run 連結

### Secrets 配置

| Secret 名稱 | 用途 |
|------------|------|
| `GHCR_TOKEN` | GitHub Container Registry push 權限（`write:packages`） |
| `DISCORD_BOT_TOKEN` | Discord Bot 發送通知 |
| `DISCORD_USER_ID` | 接收通知的 Discord 使用者 ID |

### Concurrency 控制

```yaml
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true
```

同一分支的新 push 會取消進行中的舊 run，避免多個部署同時進行造成 race condition。

### 設計決策

| 決策 | 原因 |
|------|------|
| Self-hosted runner | k8s 在內網，cloud runner 無法直接 `kubectl` |
| sha tag + latest tag 雙標籤 | sha tag 確保可追蹤回溯；latest 方便緊急手動操作 |
| Stage 3 fail-fast: false | 部分服務部署失敗不應阻擋其他服務更新 |
| Retry 2x | Docker build/push 偶發網路超時，retry 避免誤報 |
