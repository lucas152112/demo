# Just Work

工作管理系統

## 專案結構

```
Just_work/
├── frontend/          # Nuxt.js 前端
│   ├── pages/        # 頁面元件
│   ├── layouts/      # 佈局元件
│   ├── components/   # 共用元件
│   ├── composables/  # 組合式函數
│   ├── assets/       # 靜態資源
│   └── middleware/   # 中間件
│
├── backend/          # Rust 後端 (Axum)
│   └── src/
│       ├── routes/   # API 路由
│       ├── models/   # 資料模型
│       ├── services/ # 業務邏輯
│       └── middleware/ # 中間件
│
├── admin/            # Rust 管理後台 (Axum)
│   └── src/
│       ├── routes/
│       ├── models/
│       ├── services/
│       └── middleware/
│
└── documents/       # 專案文件
    ├── frontend/
    ├── backend/
    └── admin/
```

## 技術棧

### 前端
- **框架**: Nuxt.js 3
- **UI 庫**: Ant Design Vue
- **狀態管理**: Pinia
- **HTTP**: Axios

### 後端
- **語言**: Rust
- **框架**: Axum
- **資料庫**: PostgreSQL + Redis
- **ORM**: SeaORM

## 快速開始

### 前端

```bash
cd frontend
npm install
npm run dev
```

### 後端

```bash
cd backend
cargo run
```

## 環境變數

請參考 `.env.example` 檔案。

## 授權

MIT
