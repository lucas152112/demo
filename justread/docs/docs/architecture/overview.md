# 架構總覽

## 三層架構

```
┌─────────────────────────────────────────────────────────┐
│                     Layer 1: 前端層                      │
│   front (React/Vite :3000)   manage (React/Vite :3001)  │
└──────────────────────┬──────────────────────────────────┘
                       │ HTTP API
┌──────────────────────▼──────────────────────────────────┐
│                   Layer 2: 後端微服務層                   │
│  auth:8081  book:8082  user:8083  reader:8084  notif:8085│
└──────────────────────┬──────────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────────┐
│                   Layer 3: 資料層                         │
│     MySQL (justread DB)    Redis     (database ns)        │
└─────────────────────────────────────────────────────────┘
```

---

## 微服務責任劃分

### Auth Service (`backend/auth`)
- 使用者**註冊**：建立帳號、寄驗證信（Resend API）
- **Email 驗證**：token 存 MySQL，點擊後標記 `is_verified=1`
- **管理員登入**：bcrypt 驗證、簽發 RS256 JWT
- **Token 刷新**：refresh token jti 存 Redis（TTL 7d）
- JWT 公私鑰存於 k8s Secret `justread-jwt-keys`

### Book Service (`backend/book`)
- 書籍列表、詳情、上傳、刪除
- Swagger UI 掛載於 `/swagger-ui/`

### User Service (`backend/user`)
- 用戶 Profile 查詢、更新
- 使用者列表（管理用）

### Reader Service (`backend/reader`)
- 閱讀進度（progress）
- 書籤管理（bookmark）
- 閱讀歷史（history）
- 多裝置同步（sync）

### Notification Service (`backend/notification`)
- 通知列表
- 標記已讀
- 發送通知（系統觸發）

---

## 認證流程

```
前台/管理後台
    │
    ├─ POST /api/auth/register  →  建立用戶 + 寄驗證信
    ├─ GET  /api/auth/verify-email?token=xxx  →  驗證 email
    ├─ POST /api/auth/login     →  取得 access_token + refresh_token
    └─ POST /api/auth/refresh   →  用 refresh_token 換新 access_token

管理後台 Token 策略：
  - access_token:  RS256 JWT, TTL 1小時
  - refresh_token: RS256 JWT, TTL 7天, jti 存 Redis
  - 前端每 30 分鐘自動呼叫 /refresh 刷新
```

---

## Kubernetes 拓樸

```
Namespace: justread
  Deployments (各 1 replica):
    justread-auth, justread-book, justread-user,
    justread-reader, justread-notification,
    justread-front, justread-manage

  Services (NodePort):
    各服務對應 NodePort (見 README 表格)

Namespace: database
  Deployments:
    mysql  →  mysql-service.database.svc.cluster.local:3306
    redis  →  redis-service.database.svc.cluster.local:6379

Nginx (跳板機):
  /etc/nginx/sites-enabled/justread
  代理 NodePort → 對外網域
```

---

## 資料庫 Schema

### users
```sql
id           VARCHAR(36) PRIMARY KEY  -- UUID v4
username     VARCHAR(50)  UNIQUE NOT NULL
email        VARCHAR(255) UNIQUE NOT NULL
password_hash VARCHAR(255) NOT NULL
is_verified  BOOLEAN DEFAULT 0
created_at   DATETIME DEFAULT CURRENT_TIMESTAMP
```

### email_verifications
```sql
id           VARCHAR(36) PRIMARY KEY
user_id      VARCHAR(36) NOT NULL  -- FK → users.id
token        VARCHAR(255) UNIQUE NOT NULL
expires_at   DATETIME NOT NULL
created_at   DATETIME DEFAULT CURRENT_TIMESTAMP
```

### admin_users
```sql
id           VARCHAR(36) PRIMARY KEY  -- UUID v4
username     VARCHAR(50)  UNIQUE NOT NULL
password_hash VARCHAR(255) NOT NULL   -- bcrypt cost=12
created_at   DATETIME DEFAULT CURRENT_TIMESTAMP
```

### Redis Key 規格
```
refresh_jti:{jti}   →  "1"   TTL 7d    (refresh token 有效性)
```
