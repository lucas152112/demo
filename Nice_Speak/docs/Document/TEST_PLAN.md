# 測試計劃 (TEST_PLAN.md)

## 文件資訊

| 項目 | 內容 |
|------|------|
| 版本 | v1.0.0 |
| 建立日期 | 2024-XX-XX |
| 狀態 | 待審核 |

---

## 1. 測試策略總覽

### 1.1 測試金字塔

```
                    ┌─────────┐
                   /    AI     \         10% - E2E Tests
                  /   Testing   \
                 /───────────────\
                /   Integration   \     30% - Integration Tests
               /      Tests        \
              /─────────────────────\
             /        Unit           \   60% - Unit Tests
            /        Tests            \
           /───────────────────────────\
          /        Frontend             \ 50% - Frontend Tests
         /         Tests                 \
        /─────────────────────────────────\
       /           Mobile                  \ 50% - Mobile Tests
      /            Tests                    \
     └───────────────────────────────────────┘
```

### 1.2 測試覆蓋率目標

| 測試類型 | 最低覆蓋率 | 目標覆蓋率 |
|----------|------------|------------|
| **單元測試** | 80% | 90% |
| **整合測試** | 70% | 85% |
| **E2E 測試** | 6 大流程 | 100% |
| **效能測試** | 基準測試 | - |
| **安全測試** | OWASP Top 10 | - |

---

## 2. 測試環境

### 2.1 環境對照

| 環境 | 用途 | 資料庫 |
|------|------|--------|
| **dev** | 開發測試 | nice_speak_dev |
| **test** | 整合測試 | nice_speak_test |
| **pp** | 上線前測試 | nice_speak_pp |
| **prod** | 正式環境 | nice_speak_prod |

### 2.2 跳板機測試環境

| 服務 | Host | Port | 用途 |
|------|------|------|------|
| MySQL | 103.251.113.34 | 31001 | dev/test/pp/prod |
| Redis | 103.251.113.34 | 31002 | dev/test/pp/prod |
| MongoDB | 103.251.113.34 | 31003 | dev/test/pp/prod |

---

## 3. 單元測試計画

### 3.1 後端單元測試 (Rust)

#### 測試框架

```bash
# Cargo.toml
[dev-dependencies]
rstest = "0.17"
proptest = "1.0"
mockall = "0.11"
tokio-test = "0.4"
```

#### 測試項目

| # | 模組 | 測試項目 | 測試數量 | 覆蓋率目標 |
|---|------|----------|----------|------------|
| 3.1.1 | Auth | JWT 產生/驗證/過期 | 15 | 95% |
| 3.1.2 | Auth | 密碼加密/bcrypt | 10 | 95% |
| 3.1.3 | User | 用戶 CRUD | 20 | 90% |
| 3.1.4 | Scenario | 情境列表/篩選 | 15 | 90% |
| 3.1.5 | Scenario | 情境詳情 | 10 | 90% |
| 3.1.6 | Practice | 練習記錄 | 15 | 85% |
| 3.1.7 | Level | 等級計算 | 20 | 95% |
| 3.1.8 | Subscription | 訂閱狀態 | 15 | 90% |
| 3.1.9 | Speech | STT/TTS | 10 | 80% |
| 3.1.10 | Evaluation | AI 評估分數 | 15 | 85% |

#### 測試案例範例

```rust
// auth/jwt.rs
#[cfg(test)]
mod tests {
    use super::*;
    
    #[test]
    fn test_jwt_generate() {
        let claims = Claims {
            sub: "user123".to_string(),
            exp: 86400,
            iat: 0,
        };
        let token = generate_jwt(&claims);
        assert!(!token.is_empty());
    }
    
    #[test]
    fn test_jwt_validate_valid() {
        let claims = Claims {
            sub: "user123".to_string(),
            exp: 86400,
            iat: 0,
        };
        let token = generate_jwt(&claims);
        let result = validate_jwt(&token);
        assert!(result.is_ok());
    }
    
    #[test]
    fn test_jwt_expired() {
        let claims = Claims {
            sub: "user123".to_string(),
            exp: 0, // 已過期
            iat: 0,
        };
        let token = generate_jwt(&claims);
        let result = validate_jwt(&token);
        assert!(result.is_err());
    }
    
    #[test]
    fn test_password_hash() {
        let password = "Test1234!";
        let hash = hash_password(password).unwrap();
        assert_ne!(hash, password);
        assert!(verify_password(password, &hash).unwrap());
    }
}
```

### 3.2 前端單元測試 (Flutter)

#### 測試框架

```yaml
# pubspec.yaml
dev_dependencies:
  flutter_test:
    sdk: flutter
  mockito: ^5.4.0
  bloc_test: ^9.1.0
  provider_test: ^1.1.0
```

#### 測試項目

| # | 模組 | 測試項目 | 測試數量 | 覆蓋率目標 |
|---|------|----------|----------|------------|
| 3.2.1 | Auth | 登入/註冊 | 20 | 90% |
| 3.2.2 | Scenario | 情境列表 | 15 | 90% |
| 3.2.3 | Practice | 對話流程 | 20 | 85% |
| 3.2.4 | Speech | 錄音/播放 | 15 | 80% |
| 3.2.5 | Level | 等級顯示 | 10 | 90% |
| 3.2.6 | Subscription | 訂閱狀態 | 10 | 85% |

### 3.3 Web 前端單元測試 (Nuxt.js 3)

#### 測試框架

```bash
# package.json
{
  "devDependencies": {
    "vitest": "^1.0.0",
    "@vue/test-utils": "^2.4.0",
    "happy-dom": "^12.0.0",
    "msw": "^2.0.0"
  }
}
```

#### 測試項目

| # | 模組 | 測試項目 | 測試數量 | 覆蓋率目標 |
|---|------|----------|----------|------------|
| 3.3.1 | Auth | 登入/註冊 | 20 | 90% |
| 3.3.2 | Composables | useAuth | 15 | 90% |
| 3.3.3 | Composables | useSpeech | 15 | 85% |
| 3.3.4 | Composables | usePractice | 20 | 85% |
| 3.3.5 | Components | UI 元件 | 25 | 90% |

---

## 4. 整合測試計画

### 4.1 API 整合測試

#### 測試範圍

| # | API 群組 | 測試項目 | 測試數量 |
|---|----------|----------|----------|
| 4.1.1 | Auth | 登入 → Token 刷新 → 登出 | 10 |
| 4.1.2 | User | 取得資料 → 更新資料 → 刪除 | 8 |
| 4.1.3 | Scenario | 列表 → 詳情 → 篩選 | 12 |
| 4.1.4 | Practice | 開始 → 提交 → 完成 | 15 |
| 4.1.5 | Evaluation | 評估結果 → 記錄儲存 | 10 |
| 4.1.6 | Level | 等級查詢 → 進度查詢 | 8 |
| 4.1.7 | Subscription | 方案 → 購買 → 狀態 | 12 |
| 4.1.8 | Vocabulary | 單字查詢 → 用戶單字本 | 8 |

#### 整合測試範例

```rust
// tests/integration/auth_test.rs

#[tokio::test]
async fn test_full_auth_flow() {
    // 1. 註冊新用戶
    let app = spawn_app().await;
    let response = app.post_register(json!({
        "email": "test@example.com",
        "password": "Test1234!",
        "name": "Test User"
    })).await;
    assert_eq!(response.status(), 201);
    
    // 2. 登入取得 Token
    let response = app.post_login(json!({
        "email": "test@example.com",
        "password": "Test1234!"
    })).await;
    assert_eq!(response.status(), 200);
    let body: LoginResponse = response.json().await;
    assert!(!body.access_token.is_empty());
    
    // 3. 刷新 Token
    let response = app.post_refresh(json!({
        "refresh_token": body.refresh_token
    })).await;
    assert_eq!(response.status(), 200);
    
    // 4. 登出
    let response = app.post_logout().with_auth(&body.access_token).await;
    assert_eq!(response.status(), 200);
}
```

### 4.2 資料庫整合測試

#### 測試項目

| # | 測試項目 | 說明 |
|---|----------|------|
| 4.2.1 | MySQL 連線測試 | 連線池、查詢、交易 |
| 4.2.2 | Redis 快取測試 | 讀寫、TTL、過期 |
| 4.2.3 | MongoDB 日誌測試 | CRUD、聚合查詢 |
| 4.2.4 | 交易一致性測試 | ACID 特性 |

### 4.3 外部服務整合測試

| # | 服務 | 測試項目 | Mock/Stub |
|---|------|----------|-----------|
| 4.3.1 | STT | 語音轉文字 | ✅ Mock |
| 4.3.2 | TTS | 文字轉語音 | ✅ Mock |
| 4.3.3 | AI 評估 | GPT-4/Claude | ✅ Mock |
| 4.3.4 | 支付 | 綠界/Stripe | ✅ Mock |

---

## 5. E2E 測試計画

### 5.1 六大核心流程

| # | 流程名稱 | 測試步驟 | 狀態 |
|---|----------|----------|------|
| **T1** | 完整註冊流程 | 1. 首頁 → 2. 註冊 → 3. 登入 → 4. 個人設定 | ⏳ |
| **T2** | 情境練習流程 | 1. 選擇情境 → 2. 預習 → 3. 練習 → 4. 評估 → 5. 結果 | ⏳ |
| **T3** | 等級晉升流程 | 1. 多次練習 → 2. 達標 → 3. 升級通知 | ⏳ |
| **T4** | 訂閱購買流程 | 1. 選擇方案 → 2. 支付 → 3. 訂閱成功 | ⏳ |
| **T5** | 錯誤複習流程 | 1. 查看錯誤 → 2. 複習 → 3. 標記掌握 | ⏳ |
| **T6** | 權限控制流程 | 1. 未訂閱 → 2. 升級 → 3. 解鎖 | ⏳ |

### 5.2 E2E 測試詳細案例

#### T1: 完整註冊流程

```gherkin
Feature: User Registration Flow

  Scenario: New user registers and logs in successfully
    Given I am on the home page
    When I click "Sign Up"
    And I enter valid registration details
    And I submit the form
    Then I should see "Registration successful"
    And I should be redirected to login page
    
    When I enter my credentials
    And I click "Login"
    Then I should see "Welcome"
    And I should see my profile page
```

#### T2: 情境練習流程

```gherkin
Feature: Practice Flow

  Scenario: User completes a practice session
    Given I am logged in
    And I have selected a scenario
    
    When I click "Start Practice"
    Then I should see the vocabulary preview
    
    When I click "Ready"
    Then I should see the first dialogue
    
    When I record my response
    Then I should see the AI evaluation
    
    When I complete all dialogues
    Then I should see my total score
    And my practice record should be saved
```

### 5.3 行動端 E2E 測試

#### Flutter Driver / Integration Test

```dart
// test_driver/app_test.dart
void main() {
  group('End-to-End Tests', () {
    late FlutterDriver driver;
    
    setUpAll(() async {
      driver = await FlutterDriver.connect();
    });
    
    tearDownAll(() async {
      await driver.close();
    });
    
    test('Complete practice flow', () async {
      // 登入
      await driver.tap(find.byType('LoginButton'));
      await driver.enterText('test@example.com');
      await driver.enterText('password');
      await driver.tap(find.byType('SubmitButton'));
      
      // 選擇情境
      await driver.tap(find.text('Code Review'));
      await driver.tap(find.text('Start Practice'));
      
      // 錄音
      await driver.tap(find.byType('RecordButton'));
      await Future.delayed(Duration(seconds: 3));
      await driver.tap(find.byType('StopButton'));
      
      // 驗證結果
      await driver.waitFor(find.text('Score'));
    });
  });
}
```

### 5.4 Web E2E 測試

#### Playwright

```typescript
// tests/e2e/practice.spec.ts
import { test, expect } from '@playwright/test';

test('Complete practice flow', async ({ page }) => {
  // 登入
  await page.goto('/login');
  await page.fill('[data-testid=email]', 'test@example.com');
  await page.fill('[data-testid=password]', 'password');
  await page.click('[data-testid=submit]');
  
  // 選擇情境
  await page.click('[data-testid=scenario-card]:has-text("Code Review")');
  await page.click('[data-testid=start-practice]');
  
  // 錄音
  await page.click('[data-testid=record-button]');
  await page.waitForTimeout(3000);
  await page.click('[data-testid=stop-button]');
  
  // 驗證結果
  await expect(page.locator('[data-testid=score]')).toBeVisible();
});
```

---

## 6. 效能測試計画

### 6.1 測試項目

| # | 測試項目 | 目標 | 工具 |
|---|----------|------|------|
| 6.1.1 | API 響應時間 | < 200ms | k6 / wrk |
| 6.1.2 | 並發使用者 | 1000+ | k6 |
| 6.1.3 | 資料庫查詢 | < 50ms | EXPLAIN ANALYZE |
| 6.1.4 | WebSocket 延遲 | < 100ms | wscat |
| 6.1.5 | STT/TTS 延遲 | < 2s | 自定義 |
| 6.1.6 | 頁面載入時間 | < 3s | Lighthouse |

### 6.2 負載測試脚本 (k6)

```javascript
// tests/load/scenario.js
import http from 'k6/http';
import ws from 'k6/ws';
import { check, sleep } from 'k6';

export const options = {
  stages: [
    { duration: '1m', target: 100 },   // 預熱
    { duration: '5m', target: 500 },   // 負載
    { duration: '5m', target: 1000 },  // 高峰
    { duration: '2m', target: 0 },     // 降載
  ],
  thresholds: {
    http_req_duration: ['p(95)<200'],
    http_req_failed: ['rate<0.01'],
    ws_session_duration: ['p(95)<5000'],
  },
};

export default function () {
  // API 測試
  const res = http.get('http://api.nicespeak.app/v1/scenarios');
  check(res, {
    'status is 200': (r) => r.status === 200,
    'scenarios returned': (r) => JSON.parse(r.body).scenarios.length > 0,
  });
  
  sleep(1);
}
```

---

## 7. 安全測試計画

### 7.1 OWASP Top 10 測試

| # | 風險 | 測試項目 | 狀態 |
|---|------|----------|------|
| 7.1.1 | Injection | SQL Injection, XSS | ⏳ |
| 7.1.2 | Auth | JWT 弱點, Session 固定 | ⏳ |
| 7.1.3 | Sensitive Data | 密碼儲存, 傳輸加密 | ⏳ |
| 7.1.4 | XXE | XML 外部實體 | ⏳ |
| 7.1.5 | Broken Access | 未授權訪問 | ⏳ |
| 7.1.6 | Security Misconfig | CORS, Headers | ⏳ |
| 7.1.7 | IDOR | 資源 ID 操縱 | ⏳ |
| 7.1.8 | Rate Limiting | DoS 防護 | ⏳ |

### 7.2 滲透測試檢查清單

| # | 檢查項目 | 說明 |
|---|----------|------|
| 7.2.1 | SQL Injection | 測試所有輸入點 |
| 7.2.2 | XSS | 測試所有輸出點 |
| 7.2.3 | CSRF | Token 驗證 |
| 7.2.4 | JWT | 過期、簽名 |
| 7.2.5 | File Upload | 惡意檔案 |
| 7.2.6 | API Rate Limit | 暴力請求 |

---

## 8. 測試資料管理

### 8.1 測試資料產生

```sql
-- test_data.sql

-- 測試用戶
INSERT INTO users (id, email, password_hash, name) VALUES
('user-001', 'test1@nicespeak.app', '$2b$10$...', 'Test User 1'),
('user-002', 'test2@nicespeak.app', '$2b$10$...', 'Test User 2');

-- 測試情境
INSERT INTO scenarios (id, code, name, role_1, category, difficulty) VALUES
('scenario-001', 'DEV_CR_01', 'Code Review', 'Developer', 'development', 3),
('scenario-002', 'PM_REPORT_01', 'Progress Report', 'PM', 'communication', 2);
```

### 8.2 測試資料清理

```bash
# 測試前清理
DELETE FROM practice_records;
DELETE FROM error_logs;
DELETE FROM user_levels;
DELETE FROM subscriptions;
DELETE FROM users;
```

---

## 9. 自動化 CI/CD 測試

### 9.1 GitHub Actions Workflow

```yaml
# .github/workflows/test.yml
name: Tests

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    
    services:
      mysql:
        image: mysql:8.0
        ports: 3306:3306
        env:
          MYSQL_ROOT_PASSWORD: test
          MYSQL_DATABASE: nice_speak_test
      redis:
        image: redis:7
        ports: 6379:6379
      mongodb:
        image: mongo:6
        ports: 27017:27017
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Rust
        uses: actions-rs/toolchain@v1
        with:
          toolchain: stable
      
      - name: Run Rust Tests
        run: cargo test --all
      
      - name: Run Flutter Tests
        run: |
          cd frontend/mobile
          flutter test --coverage
      
      - name: Run Nuxt Tests
        run: |
          cd frontend/web
          npm install
          npm run test
      
      - name: Upload Coverage
        uses: codecov/codecov-action@v3
        with:
          files: ./coverage/lcov.info
```

### 9.2 測試閘門

| 階段 | 通過條件 |
|------|----------|
| Commit | 所有單元測試通過 (80% 覆蓋率) |
| Pull Request | 所有測試 + 整合測試 |
| Merge | 所有測試 + 安全掃描 |
| Release | 效能測試 + 滲透測試 |

---

## 10. 測試報告

### 10.1 報告格式

| 報告類型 | 頻率 | 內容 |
|----------|------|------|
| 每日測試報告 | 每天 | 測試通過率、失敗案例 |
| 週報 | 每週 | 趨勢分析、改進建議 |
| 發布報告 | 每次發布 | 完整測試摘要 |

### 10.2 報告模板

```markdown
# 測試報告 - [日期]

## 測試摘要
- 總測試數: XXX
- 通過: XXX
- 失敗: XXX
- 通過率: XX%

## 測試覆蓋率
- 單元測試: XX%
- 整合測試: XX%
- E2E 測試: XX%

## 失敗案例
| # | 測試項目 | 原因 | 狀態 |
|---|----------|------|------|
| 1 | XXX | XXX | 待修復 |

## 改進建議
- XXX
```

---

## 11. 風險與對策

| 風險 | 影響 | 機率 | 對策 |
|------|------|------|------|
| 外部服務 Mock 不完整 | 測試覆蓋不足 | 中 | 完善 Mock 服務 |
| 測試環境不穩定 | 測試失敗 | 中 | 容器化測試環境 |
| 測試資料不足 | 邊界測試不足 | 低 | 產生大量測試資料 |
| CI/CD 測試過慢 | 開發效率降低 | 中 | 並行測試執行 |

---

## 12. 審核簽核

| 角色 | 姓名 | 日期 | 簽核 |
|------|------|------|------|
| 測試負責人 | | | ⏳ |
| 技術負責人 | | | ⏳ |
| 產品負責人 | | | ⏳ |

---

**版本**: v1.0.0  
**建立日期**: 2024-XX-XX  
**最後更新**: 2024-XX-XX
