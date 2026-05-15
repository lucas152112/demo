# Email 驗證註冊流程 Implementation Plan

> **For Hermes:** Use subagent-driven-development skill to implement this plan task-by-task.

**Goal:** 前端使用者註冊時填寫 email，透過 Resend API 發送驗證信，24小時內點擊連結完成驗證後才可登入。

**Architecture:**
- auth 微服務處理 register / verify-email 邏輯，儲存 pending_verifications 到 PostgreSQL
- 呼叫 notification 微服務發送驗證信（notification 整合 Resend API）
- 前端 LoginPage 擴充 email 欄位 + 驗證等待畫面 + verify-email 路由

**Tech Stack:** Rust + Axum, Resend API (reqwest), PostgreSQL (sqlx), React + TypeScript

**寄件設定:**
- From: `support@justit.cc`
- Resend 帳號: `feng152112@gmail.com`
- RESEND_API_KEY 透過環境變數注入

---

## Task 1: notification 服務加入 Resend 發信功能

**Objective:** 新增 `POST /api/notification/email` 端點，呼叫 Resend API 發送 HTML 信件

**Files:**
- Modify: `backend/notification/Cargo.toml` — 加入 reqwest, tokio
- Create: `backend/notification/src/email.rs` — Resend client
- Modify: `backend/notification/src/routes/mod.rs` — 加入 email route
- Create: `backend/notification/src/routes/email.rs` — handler

**Step 1: 更新 Cargo.toml**

```toml
[dependencies]
# ... 現有依賴 ...
reqwest = { version = "0.12", features = ["json"] }
tokio = { version = "1", features = ["full"] }
```

**Step 2: 建立 `src/email.rs`**

```rust
use reqwest::Client;
use serde::Serialize;

#[derive(Debug, Serialize)]
pub struct SendEmailRequest {
    pub from: String,
    pub to: Vec<String>,
    pub subject: String,
    pub html: String,
}

pub struct ResendClient {
    client: Client,
    api_key: String,
}

impl ResendClient {
    pub fn new(api_key: String) -> Self {
        Self { client: Client::new(), api_key }
    }

    pub async fn send(&self, req: SendEmailRequest) -> Result<(), String> {
        let res = self.client
            .post("https://api.resend.com/emails")
            .bearer_auth(&self.api_key)
            .json(&req)
            .send()
            .await
            .map_err(|e| e.to_string())?;

        if res.status().is_success() {
            Ok(())
        } else {
            let body = res.text().await.unwrap_or_default();
            Err(format!("Resend error: {body}"))
        }
    }
}
```

**Step 3: 建立 `src/routes/email.rs`**

```rust
use axum::{routing::post, Router, Json};
use serde::{Deserialize, Serialize};
use utoipa::ToSchema;
use crate::email::{ResendClient, SendEmailRequest};

#[derive(Debug, Deserialize, ToSchema)]
pub struct SendEmailBody {
    pub to: String,
    pub subject: String,
    pub html: String,
}

#[derive(Debug, Serialize, ToSchema)]
pub struct SendEmailResponse {
    pub ok: bool,
    pub message: String,
}

pub fn router() -> Router {
    Router::new().route("/api/notification/email", post(handle))
}

/// 發送 Email（內部服務使用）
#[utoipa::path(
    post,
    path = "/api/notification/email",
    tag = "notification",
    request_body = SendEmailBody,
    responses(
        (status = 200, description = "成功", body = SendEmailResponse),
        (status = 500, description = "發信失敗"),
    )
)]
pub async fn handle(Json(body): Json<SendEmailBody>) -> Json<SendEmailResponse> {
    let api_key = std::env::var("RESEND_API_KEY").unwrap_or_default();
    let client = ResendClient::new(api_key);

    let req = SendEmailRequest {
        from: "support@justit.cc".to_string(),
        to: vec![body.to],
        subject: body.subject,
        html: body.html,
    };

    match client.send(req).await {
        Ok(_) => Json(SendEmailResponse { ok: true, message: "sent".into() }),
        Err(e) => {
            tracing::error!("email send failed: {e}");
            Json(SendEmailResponse { ok: false, message: e })
        }
    }
}
```

**Step 4: 在 `src/routes/mod.rs` 加入 email module**

```rust
pub mod email;
// 並在 router() 中 .merge(email::router())
```

**Step 5: 更新 `backend/notification/.env.example`**

```
PORT=8085
RUST_LOG=info
DATABASE_URL=postgres://user:***@localhost/justread_notification
RESEND_API_KEY=re_your_key_here
```

**Step 6: Build 驗證**

```bash
cd ~/justread/backend/notification
cargo build 2>&1
```
Expected: 編譯成功

**Step 7: Commit**

```bash
git add backend/notification/
git commit -m "feat(notification): add Resend email sender endpoint"
```

---

## Task 2: auth DB — 建立 pending_verifications table migration

**Objective:** 建立 email 驗證 token 的儲存 schema

**Files:**
- Create: `backend/auth/migrations/001_pending_verifications.sql`

**Step 1: 建立 migration**

```sql
-- backend/auth/migrations/001_pending_verifications.sql
CREATE TABLE IF NOT EXISTS pending_verifications (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email       VARCHAR(255) NOT NULL,
    username    VARCHAR(100) NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    token       VARCHAR(64)  NOT NULL UNIQUE,
    expires_at  TIMESTAMPTZ  NOT NULL,
    created_at  TIMESTAMPTZ  NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_pending_verifications_token ON pending_verifications(token);
CREATE INDEX idx_pending_verifications_email ON pending_verifications(email);
```

**Step 2: Commit**

```bash
git add backend/auth/migrations/
git commit -m "feat(auth): add pending_verifications migration"
```

---

## Task 3: auth 服務加入 sqlx + bcrypt 依賴

**Objective:** auth 服務能連接 PostgreSQL 並 hash 密碼

**Files:**
- Modify: `backend/auth/Cargo.toml`
- Modify: `backend/auth/src/main.rs` — 建立 DB pool, 注入 AppState

**Step 1: 更新 `Cargo.toml`**

```toml
sqlx = { version = "0.8", features = ["postgres", "runtime-tokio-native-tls", "uuid", "chrono", "migrate"] }
bcrypt = "0.15"
reqwest = { version = "0.12", features = ["json"] }
uuid = { version = "1", features = ["v4", "serde"] }
chrono = { version = "0.4", features = ["serde"] }
rand = "0.8"
```

**Step 2: 更新 `src/main.rs` — 建立 AppState**

```rust
use sqlx::PgPool;
use std::sync::Arc;

#[derive(Clone)]
pub struct AppState {
    pub db: PgPool,
    pub notification_url: String,
    pub frontend_url: String,
}

#[tokio::main]
async fn main() {
    // ... tracing init ...

    let database_url = std::env::var("DATABASE_URL").expect("DATABASE_URL required");
    let db = PgPool::connect(&database_url).await.expect("DB connect failed");
    sqlx::migrate!("./migrations").run(&db).await.expect("migration failed");

    let state = Arc::new(AppState {
        db,
        notification_url: std::env::var("NOTIFICATION_SERVICE_URL")
            .unwrap_or_else(|_| "http://localhost:8085".into()),
        frontend_url: std::env::var("FRONTEND_URL")
            .unwrap_or_else(|_| "http://localhost:3000".into()),
    });

    let app = Router::new()
        .merge(SwaggerUi::new("/swagger-ui").url("/api-docs/openapi.json", ApiDoc::openapi()))
        .merge(routes::router(state))
        .layer(CorsLayer::permissive());

    // ... listen ...
}
```

**Step 3: 更新 `.env.example`**

```
PORT=8081
RUST_LOG=info
DATABASE_URL=postgres://user:***@localhost/justread_auth
NOTIFICATION_SERVICE_URL=http://notification:8085
FRONTEND_URL=http://localhost:3000
```

**Step 4: Build 驗證**

```bash
cd ~/justread/backend/auth && cargo build
```

**Step 5: Commit**

```bash
git commit -am "feat(auth): add sqlx pool + AppState"
```

---

## Task 4: 實作 register handler — 儲存 pending + 觸發發信

**Objective:** POST `/api/auth/register` 寫入 pending_verifications，呼叫 notification 服務發驗證信

**Files:**
- Modify: `backend/auth/src/routes/register.rs`
- Modify: `backend/auth/src/routes/mod.rs` — 傳入 state

**Step 1: 更新 `routes/mod.rs`**

```rust
use std::sync::Arc;
use crate::AppState;

pub fn router(state: Arc<AppState>) -> Router {
    Router::new()
        .route("/health", get(health))
        .merge(login::router(state.clone()))
        .merge(register::router(state.clone()))
        .merge(refresh::router())
}
```

**Step 2: 更新 `routes/register.rs`**

```rust
use axum::{extract::State, routing::post, Router, Json, http::StatusCode};
use serde::{Deserialize, Serialize};
use utoipa::ToSchema;
use std::sync::Arc;
use crate::AppState;

#[derive(Debug, Deserialize, ToSchema)]
pub struct RegisterRequest {
    pub username: String,
    pub password: String,
    pub email: String,
}

#[derive(Debug, Serialize, ToSchema)]
pub struct RegisterResponse {
    pub message: String,
}

pub fn router(state: Arc<AppState>) -> Router {
    Router::new()
        .route("/api/auth/register", post(handle))
        .with_state(state)
}

pub async fn handle(
    State(state): State<Arc<AppState>>,
    Json(req): Json<RegisterRequest>,
) -> Result<Json<RegisterResponse>, (StatusCode, Json<serde_json::Value>)> {
    // 1. 檢查 email 格式（基本檢查）
    if !req.email.contains('@') {
        return Err((StatusCode::BAD_REQUEST, Json(serde_json::json!({"error": "email 格式錯誤"}))));
    }

    // 2. 檢查 email 是否已被使用（pending 表）
    let exists: Option<(String,)> = sqlx::query_as(
        "SELECT email FROM pending_verifications WHERE email = $1 AND expires_at > NOW()"
    )
    .bind(&req.email)
    .fetch_optional(&state.db)
    .await
    .map_err(|e| (StatusCode::INTERNAL_SERVER_ERROR, Json(serde_json::json!({"error": e.to_string()}))))?;

    if exists.is_some() {
        return Err((StatusCode::CONFLICT, Json(serde_json::json!({"error":" 此 email 已有待驗證的註冊申請"}))));
    }

    // 3. hash 密碼
    let password_hash = bcrypt::hash(&req.password, bcrypt::DEFAULT_COST)
        .map_err(|e| (StatusCode::INTERNAL_SERVER_ERROR, Json(serde_json::json!({"error": e.to_string()}))))?;

    // 4. 產生 token (64 hex chars)
    let token: String = {
        use rand::Rng;
        let bytes: [u8; 32] = rand::thread_rng().gen();
        hex::encode(bytes)  // 需加 hex crate，或用 uuid
    };
    // 或用 uuid: let token = uuid::Uuid::new_v4().to_string().replace('-', "");

    // 5. 寫入 pending_verifications（expires 24h）
    sqlx::query(
        "INSERT INTO pending_verifications (email, username, password_hash, token, expires_at)
         VALUES ($1, $2, $3, $4, NOW() + INTERVAL '24 hours')"
    )
    .bind(&req.email)
    .bind(&req.username)
    .bind(&password_hash)
    .bind(&token)
    .execute(&state.db)
    .await
    .map_err(|e| (StatusCode::INTERNAL_SERVER_ERROR, Json(serde_json::json!({"error": e.to_string()}))))?;

    // 6. 呼叫 notification 發信
    let verify_url = format!("{}/verify-email?token={}", state.frontend_url, token);
    let html = format!(r#"
        <h2>驗證您的 JustRead 帳號</h2>
        <p>您好，{username}！</p>
        <p>請點擊以下連結完成 email 驗證（24小時內有效）：</p>
        <p><a href="{url}" style="background:#000;color:#fff;padding:12px 24px;text-decoration:none;display:inline-block;">驗證 Email</a></p>
        <p>或複製此連結：{url}</p>
        <p>如果您沒有註冊 JustRead，請忽略此信。</p>
    "#, username = req.username, url = verify_url);

    let _ = reqwest::Client::new()
        .post(format!("{}/api/notification/email", state.notification_url))
        .json(&serde_json::json!({
            "to": req.email,
            "subject": "驗證您的 JustRead 帳號",
            "html": html
        }))
        .send()
        .await;  // 發信失敗不中斷流程，log 記錄即可

    Ok(Json(RegisterResponse {
        message: "驗證信已寄出，請在24小時內完成驗證".into(),
    }))
}
```

> **注意:** 用 uuid token 替代 hex（避免加 hex crate）：
> ```rust
> let token = uuid::Uuid::new_v4().to_string().replace('-', "");
> ```

**Step 5: Commit**

```bash
git commit -am "feat(auth): implement register with email verification pending"
```

---

## Task 5: 實作 verify-email handler

**Objective:** GET `/api/auth/verify-email?token=xxx` — 驗證 token，建立正式 user，清除 pending

**Files:**
- Create: `backend/auth/src/routes/verify_email.rs`
- Modify: `backend/auth/src/routes/mod.rs`
- Create: `backend/auth/migrations/002_users.sql`

**Step 1: 建立 users table migration**

```sql
-- backend/auth/migrations/002_users.sql
CREATE TABLE IF NOT EXISTS users (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    username    VARCHAR(100) NOT NULL,
    email       VARCHAR(255) NOT NULL UNIQUE,
    password_hash VARCHAR(255) NOT NULL,
    email_verified BOOLEAN NOT NULL DEFAULT TRUE,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_users_email ON users(email);
```

**Step 2: 建立 `src/routes/verify_email.rs`**

```rust
use axum::{extract::{Query, State}, routing::get, Router, Json, http::StatusCode};
use serde::{Deserialize, Serialize};
use std::sync::Arc;
use crate::AppState;

#[derive(Deserialize)]
pub struct VerifyQuery {
    pub token: String,
}

#[derive(Serialize)]
pub struct VerifyResponse {
    pub message: String,
}

pub fn router(state: Arc<AppState>) -> Router {
    Router::new()
        .route("/api/auth/verify-email", get(handle))
        .with_state(state)
}

pub async fn handle(
    State(state): State<Arc<AppState>>,
    Query(q): Query<VerifyQuery>,
) -> Result<Json<VerifyResponse>, (StatusCode, Json<serde_json::Value>)> {
    // 1. 查詢 pending
    let row: Option<(String, String, String)> = sqlx::query_as(
        "SELECT email, username, password_hash FROM pending_verifications
         WHERE token = $1 AND expires_at > NOW()"
    )
    .bind(&q.token)
    .fetch_optional(&state.db)
    .await
    .map_err(|e| (StatusCode::INTERNAL_SERVER_ERROR, Json(serde_json::json!({"error": e.to_string()}))))?;

    let (email, username, password_hash) = match row {
        Some(r) => r,
        None => return Err((StatusCode::BAD_REQUEST, Json(serde_json::json!({"error": "驗證連結無效或已過期"})))),
    };

    // 2. 寫入 users（忽略重複）
    sqlx::query(
        "INSERT INTO users (username, email, password_hash)
         VALUES ($1, $2, $3)
         ON CONFLICT (email) DO NOTHING"
    )
    .bind(&username)
    .bind(&email)
    .bind(&password_hash)
    .execute(&state.db)
    .await
    .map_err(|e| (StatusCode::INTERNAL_SERVER_ERROR, Json(serde_json::json!({"error": e.to_string()}))))?;

    // 3. 刪除 pending
    sqlx::query("DELETE FROM pending_verifications WHERE token = $1")
        .bind(&q.token)
        .execute(&state.db)
        .await
        .ok();

    Ok(Json(VerifyResponse { message: "Email 驗證成功，請重新登入".into() }))
}
```

**Step 3: 在 `routes/mod.rs` 加入 verify_email**

```rust
pub mod verify_email;
// router() 中加入 .merge(verify_email::router(state.clone()))
```

**Step 4: Commit**

```bash
git commit -am "feat(auth): implement verify-email endpoint"
```

---

## Task 6: 前端 — RegisterPage 加入 email 欄位 + 發送後等待畫面

**Objective:** 註冊表單加入 email 欄位，提交後呼叫 auth API，顯示「請查看驗證信」等待畫面

**Files:**
- Modify: `front/src/app/components/LoginPage.tsx`
- Create: `front/src/app/api/auth.ts` — API client

**Step 1: 建立 `src/app/api/auth.ts`**

```typescript
const AUTH_URL = import.meta.env.VITE_AUTH_URL ?? 'http://localhost:8081';

export async function registerWithEmail(data: {
  username: string;
  password: string;
  email: string;
}): Promise<{ message: string }> {
  const res = await fetch(`${AUTH_URL}/api/auth/register`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(data),
  });
  const json = await res.json();
  if (!res.ok) throw new Error(json.error ?? '註冊失敗');
  return json;
}

export async function login(data: {
  username: string;
  password: string;
}): Promise<{ access_token: string; refresh_token: string }> {
  const res = await fetch(`${AUTH_URL}/api/auth/login`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(data),
  });
  const json = await res.json();
  if (!res.ok) throw new Error(json.error ?? '登入失敗');
  return json;
}
```

**Step 2: 更新 `LoginPage.tsx` — 加入 email + pending 狀態**

新增 state:
```tsx
const [email, setEmail] = useState('');
const [pendingVerification, setPendingVerification] = useState(false);
const [pendingEmail, setPendingEmail] = useState('');
```

在 `isRegister` 的表單中加入 email 欄位：
```tsx
{isRegister && (
  <div className="space-y-2">
    <Label htmlFor="email">Email</Label>
    <Input
      id="email"
      type="email"
      value={email}
      onChange={(e) => setEmail(e.target.value)}
      className="border-2 border-black shadow-none rounded-none"
      placeholder="請輸入 email"
    />
  </div>
)}
```

更新 `handleSubmit` 的 register 分支：
```tsx
if (isRegister) {
  if (!email) { setError('請輸入 Email'); return; }
  try {
    await registerWithEmail({ username, password, email });
    setPendingEmail(email);
    setPendingVerification(true);
  } catch (err: unknown) {
    setError(err instanceof Error ? err.message : '註冊失敗');
  }
  return;
}
```

**Step 3: 加入驗證等待畫面**（在 return 的開頭加入）

```tsx
if (pendingVerification) {
  return (
    <div className="size-full flex items-center justify-center bg-white">
      <Card className="w-full max-w-md mx-4 border-2 border-black shadow-none">
        <CardHeader className="text-center">
          <div className="flex justify-center mb-4">
            <BookOpen className="w-16 h-16" strokeWidth={1.5} />
          </div>
          <CardTitle className="text-2xl">驗證信已寄出</CardTitle>
          <CardDescription className="text-base mt-2">
            請檢查 <strong>{pendingEmail}</strong> 的信箱<br/>
            點擊驗證連結後即可登入（24小時內有效）
          </CardDescription>
        </CardHeader>
        <CardContent>
          <Button
            className="w-full border-2 border-black rounded-none shadow-none"
            variant="outline"
            onClick={() => { setPendingVerification(false); setIsRegister(false); }}
          >
            返回登入
          </Button>
        </CardContent>
      </Card>
    </div>
  );
}
```

**Step 4: 加入 `VITE_AUTH_URL` 到 `.env.example`**

```
VITE_AUTH_URL=http://localhost:8081
```

**Step 5: Commit**

```bash
git commit -am "feat(front): add email field to register + pending verification screen"
```

---

## Task 7: 前端 — verify-email 路由頁面

**Objective:** 使用者點擊驗證信連結後，前端呼叫後端 verify-email API，顯示成功/失敗結果

**Files:**
- Create: `front/src/app/components/VerifyEmailPage.tsx`
- Modify: `front/src/app/routes.tsx` (或 routes.ts) — 加入 `/verify-email` 路由

**Step 1: 建立 `VerifyEmailPage.tsx`**

```tsx
import { useEffect, useState } from 'react';
import { useSearchParams, useNavigate } from 'react-router';
import { BookOpen } from 'lucide-react';
import { Card, CardContent, CardHeader, CardTitle, CardDescription } from './ui/card';
import { Button } from './ui/button';

const AUTH_URL = import.meta.env.VITE_AUTH_URL ?? 'http://localhost:8081';

export function VerifyEmailPage() {
  const [params] = useSearchParams();
  const navigate = useNavigate();
  const [status, setStatus] = useState<'loading' | 'success' | 'error'>('loading');
  const [message, setMessage] = useState('');

  useEffect(() => {
    const token = params.get('token');
    if (!token) { setStatus('error'); setMessage('缺少驗證 token'); return; }

    fetch(`${AUTH_URL}/api/auth/verify-email?token=${token}`)
      .then(res => res.json())
      .then(data => {
        if (data.message) { setStatus('success'); setMessage(data.message); }
        else { setStatus('error'); setMessage(data.error ?? '驗證失敗'); }
      })
      .catch(() => { setStatus('error'); setMessage('網路錯誤，請稍後再試'); });
  }, [params]);

  return (
    <div className="size-full flex items-center justify-center bg-white">
      <Card className="w-full max-w-md mx-4 border-2 border-black shadow-none">
        <CardHeader className="text-center">
          <div className="flex justify-center mb-4">
            <BookOpen className="w-16 h-16" strokeWidth={1.5} />
          </div>
          <CardTitle className="text-2xl">
            {status === 'loading' && '驗證中...'}
            {status === 'success' && '驗證成功 ✓'}
            {status === 'error' && '驗證失敗'}
          </CardTitle>
          <CardDescription className="text-base mt-2">{message}</CardDescription>
        </CardHeader>
        {status !== 'loading' && (
          <CardContent>
            <Button
              className="w-full bg-black text-white hover:bg-gray-800 rounded-none shadow-none"
              onClick={() => navigate('/')}
            >
              前往登入
            </Button>
          </CardContent>
        )}
      </Card>
    </div>
  );
}
```

**Step 2: 在路由配置加入 `/verify-email`**

找到 `front/src/app/routes.tsx`（或 routes.ts），加入：
```tsx
import { VerifyEmailPage } from './components/VerifyEmailPage';

// 在 routes array 加入:
{ path: '/verify-email', element: <VerifyEmailPage /> }
```

**Step 3: Commit**

```bash
git commit -am "feat(front): add verify-email page"
```

---

## Task 8: k8s / CI 環境變數更新

**Objective:** k8s secret / configmap 加入新的環境變數

**Files:**
- Modify: `k8s/auth/` — 加入 NOTIFICATION_SERVICE_URL, FRONTEND_URL
- Modify: `k8s/notification/` — 加入 RESEND_API_KEY (Secret)

**Step 1: 確認現有 k8s 配置**

```bash
ls ~/justread/k8s/
```

**Step 2: 在 notification deployment 加入 Secret**

```yaml
# k8s/notification/secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: notification-secret
  namespace: justread
type: Opaque
stringData:
  RESEND_API_KEY: "re_your_actual_key"  # 部署時替換
```

**Step 3: 在 auth deployment 加入 env**

```yaml
env:
  - name: NOTIFICATION_SERVICE_URL
    value: "http://notification-svc:8085"
  - name: FRONTEND_URL
    value: "https://justread.justit.cc"
```

**Step 4: Commit**

```bash
git commit -am "feat(k8s): add email verification env vars"
```

---

## 整體流程確認

```
使用者填寫 username / email / password
  → POST /api/auth/register
    → bcrypt hash password
    → INSERT pending_verifications (token, expires NOW+24h)
    → POST notification/api/notification/email (Resend API)
      → 寄信 from: support@justit.cc
      → 信中含 https://justread.justit.cc/verify-email?token=xxx
  → 前端顯示「請查看信箱」畫面

使用者點擊信中連結
  → GET /api/auth/verify-email?token=xxx
    → 查詢 pending_verifications (token 存在 + 未過期)
    → INSERT users
    → DELETE pending_verifications
  → 前端 VerifyEmailPage 顯示「驗證成功，前往登入」
```

---

## 環境變數總覽

| 服務 | 變數 | 值（生產） |
|------|------|-----------|
| notification | RESEND_API_KEY | re_xxx (Resend dashboard) |
| auth | NOTIFICATION_SERVICE_URL | http://notification-svc:8085 |
| auth | FRONTEND_URL | https://justread.justit.cc |
| front | VITE_AUTH_URL | https://justread.justit.cc (nginx proxy) |
