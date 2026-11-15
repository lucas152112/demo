# Coffee CRM 多環境配置說明

## 🎯 **配置策略**

由於 Coffee CRM 需要在不同環境中運行，我們提供兩套配置：

### **1. K8s 集群內部部署**（生產環境）
- 使用 K8s 內部 DNS 服務發現
- 自動連接到持久化資料庫服務
- 配置文件：`.env.k8s`

### **2. 本地開發測試**（開發環境）  
- 使用 localhost 或 Docker 本地服務
- 快速開發和測試
- 配置文件：`.env.local`

## 🔧 **環境切換**

```bash
# K8s 部署環境
cp .env.k8s .env

# 本地開發環境  
cp .env.local .env
```

## 🌐 **K8s 資料庫服務**

**內部服務 DNS**：
- MySQL: `mysql-service.database.svc.cluster.local:3306`
- Redis: `redis-service.database.svc.cluster.local:6379`  
- MongoDB: `mongodb-service.database.svc.cluster.local:27017`

**認證資訊**：
- MySQL: root / root123456 (gamedb)
- Redis: redis123456
- MongoDB: admin / admin123456

## 🏠 **本地資料庫設置**

```bash
# 使用 Docker 啟動本地資料庫
docker run -d --name coffee-mysql \
  -e MYSQL_ROOT_PASSWORD=root123456 \
  -e MYSQL_DATABASE=gamedb \
  -p 3306:3306 mysql:8.0

docker run -d --name coffee-redis \
  -p 6379:6379 redis:7 \
  --requirepass redis123456

docker run -d --name coffee-mongodb \
  -e MONGO_INITDB_ROOT_USERNAME=admin \
  -e MONGO_INITDB_ROOT_PASSWORD=admin123456 \
  -p 27017:27017 mongo:7
```

## 🚀 **部署流程**

### **開發測試**：
1. 啟動本地資料庫服務  
2. 使用 `.env.local` 配置
3. `cargo run` 啟動服務

### **K8s 部署**：
1. 確認 K8s 資料庫服務運行  
2. 使用 `.env.k8s` 配置
3. `docker build` + `kubectl apply`