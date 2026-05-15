# 資料庫設計 (DATABASE.md)

## 資料庫概述

- **MySQL 8**: 結構化資料 (用戶、訂閱、情境、練習記錄)
- **Redis**: 快取與會話管理
- **MongoDB**: 非結構化資料 (對話日誌、錯誤記錄)

---

## ER 圖

```
MySQL 8 (結構化資料)
┌─────────────┐       ┌─────────────┐       ┌─────────────┐
│   users     │───────│ subscriptions│───────│ payments    │
└─────────────┘       └─────────────┘       └─────────────┘
      │                     │
      │                     │
      ▼                     ▼
┌─────────────┐       ┌─────────────┐
│ user_levels │       │  scenarios   │
└─────────────┘       └─────────────┘
                            │
                            │
      ┌─────────────────────┼─────────────────────┐
      ▼                     ▼                     ▼
MongoDB                  MySQL 8                  MongoDB
┌─────────────┐       ┌─────────────┐       ┌─────────────┐
│ conversation_logs│      │ practice_records│    │ error_logs  │
└─────────────┘       └─────────────┘       └─────────────┘
```

---

## MySQL 8 資料表設計

### 1. users (用戶)

```sql
CREATE TABLE users (
    id CHAR(36) PRIMARY KEY DEFAULT (UUID()),
    email VARCHAR(255) NOT NULL UNIQUE,
    password_hash VARCHAR(255) NOT NULL,
    name VARCHAR(100),
    avatar_url VARCHAR(500),
    free_trial_used TINYINT(1) DEFAULT 0,
    registered_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    last_login_at DATETIME,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    INDEX idx_users_email (email)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

### 2. subscriptions (訂閱)

```sql
CREATE TABLE subscriptions (
    id CHAR(36) PRIMARY KEY DEFAULT (UUID()),
    user_id CHAR(36) NOT NULL,
    tier VARCHAR(50) NOT NULL,
    status VARCHAR(20) NOT NULL,
    started_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    expires_at DATETIME,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    INDEX idx_subscriptions_user_id (user_id),
    INDEX idx_subscriptions_status (status),
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

### 3. payments (支付記錄)

```sql
CREATE TABLE payments (
    id CHAR(36) PRIMARY KEY DEFAULT (UUID()),
    user_id CHAR(36) NOT NULL,
    subscription_id CHAR(36),
    amount DECIMAL(10,2) NOT NULL,
    currency VARCHAR(3) DEFAULT 'TWD',
    payment_method VARCHAR(50),
    transaction_id VARCHAR(255),
    status VARCHAR(20) NOT NULL,
    paid_at DATETIME,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_payments_user_id (user_id),
    INDEX idx_payments_status (status),
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
    FOREIGN KEY (subscription_id) REFERENCES subscriptions(id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

### 4. user_levels (用戶等級)

```sql
CREATE TABLE user_levels (
    id CHAR(36) PRIMARY KEY DEFAULT (UUID()),
    user_id CHAR(36) NOT NULL UNIQUE,
    current_level INT DEFAULT 0,
    total_score INT DEFAULT 0,
    consecutive_wins INT DEFAULT 0,
    cumulative_practices INT DEFAULT 0,
    last_practice_at DATETIME,
    level_5_unlocked TINYINT(1) DEFAULT 0,
    level_10_unlocked TINYINT(1) DEFAULT 0,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

### 5. scenarios (情境)

```sql
CREATE TABLE scenarios (
    id CHAR(36) PRIMARY KEY DEFAULT (UUID()),
    code VARCHAR(50) UNIQUE NOT NULL,
    name VARCHAR(100) NOT NULL,
    description TEXT,
    role_1 VARCHAR(50) NOT NULL,
    role_2 VARCHAR(50),
    category VARCHAR(50) NOT NULL,
    difficulty INT NOT NULL CHECK (difficulty BETWEEN 1 AND 5),
    dialogue_count INT DEFAULT 5,
    tier_required VARCHAR(50) NOT NULL DEFAULT 'free',
    is_active TINYINT(1) DEFAULT 1,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    INDEX idx_scenarios_tier (tier_required),
    INDEX idx_scenarios_category (category),
    INDEX idx_scenarios_difficulty (difficulty)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

### 6. dialogues (對話內容)

```sql
CREATE TABLE dialogues (
    id CHAR(36) PRIMARY KEY DEFAULT (UUID()),
    scenario_id CHAR(36) NOT NULL,
    sequence_number INT NOT NULL,
    speaker_role VARCHAR(50) NOT NULL,
    content TEXT NOT NULL,
    audio_url VARCHAR(500),
    vocabulary JSON,
    evaluation_points JSON,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_dialogues_scenario_id (scenario_id),
    FOREIGN KEY (scenario_id) REFERENCES scenarios(id) ON DELETE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

### 7. practice_records (練習記錄)

```sql
CREATE TABLE practice_records (
    id CHAR(36) PRIMARY KEY DEFAULT (UUID()),
    user_id CHAR(36) NOT NULL,
    scenario_id CHAR(36) NOT NULL,
    started_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    completed_at DATETIME,
    total_score INT,
    pronunciation_score INT,
    grammar_score INT,
    vocabulary_score INT,
    fluency_score INT,
    transcript TEXT,
    feedback TEXT,
    status VARCHAR(20) DEFAULT 'in_progress',
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_practice_records_user_id (user_id),
    INDEX idx_practice_records_scenario_id (scenario_id),
    INDEX idx_practice_records_status (status),
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
    FOREIGN KEY (scenario_id) REFERENCES scenarios(id) ON DELETE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

### 8. vocabulary (單字庫)

```sql
CREATE TABLE vocabulary (
    id CHAR(36) PRIMARY KEY DEFAULT (UUID()),
    word VARCHAR(100) UNIQUE NOT NULL,
    phonetic VARCHAR(50),
    definition TEXT,
    example_sentence TEXT,
    audio_url VARCHAR(500),
    category VARCHAR(50),
    difficulty INT CHECK (difficulty BETWEEN 1 AND 5),
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_vocabulary_word (word),
    INDEX idx_vocabulary_category (category)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

### 9. user_vocabulary (用戶單字本)

```sql
CREATE TABLE user_vocabulary (
    id CHAR(36) PRIMARY KEY DEFAULT (UUID()),
    user_id CHAR(36) NOT NULL,
    vocabulary_id CHAR(36) NOT NULL,
    mastery_level INT DEFAULT 0,
    last_reviewed_at DATETIME,
    review_count INT DEFAULT 0,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    UNIQUE KEY uk_user_vocab (user_id, vocabulary_id),
    INDEX idx_user_vocab_user_id (user_id),
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
    FOREIGN KEY (vocabulary_id) REFERENCES vocabulary(id) ON DELETE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

---

## MongoDB Collections

### 1. conversation_logs (對話日誌)

```javascript
{
  _id: ObjectId,
  user_id: String,
  practice_id: String,
  dialogue_sequence: Number,
  speaker_role: String,
  user_audio_url: String,
  transcript: String,
  evaluation: {
    pronunciation: Number,
    grammar: Number,
    vocabulary: Number,
    fluency: Number,
    total: Number,
    feedback: String
  },
  created_at: Date
}
```

### 2. error_logs (錯誤記錄)

```javascript
{
  _id: ObjectId,
  user_id: String,
  practice_record_id: String,
  error_type: String, // pronunciation, grammar, vocabulary, fluency
  original_text: String,
  corrected_text: String,
  suggested_text: String,
  frequency: Number,
  mastered: Boolean,
  created_at: Date,
  updated_at: Date
}
```

### 3. practice_audios (練習音頻)

```javascript
{
  _id: ObjectId,
  user_id: String,
  practice_id: String,
  dialogue_sequence: Number,
  audio_url: String,
  duration_ms: Number,
  created_at: Date
}
```

### 4. analytics (分析數據)

```javascript
{
  _id: ObjectId,
  user_id: String,
  date: Date,
  total_practices: Number,
  average_score: Number,
  practice_time_minutes: Number,
  level_progress: Number,
  category_breakdown: {
    requirement: Number,
    development: Number,
    testing: Number,
    deployment: Number,
    communication: Number
  }
}
```

---

## Redis 快取設計

### Keys

| Key Pattern | 類型 | 說明 |
|-------------|------|------|
| `user:{id}:profile` | Hash | 用戶資料快取 |
| `user:{id}:subscription` | String | 用戶訂閱狀態 |
| `scenario:{id}` | Hash | 情境資料快取 |
| `scenario:list:{tier}:{page}` | List | 情境列表 |
| `user:{id}:level` | Hash | 用戶等級資訊 |
| `practice:{id}` | Hash | 練習記錄 |
| `auth:{token}` | String | JWT Token 黑名單 |

### TTL 設定

| Key Pattern | TTL |
|-------------|-----|
| user:{id}:profile | 1 小時 |
| user:{id}:subscription | 30 分鐘 |
| scenario:{id} | 24 小時 |
| auth:{token} | Token 過期時間 |

---

## 資料庫連線設定

### MySQL 8

```toml
[database.mysql]
host = "localhost"
port = 3306
name = "nice_speak"
username = "root"
password = "password"
max_connections = 20
min_connections = 5
```

### Redis

```toml
[database.redis]
host = "localhost"
port = 6379
password = ""
db = 0
pool_size = 10
```

### MongoDB

```toml
[mongodb]
uri = "mongodb://localhost:27017"
database = "nice_speak"
max_pool_size = 10
```
