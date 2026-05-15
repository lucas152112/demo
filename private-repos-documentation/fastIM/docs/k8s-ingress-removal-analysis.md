# 移除 K8s Ingress 轉向機制影響分析

## 📅 時間：2026-02-09 11:45 (GMT+8)

## 🎯 **當前架構分析**

### **現有轉向機制**
1. **跳板機 Nginx** (`chat.justit.cc`):
   - 轉向: `localhost:8080` (本機 FastIM)
   - SSL 終止: Let's Encrypt 證書
   - WebSocket 支援: ✅ 完整配置

2. **K8s Ingress** (`im.justit.cc`):
   - 轉向: K8s Service (`fastim-backend-service:8080`)
   - 內部服務: 2 replicas FastIM pods
   - WebSocket 支援: ⚠️ 當前有問題 (301 重定向)

## 🔍 **移除 K8s Ingress 的影響評估**

### ✅ **正面影響**
1. **簡化架構**:
   - 統一入口點: 只透過跳板機 Nginx
   - 減少配置複雜度
   - 統一 SSL 證書管理

2. **解決 WebSocket 問題**:
   - 消除 K8s Ingress WebSocket 配置問題
   - 避免 301 重定向錯誤
   - 統一使用經過驗證的 Nginx 配置

3. **運維簡化**:
   - 單一外部入口點管理
   - 更容易進行流量監控和日誌分析
   - 減少故障點

### ⚠️ **潛在影響**

1. **K8s 服務隔離**:
   - 失去 K8s 原生的服務發現
   - 無法直接使用 K8s Service 負載均衡
   - 需要修改 Nginx upstream 配置

2. **擴展性考量**:
   - 跳板機成為單點入口 (需要考慮 HA)
   - K8s 服務無法直接對外暴露

## 🔧 **修改方案**

### **Option 1: 完全移除 K8s Ingress**
```yaml
# 刪除 fastim-ingress
# 保留 Service (ClusterIP)
# Nginx 直接代理到 K8s 節點
```

### **Option 2: 修改 Nginx upstream 指向 K8s**
```nginx
upstream fastim_backend {
    # 直接指向 K8s 節點
    server 103.251.113.35:30080;  # 使用 NodePort
    # 或指向 K8s Service (如果可達)
}
```

## 📊 **建議方案**

### 🎯 **推薦: 移除 K8s Ingress，統一使用 Nginx**

**理由**：
1. **當前 K8s Ingress 有問題** - WebSocket 連接失敗
2. **Nginx 配置已完善** - WebSocket 支援正常
3. **架構更簡單** - 單一入口點，易於管理
4. **性能更好** - 減少一層代理轉向

### 🔧 **具體操作步驟**

1. **修改 Nginx 配置**:
```nginx
upstream fastim_backend {
    # 選項A: 指向本機服務
    server localhost:8080;
    
    # 選項B: 指向 K8s NodePort (如需要)
    server 103.251.113.35:30080;
}
```

2. **刪除 K8s Ingress**:
```bash
kubectl delete ingress fastim-ingress -n fastim
```

3. **保留 K8s Service** (內部使用):
```yaml
# 保持 ClusterIP service
# 用於內部服務間通信
```

## 📋 **總結**

### ✅ **移除 K8s Ingress 的好處**:
- 🚀 **解決當前 WebSocket 問題**
- 🎯 **簡化架構和運維**
- 📊 **統一流量管理**
- 🔒 **統一 SSL 配置**

### 📌 **建議**:
**立即移除 K8s Ingress**，統一使用跳板機 Nginx 作為唯一外部入口點。這樣可以：
1. 立即解決 `im.justit.cc` WebSocket 連接問題
2. 簡化整體架構
3. 提高系統穩定性

**影響：幾乎沒有負面影響**，只會讓系統更簡單、更穩定！🎯