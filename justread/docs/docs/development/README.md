# 開發指南

## 專案結構

```
justread/
├── backend/
│   ├── auth/           # Rust + Axum，Port 8081
│   ├── book/           # Rust + Axum，Port 8082
│   ├── user/           # Rust + Axum，Port 8083
│   ├── reader/         # Rust + Axum，Port 8084
│   └── notification/   # Rust + Axum，Port 8085
├── front/              # React + Vite，Port 3000
├── manage/             # React + Vite，Port 3001
├── k8s/                # Kubernetes YAML
├── docs/               # 文件
└── pnpm-workspace.yaml # pnpm monorepo 設定
```

---

## 本地開發環境

### 前置需求

- Rust 1.88+
- Node.js 24+
- pnpm 11+
- Docker
- MySQL 8（本地或遠端）
- Redis

### 後端服務本地執行

```bash
cd backend/auth
DATABASE_URL="mysql://user:password@localhost/justread" \
REDIS_URL="redis://localhost:6379" \
RESEND_API_KEY="..." \
JWT_PRIVATE_KEY="$(cat /tmp/jwtkeys/private.pem)" \
JWT_PUBLIC_KEY="$(cat /tmp/jwtkeys/public.pem)" \
cargo run
```

### 前端本地開發

```bash
# 安裝依賴
pnpm install

# 啟動管理後台
pnpm --filter @justread/manage dev

# 啟動前台
pnpm --filter @justread/front dev
```

---

## Build 流程

### 後端（Docker，從專案根目錄）

```bash
# Auth
docker build -f backend/auth/Dockerfile -t ghcr.io/lucas152112/justread-auth:latest .

# Book
docker build -f backend/book/Dockerfile -t ghcr.io/lucas152112/justread-book:latest .
```

### 前端

```bash
# 管理後台
pnpm --filter @justread/manage build

# 前台
pnpm --filter @justread/front build
```

---

## 新增後端 API 端點

1. 在 `src/routes/` 新增 `.rs` 檔案
2. 加入 `#[utoipa::path(...)]` 標注
3. 在 `src/routes/mod.rs` 的 `router()` 掛載
4. 在 `main.rs` 的 `ApiDoc` paths 加入

---

## 測試

### API 測試（curl）

```bash
NODE="http://103.251.113.34"

# 健康檢查
curl $NODE:30091/health

# 註冊
curl -X POST $NODE:30091/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{"username":"test","email":"test@example.com","password":"Test1234!"}'

# 登入
curl -X POST $NODE:30091/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"aZ123456"}'
```

---

## 常見問題

### Pod CrashLoopBackOff
```bash
ssh jump "kubectl logs -n justread deployment/justread-auth --previous"
```

### DNS 解析失敗
確認使用 `mysql-service.database.svc.cluster.local`（非 `mysql`）。  
runtime 映像必須是 debian（非 Alpine）。

### JWT 驗證失敗
確認 `justread-jwt-keys` Secret 中的公私鑰一致。  
```bash
ssh jump "kubectl get secret justread-jwt-keys -n justread -o yaml"
```

### ghcr-secret 拉取失敗
GitHub PAT 需有 `read:packages` 權限，且未過期。
