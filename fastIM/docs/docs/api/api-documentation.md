# FastIM API 文檔

## 📋 目錄
1. [API 概覽](#api-概覽)
2. [認證機制](#認證機制)
3. [REST API](#rest-api)
4. [WebSocket API](#websocket-api)
5. [錯誤處理](#錯誤處理)
6. [限流和配額](#限流和配額)
7. [SDK 和客戶端庫](#sdk-和客戶端庫)
8. [API 變更日誌](#api-變更日誌)

---

## 🌐 API 概覽

### 基本信息
- **Base URL**: `https://api.fastim.com/api/v1`
- **協議**: HTTPS only
- **數據格式**: JSON
- **編碼**: UTF-8
- **API 版本**: v1
- **文檔版本**: 1.2.0

### 技術特性
- ⚡ **低延遲**: WebSocket 實時通訊 <50ms
- 🔄 **高並發**: 支持 10,000+ 併發連接
- 📊 **分佈式**: Redis 分佈式消息廣播
- 🔧 **自適應**: 動態心跳頻率優化
- 🛡️ **安全**: JWT 認證 + HTTPS

### 支持的客戶端
- 🌐 **Web**: JavaScript/TypeScript
- 📱 **移動**: iOS (Swift), Android (Kotlin)
- 💻 **桌面**: Flutter (Windows/macOS/Linux)
- 🖥️ **服務器**: Go, Python, Node.js

---

## 🔐 認證機制

### JWT Token 認證

#### 1. 獲取 Token
```http
POST /api/v1/auth/login
Content-Type: application/json

{
  "username": "johndoe",
  "password": "securepassword"
}
```

**響應**:
```json
{
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "username": "johndoe",
  "expires_in": 86400,
  "refresh_token": "refresh_token_here"
}
```

#### 2. 使用 Token
```http
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

#### 3. 刷新 Token
```http
POST /api/v1/auth/refresh
Content-Type: application/json

{
  "refresh_token": "refresh_token_here"
}
```

### API Key 認證（服務器間）
```http
X-API-Key: your_api_key_here
```

### WebSocket 認證
```javascript
const ws = new WebSocket('wss://api.fastim.com/api/v1/ws?token=your_jwt_token&username=johndoe');
```

---

## 🔄 REST API

### 認證接口

#### POST /auth/login
用戶登入

**請求體**:
```json
{
  "username": "johndoe",
  "password": "password123"
}
```

**響應**:
```json
{
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "username": "johndoe",
  "user_id": "uuid-here",
  "expires_in": 86400,
  "refresh_token": "refresh_token_here",
  "permissions": ["read", "write", "admin"]
}
```

#### POST /auth/register
用戶註冊

**請求體**:
```json
{
  "username": "newuser",
  "password": "securepassword",
  "email": "user@example.com",
  "display_name": "New User"
}
```

#### POST /auth/logout
用戶登出

```http
POST /api/v1/auth/logout
Authorization: Bearer your_jwt_token
```

#### POST /auth/refresh
刷新 Token

**請求體**:
```json
{
  "refresh_token": "your_refresh_token"
}
```

### 用戶管理

#### GET /users/profile
獲取用戶資料

**響應**:
```json
{
  "user_id": "uuid-here",
  "username": "johndoe",
  "display_name": "John Doe",
  "email": "john@example.com",
  "avatar_url": "https://cdn.fastim.com/avatars/johndoe.jpg",
  "status": "online",
  "last_seen": "2024-01-15T10:30:00Z",
  "created_at": "2024-01-01T00:00:00Z",
  "settings": {
    "notifications": true,
    "theme": "dark"
  }
}
```

#### PUT /users/profile
更新用戶資料

**請求體**:
```json
{
  "display_name": "John Smith",
  "status": "busy",
  "avatar_url": "https://cdn.fastim.com/avatars/new_avatar.jpg"
}
```

#### GET /users/search
搜索用戶

**查詢參數**:
- `q`: 搜索關鍵詞
- `limit`: 結果數量限制 (默認 20)
- `offset`: 偏移量

```http
GET /api/v1/users/search?q=john&limit=10&offset=0
```

### 頻道管理

#### GET /channels
獲取頻道列表

**響應**:
```json
{
  "channels": [
    {
      "channel_id": "uuid-here",
      "name": "general",
      "description": "General discussion",
      "type": "public",
      "member_count": 42,
      "created_at": "2024-01-01T00:00:00Z",
      "last_message": {
        "message_id": "uuid-here",
        "content": "Hello world!",
        "sender": "johndoe",
        "timestamp": "2024-01-15T10:30:00Z"
      }
    }
  ],
  "total": 10,
  "page": 1
}
```

#### POST /channels
創建頻道

**請求體**:
```json
{
  "name": "new-channel",
  "description": "A new channel for discussion",
  "type": "private",
  "members": ["user1", "user2"]
}
```

#### GET /channels/{channel_id}
獲取頻道詳情

**響應**:
```json
{
  "channel_id": "uuid-here",
  "name": "general",
  "description": "General discussion",
  "type": "public",
  "created_by": "admin",
  "created_at": "2024-01-01T00:00:00Z",
  "members": [
    {
      "user_id": "uuid-here",
      "username": "johndoe",
      "role": "member",
      "joined_at": "2024-01-01T00:00:00Z"
    }
  ],
  "settings": {
    "allow_file_upload": true,
    "message_retention_days": 30
  }
}
```

#### PUT /channels/{channel_id}
更新頻道

**請求體**:
```json
{
  "description": "Updated description",
  "settings": {
    "allow_file_upload": false
  }
}
```

#### DELETE /channels/{channel_id}
刪除頻道

```http
DELETE /api/v1/channels/uuid-here
Authorization: Bearer your_jwt_token
```

### 消息管理

#### GET /channels/{channel_id}/messages
獲取消息歷史

**查詢參數**:
- `limit`: 消息數量 (默認 50, 最大 100)
- `before`: 時間戳，獲取此時間前的消息
- `after`: 時間戳，獲取此時間後的消息

```http
GET /api/v1/channels/uuid-here/messages?limit=20&before=2024-01-15T10:30:00Z
```

**響應**:
```json
{
  "messages": [
    {
      "message_id": "uuid-here",
      "channel_id": "uuid-here",
      "sender": {
        "user_id": "uuid-here",
        "username": "johndoe",
        "display_name": "John Doe"
      },
      "content": "Hello everyone!",
      "type": "text",
      "timestamp": "2024-01-15T10:30:00Z",
      "edited_at": null,
      "attachments": [],
      "reactions": [
        {
          "emoji": "👍",
          "count": 3,
          "users": ["user1", "user2", "user3"]
        }
      ]
    }
  ],
  "has_more": true,
  "next_cursor": "cursor_token_here"
}
```

#### POST /channels/{channel_id}/messages
發送消息（REST 方式）

**請求體**:
```json
{
  "content": "Hello world!",
  "type": "text",
  "reply_to": "message_id_here"
}
```

#### PUT /messages/{message_id}
編輯消息

**請求體**:
```json
{
  "content": "Updated message content"
}
```

#### DELETE /messages/{message_id}
刪除消息

```http
DELETE /api/v1/messages/uuid-here
Authorization: Bearer your_jwt_token
```

### 文件管理

#### POST /files/upload
上傳文件

**請求** (multipart/form-data):
```
file: [binary data]
channel_id: uuid-here
filename: document.pdf
```

**響應**:
```json
{
  "file_id": "uuid-here",
  "filename": "document.pdf",
  "size": 1048576,
  "mime_type": "application/pdf",
  "url": "https://cdn.fastim.com/files/uuid-here/document.pdf",
  "thumbnail_url": "https://cdn.fastim.com/thumbs/uuid-here.jpg",
  "expires_at": "2024-02-15T10:30:00Z"
}
```

#### GET /files/{file_id}
獲取文件信息

```http
GET /api/v1/files/uuid-here
Authorization: Bearer your_jwt_token
```

#### GET /files/{file_id}/download
下載文件

```http
GET /api/v1/files/uuid-here/download
Authorization: Bearer your_jwt_token
```

### 系統監控

#### GET /health
健康檢查

**響應**:
```json
{
  "status": "healthy",
  "time": "2024-01-15T10:30:00Z",
  "mode": "progressive-heartbeat",
  "version": "1.2.0-progressive",
  "database": {
    "mongodb": "connected",
    "redis": "connected"
  },
  "services": {
    "websocket": "running",
    "file_service": "running"
  }
}
```

#### GET /stats
系統統計（需要管理員權限）

**響應**:
```json
{
  "clients": 1234,
  "channels": 56,
  "messages_today": 78912,
  "files_uploaded": 345,
  "version": "1.2.0-progressive",
  "heartbeat_stats": {
    "long_connections": 890,
    "avg_interval_seconds": 45.2,
    "efficiency_distribution": {
      "high": 234,
      "medium": 567,
      "low": 433
    }
  },
  "system": {
    "cpu_percent": 23.4,
    "memory_percent": 56.7,
    "disk_usage": 78.9
  }
}
```

#### GET /metrics
Prometheus 指標

```
# HELP fastim_websocket_connections_total Total WebSocket connections
# TYPE fastim_websocket_connections_total gauge
fastim_websocket_connections_total 1234

# HELP fastim_messages_sent_total Total messages sent
# TYPE fastim_messages_sent_total counter  
fastim_messages_sent_total 78912
```

---

## ⚡ WebSocket API

### 連接建立

#### 連接 URL
```
wss://api.fastim.com/api/v1/ws?token=JWT_TOKEN&username=USERNAME
```

#### 連接參數
- `token`: JWT 認證 token (必需)
- `username`: 用戶名 (必需)
- `client_id`: 客戶端 ID (可選，用於多設備管理)

### 消息格式

#### 基本消息結構
```json
{
  "type": "message_type",
  "data": {
    // 消息數據
  },
  "timestamp": "2024-01-15T10:30:00Z",
  "message_id": "uuid-here"
}
```

### 客戶端發送消息

#### 發送文字消息
```json
{
  "type": "message",
  "data": {
    "channel": "general",
    "content": "Hello everyone!",
    "reply_to": "message_id_here"
  }
}
```

#### 加入頻道
```json
{
  "type": "join_channel",
  "data": {
    "channel": "general"
  }
}
```

#### 離開頻道
```json
{
  "type": "leave_channel",
  "data": {
    "channel": "general"
  }
}
```

#### 用戶狀態更新
```json
{
  "type": "status_update",
  "data": {
    "status": "busy",
    "message": "In a meeting"
  }
}
```

#### 輸入狀態
```json
{
  "type": "typing",
  "data": {
    "channel": "general",
    "typing": true
  }
}
```

#### 心跳檢測
```json
{
  "type": "ping",
  "data": {
    "timestamp": "2024-01-15T10:30:00Z"
  }
}
```

### 服務器發送消息

#### 新消息通知
```json
{
  "type": "message",
  "data": {
    "message_id": "uuid-here",
    "channel": "general",
    "sender": {
      "user_id": "uuid-here",
      "username": "johndoe",
      "display_name": "John Doe"
    },
    "content": "Hello everyone!",
    "timestamp": "2024-01-15T10:30:00Z",
    "attachments": []
  }
}
```

#### 用戶上線/下線
```json
{
  "type": "user_status",
  "data": {
    "user_id": "uuid-here",
    "username": "johndoe", 
    "status": "online",
    "last_seen": "2024-01-15T10:30:00Z"
  }
}
```

#### 頻道更新
```json
{
  "type": "channel_update",
  "data": {
    "channel_id": "uuid-here",
    "name": "general",
    "member_count": 42,
    "update_type": "member_joined",
    "updated_by": "admin"
  }
}
```

#### 輸入狀態通知
```json
{
  "type": "user_typing",
  "data": {
    "channel": "general",
    "user": {
      "username": "johndoe",
      "display_name": "John Doe"
    },
    "typing": true
  }
}
```

#### 系統通知
```json
{
  "type": "system_notification",
  "data": {
    "title": "系統維護通知",
    "message": "系統將於 30 分鐘後進行維護",
    "level": "warning",
    "expires_at": "2024-01-15T11:00:00Z"
  }
}
```

#### 心跳回應
```json
{
  "type": "pong",
  "data": {
    "timestamp": "2024-01-15T10:30:00Z",
    "latency": 45
  }
}
```

#### 錯誤消息
```json
{
  "type": "error",
  "data": {
    "code": "UNAUTHORIZED",
    "message": "Invalid token",
    "details": "JWT token has expired"
  }
}
```

### 連接管理

#### 漸進式心跳機制
- **初始間隔**: 10 秒
- **最大間隔**: 120 秒
- **調整策略**: 根據連接穩定性自動調整
- **長連接檢測**: 自動識別並優化長時間連接

#### 重連機制
```javascript
const ws = new WebSocket(wsUrl);
let reconnectInterval = 1000; // 1 秒
let maxReconnectInterval = 30000; // 30 秒

ws.onclose = function() {
  setTimeout(() => {
    reconnect();
    reconnectInterval = Math.min(reconnectInterval * 1.5, maxReconnectInterval);
  }, reconnectInterval);
};
```

#### 連接狀態事件
```json
{
  "type": "connection_status",
  "data": {
    "status": "connected|reconnected|disconnected",
    "reason": "network_error|server_restart|timeout",
    "retry_after": 5000
  }
}
```

---

## ❌ 錯誤處理

### HTTP 狀態碼

| 狀態碼 | 含義 | 說明 |
|--------|------|------|
| 200 | OK | 請求成功 |
| 201 | Created | 資源創建成功 |
| 400 | Bad Request | 請求參數錯誤 |
| 401 | Unauthorized | 未認證或認證失效 |
| 403 | Forbidden | 權限不足 |
| 404 | Not Found | 資源不存在 |
| 429 | Too Many Requests | 請求過於頻繁 |
| 500 | Internal Server Error | 服務器內部錯誤 |
| 502 | Bad Gateway | 網關錯誤 |
| 503 | Service Unavailable | 服務不可用 |

### 錯誤響應格式

```json
{
  "error": {
    "code": "ERROR_CODE",
    "message": "Human readable error message",
    "details": "Additional error details",
    "timestamp": "2024-01-15T10:30:00Z",
    "request_id": "uuid-here"
  }
}
```

### 常見錯誤碼

#### 認證錯誤
- `INVALID_TOKEN`: JWT token 無效
- `EXPIRED_TOKEN`: JWT token 已過期
- `MISSING_TOKEN`: 缺少認證 token
- `INVALID_CREDENTIALS`: 用戶名或密碼錯誤

#### 權限錯誤
- `INSUFFICIENT_PERMISSIONS`: 權限不足
- `CHANNEL_ACCESS_DENIED`: 頻道訪問被拒絕
- `ADMIN_REQUIRED`: 需要管理員權限

#### 業務錯誤
- `CHANNEL_NOT_FOUND`: 頻道不存在
- `USER_NOT_FOUND`: 用戶不存在
- `MESSAGE_NOT_FOUND`: 消息不存在
- `DUPLICATE_CHANNEL_NAME`: 頻道名稱重複
- `INVALID_FILE_TYPE`: 不支持的文件類型
- `FILE_TOO_LARGE`: 文件過大

#### 系統錯誤
- `DATABASE_ERROR`: 資料庫錯誤
- `REDIS_ERROR`: Redis 連接錯誤
- `FILE_UPLOAD_ERROR`: 文件上傳失敗
- `WEBSOCKET_ERROR`: WebSocket 連接錯誤

### WebSocket 錯誤處理

```json
{
  "type": "error",
  "data": {
    "code": "WEBSOCKET_ERROR_CODE",
    "message": "Error description",
    "should_reconnect": true,
    "retry_after": 5000
  }
}
```

### 錯誤處理最佳實踐

#### 客戶端錯誤處理
```javascript
// REST API 錯誤處理
async function apiCall(url, options) {
  try {
    const response = await fetch(url, options);
    
    if (!response.ok) {
      const error = await response.json();
      throw new APIError(error.error.code, error.error.message, response.status);
    }
    
    return await response.json();
  } catch (error) {
    console.error('API Error:', error);
    throw error;
  }
}

// WebSocket 錯誤處理
ws.onmessage = function(event) {
  const message = JSON.parse(event.data);
  
  if (message.type === 'error') {
    console.error('WebSocket Error:', message.data);
    
    if (message.data.should_reconnect) {
      setTimeout(() => {
        reconnectWebSocket();
      }, message.data.retry_after || 5000);
    }
  }
};
```

---

## 🚦 限流和配額

### 請求限制

#### REST API 限制
- **通用接口**: 100 請求/分鐘/用戶
- **認證接口**: 5 請求/分鐘/IP
- **文件上傳**: 10 次/分鐘/用戶
- **搜索接口**: 20 次/分鐘/用戶

#### WebSocket 限制
- **消息發送**: 60 消息/分鐘/用戶
- **心跳檢測**: 不限制
- **狀態更新**: 10 次/分鐘/用戶

### 配額管理

#### 存儲配額
- **個人文件**: 1GB/用戶
- **頻道文件**: 10GB/頻道
- **消息歷史**: 10,000 條/頻道

#### 連接配額
- **併發連接**: 5 連接/用戶
- **頻道數量**: 50 個/用戶
- **好友數量**: 1000 個/用戶

### 限制響應

```json
{
  "error": {
    "code": "RATE_LIMIT_EXCEEDED",
    "message": "Too many requests",
    "details": "You have exceeded the rate limit of 100 requests per minute",
    "retry_after": 60
  }
}
```

### 響應標頭

```http
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 95
X-RateLimit-Reset: 1642248000
Retry-After: 60
```

---

## 📚 SDK 和客戶端庫

### 官方 SDK

#### JavaScript/TypeScript
```bash
npm install @fastim/js-sdk
```

```javascript
import { FastIMClient } from '@fastim/js-sdk';

const client = new FastIMClient({
  baseUrl: 'https://api.fastim.com',
  token: 'your_jwt_token'
});

// 連接 WebSocket
await client.connect();

// 發送消息
await client.sendMessage('general', 'Hello world!');

// 監聽消息
client.on('message', (message) => {
  console.log('New message:', message);
});
```

#### Python
```bash
pip install fastim-python-sdk
```

```python
from fastim import FastIMClient

client = FastIMClient(
    base_url='https://api.fastim.com',
    token='your_jwt_token'
)

# 連接 WebSocket  
client.connect()

# 發送消息
client.send_message('general', 'Hello world!')

# 監聽消息
@client.on('message')
def on_message(message):
    print('New message:', message)
```

#### Go
```bash
go get github.com/fastim/go-sdk
```

```go
package main

import (
    "github.com/fastim/go-sdk"
)

func main() {
    client := fastim.NewClient(&fastim.Config{
        BaseURL: "https://api.fastim.com",
        Token:   "your_jwt_token",
    })
    
    // 連接 WebSocket
    err := client.Connect()
    if err != nil {
        panic(err)
    }
    
    // 發送消息
    err = client.SendMessage("general", "Hello world!")
    if err != nil {
        panic(err)
    }
    
    // 監聽消息
    client.OnMessage(func(message *fastim.Message) {
        fmt.Printf("New message: %+v\n", message)
    })
}
```

### 社區 SDK

#### Rust
```toml
[dependencies]
fastim-rs = "0.1.0"
```

#### PHP
```bash
composer require fastim/php-sdk
```

#### C#
```bash
dotnet add package FastIM.Client
```

---

## 📝 API 變更日誌

### v1.2.0 (2024-01-15)
**新增**
- 漸進式心跳機制
- 長連接自動優化
- 文件縮略圖生成
- 消息反應功能

**改進**
- WebSocket 性能優化
- 錯誤處理增強
- API 響應時間優化

**修復**
- 修復併發連接問題
- 修復消息順序問題

### v1.1.0 (2024-01-01)
**新增**
- 用戶狀態管理
- 消息編輯/刪除
- 文件上傳功能
- 系統監控接口

**改進**
- 認證機制增強
- 數據庫性能優化

### v1.0.0 (2023-12-01)
**初始發布**
- 基本消息功能
- 用戶認證
- 頻道管理
- WebSocket 實時通訊

---

## 🔗 相關資源

### 官方文檔
- [快速開始指南](../user-guide/user-manual.md)
- [部署文檔](../deployment/deployment-guide.md)
- [系統架構](../architecture/system-architecture.md)

### 開發工具
- **API 測試**: [Postman Collection](https://www.postman.com/fastim)
- **WebSocket 測試**: [在線測試工具](https://websocket.org/echo.html)
- **SDK 文檔**: [開發者中心](https://developers.fastim.com)

### 支持社區
- **GitHub**: https://github.com/fastim/fastim
- **Discord**: https://discord.gg/fastim
- **論壇**: https://forum.fastim.com

---

**API 文檔最後更新**: 2024-01-15
**版本**: v1.2.0-progressive

如有問題或建議，請聯繫 API 支持團隊 📧 api-support@fastim.com