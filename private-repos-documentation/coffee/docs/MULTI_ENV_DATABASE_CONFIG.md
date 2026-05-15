# Coffee CRM 多環境資料庫配置

## 🏗️ **K8s 多環境架構** (2026-02-14 17:40)

### 環境分離設計
```
DEV 環境    → coffee_dev_*   資料庫
TEST 環境   → coffee_test_*  資料庫  
PP 環境     → coffee_pp_*    資料庫
PROD 環境   → coffee_prod_*  資料庫
```

## 🗄️ **K8s 資料庫服務連接**

### MySQL 8.0 (各環境獨立)
```bash
# K8s Service
HOST: mysql-service.database.svc.cluster.local:3306
USER: root
PASS: root123456
DATABASE: gamedb
```

### Redis 7.0 (環境分離)
```bash
# K8s Service  
HOST: redis-service.database.svc.cluster.local:6379
PASS: redis123456
# Key前綴: coffee:{env}:*
```

### MongoDB 7.0 (環境分離)
```bash
# K8s Service
HOST: mongodb-service.database.svc.cluster.local:27017
USER: admin
PASS: admin123456
# Database: coffee_{env}
```

## 🔧 **環境變數配置**

### Rust 環境配置 (.env 模式)

#### DEV 環境 (.env.dev)
```env
# Coffee CRM - DEV Environment
ENVIRONMENT=dev
APP_NAME=coffee-crm-dev

# MySQL - K8s Service
MYSQL_HOST=mysql-service.database.svc.cluster.local
MYSQL_PORT=3306  
MYSQL_USER=root
MYSQL_PASSWORD=root123456
MYSQL_DATABASE=gamedb
TABLE_PREFIX=coffee_dev_

# Redis - K8s Service
REDIS_HOST=redis-service.database.svc.cluster.local
REDIS_PORT=6379
REDIS_PASSWORD=redis123456
REDIS_PREFIX=coffee:dev:

# MongoDB - K8s Service
MONGODB_HOST=mongodb-service.database.svc.cluster.local
MONGODB_PORT=27017
MONGODB_USER=admin
MONGODB_PASSWORD=admin123456
MONGODB_DATABASE=coffee_dev

# Application
JWT_SECRET=coffee-dev-secret-2026
LOG_LEVEL=debug
PORT=8080
```

#### TEST 環境 (.env.test)
```env
# Coffee CRM - TEST Environment
ENVIRONMENT=test
APP_NAME=coffee-crm-test

# MySQL - K8s Service
MYSQL_HOST=mysql-service.database.svc.cluster.local
MYSQL_PORT=3306
MYSQL_USER=root
MYSQL_PASSWORD=root123456
MYSQL_DATABASE=gamedb
TABLE_PREFIX=coffee_test_

# Redis - K8s Service
REDIS_HOST=redis-service.database.svc.cluster.local
REDIS_PORT=6379
REDIS_PASSWORD=redis123456
REDIS_PREFIX=coffee:test:

# MongoDB - K8s Service
MONGODB_HOST=mongodb-service.database.svc.cluster.local
MONGODB_PORT=27017
MONGODB_USER=admin
MONGODB_PASSWORD=admin123456
MONGODB_DATABASE=coffee_test

# Application
JWT_SECRET=coffee-test-secret-2026
LOG_LEVEL=info
PORT=8080
```

#### PP 環境 (.env.pp)
```env
# Coffee CRM - PP (Pre-Production) Environment
ENVIRONMENT=pp
APP_NAME=coffee-crm-pp

# MySQL - K8s Service
MYSQL_HOST=mysql-service.database.svc.cluster.local
MYSQL_PORT=3306
MYSQL_USER=root
MYSQL_PASSWORD=root123456
MYSQL_DATABASE=gamedb
TABLE_PREFIX=coffee_pp_

# Redis - K8s Service
REDIS_HOST=redis-service.database.svc.cluster.local
REDIS_PORT=6379
REDIS_PASSWORD=redis123456
REDIS_PREFIX=coffee:pp:

# MongoDB - K8s Service
MONGODB_HOST=mongodb-service.database.svc.cluster.local
MONGODB_PORT=27017
MONGODB_USER=admin
MONGODB_PASSWORD=admin123456
MONGODB_DATABASE=coffee_pp

# Application
JWT_SECRET=coffee-pp-secret-2026
LOG_LEVEL=warn
PORT=8080
```

#### PROD 環境 (.env.prod)
```env
# Coffee CRM - PRODUCTION Environment
ENVIRONMENT=prod
APP_NAME=coffee-crm

# MySQL - K8s Service
MYSQL_HOST=mysql-service.database.svc.cluster.local
MYSQL_PORT=3306
MYSQL_USER=root
MYSQL_PASSWORD=root123456
MYSQL_DATABASE=gamedb
TABLE_PREFIX=coffee_prod_

# Redis - K8s Service
REDIS_HOST=redis-service.database.svc.cluster.local
REDIS_PORT=6379
REDIS_PASSWORD=redis123456
REDIS_PREFIX=coffee:prod:

# MongoDB - K8s Service
MONGODB_HOST=mongodb-service.database.svc.cluster.local
MONGODB_PORT=27017
MONGODB_USER=admin
MONGODB_PASSWORD=admin123456
MONGODB_DATABASE=coffee_prod

# Application
JWT_SECRET=coffee-production-secret-2026-ultra-secure
LOG_LEVEL=error
PORT=8080
```

## 🚀 **K8s Deployment 環境配置**

### ConfigMap 配置
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: coffee-config-dev
  namespace: coffee
data:
  ENVIRONMENT: "dev"
  APP_NAME: "coffee-crm-dev"
  MYSQL_HOST: "mysql-service.database.svc.cluster.local"
  MYSQL_PORT: "3306"
  MYSQL_DATABASE: "gamedb"
  TABLE_PREFIX: "coffee_dev_"
  REDIS_HOST: "redis-service.database.svc.cluster.local"
  REDIS_PREFIX: "coffee:dev:"
  MONGODB_HOST: "mongodb-service.database.svc.cluster.local"
  MONGODB_DATABASE: "coffee_dev"
  LOG_LEVEL: "debug"
---
# 其他環境的 ConfigMap (test/pp/prod)
```

### Secret 配置
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: coffee-secrets-dev
  namespace: coffee
type: Opaque
data:
  MYSQL_PASSWORD: cm9vdDEyMzQ1Ng==  # root123456
  REDIS_PASSWORD: cmVkaXMxMjM0NTY=  # redis123456  
  MONGODB_PASSWORD: YWRtaW4xMjM0NTY= # admin123456
  JWT_SECRET: Y29mZmVlLWRldi1zZWNyZXQtMjAyNg== # coffee-dev-secret-2026
```

## 📋 **多環境初始化腳本**

### 自動化環境部署
```bash
#!/bin/bash
# deploy_coffee_env.sh - Coffee CRM 多環境部署

ENVIRONMENT=${1:-dev}
VALID_ENVS=("dev" "test" "pp" "prod")

if [[ ! " ${VALID_ENVS[@]} " =~ " ${ENVIRONMENT} " ]]; then
    echo "❌ 無效環境: $ENVIRONMENT"
    echo "✅ 有效環境: dev, test, pp, prod"
    exit 1
fi

echo "🚀 部署 Coffee CRM - $ENVIRONMENT 環境"

# 1. 建立 namespace
kubectl create namespace coffee --dry-run=client -o yaml | kubectl apply -f -

# 2. 部署 ConfigMap 和 Secret
kubectl apply -f k8s/coffee-config-${ENVIRONMENT}.yaml
kubectl apply -f k8s/coffee-secrets-${ENVIRONMENT}.yaml

# 3. 初始化資料庫
kubectl apply -f k8s/coffee-db-init-${ENVIRONMENT}.yaml

# 4. 部署服務
kubectl apply -f k8s/coffee-services-${ENVIRONMENT}.yaml

echo "✅ Coffee CRM $ENVIRONMENT 環境部署完成"
```

## 🔧 **Rust 多環境配置支援**

### 環境配置結構
```rust
// src/config.rs
use serde::Deserialize;
use std::env;

#[derive(Debug, Deserialize, Clone)]
pub struct DatabaseConfig {
    pub mysql_host: String,
    pub mysql_port: u16,
    pub mysql_user: String,
    pub mysql_password: String,
    pub mysql_database: String,
    pub table_prefix: String,
}

#[derive(Debug, Deserialize, Clone)]
pub struct RedisConfig {
    pub host: String,
    pub port: u16,
    pub password: String,
    pub prefix: String,
}

#[derive(Debug, Deserialize, Clone)]
pub struct MongoConfig {
    pub host: String,
    pub port: u16,
    pub user: String,
    pub password: String,
    pub database: String,
}

#[derive(Debug, Deserialize, Clone)]
pub struct AppConfig {
    pub environment: String,
    pub app_name: String,
    pub jwt_secret: String,
    pub log_level: String,
    pub port: u16,
    pub database: DatabaseConfig,
    pub redis: RedisConfig,
    pub mongodb: MongoConfig,
}

impl AppConfig {
    pub fn from_env() -> Result<Self, Box<dyn std::error::Error>> {
        let environment = env::var("ENVIRONMENT").unwrap_or_else(|_| "dev".to_string());
        
        // 根據環境動態構建配置
        Ok(AppConfig {
            environment: environment.clone(),
            app_name: env::var("APP_NAME").unwrap_or_else(|_| "coffee-crm".to_string()),
            jwt_secret: env::var("JWT_SECRET").unwrap_or_else(|_| "default-secret".to_string()),
            log_level: env::var("LOG_LEVEL").unwrap_or_else(|_| "info".to_string()),
            port: env::var("PORT").unwrap_or_else(|_| "8080".to_string()).parse()?,
            
            database: DatabaseConfig {
                mysql_host: env::var("MYSQL_HOST")?,
                mysql_port: env::var("MYSQL_PORT").unwrap_or_else(|_| "3306".to_string()).parse()?,
                mysql_user: env::var("MYSQL_USER").unwrap_or_else(|_| "root".to_string()),
                mysql_password: env::var("MYSQL_PASSWORD")?,
                mysql_database: env::var("MYSQL_DATABASE")?,
                table_prefix: env::var("TABLE_PREFIX").unwrap_or_else(|_| format!("coffee_{}_", environment)),
            },
            
            redis: RedisConfig {
                host: env::var("REDIS_HOST")?,
                port: env::var("REDIS_PORT").unwrap_or_else(|_| "6379".to_string()).parse()?,
                password: env::var("REDIS_PASSWORD")?,
                prefix: env::var("REDIS_PREFIX").unwrap_or_else(|_| format!("coffee:{}:", environment)),
            },
            
            mongodb: MongoConfig {
                host: env::var("MONGODB_HOST")?,
                port: env::var("MONGODB_PORT").unwrap_or_else(|_| "27017".to_string()).parse()?,
                user: env::var("MONGODB_USER").unwrap_or_else(|_| "admin".to_string()),
                password: env::var("MONGODB_PASSWORD")?,
                database: env::var("MONGODB_DATABASE").unwrap_or_else(|_| format!("coffee_{}", environment)),
            },
        })
    }
    
    pub fn mysql_url(&self) -> String {
        format!(
            "mysql://{}:{}@{}:{}/{}",
            self.database.mysql_user,
            self.database.mysql_password,
            self.database.mysql_host,
            self.database.mysql_port,
            self.database.mysql_database
        )
    }
    
    pub fn redis_url(&self) -> String {
        format!(
            "redis://:{}@{}:{}",
            self.redis.password,
            self.redis.host,
            self.redis.port
        )
    }
}
```

---

**配置完成時間**: 2026-02-14 17:40  
**環境支援**: dev / test / pp / prod  
**資料庫**: 完全K8s持久化部署  
**隔離性**: 表前綴 + 環境分離  
**部署方式**: 自動化腳本 + K8s資源  
**負責人**: 芊芊 AI