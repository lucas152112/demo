# API 規格 (API.md)

## 基礎資訊

- **Base URL**: `https://api.nicespeak.app/v1`
- **認證**: Bearer Token (JWT)
- **Content-Type**: application/json

## 錯誤格式

```json
{
  "error": {
    "code": "ERROR_CODE",
    "message": "錯誤說明",
    "details": {}
  }
}
```

## 錯誤碼

| Code | HTTP Status | 說明 |
|------|-------------|------|
| UNAUTHORIZED | 401 | 未登入或 Token 過期 |
| FORBIDDEN | 403 | 無權限訪問 |
| NOT_FOUND | 404 | 資源不存在 |
| VALIDATION_ERROR | 422 | 參數驗證錯誤 |
| SUBSCRIPTION_REQUIRED | 403 | 需要訂閱 |
| RATE_LIMIT_EXCEEDED | 429 | 請求次數過多 |
| INTERNAL_ERROR | 500 | 伺服器錯誤 |

---

## API 列表

### 1. 認證 (Auth)

#### 1.1 POST /auth/register
註冊新帳號

**Request:**
```json
{
  "email": "user@example.com",
  "password": "password123",
  "name": "User Name"
}
```

**Response:**
```json
{
  "user": {
    "id": "uuid",
    "email": "user@example.com",
    "name": "User Name",
    "tier": "free",
    "registered_at": "2024-01-01T00:00:00Z"
  },
  "access_token": "jwt_token",
  "refresh_token": "refresh_token"
}
```

#### 1.2 POST /auth/login
登入

**Request:**
```json
{
  "email": "user@example.com",
  "password": "password123"
}
```

**Response:** 同 register

#### 1.3 POST /auth/refresh
刷新 Token

**Request:**
```json
{
  "refresh_token": "refresh_token"
}
```

**Response:**
```json
{
  "access_token": "new_jwt_token"
}
```

#### 1.4 POST /auth/logout
登出

**Response:**
```json
{
  "success": true
}
```

---

### 2. 用戶 (User)

#### 2.1 GET /user/profile
取得用戶資料

**Response:**
```json
{
  "user": {
    "id": "uuid",
    "email": "user@example.com",
    "name": "User Name",
    "avatar_url": "https://...",
    "level": {
      "current": 5,
      "total_score": 250,
      "consecutive_wins": 3,
      "cumulative_practices": 45,
      "level_5_unlocked": true,
      "level_10_unlocked": false
    },
    "subscription": {
      "tier": "basic",
      "status": "active",
      "expires_at": "2024-02-01T00:00:00Z"
    }
  }
}
```

#### 2.2 PUT /user/profile
更新用戶資料

**Request:**
```json
{
  "name": "New Name",
  "avatar_url": "https://..."
}
```

#### 2.3 GET /user/level
取得用戶等級

**Response:**
```json
{
  "level": {
    "current": 5,
    "title": "Intermediate",
    "total_score": 250,
    "progress": {
      "to_next": 50,
      "current": 250,
      "next": 300
    },
    "consecutive_wins": 3,
    "cumulative_practices": 45,
    "discount": {
      "level_5": true,
      "level_10": false
    }
  }
}
```

#### 2.4 GET /user/practice-history
練習歷史

**Query Parameters:**
- `page`: 頁碼，預設 1
- `limit`: 每頁數量，預設 20

**Response:**
```json
{
  "records": [
    {
      "id": "uuid",
      "scenario": {
        "id": "uuid",
        "name": "Code Review",
        "role": "Developer"
      },
      "score": 85,
      "completed_at": "2024-01-01T00:00:00Z"
    }
  ],
  "pagination": {
    "page": 1,
    "limit": 20,
    "total": 100
  }
}
```

#### 2.5 GET /user/errors
錯誤複習

**Response:**
```json
{
  "errors": [
    {
      "id": "uuid",
      "type": "pronunciation",
      "original": "I want to",
      "corrected": "I want to",
      "suggested": "I'd like to",
      "frequency": 3,
      "mastered": false
    }
  ]
}
```

---

### 3. 情境 (Scenarios)

#### 3.1 GET /scenarios
取得情境列表

**Query Parameters:**
- `tier`: 訂閱等級過濾
- `category`: 分類過濾
- `role`: 角色過濾
- `difficulty`: 難度過濾
- `page`: 頁碼
- `limit`: 每頁數量

**Response:**
```json
{
  "scenarios": [
    {
      "id": "uuid",
      "code": "DEV_CR_01",
      "name": "Code Review Discussion",
      "description": "Discuss code changes with team lead",
      "role_1": "Developer",
      "role_2": "Tech Lead",
      "category": "development",
      "difficulty": 3,
      "dialogue_count": 8,
      "tier_required": "free",
      "is_unlocked": true
    }
  ],
  "pagination": {
    "page": 1,
    "limit": 20,
    "total": 180
  }
}
```

#### 3.2 GET /scenarios/{id}
取得情境詳情

**Response:**
```json
{
  "scenario": {
    "id": "uuid",
    "code": "DEV_CR_01",
    "name": "Code Review Discussion",
    "description": "...",
    "role_1": "Developer",
    "role_2": "Tech Lead",
    "category": "development",
    "difficulty": 3,
    "dialogues": [
      {
        "sequence": 1,
        "speaker": "Tech Lead",
        "content": "Can you walk me through this change?",
        "vocabulary": ["walk through", "change"],
        "audio_url": "https://..."
      }
    ],
    "vocabulary": [
      {
        "word": "refactor",
        "definition": "Restructure code",
        "example": "We need to refactor this module"
      }
    ],
    "is_unlocked": true
  }
}
```

#### 3.3 GET /scenarios/categories
取得分類列表

**Response:**
```json
{
  "categories": [
    {
      "code": "requirement",
      "name": "需求分析",
      "count": 20
    },
    {
      "code": "development",
      "name": "開發實作",
      "count": 30
    },
    {
      "code": "testing",
      "name": "測試相關",
      "count": 25
    },
    {
      "code": "deployment",
      "name": "部署上線",
      "count": 20
    },
    {
      "code": "communication",
      "name": "跨國溝通",
      "count": 25
    }
  ]
}
```

---

### 4. 練習 (Practice)

#### 4.1 POST /practice/start
開始練習

**Request:**
```json
{
  "scenario_id": "uuid"
}
```

**Response:**
```json
{
  "practice": {
    "id": "uuid",
    "scenario_id": "uuid",
    "status": "in_progress",
    "started_at": "2024-01-01T00:00:00Z"
  },
  "dialogue": {
    "sequence": 1,
    "speaker": "Tech Lead",
    "content": "Can you walk me through this change?",
    "audio_url": "https://..."
  }
}
```

#### 4.2 POST /practice/{id}/submit
提交語音答案

**Request:**
```json
{
  "audio": "<base64 or audio_url>",
  "transcript": "Can you walk me through this change?"
}
```

**Response:**
```json
{
  "evaluation": {
    "pronunciation": 28,
    "grammar": 29,
    "vocabulary": 18,
    "fluency": 17,
    "total": 92,
    "feedback": "Great pronunciation! Minor improvement needed in fluency.",
    "suggestions": ["Try to speak more naturally", "Add pauses between phrases"]
  },
  "next_dialogue": {
    "sequence": 2,
    "speaker": "Developer",
    "content": "Sure, this change adds a new authentication module.",
    "audio_url": "https://..."
  }
}
```

#### 4.3 POST /practice/{id}/complete
完成練習

**Response:**
```json
{
  "practice": {
    "id": "uuid",
    "status": "completed",
    "completed_at": "2024-01-01T00:10:00Z",
    "total_score": 85,
    "evaluation": {
      "pronunciation": 28,
      "grammar": 29,
      "vocabulary": 18,
      "fluency": 17
    },
    "level_up": {
      "leveled_up": true,
      "new_level": 6,
      "message": "Congratulations! You've reached Level 6!"
    }
  },
  "errors": [
    {
      "type": "fluency",
      "original": "...",
      "suggested": "..."
    }
  ]
}
```

---

### 5. 訂閱 (Subscription)

#### 5.1 GET /subscription/plans
取得訂閱方案

**Response:**
```json
{
  "plans": [
    {
      "tier": "basic",
      "name": "入門版",
      "price": {
        "monthly": 100,
        "yearly": 1000,
        "discount": 0.8 // 5 級折扣
      },
      "features": {
        "scenarios": 180,
        "roles": ["Developer", "SA", "PM", "QA", "Tech Lead", "CTO"],
        "dialogue_partners": 3
      },
      "is_current": false
    },
    {
      "tier": "advanced",
      "name": "進階版",
      "price": {
        "monthly": 300,
        "yearly": 3000,
        "discount": 0.8
      },
      "features": {
        "scenarios": 540,
        "roles": [...],
        "dialogue_partners": 3
      },
      "is_current": false
    }
  ]
}
```

#### 5.2 POST /subscription/purchase
購買訂閱

**Request:**
```json
{
  "plan_tier": "basic",
  "billing_cycle": "monthly", // monthly, yearly
  "payment_method": "credit_card"
}
```

**Response:**
```json
{
  "payment_url": "https://payment.example.com/checkout/...",
  "order_id": "uuid"
}
```

#### 5.3 POST /subscription/cancel
取消訂閱

#### 5.4 GET /subscription/status
取得訂閱狀態

**Response:**
```json
{
  "subscription": {
    "tier": "basic",
    "status": "active",
    "started_at": "2024-01-01T00:00:00Z",
    "expires_at": "2024-02-01T00:00:00Z",
    "auto_renew": true
  },
  "benefits": {
    "scenarios_available": 180,
    "scenarios_used": 45,
    "discount_available": true,
    "discount_percentage": 20
  }
}
```

---

### 6. 統計 (Statistics)

#### 6.1 GET /statistics/overview
取得總覽

**Response:**
```json
{
  "overview": {
    "total_practices": 100,
    "average_score": 82,
    "total_practice_time": 5400, // 秒
    "current_streak": 7, // 天
    "level_progress": {
      "current": 5,
      "next": 6,
      "progress_percent": 83
    }
  },
  "breakdown": {
    "by_category": [
      {
        "category": "development",
        "count": 45,
        "average_score": 85
      }
    ],
    "by_difficulty": [
      {
        "difficulty": 3,
        "count": 30,
        "average_score": 78
      }
    ]
  }
}
```

---

## WebSocket API

### 連接

```
wss://api.nicespeak.app/ws/practice
```

### 認證

```json
{
  "type": "auth",
  "token": "jwt_token"
}
```

### 對話流程

```json
// Client → Server (語音數據)
{
  "type": "audio",
  "audio": "<base64>",
  "dialogue_id": "uuid"
}

// Server → Client (AI 評估)
{
  "type": "evaluation",
  "dialogue_id": "uuid",
  "pronunciation": 28,
  "grammar": 29,
  "vocabulary": 18,
  "fluency": 17,
  "total": 92,
  "feedback": "..."
}

// Server → Client (下一輪對話)
{
  "type": "next_dialogue",
  "dialogue": {
    "id": "uuid",
    "speaker": "Tech Lead",
    "content": "...",
    "audio_url": "..."
  }
}
```
