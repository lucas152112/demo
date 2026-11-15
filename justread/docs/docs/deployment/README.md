# 部署指南

## 基礎設施

| 項目 | 位置 | 說明 |
|------|------|------|
| k8s Node | `103.251.113.34` | 工作節點，NodePort 對外 |
| k8s Control Plane | `103.251.113.35:6443` | |
| 跳板機 | `ssh jump` | 透過 `~/.ssh/config` 配置 |
| MySQL | `mysql-service.database.svc.cluster.local:3306` | Namespace: database |
| Redis | `redis-service.database.svc.cluster.local:6379` | Namespace: database |
| Nginx | 跳板機 `/etc/nginx/sites-enabled/justread` | 反向代理 |

---

## Kubernetes Secrets & ConfigMaps

### Secrets

| 名稱 | Namespace | 內容 |
|------|-----------|------|
| `justread-secrets` | justread | `RESEND_API_KEY` |
| `justread-jwt-keys` | justread | `JWT_PRIVATE_KEY`, `JWT_PUBLIC_KEY` |
| `ghcr-secret` | justread | GitHub Container Registry 拉取憑證 |

### ConfigMaps

| 名稱 | Namespace | 內容 |
|------|-----------|------|
| `justread-config` | justread | `DATABASE_URL`, `REDIS_URL`, `PORT` 等 |

---

## 部署流程

### Build & Push 映像檔（從專案根目錄執行）

```bash
# Auth service
docker build -f backend/auth/Dockerfile -t ghcr.io/lucas152112/justread-auth:latest .
docker push ghcr.io/lucas152112/justread-auth:latest

# 管理後台
docker build -f manage/Dockerfile -t ghcr.io/lucas152112/justread-manage:latest .
docker push ghcr.io/lucas152112/justread-manage:latest
```

> ⚠️ Dockerfile 路徑依賴專案根目錄的 `COPY` 指令，必須從根目錄執行

### Rolling Update

```bash
ssh jump "kubectl rollout restart deployment/justread-auth -n justread"
ssh jump "kubectl rollout restart deployment/justread-manage -n justread"

# 確認狀態
ssh jump "kubectl rollout status deployment/justread-auth -n justread --timeout=60s"
```

### 查看 Pod 狀態

```bash
ssh jump "kubectl get pods -n justread"
ssh jump "kubectl logs -n justread deployment/justread-auth --tail=50"
```

---

## 資料庫操作

### 連入 MySQL

```bash
ssh jump "kubectl exec -n database deploy/mysql -- bash -c 'mysql -u root -p\$MYSQL_ROOT_PASSWORD justread'"
```

### 連入 Redis

```bash
ssh jump "kubectl exec -n database deploy/redis -- redis-cli"
```

---

## JWT 金鑰管理

### 生成新金鑰對

```bash
mkdir -p /tmp/jwtkeys
openssl genrsa -out /tmp/jwtkeys/private.pem 2048
openssl rsa -in /tmp/jwtkeys/private.pem -pubout -out /tmp/jwtkeys/public.pem
```

### 更新 k8s Secret

```bash
PRIV=$(cat /tmp/jwtkeys/private.pem | base64 -w 0)
PUB=$(cat /tmp/jwtkeys/public.pem | base64 -w 0)

ssh jump "kubectl patch secret justread-jwt-keys -n justread \
  --type='json' \
  -p='[
    {\"op\":\"replace\",\"path\":\"/data/JWT_PRIVATE_KEY\",\"value\":\"'$PRIV'\"},
    {\"op\":\"replace\",\"path\":\"/data/JWT_PUBLIC_KEY\",\"value\":\"'$PUB'\"}
  ]'"
```

---

## 管理員帳號管理

預設帳號：`admin` / 密碼：`aZ123456`

### 新增管理員

```sql
INSERT INTO admin_users (id, username, password_hash)
VALUES (UUID(), 'newadmin', '$2b$12$...');  -- bcrypt cost=12
```

---

## k8s 配置檔案

```
k8s/
├── namespace-rbac.yaml   # Namespace 與 RBAC
├── configmap.yaml        # 環境變數 ConfigMap
├── backend.yaml          # 所有後端 Deployment + Service
├── frontend.yaml         # front + manage Deployment + Service
└── ghcr-secret.yaml      # Container Registry 拉取 Secret
```
