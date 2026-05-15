# Just Work 後端技術規格

## 概述

本文檔描述 Just Work 系統的後端技術架構和規格。

---

## 1. 微服務框架架構

### 1.1 服務結構

```
backend/
├── services/                  # 微服務目錄
│   ├── user-service/          # 用戶服務
│   │   ├── src/
│   │   ├── Cargo.toml
│   │   └── Dockerfile
│   ├── work-service/          # 工作管理服務
│   ├── team-service/          # 團隊服務
│   ├── auth-service/          # 認證服務
│   └── notification-service/  # 通知服務
│
├── common/                    # 共用模組
│   ├── models/
│   ├── error/
│   ├── middleware/
│   └── utils/
│
├── gateway/                   # API 閘道
│
└── registry/                  # 服務註冊
```

### 1.2 服務通訊

| 通訊方式 | 用途 |
|----------|------|
| REST API | 外部客戶端通訊 |
| gRPC | 內部服務間通訊 |
| Message Queue | 異步事件處理 |

### 1.3 熱更新機制

```rust
// 服務監控器
pub struct ServiceWatcher {
    services: HashMap<String, ServiceInstance>,
    notify: WatchNotifier,
}

impl ServiceWatcher {
    /// 監控服務目錄變化
    pub async fn watch(&self, path: &Path) {
        let mut watcher = notify::Watcher::new_immediate(move |event| {
            if let notify::EventKind::Modify(_) = event.kind {
                // 觸發熱更新
                self.trigger_hot_reload(event.paths[0].clone());
            }
        }).unwrap();
        
        watcher.watch(path, Recursive::true).unwrap();
    }
    
    /// 觸發熱更新
    async fn trigger_hot_reload(&self, service_path: PathBuf) {
        let service_name = extract_service_name(&service_path);
        
        // 檢查服務狀態
        if self.get_service_status(&service_name) == ServiceStatus::Online {
            // 執行熱更新
            self.hot_reload(&service_name).await;
        }
    }
}
```

---

## 2. 資料庫架構

### 2.1 資料庫清單

| 資料庫 | 類型 | 用途 |
|--------|------|------|
| MySQL 8 | 關聯式 | 用戶、工作、團隊等核心資料 |
| Redis | Key-Value | 快取、Session、訊息佇列 |
| MongoDB | 文件 | 日誌、通知、歷史記錄 |

### 2.2 環境資料庫命名

| 環境 | MySQL | Redis | MongoDB |
|------|-------|-------|---------|
| dev | justwork_dev | justwork:dev | justwork_dev |
| test | justwork_test | justwork:test | justwork_test |
| pp | justwork_pp | justwork:pp | justwork_pp |
| prod | justwork | justwork:prod | justwork_prod |

### 2.3 MySQL Schema 範例

```sql
-- 用戶表
CREATE TABLE users (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    uuid VARCHAR(36) NOT NULL UNIQUE,
    email VARCHAR(255) NOT NULL UNIQUE,
    password_hash VARCHAR(255) NOT NULL,
    name VARCHAR(100) NOT NULL,
    status TINYINT DEFAULT 1 COMMENT '1: active, 0: inactive',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    INDEX idx_email (email),
    INDEX idx_uuid (uuid)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- 工作表
CREATE TABLE works (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    uuid VARCHAR(36) NOT NULL UNIQUE,
    title VARCHAR(255) NOT NULL,
    description TEXT,
    status TINYINT DEFAULT 1,
    assignee_id BIGINT UNSIGNED,
    creator_id BIGINT UNSIGNED NOT NULL,
    due_date DATETIME,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    FOREIGN KEY (assignee_id) REFERENCES users(id),
    FOREIGN KEY (creator_id) REFERENCES users(id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

### 2.4 Redis 用途

```
# 快取鍵命名規範
justwork:{env}:{service}:{resource}:{id}

# 範例
justwork:dev:user:1001          # 用戶快取
justwork:prod:session:abc123    # Session
justwork:test:work:queue         # 工作佇列
```

### 2.5 MongoDB Collections

```
# 日誌集合
{env}_logs

# 通知集合
{env}_notifications

# 審計集合
{env}_audit_logs
```

---

## 3. API 設計

### 3.1 RESTful 規範

```
GET    /api/v1/users              # 列出用戶
GET    /api/v1/users/:id          # 取得用戶
POST   /api/v1/users              # 建立用戶
PUT    /api/v1/users/:id          # 更新用戶
DELETE /api/v1/users/:id          # 刪除用戶
```

### 3.2 請求/響應格式

```json
// 請求格式
{
  "data": { ... },
  "meta": {
    "request_id": "uuid",
    "timestamp": "2024-01-15T10:30:00Z"
  }
}

// 成功響應
{
  "success": true,
  "data": { ... },
  "meta": {
    "request_id": "uuid",
    "timestamp": "2024-01-15T10:30:00Z"
  }
}

// 錯誤響應
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Invalid input data",
    "details": [...]
  },
  "meta": {
    "request_id": "uuid",
    "timestamp": "2024-01-15T10:30:00Z"
  }
}
```

### 3.3 Swagger/OpenAPI

```rust
// 使用utoipa生成Swagger文檔
use utoipa::{OpenApi, Modify};

#[derive(OpenApi)]
#[openapi(
    paths(
        api::users::list_users,
        api::users::create_user,
        api::works::list_works,
    ),
    components(
        schemas(User, Work, ErrorResponse)
    ),
    tags(
        name = "users",
        description = "用戶管理 API"
    ),
)]
struct ApiDoc;

#[utoipa::path(
    get,
    path = "/api/v1/users",
    responses(
        (status = 200, description = "List of users", body = Vec<User>),
    )
)]
async fn list_users() -> Json<Vec<User>> {
    // ...
}
```

---

## 4. 服務狀態管理

### 4.1 狀態定義

| 狀態 | 說明 | 允許操作 |
|------|------|----------|
| **Online** | 上架，可接受請求 | 下架、停止 |
| **Offline** | 下架，不接受請求 | 上架、停止 |
| **Stopped** | 停止使用 | 無 |
| **Deprecated** | 已廢棄 | 無 |

### 4.2 狀態機

```rust
#[derive(Debug, Clone, PartialEq, Serialize, Deserialize)]
pub enum ServiceStatus {
    Online,
    Offline,
    Stopped,
    Deprecated,
}

pub struct StatusMachine {
    transitions: HashMap<(ServiceStatus, ServiceStatus), bool>,
}

impl StatusMachine {
    pub fn new() -> Self {
        let mut transitions = HashMap::new();
        
        // 允許的轉換
        transitions.insert((ServiceStatus::Offline, ServiceStatus::Online), true);
        transitions.insert((ServiceStatus::Online, ServiceStatus::Offline), true);
        transitions.insert((ServiceStatus::Offline, ServiceStatus::Stopped), true);
        
        Self { transitions }
    }
    
    pub fn can_transition(&self, from: &ServiceStatus, to: &ServiceStatus) -> bool {
        self.transitions
            .get(&(*from, *to))
            .copied()
            .unwrap_or(false)
    }
}
```

### 4.3 熱更新條件

```rust
fn can_hot_reload(service: &Service) -> bool {
    service.status == ServiceStatus::Online
}
```

---

## 5. 環境配置

### 5.1 環境變數

```bash
# 開發環境 (.env.dev)
DATABASE_URL=mysql://user:pass@localhost:3306/justwork_dev
REDIS_URL=redis://localhost:6379/0
MONGODB_URL=mongodb://localhost:27017/justwork_dev
ENVIRONMENT=dev

# 測試環境 (.env.test)
DATABASE_URL=mysql://user:pass@mysql:3306/justwork_test
REDIS_URL=redis://redis:6379/0
MONGODB_URL=mongodb://mongo:27017/justwork_test
ENVIRONMENT=test

# 預發布 (.env.pp)
DATABASE_URL=mysql://user:pass@mysql:3306/justwork_pp
REDIS_URL=redis://redis:6379/0
MONGODB_URL=mongodb://mongo:27017/justwork_pp
ENVIRONMENT=pp

# 生產環境 (.env.prod)
DATABASE_URL=mysql://user:pass@mysql:3306/justwork
REDIS_URL=redis://redis:6379/0
MONGODB_URL=mongodb://mongo:27017/justwork_prod
ENVIRONMENT=prod
```

### 5.2 K8s 配置

```yaml
# backend/services/user-service/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: user-service
  labels:
    app: user-service
    environment: dev
spec:
  replicas: 1
  selector:
    matchLabels:
      app: user-service
  template:
    spec:
      containers:
      - name: user-service
        image: justwork/user-service:v1.0.0
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: database-credentials
              key: url
        - name: REDIS_URL
          valueFrom:
            configMapKeyRef:
              name: redis-config
              key: url
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "512Mi"
            cpu: "500m"
---
apiVersion: v1
kind: Service
metadata:
  name: user-service
spec:
  selector:
    app: user-service
  ports:
  - port: 8080
    targetPort: 8080
  type: ClusterIP
```

---

## 6. 版本號規範

### 6.1 版本格式

```
主版號.大版本號.小版本號.測試版本號.維護版本號
   |         |        |         |          |
   1         0        0         0          0
```

### 6.2 版本意義

| 位置 | 名稱 | 說明 |
|------|------|------|
| 1 | 主版號 | 正式版本重大變更 |
| 0 | 大版本號 | 開發大版本 |
| 0 | 小版本號 | 功能更新 |
| 0 | 測試版本號 | 測試版本 |
| 0 | 維護版本號 | 維護更新 |

### 6.3 版本範例

| 版本號 | 意義 |
|--------|------|
| `1.0.0.0.0` | 正式版 v1.0 |
| `1.1.0.0.0` | 正式版 v1.1 |
| `0.1.0.0.0` | 開發版 v0.1 |
| `0.2.0.0.0` | 開發版 v0.2 |
| `0.2.1.0.0` | 開發版 v0.2.1 (小更新) |
| `0.2.0.1.0` | 測試版 v0.2.0-t1 |
| `0.2.0.1.1` | 維護版 v0.2.0-t1-m1 |

---

## 7. 技術棧版本

| 技術 | 版本 |
|------|------|
| Rust | 1.75+ |
| Axum | 0.7+ |
| SeaORM | 0.12+ |
| Redis | 7.x |
| MySQL | 8.0+ |
| MongoDB | 6.0+ |

---

## 8. 監控指標

### 8.1 Prometheus 指標

```
# 服務指標
http_requests_total{method="GET",path="/api/v1/users",status="200"}
http_request_duration_seconds_bucket{le="0.1"}
service_status{service="user-service",status="online"}

# 資料庫指標
db_connections_active{db="mysql"}
db_connections_idle{db="mysql"}
redis_commands_total{command="GET"}
mongodb_operations_total{operation="insert"}
```

---

**文件版本**: 0.1.0.0.0
**最後更新**: 2026-02-14
**維護團隊**: Backend Team
