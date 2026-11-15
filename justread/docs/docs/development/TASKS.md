# 開發任務清單

> 最後更新：2026-05-11

---

## ✅ 已完成

### 基礎設施
- [x] Kubernetes namespace `justread` 建立
- [x] MySQL `justread` 資料庫 + tables (users, email_verifications, admin_users)
- [x] Redis 連線池（deadpool-redis）
- [x] k8s ConfigMap / Secret 配置
- [x] GHCR 映像 push / pull 流程
- [x] Nginx 反向代理配置

### Auth 服務
- [x] 使用者註冊 API (`POST /api/auth/register`)
- [x] Email 驗證信發送（Resend API + rustls）
- [x] Email 驗證端點（HTML 回傳頁面）
- [x] 管理員登入 API（RS256 JWT）
- [x] Access Token（1小時 TTL）
- [x] Refresh Token（7天 TTL，jti 存 Redis）
- [x] Token 刷新端點 (`POST /api/auth/refresh`)
- [x] RSA 2048 金鑰對生成 + k8s Secret 儲存
- [x] 預建管理員帳號（admin / aZ123456，bcrypt cost=12）

### 管理後台（manage）
- [x] Login 頁面 + JWT 登入流程
- [x] Dashboard（用戶數、書籍數統計）
- [x] Users 頁面（列表、停用、刪除）
- [x] Books 頁面（列表、刪除）
- [x] Sidebar Layout 元件
- [x] axios client（讀取 `manage_token`）
- [x] TokenRefresher（每 30 分鐘自動刷新）
- [x] PrivateRoute（未登入導向 /login）
- [x] 登出清除 token

### 其他服務（骨架）
- [x] Book Service 骨架（list, detail, upload, delete routes）
- [x] User Service 骨架（profile, update, list routes）
- [x] Reader Service 骨架（progress, bookmark, history, sync routes）
- [x] Notification Service 骨架（list, read, send routes）
- [x] 所有服務 Swagger UI 整合

---

## 🚧 進行中 / 待開發

### Auth 服務
- [ ] 一般使用者登入（`users` 表，非 admin_users）
- [ ] 登出端點（invalidate refresh token）
- [ ] 忘記密碼 / 重設密碼流程
- [ ] Token Blacklist（強制登出）

### Book 服務
- [ ] 連接 MySQL，實作書籍 CRUD 業務邏輯
- [ ] 書籍封面/檔案上傳（S3 or MinIO）
- [ ] 書籍搜尋、分類、標籤功能
- [ ] 分頁查詢

### User 服務
- [ ] 連接 MySQL，實作 Profile CRUD
- [ ] 帳號停用 / 刪除
- [ ] 使用者統計資料

### Reader 服務
- [ ] 連接 MySQL，實作閱讀進度儲存
- [ ] 書籤 CRUD
- [ ] 閱讀歷史記錄
- [ ] 多裝置進度同步（Redis 快取）

### Notification 服務
- [ ] 連接 MySQL，實作通知儲存
- [ ] 標記已讀邏輯
- [ ] 系統事件觸發通知（如：新書上架）
- [ ] Email 通知整合（Resend）

### 前台（front）
- [ ] 使用者登入/註冊頁面
- [ ] Email 驗證完成後導向
- [ ] 書籍列表瀏覽
- [ ] 書籍閱讀器（PDF/EPUB 渲染）
- [ ] 閱讀進度顯示
- [ ] 書籤管理介面
- [ ] 通知中心

### 管理後台擴充
- [ ] Books 頁面：上傳書籍功能
- [ ] Users 頁面：接通真實 User API
- [ ] 系統設定頁面
- [ ] 管理員帳號管理（新增、修改密碼）
- [ ] 操作日誌

### 基礎設施
- [ ] CI/CD 自動化（GitHub Actions → build → push → rollout）
- [ ] 多 replica 部署 + HPA
- [ ] Ingress Controller（取代 NodePort）
- [ ] TLS 憑證（Let's Encrypt）
- [ ] 集中式日誌（Loki or ELK）
- [ ] Prometheus + Grafana 監控
- [ ] 資料庫 Migration 工具（sqlx-migrate）
- [ ] 自動備份策略

---

## 📋 優先順序

| 優先 | 任務 |
|------|------|
| P1 | 一般使用者登入 + 前台登入頁面 |
| P1 | Book / User Service 連接 DB |
| P2 | 前台書籍列表 + 閱讀器 |
| P2 | Reader Service 進度同步 |
| P3 | CI/CD 自動化 |
| P3 | Notification 完整實作 |
| P4 | TLS / Ingress / 監控 |
