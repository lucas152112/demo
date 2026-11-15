# API 規格總覽

所有服務均掛載 Swagger UI，可於 `http://{NODE_IP}:{NodePort}/swagger-ui/` 查看互動式文件。

> NODE_IP: `103.251.113.34`

---

## Auth Service (Port 30091)

Base URL: `http://103.251.113.34:30091/api/auth`

| Method | Path | 說明 | Auth |
|--------|------|------|------|
| POST | `/register` | 使用者註冊，寄驗證信 | ❌ |
| GET | `/verify-email` | Email 驗證（?token=xxx） | ❌ |
| POST | `/login` | 管理員登入，回傳 JWT | ❌ |
| POST | `/refresh` | 刷新 access_token | refresh_token |

### POST /register
```json
Request:
{ "username": "string", "email": "string", "password": "string" }

Response 200:
{ "user_id": "uuid", "message": "註冊成功！請檢查您的電子郵件並完成驗證。" }
```

### GET /verify-email?token={token}
```
Response: HTML 頁面（成功 / 過期 / 無效）
```

### POST /login
```json
Request:
{ "username": "string", "password": "string" }

Response 200:
{
  "access_token": "eyJ...",
  "refresh_token": "eyJ...",
  "expires_in": 3600
}
```

### POST /refresh
```json
Request:
{ "refresh_token": "eyJ..." }

Response 200:
{ "access_token": "eyJ...", "expires_in": 3600 }
```

---

## Book Service (Port 30082)

Base URL: `http://103.251.113.34:30082`

| Method | Path | 說明 |
|--------|------|------|
| GET | `/api/books` | 書籍列表 |
| GET | `/api/books/{id}` | 書籍詳情 |
| POST | `/api/books` | 上傳書籍 |
| DELETE | `/api/books/{id}` | 刪除書籍 |
| GET | `/health` | 健康檢查 |

---

## User Service (Port 30083)

Base URL: `http://103.251.113.34:30083`

| Method | Path | 說明 |
|--------|------|------|
| GET | `/api/users` | 用戶列表（管理用） |
| GET | `/api/users/{id}` | 用戶 Profile |
| PUT | `/api/users/{id}` | 更新 Profile |
| GET | `/health` | 健康檢查 |

---

## Reader Service (Port 30084)

Base URL: `http://103.251.113.34:30084`

| Method | Path | 說明 |
|--------|------|------|
| GET | `/api/reader/progress/{book_id}` | 閱讀進度 |
| PUT | `/api/reader/progress/{book_id}` | 更新進度 |
| GET | `/api/reader/bookmarks/{book_id}` | 書籤列表 |
| POST | `/api/reader/bookmarks` | 新增書籤 |
| GET | `/api/reader/history` | 閱讀歷史 |
| POST | `/api/reader/sync` | 多裝置同步 |
| GET | `/health` | 健康檢查 |

---

## Notification Service (Port 30085)

Base URL: `http://103.251.113.34:30085`

| Method | Path | 說明 |
|--------|------|------|
| GET | `/api/notifications` | 通知列表 |
| PUT | `/api/notifications/{id}/read` | 標記已讀 |
| POST | `/api/notifications/send` | 發送通知（系統內部） |
| GET | `/health` | 健康檢查 |
