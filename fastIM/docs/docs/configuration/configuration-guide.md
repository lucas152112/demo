# FastIM 配置參數說明

## 📋 目錄
1. [配置概覽](#配置概覽)
2. [服務器配置](#服務器配置)
3. [數據庫配置](#數據庫配置)
4. [WebSocket 配置](#websocket-配置)
5. [認證配置](#認證配置)
6. [日誌配置](#日誌配置)
7. [性能調優](#性能調優)
8. [部署環境配置](#部署環境配置)

---

## 🔧 配置概覽

### 配置文件結構

FastIM 支持多種配置方式：
1. **YAML 配置文件** (推薦)
2. **環境變數**
3. **命令行參數**
4. **默認配置**

### 配置優先級
```
命令行參數 > 環境變數 > YAML 配置文件 > 默認配置
```

### 主要配置文件

#### 完整配置示例 (config.yaml)
```yaml
# FastIM 完整配置文件
# 版本: 1.2.0-progressive

# ================================
# 服務器基本配置
# ================================
server:
  # 服務器監聽配置
  host: "0.0.0.0"                    # 監聽地址，生產環境建議 0.0.0.0
  port: "8080"                       # 監聽端口
  mode: "release"                    # 運行模式: debug, release, test
  
  # 超時配置
  timeout:
    read: 15s                        # 讀取超時
    write: 15s                       # 寫入超時  
    idle: 60s                        # 空閒連接超時
    shutdown: 30s                    # 優雅關閉超時
  
  # TLS/SSL 配置
  tls:
    enabled: true                    # 啟用 HTTPS
    cert_file: "/etc/ssl/certs/fastim.crt"
    key_file: "/etc/ssl/private/fastim.key"
    min_version: "1.2"               # 最低 TLS 版本

# ================================
# 數據庫配置
# ================================
database:
  # MongoDB 配置
  mongodb:
    url: "mongodb://localhost:27017/fastim"  # 連接字符串
    database: "fastim"                      # 數據庫名稱
    timeout: 10s                           # 連接超時
    max_pool_size: 100                     # 最大連接池大小
    min_pool_size: 10                      # 最小連接池大小
    max_idle_time: 5m                      # 最大空閒時間
    
    # 認證配置
    auth:
      username: "fastim_user"
      password: "secure_password"
      source: "admin"                      # 認證數據庫
    
    # 複製集配置 (可選)
    replica_set:
      enabled: false
      name: "fastim-replica"
      hosts:
        - "mongo1:27017"
        - "mongo2:27017" 
        - "mongo3:27017"
  
  # Redis 配置
  redis:
    # 基本連接
    url: "redis://localhost:6379/0"         # 連接字符串
    password: ""                            # 密碼
    db: 0                                  # 數據庫編號
    
    # 連接池配置
    pool_size: 100                         # 連接池大小
    min_idle_conns: 10                     # 最小空閒連接
    max_retries: 3                         # 重試次數
    
    # 超時配置
    dial_timeout: 5s                       # 連接超時
    read_timeout: 3s                       # 讀取超時
    write_timeout: 3s                      # 寫入超時
    
    # 集群配置 (可選)
    cluster:
      enabled: false
      addresses:
        - "redis1:6379"
        - "redis2:6379"
        - "redis3:6379"

# ================================
# WebSocket 配置
# ================================
websocket:
  # 連接限制
  max_connections: 10000                   # 最大併發連接數
  max_connections_per_ip: 100              # 每 IP 最大連接數
  
  # 消息配置
  message_buffer_size: 256                 # 消息緩衝區大小
  max_message_size: 32768                  # 最大消息大小 (bytes)
  
  # 心跳配置
  heartbeat:
    enabled: true                          # 啟用心跳檢測
    interval: 30s                          # 默認心跳間隔
    timeout: 90s                           # 心跳超時時間
    
    # 漸進式心跳配置
    progressive:
      enabled: true                        # 啟用漸進式心跳
      base_interval: 10s                   # 基礎間隔
      max_interval: 120s                   # 最大間隔
      long_connection_threshold: 300s      # 長連接判定閾值
      adjustment_factor: 1.2               # 調整係數
  
  # CORS 配置
  cors:
    allow_origins:
      - "https://fastim.com"
      - "https://*.fastim.com"
      - "http://localhost:3000"            # 開發環境
    allow_methods:
      - "GET"
      - "POST"
      - "PUT" 
      - "DELETE"
      - "OPTIONS"
    allow_headers:
      - "Authorization"
      - "Content-Type"
      - "X-Requested-With"
  
  # 壓縮配置
  compression:
    enabled: true                          # 啟用消息壓縮
    level: 1                              # 壓縮級別 (1-9)
    threshold: 1024                       # 壓縮閾值 (bytes)

# ================================
# 認證和安全配置
# ================================
security:
  # JWT 配置
  jwt:
    secret: "your-256-bit-secret-key"      # JWT 簽名密鑰 (生產環境必須更改)
    issuer: "fastim-server"                # JWT 發行者
    expiry: 24h                           # Token 過期時間
    refresh_expiry: 168h                  # Refresh Token 過期時間 (7天)
    algorithm: "HS256"                    # 簽名算法
  
  # 密碼策略
  password:
    min_length: 8                         # 最小長度
    require_uppercase: true               # 需要大寫字母
    require_lowercase: true               # 需要小寫字母
    require_numbers: true                 # 需要數字
    require_symbols: false                # 需要特殊字符
    max_age: 90                          # 密碼最大使用天數 (0=無限制)
  
  # API 限流
  rate_limit:
    enabled: true                         # 啟用限流
    
    # 全局限制
    global:
      requests_per_minute: 1000           # 每分鐘全局請求數
      burst: 100                         # 突發請求數
    
    # 用戶限制
    per_user:
      requests_per_minute: 100            # 每用戶每分鐘請求數
      burst: 20                          # 用戶突發請求數
    
    # API 特定限制
    endpoints:
      "/api/v1/auth/login":
        requests_per_minute: 10           # 登入接口限制
        window: 15m                       # 滑動窗口
      "/api/v1/files/upload":
        requests_per_minute: 5            # 文件上傳限制
        max_size: 50MB                    # 最大文件大小

# ================================
# 日誌配置
# ================================
logging:
  # 基本配置
  level: "info"                           # 日誌級別: debug, info, warn, error
  format: "json"                          # 日誌格式: text, json
  output: "stdout"                        # 輸出: stdout, stderr, file
  
  # 文件輸出配置
  file:
    enabled: false                        # 啟用文件輸出
    path: "/var/log/fastim/app.log"       # 日誌文件路徑
    max_size: 100MB                       # 最大文件大小
    max_age: 7                           # 最大保存天數
    max_backups: 10                      # 最大備份文件數
    compress: true                       # 壓縮舊日誌
  
  # 結構化日誌字段
  fields:
    service: "fastim"                     # 服務名稱
    version: "1.2.0"                      # 版本號
    environment: "production"             # 環境
  
  # 敏感信息過濾
  redact:
    enabled: true                         # 啟用敏感信息過濾
    fields:                              # 需要過濾的字段
      - "password"
      - "token"
      - "authorization"
      - "cookie"

# ================================
# 消息處理配置
# ================================
message:
  # 處理配置
  processing:
    worker_count: 10                      # Worker 數量
    queue_size: 10000                     # 隊列大小
    batch_size: 100                       # 批處理大小
    flush_interval: 5s                    # 刷新間隔
  
  # 消息持久化
  persistence:
    enabled: true                         # 啟用持久化
    async: true                          # 異步保存
    compression: true                     # 壓縮存儲
    
    # 保留策略
    retention:
      max_age: 90                        # 最大保存天數 (0=永久保存)
      max_messages: 1000000              # 每頻道最大消息數
      cleanup_interval: 24h              # 清理檢查間隔
  
  # 消息過濾
  filter:
    enabled: true                         # 啟用內容過濾
    profanity: true                      # 過濾不良內容
    spam_detection: true                 # 垃圾消息檢測
    max_length: 4096                     # 最大消息長度
    
    # 自定義過濾規則
    custom_rules:
      - pattern: "\\b(spam|advertisement)\\b"
        action: "block"
      - pattern: "http[s]?://(?!fastim\\.com)"
        action: "warn"

# ================================
# 文件處理配置
# ================================
file:
  # 上傳配置
  upload:
    enabled: true                         # 啟用文件上傳
    max_size: 50MB                        # 最大文件大小
    max_files_per_user: 1000              # 每用戶最大文件數
    
    # 允許的文件類型
    allowed_types:
      - "image/jpeg"
      - "image/png"
      - "image/gif"
      - "image/webp"
      - "application/pdf"
      - "text/plain"
      - "application/zip"
    
    # 禁止的文件類型
    blocked_types:
      - "application/x-executable"
      - "application/x-msdos-program"
  
  # 存儲配置
  storage:
    type: "local"                         # 存儲類型: local, s3, gcs
    path: "/var/lib/fastim/uploads"       # 本地存儲路徑
    
    # 雲存儲配置 (S3 兼容)
    s3:
      endpoint: ""                        # S3 端點
      region: "us-east-1"                 # 區域
      bucket: "fastim-files"              # 存儲桶
      access_key: ""                      # 訪問密鑰
      secret_key: ""                      # 密鑰
      use_ssl: true                       # 使用 SSL
    
    # CDN 配置
    cdn:
      enabled: false                      # 啟用 CDN
      base_url: "https://cdn.fastim.com"  # CDN 基礎 URL
      cache_ttl: 86400                    # 緩存 TTL (秒)
  
  # 圖片處理
  image:
    thumbnail:
      enabled: true                       # 啟用縮略圖
      sizes:                             # 縮略圖尺寸
        - name: "small"
          width: 150
          height: 150
        - name: "medium" 
          width: 300
          height: 300
      quality: 85                        # JPEG 質量
      format: "webp"                     # 輸出格式

# ================================
# 監控和指標配置
# ================================
monitoring:
  # 健康檢查
  health:
    enabled: true                         # 啟用健康檢查
    endpoint: "/api/v1/health"            # 健康檢查端點
    interval: 30s                        # 檢查間隔
    timeout: 5s                          # 檢查超時
  
  # Prometheus 指標
  metrics:
    enabled: true                         # 啟用指標收集
    endpoint: "/api/v1/metrics"           # 指標端點
    namespace: "fastim"                   # 指標命名空間
    
    # 自定義標籤
    labels:
      service: "fastim-backend"
      version: "1.2.0"
  
  # 分佈式追蹤
  tracing:
    enabled: false                        # 啟用追蹤
    provider: "jaeger"                    # 追蹤提供商: jaeger, zipkin
    endpoint: "http://jaeger:14268"       # 追蹤端點
    sample_rate: 0.1                      # 採樣率
  
  # 性能分析
  profiling:
    enabled: false                        # 啟用性能分析 (僅開發環境)
    endpoint: "/debug/pprof"              # pprof 端點

# ================================
# 緩存配置
# ================================
cache:
  # 內存緩存
  memory:
    enabled: true                         # 啟用內存緩存
    max_size: 100MB                       # 最大緩存大小
    ttl: 1h                              # 默認 TTL
    cleanup_interval: 10m                 # 清理間隔
  
  # Redis 緩存
  redis:
    enabled: true                         # 啟用 Redis 緩存
    key_prefix: "fastim:cache:"           # 鍵前綴
    default_ttl: 3600                     # 默認 TTL (秒)
    
    # 緩存策略
    strategies:
      user_sessions:
        ttl: 86400                        # 用戶會話緩存
        max_entries: 10000
      message_cache:
        ttl: 3600                         # 消息緩存
        max_entries: 50000
      channel_info:
        ttl: 1800                         # 頻道信息緩存
        max_entries: 1000

# ================================
# 開發和調試配置
# ================================
development:
  # 調試配置
  debug:
    enabled: false                        # 啟用調試模式
    verbose: false                        # 詳細日誌
    profiling: false                      # 性能分析
  
  # 熱重載
  hot_reload:
    enabled: false                        # 啟用熱重載 (僅開發)
    watch_paths:                          # 監聽路徑
      - "./internal"
      - "./pkg"
      - "./config"
  
  # 測試配置
  testing:
    mock_external_services: false        # 模擬外部服務
    test_data_path: "./testdata"          # 測試數據路徑

# ================================
# 第三方服務配置
# ================================
external:
  # 郵件服務
  email:
    enabled: true                         # 啟用郵件服務
    provider: "smtp"                      # 提供商: smtp, sendgrid
    
    smtp:
      host: "smtp.gmail.com"
      port: 587
      username: "noreply@fastim.com"
      password: "app_password"
      tls: true
  
  # 推送通知
  push:
    enabled: false                        # 啟用推送通知
    
    fcm:
      enabled: false
      server_key: ""
      project_id: ""
    
    apns:
      enabled: false
      key_file: ""
      key_id: ""
      team_id: ""
  
  # 短信服務
  sms:
    enabled: false                        # 啟用短信服務
    provider: "twilio"                    # 提供商
    
    twilio:
      account_sid: ""
      auth_token: ""
      from_number: ""
```

---

## 🌐 環境變數配置

### 基本環境變數
```bash
# 服務器配置
export SERVER_HOST=0.0.0.0
export SERVER_PORT=8080
export SERVER_MODE=release

# 數據庫連接
export MONGODB_URL="mongodb://user:pass@localhost:27017/fastim"
export REDIS_URL="redis://:password@localhost:6379/0"

# 安全配置
export JWT_SECRET="your-very-secure-256-bit-secret-key-here"
export ENCRYPTION_KEY="32-byte-encryption-key-for-sensitive-data"

# 日誌配置
export LOG_LEVEL=info
export LOG_FORMAT=json

# 性能配置
export MAX_CONNECTIONS=10000
export WORKER_COUNT=10
export GOGC=100
export GOMEMLIMIT=1GiB
```

### Docker 環境變數
```dockerfile
# Dockerfile 環境變數設置
ENV SERVER_PORT=8080
ENV SERVER_MODE=release
ENV LOG_LEVEL=info
ENV MONGODB_URL="mongodb://fastim-user:secure-password@mongodb:27017/fastim"
ENV REDIS_URL="redis://:redis-password@redis:6379/0"
ENV JWT_SECRET="production-jwt-secret-key"
```

### Kubernetes ConfigMap
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: fastim-config
  namespace: fastim
data:
  # 服務配置
  SERVER_HOST: "0.0.0.0"
  SERVER_PORT: "8080"
  SERVER_MODE: "release"
  
  # 數據庫配置
  MONGODB_URL: "mongodb://fastim-user:secure-password@mongodb-service:27017/fastim"
  REDIS_URL: "redis://:redis-password@redis-service:6379/0"
  
  # 應用配置
  MAX_CONNECTIONS: "10000"
  WORKER_COUNT: "20"
  LOG_LEVEL: "info"
  
  # config.yaml 內容
  config.yaml: |
    server:
      host: "0.0.0.0"
      port: "8080"
      mode: "release"
    # ... 其他配置內容
```

---

## 🔧 配置驗證和載入

### Go 配置結構體
```go
package config

import (
    "fmt"
    "os"
    "time"
    "gopkg.in/yaml.v2"
)

// Config 完整配置結構體
type Config struct {
    Server      ServerConfig      `yaml:"server"`
    Database    DatabaseConfig    `yaml:"database"`
    WebSocket   WebSocketConfig   `yaml:"websocket"`
    Security    SecurityConfig    `yaml:"security"`
    Logging     LoggingConfig     `yaml:"logging"`
    Message     MessageConfig     `yaml:"message"`
    File        FileConfig        `yaml:"file"`
    Monitoring  MonitoringConfig  `yaml:"monitoring"`
    Cache       CacheConfig       `yaml:"cache"`
    Development DevelopmentConfig `yaml:"development"`
    External    ExternalConfig    `yaml:"external"`
}

type ServerConfig struct {
    Host    string        `yaml:"host"`
    Port    string        `yaml:"port"`
    Mode    string        `yaml:"mode"`
    Timeout TimeoutConfig `yaml:"timeout"`
    TLS     TLSConfig     `yaml:"tls"`
}

type TimeoutConfig struct {
    Read     time.Duration `yaml:"read"`
    Write    time.Duration `yaml:"write"`
    Idle     time.Duration `yaml:"idle"`
    Shutdown time.Duration `yaml:"shutdown"`
}

type DatabaseConfig struct {
    MongoDB MongoDBConfig `yaml:"mongodb"`
    Redis   RedisConfig   `yaml:"redis"`
}

type WebSocketConfig struct {
    MaxConnections       int           `yaml:"max_connections"`
    MaxConnectionsPerIP  int           `yaml:"max_connections_per_ip"`
    MessageBufferSize    int           `yaml:"message_buffer_size"`
    MaxMessageSize       int           `yaml:"max_message_size"`
    Heartbeat           HeartbeatConfig `yaml:"heartbeat"`
    CORS                CORSConfig      `yaml:"cors"`
    Compression         CompressionConfig `yaml:"compression"`
}

// 配置載入函數
func LoadConfig(configPath string) (*Config, error) {
    // 1. 設置默認值
    config := getDefaultConfig()
    
    // 2. 載入 YAML 文件
    if configPath != "" {
        if err := loadYAMLConfig(config, configPath); err != nil {
            return nil, fmt.Errorf("failed to load YAML config: %w", err)
        }
    }
    
    // 3. 覆蓋環境變數
    overrideWithEnvVars(config)
    
    // 4. 驗證配置
    if err := validateConfig(config); err != nil {
        return nil, fmt.Errorf("config validation failed: %w", err)
    }
    
    return config, nil
}

// 載入 YAML 配置
func loadYAMLConfig(config *Config, path string) error {
    data, err := os.ReadFile(path)
    if err != nil {
        return err
    }
    
    return yaml.Unmarshal(data, config)
}

// 環境變數覆蓋
func overrideWithEnvVars(config *Config) {
    if host := os.Getenv("SERVER_HOST"); host != "" {
        config.Server.Host = host
    }
    if port := os.Getenv("SERVER_PORT"); port != "" {
        config.Server.Port = port
    }
    if mongoURL := os.Getenv("MONGODB_URL"); mongoURL != "" {
        config.Database.MongoDB.URL = mongoURL
    }
    if redisURL := os.Getenv("REDIS_URL"); redisURL != "" {
        config.Database.Redis.URL = redisURL
    }
    if jwtSecret := os.Getenv("JWT_SECRET"); jwtSecret != "" {
        config.Security.JWT.Secret = jwtSecret
    }
    if logLevel := os.Getenv("LOG_LEVEL"); logLevel != "" {
        config.Logging.Level = logLevel
    }
}

// 配置驗證
func validateConfig(config *Config) error {
    // 驗證必需字段
    if config.Database.MongoDB.URL == "" {
        return fmt.Errorf("mongodb URL is required")
    }
    if config.Database.Redis.URL == "" {
        return fmt.Errorf("redis URL is required")
    }
    if config.Security.JWT.Secret == "" {
        return fmt.Errorf("JWT secret is required")
    }
    
    // 驗證數值範圍
    if config.WebSocket.MaxConnections < 1 || config.WebSocket.MaxConnections > 100000 {
        return fmt.Errorf("max_connections must be between 1 and 100000")
    }
    if config.Message.Processing.WorkerCount < 1 || config.Message.Processing.WorkerCount > 1000 {
        return fmt.Errorf("worker_count must be between 1 and 1000")
    }
    
    // 驗證枚舉值
    validLogLevels := map[string]bool{"debug": true, "info": true, "warn": true, "error": true}
    if !validLogLevels[config.Logging.Level] {
        return fmt.Errorf("invalid log level: %s", config.Logging.Level)
    }
    
    return nil
}

// 默認配置
func getDefaultConfig() *Config {
    return &Config{
        Server: ServerConfig{
            Host: "0.0.0.0",
            Port: "8080",
            Mode: "release",
            Timeout: TimeoutConfig{
                Read:     15 * time.Second,
                Write:    15 * time.Second,
                Idle:     60 * time.Second,
                Shutdown: 30 * time.Second,
            },
        },
        Database: DatabaseConfig{
            MongoDB: MongoDBConfig{
                URL:         "mongodb://localhost:27017/fastim",
                Database:    "fastim",
                Timeout:     10 * time.Second,
                MaxPoolSize: 100,
                MinPoolSize: 10,
            },
            Redis: RedisConfig{
                URL:        "redis://localhost:6379/0",
                PoolSize:   100,
                MaxRetries: 3,
                DialTimeout: 5 * time.Second,
            },
        },
        WebSocket: WebSocketConfig{
            MaxConnections:      10000,
            MaxConnectionsPerIP: 100,
            MessageBufferSize:   256,
            MaxMessageSize:      32768,
            Heartbeat: HeartbeatConfig{
                Enabled:  true,
                Interval: 30 * time.Second,
                Timeout:  90 * time.Second,
            },
        },
        Security: SecurityConfig{
            JWT: JWTConfig{
                Secret: "default-jwt-secret-change-in-production",
                Expiry: 24 * time.Hour,
            },
        },
        Logging: LoggingConfig{
            Level:  "info",
            Format: "json",
            Output: "stdout",
        },
    }
}
```

---

## 🚀 性能調優配置

### Go 運行時優化
```bash
# Go 垃圾回收器優化
export GOGC=100                    # GC 目標百分比 (默認 100)
export GOMEMLIMIT=2GiB            # 記憶體限制
export GOMAXPROCS=8               # 最大 CPU 核心數

# Go 調試和 pprof
export GODEBUG=gctrace=1          # GC 跟蹤
export GODEBUG=schedtrace=1000    # 調度器跟蹤
```

### 系統級優化
```bash
# Linux 內核參數調優
# /etc/sysctl.conf

# 網路參數
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216
net.ipv4.tcp_rmem = 4096 65536 16777216
net.ipv4.tcp_wmem = 4096 65536 16777216
net.core.netdev_max_backlog = 5000
net.ipv4.tcp_congestion_control = bbr

# 文件描述符限制
fs.file-max = 2097152
fs.nr_open = 2097152

# 進程限制
kernel.pid_max = 4194304

# 應用到系統
sudo sysctl -p
```

### Docker 容器優化
```yaml
# docker-compose.yml 性能配置
version: '3.8'
services:
  fastim-backend:
    image: fastim:latest
    deploy:
      resources:
        limits:
          cpus: '2.0'
          memory: 2G
        reservations:
          cpus: '1.0'
          memory: 1G
    environment:
      - GOGC=100
      - GOMEMLIMIT=1GiB
      - GOMAXPROCS=2
    ulimits:
      nofile:
        soft: 65536
        hard: 65536
    sysctls:
      - net.core.rmem_max=16777216
      - net.core.wmem_max=16777216
```

---

## 📊 監控配置詳解

### Prometheus 指標配置
```yaml
# 在主配置文件中的監控部分
monitoring:
  metrics:
    enabled: true
    endpoint: "/metrics"
    namespace: "fastim"
    
    # 自定義指標配置
    custom_metrics:
      # 連接指標
      connections:
        enabled: true
        labels: ["instance", "channel", "client_type"]
      
      # 消息指標
      messages:
        enabled: true
        labels: ["type", "channel", "status"]
        buckets: [0.01, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0, 10.0]
      
      # 系統指標
      system:
        enabled: true
        collect_interval: 15s
        
    # 指標保留策略
    retention:
      default: 15d
      detailed: 7d
      summary: 30d
```

### 日誌聚合配置
```yaml
logging:
  # ELK Stack 配置
  elasticsearch:
    enabled: false
    hosts:
      - "http://elasticsearch:9200"
    index_pattern: "fastim-logs-{yyyy.MM.dd}"
    
  # Fluentd 配置
  fluentd:
    enabled: false
    host: "fluentd.logging"
    port: 24224
    tag: "fastim.application"
    
  # 結構化日誌配置
  structured:
    enabled: true
    fields:
      timestamp: "@timestamp"
      level: "level"
      message: "message"
      service: "service"
      trace_id: "trace_id"
      span_id: "span_id"
```

---

## 🛡️ 安全配置詳解

### 高級認證配置
```yaml
security:
  # 多因素認證
  mfa:
    enabled: false                    # 啟用多因素認證
    methods:
      - "totp"                        # TOTP (Google Authenticator)
      - "sms"                         # 短信驗證
    grace_period: 24h                 # 信任設備期限
    
  # 會話管理
  session:
    max_sessions_per_user: 5          # 每用戶最大會話數
    concurrent_login_policy: "allow"  # 併發登入策略: allow, deny, replace
    idle_timeout: 30m                 # 空閒超時
    absolute_timeout: 8h              # 絕對超時
    
  # 審計日誌
  audit:
    enabled: true                     # 啟用審計日誌
    events:
      - "login"
      - "logout"
      - "password_change"
      - "admin_action"
      - "file_upload"
    retention_days: 90                # 審計日誌保留天數
    
  # IP 白名單/黑名單
  ip_filter:
    enabled: false                    # 啟用 IP 過濾
    whitelist:                        # IP 白名單
      - "192.168.1.0/24"
      - "10.0.0.0/8"
    blacklist:                        # IP 黑名單
      - "192.168.100.0/24"
    
  # CSRF 保護
  csrf:
    enabled: true                     # 啟用 CSRF 保護
    token_length: 32                  # Token 長度
    cookie_name: "fastim_csrf_token"  # Cookie 名稱
    header_name: "X-CSRF-Token"       # 頭部名稱
```

---

## 📱 客戶端配置

### Flutter 客戶端配置
```dart
// lib/config/app_config.dart
class AppConfig {
  // 環境配置
  static const String environment = String.fromEnvironment(
    'ENVIRONMENT',
    defaultValue: 'development',
  );
  
  // API 配置
  static const String apiBaseUrl = String.fromEnvironment(
    'API_BASE_URL',
    defaultValue: 'http://localhost:8080/api/v1',
  );
  
  static const String wsBaseUrl = String.fromEnvironment(
    'WS_BASE_URL',
    defaultValue: 'ws://localhost:8080/api/v1/ws',
  );
  
  // 應用配置
  static const int maxReconnectAttempts = 5;
  static const Duration reconnectDelay = Duration(seconds: 5);
  static const Duration heartbeatInterval = Duration(seconds: 30);
  static const int maxMessageLength = 4096;
  
  // 緩存配置
  static const int messageCacheSize = 1000;
  static const Duration cacheExpiry = Duration(hours: 24);
  
  // 文件上傳配置
  static const int maxFileSize = 50 * 1024 * 1024; // 50MB
  static const List<String> allowedImageTypes = [
    'image/jpeg',
    'image/png',
    'image/gif',
    'image/webp',
  ];
  
  // 根據環境獲取配置
  static String get effectiveApiBaseUrl {
    switch (environment) {
      case 'production':
        return 'https://api.fastim.com/api/v1';
      case 'staging':
        return 'https://staging-api.fastim.com/api/v1';
      default:
        return apiBaseUrl;
    }
  }
  
  static String get effectiveWsBaseUrl {
    switch (environment) {
      case 'production':
        return 'wss://api.fastim.com/api/v1/ws';
      case 'staging':
        return 'wss://staging-api.fastim.com/api/v1/ws';
      default:
        return wsBaseUrl;
    }
  }
}
```

### Web 客戶端配置
```typescript
// src/config/app.config.ts
export interface AppConfig {
  api: {
    baseUrl: string;
    timeout: number;
    retries: number;
  };
  websocket: {
    url: string;
    reconnectDelay: number;
    maxReconnectAttempts: number;
    heartbeatInterval: number;
  };
  ui: {
    theme: 'light' | 'dark' | 'auto';
    language: string;
    dateFormat: string;
  };
  features: {
    fileUpload: boolean;
    voiceMessages: boolean;
    videoChat: boolean;
  };
}

const config: AppConfig = {
  api: {
    baseUrl: process.env.REACT_APP_API_BASE_URL || 'http://localhost:8080/api/v1',
    timeout: parseInt(process.env.REACT_APP_API_TIMEOUT || '10000'),
    retries: parseInt(process.env.REACT_APP_API_RETRIES || '3'),
  },
  websocket: {
    url: process.env.REACT_APP_WS_URL || 'ws://localhost:8080/api/v1/ws',
    reconnectDelay: parseInt(process.env.REACT_APP_WS_RECONNECT_DELAY || '5000'),
    maxReconnectAttempts: parseInt(process.env.REACT_APP_WS_MAX_RECONNECT || '5'),
    heartbeatInterval: parseInt(process.env.REACT_APP_WS_HEARTBEAT || '30000'),
  },
  ui: {
    theme: (process.env.REACT_APP_THEME as any) || 'auto',
    language: process.env.REACT_APP_LANGUAGE || 'en',
    dateFormat: process.env.REACT_APP_DATE_FORMAT || 'yyyy-MM-dd HH:mm:ss',
  },
  features: {
    fileUpload: process.env.REACT_APP_ENABLE_FILE_UPLOAD === 'true',
    voiceMessages: process.env.REACT_APP_ENABLE_VOICE === 'true',
    videoChat: process.env.REACT_APP_ENABLE_VIDEO === 'true',
  },
};

export default config;
```

---

## 📋 配置檢查清單

### 部署前檢查
- [ ] **安全配置**
  - [ ] JWT 密鑰已更改為生產密鑰
  - [ ] 數據庫認證憑據已設置
  - [ ] HTTPS/TLS 證書已配置
  - [ ] CORS 允許來源已正確設置
  
- [ ] **性能配置**
  - [ ] 連接數限制適合硬件配置
  - [ ] 數據庫連接池大小合理
  - [ ] 緩存策略已啟用
  - [ ] 日誌級別設置為 info 或 warn
  
- [ ] **監控配置**
  - [ ] 健康檢查端點可訪問
  - [ ] 指標收集已啟用
  - [ ] 日誌輸出已配置
  - [ ] 告警規則已設置

### 安全配置檢查
- [ ] **認證安全**
  - [ ] 密碼複雜度策略已啟用
  - [ ] JWT Token 過期時間合理
  - [ ] 刷新 Token 機制已實施
  - [ ] API 限流已啟用
  
- [ ] **數據安全**
  - [ ] 敏感信息已加密存儲
  - [ ] 日誌中敏感信息已過濾
  - [ ] 文件上傳類型已限制
  - [ ] 輸入驗證已實施

### 性能配置檢查
- [ ] **資源配置**
  - [ ] CPU 和記憶體限制已設置
  - [ ] 垃圾回收參數已調優
  - [ ] 網路參數已優化
  - [ ] 文件描述符限制已提高
  
- [ ] **數據庫優化**
  - [ ] 數據庫索引已創建
  - [ ] 連接池配置合理
  - [ ] 查詢超時已設置
  - [ ] 慢查詢監控已啟用

---

## 🔗 相關文檔

- [部署指南](../deployment/deployment-guide.md) - 完整部署流程
- [系統架構](../architecture/system-architecture.md) - 架構設計說明
- [故障排除](../troubleshooting/troubleshooting-guide.md) - 配置問題排查
- [API 文檔](../api/api-documentation.md) - API 接口說明

---

**配置文檔最後更新**: 2024-01-15  
**版本**: v1.2.0-progressive

正確的配置是 FastIM 穩定運行的基礎。請根據實際部署環境調整相應參數，並定期檢查和更新配置。🔧