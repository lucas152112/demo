# 架構設計 (ARCHITECTURE.md)

## 系統架構

```
┌─────────────────────────────────────────────────────────────────────┐
│                         Nice Speak App                              │
├─────────────────────────────────────────────────────────────────────┤
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐               │
│  │   iOS App   │  │ Android App │  │   Web App    │               │
│  │  (Flutter)  │  │  (Flutter)  │  │ (Nuxt.js 3)  │               │
│  └─────────────┘  └─────────────┘  └─────────────┘               │
└────────────────────────────┬──────────────────────────────────────┘
                             │ HTTPS + WebSocket
                             ▼
┌─────────────────────────────────────────────────────────────────────┐
│                         API Gateway                                  │
│                    (Authentication & Rate Limit)                     │
└────────────────────────────┬──────────────────────────────────────┘
                             │
         ┌───────────────────┼───────────────────┐
         ▼                   ▼                   ▼
┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐
│   Auth Service  │ │  API Service    │ │  WebSocket      │
│   (JWT)         │ │  (REST)         │ │  Service        │
├─────────────────┤ ├─────────────────┤ ├─────────────────┤
│ - Login         │ │ - Scenarios     │ │ - Real-time     │
│ - Register      │ │ - Conversations │ │   dialogue      │
│ - Profile       │ │ - Practice      │ │ - Speech stream │
│ - Subscription  │ │ - Evaluation    │ └─────────────────┘
└─────────────────┘ └─────────────────┘
         │                   │
         ▼                   ▼
┌─────────────────────────────────────────────────────────────────────┐
│                           Backend Services                            │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐            │
│  │ Speech       │  │ Evaluation    │  │ Subscription  │            │
│  │ Service      │  │ Service       │  │ Service       │            │
│  │ (STT/TTS)    │  │ (AI/LLM)      │  │ (Payments)    │            │
│  └──────────────┘  └──────────────┘  └──────────────┘            │
└────────────────────────────┬──────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────────┐
│                           Data Layer                                 │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐            │
│  │    MySQL 8   │  │     Redis     │  │   MongoDB    │            │
│  │  (主資料庫)   │  │  (Cache/Queue)│  │  (日誌/對話)  │            │
│  └──────────────┘  └──────────────┘  └──────────────┘            │
└─────────────────────────────────────────────────────────────────────┘
```

## 資料庫規劃

| 資料庫 | 用途 | 說明 |
|--------|------|------|
| **MySQL 8** | 結構化資料 | 用戶、訂閱、情境、練習記錄 |
| **Redis** | 快取/會話 | Token、Session、Cache |
| **MongoDB** | 非結構化資料 | 對話日誌、錯誤記錄、練習音頻 |

### MySQL 8 表結構

```
Tables:
├── users                    # 用戶
├── subscriptions           # 訂閱
├── payments                # 支付
├── user_levels             # 等級
├── scenarios               # 情境
├── dialogues               # 對話
├── practice_records        # 練習記錄
└── vocabulary              # 單字
```

### MongoDB Collections

```
Collections:
├── conversation_logs       # 對話日誌
├── error_logs             # 錯誤記錄
├── practice_audios        # 練習音頻
└── analytics              # 分析數據
```

### Redis Keys

```
Keys:
├── user:{id}:profile       # 用戶資料快取
├── user:{id}:subscription # 訂閱狀態
├── user:{id}:session      # 會話
├── scenario:{id}           # 情境快取
├── auth:{token}           # Token 黑名單
└── rate_limit:{ip}        # 限流計數
```

---

## 前端架構 (Flutter + Nuxt.js 3)

```
frontend/
├── mobile/                   # Flutter (iOS/Android)
│   ├── lib/
│   │   ├── main.dart              # 入口
│   │   ├── app.dart               # App 配置
│   │   ├── screens/               # 頁面
│   │   ├── components/            # 通用元件
│   │   ├── services/              # API 服務
│   │   ├── models/                # 資料模型
│   │   ├── providers/             # 狀態管理
│   │   └── utils/                 # 工具函數
│   ├── pubspec.yaml
│   ├── android/
│   └── ios/
│
└── web/                       # Nuxt.js 3 (H5)
    ├── src/
    │   ├── app.vue              # 根元件
    │   ├── pages/               # 頁面
    │   ├── components/           # 元件
    │   ├── composables/         # Composition API
    │   ├── server/              # API 代理
    │   ├── utils/               # 工具函數
    │   ├── types/               # TypeScript 類型
    │   ├── stores/              # Pinia 狀態管理
    │   └── middleware/          # 路由中介
    ├── nuxt.config.ts
    ├── package.json
    ├── tailwind.config.js
    └── public/
```

## Flutter Tech Stack

| 項目 | 技術 |
|------|------|
| Framework | Flutter 3.x |
| 狀態管理 | Provider / Riverpod |
| HTTP | Dio |
| WebSocket | web_socket_channel |
| 語音 | flutter_sound / record |

## Nuxt.js 3 Tech Stack

| 項目 | 技術 |
|------|------|
| Framework | Nuxt.js 3 |
| 語言 | TypeScript |
| UI 框架 | TailwindCSS |
| 狀態管理 | Pinia |
| HTTP | $fetch / ofetch |

## 後端架構 (Rust + Axum)

```
backend/
├── src/
│   ├── main.rs                 # 入口
│   ├── lib.rs                  # 庫入口
│   ├── config.rs               # 配置
│   ├── auth/                   # 認證服務
│   ├── conversation/          # 對話服務
│   ├── speech/                 # 語音服務
│   ├── evaluation/             # 評估服務
│   ├── subscription/           # 訂閱服務
│   ├── user/                   # 用戶服務
│   ├── database/               # 資料庫
│   │   ├── mod.rs
│   │   ├── mysql.rs
│   │   ├── redis.rs
│   │   └── mongodb.rs
│   └── middleware/             # 中介軟體
├── Cargo.toml
└── migrations/
```

## 後端 Tech Stack

| 項目 | 技術 |
|------|------|
| Runtime | Rust 1.70+ |
| Web Framework | Axum |
| MySQL ORM | SQLx |
| MongoDB Driver | mongodb |
| Cache | Redis (redis-rs) |
| 認證 | JWT |

---

## 部署架構 (跳板機 + K8s)

```
┌─────────────────────────────────────────────────────────────────────┐
│                           外部網路                                   │
│                                                                      │
│  ┌─────────────┐                                                   │
│  │   用戶終端   │                                                   │
│  │ (App/Web)   │                                                   │
│  └──────┬──────┘                                                   │
│         │                                                          │
└─────────┼──────────────────────────────────────────────────────────┘
          │ HTTPS / WSS
          ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      Load Balancer                                  │
│                  (Cloud Load Balancer)                             │
└────────────────────────────┬──────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    K8s Ingress / NodePort                           │
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  Nice-Speak-Fontend (Flutter Web)                         │   │
│  │  Service: NodePort 30080                                   │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  Nice-Speak-Web (Nuxt.js 3)                               │   │
│  │  Service: NodePort 30000                                   │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  Nice-Speak-Backend (Rust Axum)                           │   │
│  │  Service: NodePort 31000                                   │   │
│  └─────────────────────────────────────────────────────────────┘   │
└────────────────────────────┬──────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────────┐
│                         跳板機 (Bastion Host)                        │
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  SSH (Port 22)                                            │   │
│  │  + MySQL (Port 3306)                                      │   │
│  │  + Redis (Port 6379)                                      │   │
│  │  + MongoDB (Port 27017)                                   │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  K8s Cluster                                              │   │
│  │  ├── MySQL Pod (NodePort 32000)                           │   │
│  │  ├── Redis Pod (NodePort 32001)                           │   │
│  │  └── MongoDB Pod (NodePort 32002)                         │   │
│  └─────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
```

### NodePort 對照表

| Service | Port | NodePort | 說明 |
|---------|------|----------|------|
| Frontend (Flutter Web) | 80 | 30080 | Web |
| Web (Nuxt.js) | 3000 | 30000 | Web |
| Backend (Rust) | 3000 | 31000 | API |
| MySQL | 3306 | 32000 | 資料庫 |
| Redis | 6379 | 32001 | 快取 |
| MongoDB | 27017 | 32002 | 日誌 |

---

## 安全性

- JWT 認證 + Refresh Token
- HTTPS/TLS 加密 (Let's Encrypt)
- Rate Limiting
- 跳板機隔離資料庫
- K8s 網路隔離

---

## 外部服務

| 項目 | 服務商 |
|------|--------|
| STT | Google Speech / Azure Speech |
| TTS | Azure Speech / ElevenLabs |
| AI 評估 | OpenAI GPT-4 / Anthropic Claude |
| 支付 | Stripe / 綠界科技 |
| K8s | 自建 / GKE / EKS / AKS |
