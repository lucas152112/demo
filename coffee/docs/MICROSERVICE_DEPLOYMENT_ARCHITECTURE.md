# Coffee CRM 微服務管理系統

## 獨立微服務架構設計

基於JustProject經驗，每個微服務完全獨立部署和管理。

### 🏗️ 微服務部署架構

#### 平台級微服務 (2個)
1. **tenant-management-service** (端口 20001)
2. **platform-ops-service** (端口 20002)

#### 租戶級微服務 (14個)  
3. **auth-service** (端口 20003)
4. **customer-service** (端口 20004)
5. **sales-service** (端口 20005)
6. **inventory-service** (端口 20006)
7. **purchase-service** (端口 20007)
8. **finance-service** (端口 20008)
9. **supplier-service** (端口 20009)
10. **report-service** (端口 20010)
11. **notification-service** (端口 20011)
12. **audit-service** (端口 20012)
13. **file-service** (端口 20013)
14. **workflow-service** (端口 20014)
15. **ai-chat-service** (端口 20015)
16. **gateway-service** (端口 20000)

### 🔧 獨立部署特性

#### ✅ 每個微服務包含：
- **獨立Dockerfile** - 單獨容器化
- **獨立K8s配置** - 獨立Service/Deployment
- **健康檢查端點** - `/health`
- **狀態監控API** - `/status` 
- **版本信息API** - `/version`
- **獨立環境變數** - 服務專用配置

#### ✅ 微服務管理功能：
- **狀態感知** - 即時監控API可用性
- **自動重啟** - 故障自動恢復
- **滾動更新** - 零停機部署
- **服務發現** - 自動註冊和發現
- **負載均衡** - 多實例分流

#### ✅ CI/CD自動化：
- **Git觸發** - 代碼提交自動觸發
- **自動構建** - Docker鏡像自動構建
- **自動測試** - 單元測試和整合測試
- **熱更新部署** - K8s滾動部署
- **部署通知** - Discord/郵件通知

### 📊 部署狀態監控

#### 實時監控指標：
- **服務健康狀態** - UP/DOWN/DEGRADED
- **響應時間** - API延遲監控  
- **錯誤率** - 4xx/5xx錯誤統計
- **資源使用** - CPU/Memory使用率
- **並發請求數** - 當前活躍連接
- **版本信息** - 部署版本和時間

#### 自動化操作：
- **異常告警** - 服務異常立即通知
- **自動擴容** - 負載過高自動擴展
- **故障轉移** - 實例故障自動切換
- **版本回滾** - 部署失敗自動回滾

### 🚀 部署流程

#### 開發階段：
```bash
# 本地開發 - Docker Compose
docker-compose up auth-service customer-service

# 單獨服務重啟
docker-compose restart auth-service
```

#### 測試階段：
```bash
# K8s測試環境部署
kubectl apply -f k8s/auth-service/

# 服務狀態檢查
kubectl get pods -l app=auth-service
```

#### 生產階段：
```bash
# 滾動更新部署
kubectl set image deployment/auth-service auth-service=coffee/auth-service:v1.2.0

# 部署狀態監控
kubectl rollout status deployment/auth-service
```

### 🔄 Git自動化工作流

#### 分支策略：
- **main** - 生產環境自動部署
- **develop** - 測試環境自動部署  
- **feature/** - 功能分支觸發構建

#### 自動化階段：
1. **代碼提交** → 觸發CI管道
2. **自動測試** → 運行單元測試
3. **構建鏡像** → Docker鏡像構建
4. **推送Registry** → 鏡像推送倉庫
5. **部署K8s** → 滾動更新部署
6. **健康檢查** → 驗證部署狀態
7. **通知結果** → Discord通知完成

**完全依照JustProject模式，實現微服務獨立管理和自動化部署！**

### 🛠️ 開發工具配置

#### ✅ 已配置文件：
- **Docker配置** - 每個微服務獨立Dockerfile
- **K8s部署配置** - 獨立的Deployment/Service/HPA
- **CI/CD管道** - GitHub Actions自動化部署
- **Docker Compose** - 本地開發環境
- **服務管理器** - 微服務狀態監控系統
- **部署腳本** - 一鍵部署管理工具
- **快速啟動** - 開發環境快速啟動

#### 🚀 快速開始：

**本地開發環境：**
```bash
# 快速啟動開發環境
./start-dev.sh

# 手動選擇服務
docker-compose -f docker-compose.dev.yml up mysql redis auth-service
```

**生產環境部署：**
```bash
# 全量部署
./deploy.sh deploy-all latest

# 單獨部署服務
./deploy.sh deploy auth-service v1.0.1

# 檢查服務狀態
./deploy.sh status
```

**服務管理：**
```bash
# 重啟服務
./deploy.sh restart auth-service

# 擴容服務
./deploy.sh scale auth-service 3

# 查看日誌
./deploy.sh logs auth-service 200
```

### 📡 服務監控端點

#### 微服務管理器 (端口 20002)：
- **系統總覽**: `GET /system/overview`
- **服務列表**: `GET /services`
- **服務狀態**: `GET /services/{name}`
- **重啟服務**: `POST /services/{name}/restart`
- **部署服務**: `POST /deploy`
- **系統指標**: `GET /system/metrics`

#### 每個微服務標準端點：
- **健康檢查**: `GET /health`
- **服務狀態**: `GET /status` 
- **版本信息**: `GET /version`

### 🔄 自動化部署流程

#### Git觸發流程：
1. **代碼提交** → 自動檢測變更服務
2. **並行構建** → 多個服務同時構建測試
3. **Docker鏡像** → 自動構建並推送Registry
4. **K8s部署** → 滾動更新零停機部署
5. **健康檢查** → 自動驗證服務可用性
6. **Discord通知** → 即時通知部署結果

#### 環境策略：
- **main分支** → 自動部署到生產環境
- **develop分支** → 自動部署到測試環境
- **feature分支** → 只構建不部署

### 💎 企業級特性

#### ✅ 可觀測性：
- **分布式追蹤** - 跨服務請求追蹤
- **指標收集** - Prometheus/Grafana監控
- **日誌聚合** - 結構化日誌收集
- **告警系統** - 異常自動通知

#### ✅ 可靠性：
- **健康檢查** - 自動故障檢測
- **自動重啟** - Pod異常自動恢復
- **負載均衡** - 多實例分流
- **優雅關閉** - 零停機滾動更新

#### ✅ 安全性：
- **網絡隔離** - K8s NetworkPolicy
- **資源限制** - CPU/Memory限制
- **密鑰管理** - K8s Secrets
- **RBAC權限** - 服務帳戶權限控制

#### ✅ 擴展性：
- **水平擴展** - HPA自動擴容
- **垂直擴展** - 資源動態調整
- **多環境** - Dev/Test/Prod環境隔離
- **藍綠部署** - 零風險部署策略

**Coffee CRM 微服務架構完全具備企業級生產能力！**