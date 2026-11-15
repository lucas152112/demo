# Nice Speak

**軟體開發英語口語學習 App**

## 專案概述

幫助軟體開發人員透過情境對話練習提升英語口說能力。

## 功能特色

- 🎯 6 種開發者角色對話
- 📚 情境式學習
- 🎤 語音輸入輸出 (STT/TTS)
- 🤖 AI 發音與語法評估
- 📊 等級系統 (0-10 級)
- 💰 訂閱制收費

## 技術架構

| 項目 | 技術 |
|------|------|
| 前端 | Flutter (iOS/Android/H5) |
| 後端 | Rust + Axum |
| 資料庫 | MySQL 8 + Redis + MongoDB |
| 語音 | STT/TTS API |
| AI 評估 | LLM (GPT-4/Claude) |
| 認證 | JWT |

## 目錄結構

```
Nice_Speak/
├── Document/           # 專案文件
├── frontend/          # Flutter 前端
└── backend/           # Rust 後端
```

## 快速開始

### 前端

```bash
cd frontend
flutter pub get
flutter run
```

### 後端

```bash
cd backend
cargo run
```

## 收費方案

| 等級 | 情境數 | 價格 |
|------|--------|------|
| 免費版 | 10 | NT$0 |
| 評估版 | 10 | NT$39 |
| 入門版 | 180 | NT$100/月 |
| 進階版 | 540 | NT$300/月 |
| 高階版 | 1,080 | NT$1,000/月 |
| 白金版 | 3,600 | NT$3,000/月 |
| 無限版 | 10,800 | NT$10,000/月 |

## 等級系統

- 練習評分：0-100 分
- 等級：0-10 級
- 升級：連續 3 次 或 累計 6 次 平均達標
- 5 級：付費 80%
- 10 級：付費 60%

## 文件

- [架構設計](ARCHITECTURE.md)
- [資料庫設計](DATABASE.md)
- [API 規格](API.md)
- [使用者流程](USER_FLOW.md)
- [收費方案](PRICING.md)
- [等級系統](LEVEL_SYSTEM.md)
- [開發計劃](DEVELOPMENT_PLAN.md)
- [開發日誌](DEVELOPMENT_LOG.md)
- [執行追蹤](DEVELOPMENT_EXECUTION.md)
- [測試計劃](TEST_PLAN.md)

## 授權

MIT License
