# Just Work 前端開發原則

本文檔定義 Just Work 前端開發的原則、規範和流程。

---

**文件版本**: 0.1.0.0.0
**最後更新**: 2026-02-14
**維護團隊**: Frontend Team

---

## 1. 版本號規範

### 1.1 版本格式

```
主版號.大版本號.小版本號.測試版本號.維護版本號
   |         |        |         |          |
   1         0        0         0          0
```

### 1.2 版本意義

| 位置 | 名稱 | 說明 |
|------|------|------|
| 1 | 主版號 | 正式版本重大變更 |
| 0 | 大版本號 | 開發大版本 |
| 0 | 小版本號 | 功能更新 |
| 0 | 測試版本號 | 測試版本 |
| 0 | 維護版本號 | 維護更新 |

### 1.3 版本範例

| 版本號 | 意義 |
|--------|------|
| `1.0.0.0.0` | 正式版 v1.0 |
| `0.1.0.0.0` | 開發版 v0.1 |
| `0.2.1.0.0` | 開發版 v0.2.1 |
| `0.2.0.1.0` | 測試版 v0.2.0-t1 |
| `0.2.0.0.1` | 維護版 v0.2.0-m1 |

---

## 2. Git 分支策略

### 2.1 分支結構

```
frontend/
├── main (主分支)
│   └── releases/ (發布分支)
│
├── develop (開發分支)
│   ├── feature/login-page (登入頁功能)
│   ├── feature/dashboard (儀表板功能)
│   └── fix/ui-bug (UI 修復)
│
├── test (測試分支)
│   ├── feature/login-page
│   └── feature/dashboard
│
├── pp (預發布分支)
│
└── prod (正式分支)
    └── hotfix/critical-fix
```

### 2.2 分支命名規範

| 分支類型 | 命名範例 | 說明 |
|----------|----------|------|
| 功能分支 | `feature/login-page` | 新功能 |
| 修復分支 | `fix/ui-bug` | UI 修復 |
| 熱修復 | `hotfix/login-security` | 緊急修復 |
| 發布分支 | `release/v1.0.0` | 版本發布 |

### 2.3 合併規則

| 來源 | 目標 | 說明 |
|------|------|------|
| feature/xxx | develop | 新功能開發 |
| develop | test | 功能完成 |
| test | pp | 測試通過 |
| pp | prod | 預發布驗證 |
| hotfix/xxx | prod | 緊急修復 |
| prod | develop | 同步修復 |

---

## 3. 開發原則

### 3.1 永不崩潰 (Zero Crash)

| 原則 | 說明 |
|------|------|
| 錯誤邊界 | 使用 Error Boundary 捕捉錯誤 |
| 優雅降級 | 降級到基本功能 |
| 錯誤提示 | 用戶友好的錯誤訊息 |

### 3.2 效能要求

| 指標 | 目標 |
|------|------|
| 頁面載入 | < 2s |
| API 響應 | < 500ms |
| 互動延遲 | < 100ms |
| 首屏時間 | < 1s |

---

## 4. 程式碼規範

### 4.1 Nuxt.js 結構

```
frontend/
├── pages/
│   ├── index.vue
│   ├── login.vue
│   ├── register.vue
│   └── dashboard/
│       └── index.vue
│
├── components/
│   ├── common/
│   │   ├── Button.vue
│   │   └── Input.vue
│   └── business/
│       ├── UserCard.vue
│       └── WorkItem.vue
│
├── layouts/
│   ├── default.vue
│   └── auth.vue
│
├── composables/
│   ├── useAuth.ts
│   ├── useApi.ts
│   └── useForm.ts
│
├── middleware/
│   ├── auth.ts
│   └── guest.ts
│
├── stores/
│   ├── auth.ts
│   └── user.ts
│
├── assets/
│   ├── css/
│   └── images/
│
└── plugins/
    └── antd.ts
```

### 4.2 元件命名

```vue
<!-- ✅ 好的範例 -->
<template>
  <div>
    <BaseButton />
    <UserProfileCard />
    <WorkList />
  </div>
</template>

<!-- ❌ 不好的範例 -->
<template>
  <div>
    <button />           <!-- 缺少前綴 -->
    <user-card />        <!-- 駝峰式 -->
    <worklist />         <!-- 縮寫 -->
  </div>
</template>
```

### 4.3 Composables 命名

```typescript
// ✅ 好的範例
useAuth.ts      // 認證相關
useUser.ts      // 用戶相關
useWork.ts      // 工作相關
useNotification.ts  // 通知相關

// ❌ 不好的範例
auth.ts         // 缺少 use 前綴
userData.ts     // 缺少 use 前綴
```

---

## 5. 測試規範

### 5.1 測試覆蓋率目標

| 測試類型 | 覆蓋率目標 |
|----------|-----------|
| 單元測試 | ≥ 70% |
| 元件測試 | ≥ 60% |
| E2E 測試 | 核心流程 100% |

### 5.2 測試範例

```typescript
// components/__tests__/Button.spec.ts
import { describe, it, expect } from 'vitest'
import { mount } from '@vue/test-utils'
import Button from '../Button.vue'

describe('Button', () => {
  it('renders correctly', () => {
    const wrapper = mount(Button, {
      props: {
        type: 'primary'
      }
    })
    expect(wrapper.classes()).toContain('ant-btn-primary')
  })
  
  it('emits click event', async () => {
    const wrapper = mount(Button)
    await wrapper.trigger('click')
    expect(wrapper.emitted('click')).toBeTruthy()
  })
})
```

---

## 6. 環境配置

### 6.1 環境變數

| 環境 | .env 檔案 | API Base URL |
|------|-----------|-------------|
| dev | `.env.dev` | http://localhost:8080/api |
| test | `.env.test` | https://api.test.justwork.cc/api |
| pp | `.env.pp` | https://api.pp.justwork.cc/api |
| prod | `.env.prod` | https://api.justwork.cc/api |

### 6.2 Nuxt Runtime Config

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  runtimeConfig: {
    public: {
      apiBase: process.env.API_BASE || '/api',
      appVersion: process.env.APP_VERSION || '0.1.0.0.0'
    }
  }
})
```

---

## 7. 發布規範

### 7.1 發布流程

```bash
# 1. 建立發布分支
git checkout -b release/v0.2.0

# 2. 更新版本號
npm version patch  # 0.2.0 -> 0.2.1

# 3. 構建
npm run build

# 4. 測試通過後合併
git checkout prod
git merge release/v0.2.0
git push

# 5. 刪除發布分支
git branch -d release/v0.2.0
```

### 7.2 版本發布紀錄

```markdown
# Changelog

## [0.2.0] - 2024-01-15

### Added
- Login page with form validation
- Register page with form validation
- Dashboard layout with sidebar

### Changed
- Updated Ant Design Vue to v4.0
- Improved responsive design

### Fixed
- Fixed mobile layout issue
- Fixed form validation bug

### Security
- Updated JWT handling
```

---

## 8. 文件版本管理

### 8.1 文件版本格式

每個文件開頭包含版本資訊：

```markdown
---
title: 前端開發原則
version: 0.1.0.0.0
last_updated: 2024-01-15
author: Frontend Team
---
```

### 8.2 版本變更紀錄

| 版本 | 日期 | 變更 |
|------|------|------|
| 0.1.0.0.0 | 2024-01-15 | 初始版本 |

---

## 9. 技術棧版本

| 技術 | 版本 |
|------|------|
| Node.js | 20.x LTS |
| Nuxt.js | 3.8+ |
| Vue.js | 3.3+ |
| TypeScript | 5.3+ |
| Ant Design Vue | 4.x |
| Pinia | 2.x |
| Vitest | 1.x |

---

## 參考文件

- [後端技術規格](../backend/TECHNICAL_SPECIFICATION.md)
- [後端開發原則](../backend/DEVELOPMENT.md)
- [管理後台開發原則](../admin/DEVELOPMENT.md)

---

**文件版本**: 0.1.0.0.0
**最後更新**: 2026-02-14
**維護團隊**: Frontend Team
