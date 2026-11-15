# JustRead

**JustRead** 是一個多租戶線上閱讀平台，採用三層架構 + 微服務設計，支援使用者閱讀書籍、書籤管理、閱讀進度同步，並提供完整管理後台與通知系統。

---

## 快速導覽

| 文件 | 說明 |
|------|------|
| [架構總覽](docs/architecture/overview.md) | 三層架構、微服務拓樸、資料流 |
| [技術規格](docs/architecture/tech-spec.md) | 技術棧、框架版本、設計決策 |
| [API 規格](docs/api/README.md) | 各服務端點總覽 |
| [部署指南](docs/deployment/README.md) | Kubernetes 部署、環境變數、Secret 管理 |
| [開發指南](docs/development/README.md) | 本地開發、build、測試流程 |
| [任務清單](docs/development/TASKS.md) | 開發進度追蹤 |

---

## 系統組成

```
justread/
├── backend/
│   ├── auth/           # 認證服務 (Port 8081)
│   ├── book/           # 書籍服務 (Port 8082)
│   ├── user/           # 用戶服務 (Port 8083)
│   ├── reader/         # 閱讀器服務 (Port 8084)
│   └── notification/   # 通知服務 (Port 8085)
├── front/              # 前台 React 應用 (Port 3000)
├── manage/             # 管理後台 React 應用 (Port 3001)
├── k8s/                # Kubernetes 配置
└── docs/               # 文件目錄
```

---

## 服務存取

| 服務 | 內部 Port | NodePort | 說明 |
|------|-----------|----------|------|
| 前台 | 3000 | 30200 | 使用者閱讀介面 |
| 管理後台 | 3001 | 30201 | 管理員操作介面 |
| Auth | 8081 | 30091 | 認證、註冊、JWT |
| Book | 8082 | 30082 | 書籍 CRUD |
| User | 8083 | 30083 | 用戶 Profile |
| Reader | 8084 | 30084 | 閱讀進度、書籤 |
| Notification | 8085 | 30085 | 通知推送 |

> Node IP: `103.251.113.34`

---

## 開發狀態

當前版本：`v0.2.0-dev`

- ✅ Auth 服務完整實作（註冊、驗證信、登入、JWT RS256、Refresh Token）
- ✅ 管理後台前端（Dashboard、Users、Books、Login）
- ✅ 管理員帳號 JWT 認證（30分鐘自動刷新、1小時時效）
- 🚧 Book / User / Reader / Notification 服務（骨架完成，業務邏輯待實作）
- 📋 前台使用者功能待開發
