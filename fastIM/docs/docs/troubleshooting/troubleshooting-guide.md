# FastIM 故障排除手冊

## 📋 目錄
1. [診斷工具](#診斷工具)
2. [連接問題](#連接問題)
3. [性能問題](#性能問題)
4. [部署問題](#部署問題)
5. [客戶端問題](#客戶端問題)
6. [資料庫問題](#資料庫問題)
7. [監控和日誌](#監控和日誌)
8. [緊急處理](#緊急處理)

---

## 🔧 診斷工具

### 系統健康檢查

#### 快速診斷腳本
```bash
#!/bin/bash
# fastim-health-check.sh

echo "🏥 FastIM 系統健康檢查"
echo "========================"

# 檢查服務狀態
echo "📊 檢查服務狀態..."
systemctl status fastim || echo "❌ FastIM 服務未運行"
systemctl status mongodb || echo "❌ MongoDB 服務未運行"
systemctl status redis-server || echo "❌ Redis 服務未運行"

# 檢查端口
echo "🔌 檢查端口狀態..."
netstat -tuln | grep :8080 || echo "❌ FastIM 端口 8080 未開放"
netstat -tuln | grep :27017 || echo "❌ MongoDB 端口 27017 未開放"
netstat -tuln | grep :6379 || echo "❌ Redis 端口 6379 未開放"

# 檢查 API 健康
echo "🌐 檢查 API 健康..."
HTTP_CODE=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:8080/api/v1/health)
if [ "$HTTP_CODE" = "200" ]; then
    echo "✅ API 服務正常"
else
    echo "❌ API 服務異常 (HTTP $HTTP_CODE)"
fi

# 檢查系統資源
echo "💻 檢查系統資源..."
echo "CPU 使用率: $(top -bn1 | grep "Cpu(s)" | awk '{print $2}' | awk -F'%' '{print $1}')"
echo "記憶體使用: $(free | grep Mem | awk '{printf("%.1f%%", $3/$2 * 100.0)}')"
echo "磁盤使用: $(df -h / | awk 'NR==2{printf "%s", $5}')"

# 檢查日誌錯誤
echo "📋 檢查最近錯誤..."
journalctl -u fastim --since "10 minutes ago" | grep -i error | tail -5
```

#### 網路連接測試
```bash
#!/bin/bash
# network-test.sh

DOMAIN="fastim.yourdomain.com"
API_URL="https://${DOMAIN}/api/v1"
WS_URL="wss://${DOMAIN}/api/v1/ws"

echo "🌐 網路連接測試"
echo "================="

# DNS 解析測試
echo "🔍 DNS 解析測試..."
nslookup $DOMAIN || echo "❌ DNS 解析失敗"

# HTTP 連接測試
echo "📡 HTTP 連接測試..."
curl -I $API_URL/health || echo "❌ HTTP 連接失敗"

# HTTPS 證書檢查
echo "🔒 SSL 證書檢查..."
echo | openssl s_client -servername $DOMAIN -connect $DOMAIN:443 2>/dev/null | openssl x509 -noout -dates

# WebSocket 連接測試
echo "⚡ WebSocket 連接測試..."
if command -v websocat &> /dev/null; then
    timeout 5 websocat $WS_URL --text <<< '{"type":"ping"}' || echo "❌ WebSocket 連接失敗"
else
    echo "⚠️ 請安裝 websocat 進行 WebSocket 測試"
fi

# 延遲測試
echo "⏱️ 網路延遲測試..."
ping -c 4 $DOMAIN | tail -1 | awk -F '/' '{print "平均延遲: " $5 "ms"}'
```

### 性能監控工具

#### 實時監控腳本
```bash
#!/bin/bash
# fastim-monitor.sh

watch -n 1 '
echo "FastIM 實時監控 - $(date)"
echo "=========================="
echo "🖥️ 系統狀態:"
echo "CPU: $(top -bn1 | grep "Cpu(s)" | awk "{print \$2}" | awk -F"%" "{print \$1}")%"
echo "內存: $(free | grep Mem | awk "{printf(\"%.1f%%\", \$3/\$2 * 100.0)}")"
echo "磁盤: $(df -h / | awk "NR==2{printf \"%s\", \$5}")"
echo ""
echo "📊 FastIM 服務:"
systemctl is-active fastim && echo "✅ FastIM: 運行中" || echo "❌ FastIM: 已停止"
echo "連接數: $(ss -tun | grep :8080 | wc -l)"
echo ""
echo "🗄️ 資料庫狀態:"
systemctl is-active mongodb && echo "✅ MongoDB: 運行中" || echo "❌ MongoDB: 已停止"
systemctl is-active redis-server && echo "✅ Redis: 運行中" || echo "❌ Redis: 已停止"
echo ""
echo "📋 最新日誌:"
journalctl -u fastim --since "1 minute ago" --no-pager | tail -3
'
```

---

## 🌐 連接問題

### WebSocket 連接失敗

#### 問題現象
- 客戶端無法建立 WebSocket 連接
- 連接建立後立即斷開
- 消息發送失敗

#### 診斷步驟
```bash
# 1. 檢查 WebSocket 端口
netstat -tuln | grep :8080

# 2. 檢查防火牆設置
ufw status
iptables -L | grep 8080

# 3. 測試 WebSocket 連接
websocat ws://localhost:8080/api/v1/ws

# 4. 檢查 Nginx 配置（如果使用代理）
nginx -t
cat /etc/nginx/sites-enabled/fastim
```

#### 解決方案
**問題**: 防火牆阻擋
```bash
# Ubuntu/Debian
sudo ufw allow 8080
sudo ufw allow 443

# CentOS/RHEL
sudo firewall-cmd --permanent --add-port=8080/tcp
sudo firewall-cmd --permanent --add-port=443/tcp
sudo firewall-cmd --reload
```

**問題**: Nginx 代理配置錯誤
```nginx
# 正確的 WebSocket 代理配置
location /api/v1/ws {
    proxy_pass http://localhost:8080;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_read_timeout 86400;
    proxy_send_timeout 86400;
}
```

**問題**: SSL/TLS 證書問題
```bash
# 檢查證書有效性
openssl x509 -in /etc/letsencrypt/live/yourdomain.com/cert.pem -text -noout

# 更新證書
sudo certbot renew --dry-run
sudo certbot renew
```

### 認證失敗

#### 問題現象
- 登入返回 401 錯誤
- JWT Token 無效
- 認證後立即失效

#### 診斷步驟
```bash
# 1. 檢查系統時間
date
timedatectl status

# 2. 測試認證接口
curl -X POST http://localhost:8080/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"test","password":"test"}'

# 3. 驗證 JWT Token
# 使用 jwt.io 或 jwt-cli 工具驗證
```

#### 解決方案
**問題**: 系統時間不同步
```bash
# 同步系統時間
sudo ntpdate -s time.nist.gov
sudo timedatectl set-ntp true
```

**問題**: JWT 密鑰配置錯誤
```bash
# 檢查配置文件
cat /opt/fastim/config/config.yaml | grep jwt_secret

# 生成新的 JWT 密鑰
openssl rand -base64 32
```

### 數據庫連接問題

#### MongoDB 連接失敗
```bash
# 檢查 MongoDB 狀態
sudo systemctl status mongodb
mongosh --eval "db.adminCommand('ping')"

# 檢查連接配置
cat /opt/fastim/config/config.yaml | grep mongodb

# 測試連接
mongosh "mongodb://localhost:27017/fastim"
```

#### Redis 連接失敗
```bash
# 檢查 Redis 狀態
sudo systemctl status redis-server
redis-cli ping

# 檢查 Redis 配置
cat /etc/redis/redis.conf | grep bind
cat /etc/redis/redis.conf | grep requirepass

# 測試連接
redis-cli -a your_password ping
```

---

## ⚡ 性能問題

### 高 CPU 使用率

#### 診斷工具
```bash
# 查看進程 CPU 使用
top -p $(pgrep fastim)
htop -p $(pgrep fastim)

# 查看 Go 程序性能分析
curl http://localhost:8080/debug/pprof/profile?seconds=30 > cpu.prof
go tool pprof cpu.prof
```

#### 解決方案
```bash
# 1. 調整 Go GC 參數
export GOGC=100
export GOMEMLIMIT=1GiB

# 2. 限制併發連接數
# 在配置文件中設置
websocket:
  max_connections: 1000
  worker_pool_size: 100

# 3. 優化心跳頻率
websocket:
  heartbeat_interval: 30s
  adaptive_heartbeat: true
```

### 高記憶體使用

#### 記憶體洩漏檢測
```bash
# 檢查記憶體使用趨勢
ps aux | grep fastim
cat /proc/$(pgrep fastim)/status | grep VmRSS

# Go 記憶體分析
curl http://localhost:8080/debug/pprof/heap > heap.prof
go tool pprof heap.prof
```

#### 解決方案
```yaml
# 配置記憶體限制
version: '3.8'
services:
  fastim:
    deploy:
      resources:
        limits:
          memory: 1G
        reservations:
          memory: 512M
```

### 數據庫性能問題

#### MongoDB 優化
```javascript
// 創建索引
db.messages.createIndex({ "channel": 1, "timestamp": -1 })
db.messages.createIndex({ "sender": 1, "timestamp": -1 })

// 檢查慢查詢
db.setProfilingLevel(2, { slowms: 100 })
db.system.profile.find().sort({ ts: -1 }).limit(5)

// 統計信息
db.messages.stats()
db.messages.totalIndexSize()
```

#### Redis 優化
```bash
# 檢查 Redis 記憶體使用
redis-cli info memory

# 設置記憶體策略
redis-cli config set maxmemory 2gb
redis-cli config set maxmemory-policy allkeys-lru

# 檢查鍵空間
redis-cli info keyspace
```

---

## 🚀 部署問題

### Docker 部署問題

#### 容器無法啟動
```bash
# 檢查容器日誌
docker logs fastim-backend
docker logs fastim-mongodb
docker logs fastim-redis

# 檢查容器狀態
docker ps -a
docker inspect fastim-backend

# 檢查資源使用
docker stats
```

#### 網絡連接問題
```bash
# 檢查 Docker 網絡
docker network ls
docker network inspect fastim-network

# 測試容器間連接
docker exec fastim-backend ping mongodb
docker exec fastim-backend ping redis
```

### Kubernetes 部署問題

#### Pod 啟動失敗
```bash
# 檢查 Pod 狀態
kubectl get pods -n fastim
kubectl describe pod fastim-backend-xxx -n fastim

# 檢查日誌
kubectl logs fastim-backend-xxx -n fastim
kubectl logs -f deployment/fastim-backend -n fastim

# 檢查資源
kubectl top pods -n fastim
kubectl get events -n fastim
```

#### 服務發現問題
```bash
# 檢查服務
kubectl get svc -n fastim
kubectl describe svc fastim-backend-service -n fastim

# 檢查端點
kubectl get endpoints -n fastim

# 測試服務連接
kubectl exec -it fastim-backend-xxx -n fastim -- curl mongodb-service:27017
```

---

## 💻 客戶端問題

### Flutter 客戶端問題

#### 編譯錯誤
```bash
# 清理編譯緩存
flutter clean
flutter pub get

# 檢查依賴
flutter doctor -v
flutter pub deps

# 重建項目
flutter build linux --verbose
flutter build windows --verbose
```

#### 運行時錯誤
```bash
# 檢查日誌
flutter logs

# Debug 模式運行
flutter run --debug --verbose

# 檢查網絡權限
# Linux: 檢查防火牆設置
# Windows: 檢查 Windows Defender
# macOS: 檢查系統偏好設置
```

### Web 客戶端問題

#### 瀏覽器兼容性
```javascript
// 檢查 WebSocket 支持
if (!window.WebSocket) {
    console.error('WebSocket not supported');
}

// 檢查本地存儲支持
if (!window.localStorage) {
    console.error('LocalStorage not supported');
}

// 檢查通知權限
if (!('Notification' in window)) {
    console.error('Notifications not supported');
}
```

#### CORS 問題
```javascript
// 客戶端設置
const config = {
    headers: {
        'Content-Type': 'application/json',
        'Access-Control-Allow-Origin': '*'
    },
    withCredentials: false
};

// 服務器端設置 (已在後端配置)
```

---

## 🗄️ 資料庫問題

### MongoDB 問題

#### 連接數過多
```bash
# 檢查當前連接
mongosh --eval "db.serverStatus().connections"

# 設置連接池大小
# 在配置文件中設置
database:
  mongodb:
    max_pool_size: 100
    min_pool_size: 10
```

#### 磁盤空間不足
```bash
# 檢查數據庫大小
mongosh --eval "db.stats()"

# 清理舊數據
mongosh fastim --eval "db.messages.deleteMany({timestamp: {$lt: new Date(Date.now() - 30*24*60*60*1000)}})"

# 壓縮數據庫
mongosh fastim --eval "db.runCommand({compact: 'messages'})"
```

### Redis 問題

#### 記憶體不足
```bash
# 檢查記憶體使用
redis-cli info memory

# 清理過期鍵
redis-cli --scan --pattern "session:*" | head -10000 | xargs redis-cli del

# 設置自動過期
redis-cli config set maxmemory-policy allkeys-lru
```

#### 持久化問題
```bash
# 檢查 RDB 備份
ls -la /var/lib/redis/

# 手動保存
redis-cli bgsave

# 檢查 AOF 日誌
redis-cli config get appendonly
tail -f /var/lib/redis/appendonly.aof
```

---

## 📊 監控和日誌

### 日誌分析

#### 系統日誌
```bash
# FastIM 服務日誌
journalctl -u fastim -f
journalctl -u fastim --since "1 hour ago" | grep ERROR

# 應用程序日誌
tail -f /opt/fastim/logs/app.log
grep -n "ERROR\|FATAL" /opt/fastim/logs/app.log

# Web 服務器日誌
tail -f /var/log/nginx/access.log
tail -f /var/log/nginx/error.log
```

#### 日誌分析腳本
```bash
#!/bin/bash
# log-analysis.sh

LOG_FILE="/opt/fastim/logs/app.log"
TIMEFRAME="1 hour ago"

echo "📊 FastIM 日誌分析報告"
echo "======================="

# 錯誤統計
echo "❌ 錯誤統計:"
journalctl -u fastim --since "$TIMEFRAME" | grep -c "ERROR"

# 連接統計
echo "🔌 WebSocket 連接:"
grep "WebSocket connection" $LOG_FILE | tail -10

# 性能指標
echo "⚡ 性能指標:"
grep "Response time" $LOG_FILE | tail -5

# 最頻繁的錯誤
echo "🔥 最頻繁錯誤:"
grep ERROR $LOG_FILE | cut -d' ' -f4- | sort | uniq -c | sort -nr | head -5
```

### 監控設置

#### Prometheus 配置
```yaml
# prometheus.yml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'fastim'
    static_configs:
      - targets: ['localhost:8080']
    metrics_path: '/api/v1/metrics'
    scrape_interval: 10s
```

#### Grafana 報警規則
```yaml
# alerting rules
groups:
  - name: fastim_alerts
    rules:
      - alert: HighCPUUsage
        expr: fastim_cpu_usage_percent > 80
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "FastIM CPU 使用率過高"
          
      - alert: WebSocketConnectionDrop
        expr: rate(fastim_websocket_disconnections_total[5m]) > 10
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "WebSocket 連接大量斷開"
```

---

## 🚨 緊急處理

### 服務器過載

#### 緊急處理步驟
```bash
# 1. 立即檢查系統負載
top
free -h
df -h

# 2. 重啟 FastIM 服務
sudo systemctl restart fastim

# 3. 清理系統緩存
sudo sync && sudo sysctl vm.drop_caches=3

# 4. 暫時限制連接數
# 修改配置文件
websocket:
  max_connections: 500  # 降低到 500

# 5. 通知用戶
curl -X POST http://localhost:8080/api/v1/admin/broadcast \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -d '{"message": "系統維護中，請稍後重試"}'
```

### 資料庫崩潰

#### MongoDB 恢復
```bash
# 1. 停止 FastIM 服務
sudo systemctl stop fastim

# 2. 檢查 MongoDB 狀態
sudo systemctl status mongodb

# 3. 嘗試修復數據庫
mongod --repair --dbpath /var/lib/mongodb

# 4. 從備份恢復（如果需要）
mongorestore --db fastim /backup/mongodb/fastim

# 5. 重啟服務
sudo systemctl start mongodb
sudo systemctl start fastim
```

#### Redis 恢復
```bash
# 1. 檢查 Redis 數據文件
ls -la /var/lib/redis/

# 2. 從 RDB 備份恢復
sudo cp /backup/redis/dump.rdb /var/lib/redis/
sudo chown redis:redis /var/lib/redis/dump.rdb

# 3. 重啟 Redis
sudo systemctl restart redis-server

# 4. 驗證數據
redis-cli info keyspace
```

### 安全事件處理

#### 檢測到異常流量
```bash
# 1. 查看訪問日誌
tail -f /var/log/nginx/access.log | grep -E "(404|500|POST)"

# 2. 阻擋可疑 IP
sudo ufw deny from suspicious_ip_address

# 3. 啟用更嚴格的限流
# 在 Nginx 配置中添加
limit_req_zone $binary_remote_addr zone=api:10m rate=10r/m;
limit_req_zone $binary_remote_addr zone=login:10m rate=1r/m;

# 4. 檢查用戶活動
grep "Failed login" /opt/fastim/logs/security.log
```

### 災難恢復計劃

#### 完整系統恢復
```bash
#!/bin/bash
# disaster-recovery.sh

echo "🚨 開始災難恢復程序"

# 1. 停止所有服務
sudo systemctl stop fastim nginx mongodb redis-server

# 2. 恢復數據庫
echo "恢復 MongoDB..."
mongorestore --drop --db fastim /backup/mongodb/latest/

echo "恢復 Redis..."
sudo cp /backup/redis/latest/dump.rdb /var/lib/redis/
sudo chown redis:redis /var/lib/redis/dump.rdb

# 3. 恢復配置文件
sudo cp /backup/config/fastim-config.yaml /opt/fastim/config/
sudo cp /backup/config/nginx.conf /etc/nginx/

# 4. 重新啟動服務
sudo systemctl start redis-server
sudo systemctl start mongodb
sleep 10
sudo systemctl start fastim
sudo systemctl start nginx

# 5. 驗證服務
curl -f http://localhost:8080/api/v1/health

echo "✅ 災難恢復完成"
```

---

## 📞 技術支持

### 收集診斷信息

#### 系統信息收集腳本
```bash
#!/bin/bash
# collect-diagnostics.sh

REPORT_FILE="fastim-diagnostics-$(date +%Y%m%d_%H%M%S).txt"

echo "FastIM 診斷報告" > $REPORT_FILE
echo "生成時間: $(date)" >> $REPORT_FILE
echo "====================" >> $REPORT_FILE

# 系統信息
echo "系統信息:" >> $REPORT_FILE
uname -a >> $REPORT_FILE
cat /etc/os-release >> $REPORT_FILE

# 服務狀態
echo "服務狀態:" >> $REPORT_FILE
systemctl status fastim mongodb redis-server >> $REPORT_FILE

# 資源使用
echo "資源使用:" >> $REPORT_FILE
free -h >> $REPORT_FILE
df -h >> $REPORT_FILE

# 網絡狀態
echo "網絡狀態:" >> $REPORT_FILE
netstat -tuln | grep -E "(8080|27017|6379)" >> $REPORT_FILE

# 最近日誌
echo "最近日誌:" >> $REPORT_FILE
journalctl -u fastim --since "1 hour ago" >> $REPORT_FILE

# 配置文件
echo "配置文件:" >> $REPORT_FILE
cat /opt/fastim/config/config.yaml >> $REPORT_FILE

echo "✅ 診斷信息已保存到 $REPORT_FILE"
echo "請將此文件發送給技術支持團隊"
```

### 聯繫支持

**技術支持渠道**:
- 📧 **郵箱**: support@fastim.com
- 💬 **實時聊天**: https://support.fastim.com
- 🐛 **Bug 報告**: https://github.com/fastim/fastim/issues
- 📱 **緊急支持**: +1-800-FASTIM (24/7)

**提供信息**:
1. 問題描述和重現步驟
2. 系統診斷報告
3. 錯誤日誌片段
4. 系統配置信息
5. 預期行為和實際行為

---

**故障排除手冊最後更新**: 2024-01-15  
**版本**: v1.2.0

記住：當遇到問題時，先檢查日誌，再嘗試重啟，最後聯繫支持！🔧