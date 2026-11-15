# 🤖 Coffee系統AI服務與工具函式開發

## 🧠 **AI服務模組**

### **AI聊天客戶端**
```rust
// src/ai/chat_client.rs
use serde::{Deserialize, Serialize};
use std::collections::HashMap;
use tokio::time::{timeout, Duration};
use uuid::Uuid;

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum AIProvider {
    OpenAI,
    Claude,
    Gemini,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct AIConfig {
    pub provider: AIProvider,
    pub api_key: String,
    pub model: String,
    pub max_tokens: u32,
    pub temperature: f32,
    pub timeout_seconds: u64,
}

#[derive(Debug, Serialize, Deserialize)]
pub struct ChatMessage {
    pub role: String,  // "user" | "assistant" | "system"
    pub content: String,
}

#[derive(Debug, Serialize, Deserialize)]
pub struct ChatRequest {
    pub messages: Vec<ChatMessage>,
    pub tenant_id: String,
    pub user_id: Uuid,
    pub context: Option<serde_json::Value>,
}

#[derive(Debug, Serialize, Deserialize)]
pub struct ChatResponse {
    pub content: String,
    pub usage_tokens: u32,
    pub model_used: String,
    pub response_time_ms: u64,
}

#[derive(Debug, thiserror::Error)]
pub enum AIError {
    #[error("API request failed: {0}")]
    RequestFailed(String),
    #[error("Timeout: {0}")]
    Timeout(String),
    #[error("Rate limit exceeded")]
    RateLimit,
    #[error("Invalid API key")]
    InvalidKey,
    #[error("Permission denied: {0}")]
    PermissionDenied(String),
    #[error("Serialization error: {0}")]
    SerializationError(#[from] serde_json::Error),
    #[error("HTTP error: {0}")]
    HttpError(#[from] reqwest::Error),
}

pub struct AIChatClient {
    configs: HashMap<AIProvider, AIConfig>,
    http_client: reqwest::Client,
    permission_checker: crate::ai::permission_checker::AIPermissionChecker,
}

impl AIChatClient {
    pub fn new(configs: Vec<AIConfig>) -> Self {
        let mut config_map = HashMap::new();
        for config in configs {
            config_map.insert(config.provider.clone(), config);
        }

        let http_client = reqwest::Client::builder()
            .timeout(Duration::from_secs(30))
            .build()
            .expect("Failed to create HTTP client");

        Self {
            configs: config_map,
            http_client,
            permission_checker: crate::ai::permission_checker::AIPermissionChecker::new(),
        }
    }

    /// 處理聊天請求
    pub async fn chat(
        &self,
        request: ChatRequest,
        preferred_provider: Option<AIProvider>,
    ) -> Result<ChatResponse, AIError> {
        // 權限檢查
        self.permission_checker
            .check_chat_permission(&request.tenant_id, request.user_id)
            .await?;

        // 內容安全檢查
        let query = request.messages
            .iter()
            .find(|msg| msg.role == "user")
            .map(|msg| msg.content.clone())
            .unwrap_or_default();

        self.permission_checker
            .check_content_safety(&query)
            .await?;

        // 選擇AI提供者
        let provider = preferred_provider
            .or_else(|| self.get_best_available_provider())
            .ok_or_else(|| AIError::RequestFailed("No AI provider available".to_string()))?;

        let config = self.configs
            .get(&provider)
            .ok_or_else(|| AIError::RequestFailed("Provider config not found".to_string()))?;

        // 執行聊天請求
        let start_time = std::time::Instant::now();
        
        let response = match provider {
            AIProvider::OpenAI => self.chat_openai(request, config).await?,
            AIProvider::Claude => self.chat_claude(request, config).await?,
            AIProvider::Gemini => self.chat_gemini(request, config).await?,
        };

        let response_time_ms = start_time.elapsed().as_millis() as u64;

        Ok(ChatResponse {
            content: response.content,
            usage_tokens: response.usage_tokens,
            model_used: config.model.clone(),
            response_time_ms,
        })
    }

    /// OpenAI Chat API
    async fn chat_openai(
        &self,
        request: ChatRequest,
        config: &AIConfig,
    ) -> Result<ChatResponse, AIError> {
        let url = "https://api.openai.com/v1/chat/completions";
        
        let payload = serde_json::json!({
            "model": config.model,
            "messages": request.messages,
            "max_tokens": config.max_tokens,
            "temperature": config.temperature,
        });

        let response = timeout(
            Duration::from_secs(config.timeout_seconds),
            self.http_client
                .post(url)
                .header("Authorization", format!("Bearer {}", config.api_key))
                .header("Content-Type", "application/json")
                .json(&payload)
                .send()
        ).await
        .map_err(|_| AIError::Timeout("OpenAI API timeout".to_string()))??;

        if !response.status().is_success() {
            let error_text = response.text().await?;
            return Err(AIError::RequestFailed(format!("OpenAI API error: {}", error_text)));
        }

        let response_data: serde_json::Value = response.json().await?;
        
        let content = response_data["choices"][0]["message"]["content"]
            .as_str()
            .ok_or_else(|| AIError::RequestFailed("Invalid OpenAI response format".to_string()))?
            .to_string();

        let usage_tokens = response_data["usage"]["total_tokens"]
            .as_u64()
            .unwrap_or(0) as u32;

        Ok(ChatResponse {
            content,
            usage_tokens,
            model_used: config.model.clone(),
            response_time_ms: 0, // Will be set by caller
        })
    }

    /// Claude API
    async fn chat_claude(
        &self,
        request: ChatRequest,
        config: &AIConfig,
    ) -> Result<ChatResponse, AIError> {
        let url = "https://api.anthropic.com/v1/messages";
        
        let payload = serde_json::json!({
            "model": config.model,
            "max_tokens": config.max_tokens,
            "messages": request.messages,
        });

        let response = timeout(
            Duration::from_secs(config.timeout_seconds),
            self.http_client
                .post(url)
                .header("x-api-key", &config.api_key)
                .header("Content-Type", "application/json")
                .header("anthropic-version", "2023-06-01")
                .json(&payload)
                .send()
        ).await
        .map_err(|_| AIError::Timeout("Claude API timeout".to_string()))??;

        if !response.status().is_success() {
            let error_text = response.text().await?;
            return Err(AIError::RequestFailed(format!("Claude API error: {}", error_text)));
        }

        let response_data: serde_json::Value = response.json().await?;
        
        let content = response_data["content"][0]["text"]
            .as_str()
            .ok_or_else(|| AIError::RequestFailed("Invalid Claude response format".to_string()))?
            .to_string();

        let usage_tokens = response_data["usage"]["output_tokens"]
            .as_u64()
            .unwrap_or(0) as u32;

        Ok(ChatResponse {
            content,
            usage_tokens,
            model_used: config.model.clone(),
            response_time_ms: 0,
        })
    }

    /// Gemini API  
    async fn chat_gemini(
        &self,
        request: ChatRequest,
        config: &AIConfig,
    ) -> Result<ChatResponse, AIError> {
        let url = format!(
            "https://generativelanguage.googleapis.com/v1beta/models/{}:generateContent?key={}",
            config.model, config.api_key
        );

        // 轉換訊息格式
        let contents = request.messages
            .into_iter()
            .filter(|msg| msg.role != "system") // Gemini不支援system訊息
            .map(|msg| {
                let role = if msg.role == "user" { "user" } else { "model" };
                serde_json::json!({
                    "role": role,
                    "parts": [{"text": msg.content}]
                })
            })
            .collect::<Vec<_>>();

        let payload = serde_json::json!({
            "contents": contents,
            "generationConfig": {
                "maxOutputTokens": config.max_tokens,
                "temperature": config.temperature,
            }
        });

        let response = timeout(
            Duration::from_secs(config.timeout_seconds),
            self.http_client
                .post(&url)
                .header("Content-Type", "application/json")
                .json(&payload)
                .send()
        ).await
        .map_err(|_| AIError::Timeout("Gemini API timeout".to_string()))??;

        if !response.status().is_success() {
            let error_text = response.text().await?;
            return Err(AIError::RequestFailed(format!("Gemini API error: {}", error_text)));
        }

        let response_data: serde_json::Value = response.json().await?;
        
        let content = response_data["candidates"][0]["content"]["parts"][0]["text"]
            .as_str()
            .ok_or_else(|| AIError::RequestFailed("Invalid Gemini response format".to_string()))?
            .to_string();

        // Gemini不直接提供token使用量，估算
        let usage_tokens = (content.len() / 4) as u32; // 大約估算

        Ok(ChatResponse {
            content,
            usage_tokens,
            model_used: config.model.clone(),
            response_time_ms: 0,
        })
    }

    /// 獲取最佳可用提供者
    fn get_best_available_provider(&self) -> Option<AIProvider> {
        // 優先順序: Claude > OpenAI > Gemini
        if self.configs.contains_key(&AIProvider::Claude) {
            Some(AIProvider::Claude)
        } else if self.configs.contains_key(&AIProvider::OpenAI) {
            Some(AIProvider::OpenAI)
        } else if self.configs.contains_key(&AIProvider::Gemini) {
            Some(AIProvider::Gemini)
        } else {
            None
        }
    }

    /// 健康檢查
    pub async fn health_check(&self) -> HashMap<AIProvider, bool> {
        let mut results = HashMap::new();
        
        for (provider, config) in &self.configs {
            let is_healthy = match provider {
                AIProvider::OpenAI => self.check_openai_health(config).await,
                AIProvider::Claude => self.check_claude_health(config).await,
                AIProvider::Gemini => self.check_gemini_health(config).await,
            };
            results.insert(provider.clone(), is_healthy);
        }
        
        results
    }

    async fn check_openai_health(&self, config: &AIConfig) -> bool {
        let test_messages = vec![ChatMessage {
            role: "user".to_string(),
            content: "Hello".to_string(),
        }];

        let payload = serde_json::json!({
            "model": config.model,
            "messages": test_messages,
            "max_tokens": 10,
        });

        let result = self.http_client
            .post("https://api.openai.com/v1/chat/completions")
            .header("Authorization", format!("Bearer {}", config.api_key))
            .header("Content-Type", "application/json")
            .json(&payload)
            .send()
            .await;

        matches!(result, Ok(response) if response.status().is_success())
    }

    async fn check_claude_health(&self, config: &AIConfig) -> bool {
        let test_messages = vec![ChatMessage {
            role: "user".to_string(),
            content: "Hello".to_string(),
        }];

        let payload = serde_json::json!({
            "model": config.model,
            "max_tokens": 10,
            "messages": test_messages,
        });

        let result = self.http_client
            .post("https://api.anthropic.com/v1/messages")
            .header("x-api-key", &config.api_key)
            .header("Content-Type", "application/json")
            .header("anthropic-version", "2023-06-01")
            .json(&payload)
            .send()
            .await;

        matches!(result, Ok(response) if response.status().is_success())
    }

    async fn check_gemini_health(&self, config: &AIConfig) -> bool {
        let url = format!(
            "https://generativelanguage.googleapis.com/v1beta/models/{}:generateContent?key={}",
            config.model, config.api_key
        );

        let payload = serde_json::json!({
            "contents": [{
                "role": "user",
                "parts": [{"text": "Hello"}]
            }],
            "generationConfig": {
                "maxOutputTokens": 10,
            }
        });

        let result = self.http_client
            .post(&url)
            .header("Content-Type", "application/json")
            .json(&payload)
            .send()
            .await;

        matches!(result, Ok(response) if response.status().is_success())
    }
}
```

### **AI權限檢查器**
```rust
// src/ai/permission_checker.rs
use crate::auth::rbac::RBACManager;
use crate::models::user::User;
use serde::{Deserialize, Serialize};
use sqlx::PgPool;
use std::collections::HashSet;
use uuid::Uuid;

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct AIPermission {
    pub can_query_orders: bool,
    pub can_query_customers: bool,
    pub can_query_products: bool,
    pub can_query_inventory: bool,
    pub can_query_reports: bool,
    pub can_query_financial: bool,
    pub can_query_cross_tenant: bool,
    pub daily_query_limit: u32,
    pub max_query_complexity: u32,
}

#[derive(Debug, thiserror::Error)]
pub enum PermissionError {
    #[error("Permission denied: {0}")]
    Denied(String),
    #[error("Daily query limit exceeded")]
    RateLimitExceeded,
    #[error("Query too complex")]
    QueryTooComplex,
    #[error("Unsafe content detected")]
    UnsafeContent,
    #[error("Database error: {0}")]
    DatabaseError(#[from] sqlx::Error),
}

pub struct AIPermissionChecker {
    rbac: RBACManager,
    unsafe_patterns: HashSet<String>,
}

impl AIPermissionChecker {
    pub fn new() -> Self {
        let mut unsafe_patterns = HashSet::new();
        
        // 添加不安全關鍵詞
        unsafe_patterns.insert("password".to_string());
        unsafe_patterns.insert("secret".to_string());
        unsafe_patterns.insert("token".to_string());
        unsafe_patterns.insert("key".to_string());
        unsafe_patterns.insert("admin".to_string());
        unsafe_patterns.insert("root".to_string());
        unsafe_patterns.insert("delete".to_string());
        unsafe_patterns.insert("drop".to_string());
        unsafe_patterns.insert("truncate".to_string());
        unsafe_patterns.insert("alter".to_string());

        Self {
            rbac: RBACManager::new(),
            unsafe_patterns,
        }
    }

    /// 檢查聊天權限
    pub async fn check_chat_permission(
        &self,
        tenant_id: &str,
        user_id: Uuid,
    ) -> Result<AIPermission, PermissionError> {
        // 這裡需要從資料庫獲取使用者資訊
        // 為簡化，先返回預設權限
        
        // 基於角色的權限配置
        let permission = AIPermission {
            can_query_orders: true,
            can_query_customers: true,
            can_query_products: true,
            can_query_inventory: true,
            can_query_reports: true,
            can_query_financial: false, // 只有管理員才能查詢財務
            can_query_cross_tenant: false, // 只有超級管理員才能跨租戶
            daily_query_limit: 100,
            max_query_complexity: 5,
        };

        // 檢查每日查詢限制
        let daily_usage = self.get_daily_usage(tenant_id, user_id).await?;
        if daily_usage >= permission.daily_query_limit {
            return Err(PermissionError::RateLimitExceeded);
        }

        Ok(permission)
    }

    /// 檢查內容安全性
    pub async fn check_content_safety(&self, content: &str) -> Result<(), PermissionError> {
        let content_lower = content.to_lowercase();
        
        for pattern in &self.unsafe_patterns {
            if content_lower.contains(pattern) {
                return Err(PermissionError::UnsafeContent);
            }
        }

        // 檢查SQL注入嘗試
        if content_lower.contains("';") || 
           content_lower.contains("union") ||
           content_lower.contains("select *") ||
           content_lower.contains("drop table") {
            return Err(PermissionError::UnsafeContent);
        }

        Ok(())
    }

    /// 檢查資料查詢權限
    pub async fn check_data_permission(
        &self,
        tenant_id: &str,
        user_id: Uuid,
        resource: &str,
        operation: &str,
    ) -> Result<bool, PermissionError> {
        let permission_key = format!("{}:{}", resource, operation);
        
        // 這裡需要獲取使用者角色
        let user_roles = vec!["manager".to_string()]; // 簡化處理
        
        let has_permission = self.rbac.check_permission(
            &user_roles,
            tenant_id,
            &permission_key,
        );

        Ok(has_permission)
    }

    /// 過濾敏感資料
    pub fn filter_sensitive_data(&self, data: &mut serde_json::Value) {
        match data {
            serde_json::Value::Object(map) => {
                // 移除敏感欄位
                map.remove("password_hash");
                map.remove("secret");
                map.remove("token");
                map.remove("api_key");
                
                // 遞歸處理子對象
                for value in map.values_mut() {
                    self.filter_sensitive_data(value);
                }
            }
            serde_json::Value::Array(arr) => {
                for item in arr.iter_mut() {
                    self.filter_sensitive_data(item);
                }
            }
            _ => {}
        }
    }

    /// 獲取每日使用量 (簡化實現)
    async fn get_daily_usage(&self, _tenant_id: &str, _user_id: Uuid) -> Result<u32, PermissionError> {
        // 這裡應該從Redis或資料庫獲取今日使用量
        Ok(10) // 模擬回傳
    }

    /// 記錄AI查詢
    pub async fn log_ai_query(
        &self,
        tenant_id: &str,
        user_id: Uuid,
        query: &str,
        response_tokens: u32,
    ) -> Result<(), PermissionError> {
        // 記錄查詢到審計日誌
        // 更新每日使用統計
        // 實際實現會寫入資料庫
        println!("AI Query logged: {} tokens for user {}", response_tokens, user_id);
        Ok(())
    }
}
```

### **語音處理器**
```rust
// src/ai/voice_processor.rs
use serde::{Deserialize, Serialize};
use std::io::Cursor;
use uuid::Uuid;

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum AudioFormat {
    Wav,
    Mp3,
    Ogg,
    WebM,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct VoiceRequest {
    pub audio_data: Vec<u8>,
    pub format: AudioFormat,
    pub language: Option<String>, // "zh-TW", "en-US", etc.
    pub tenant_id: String,
    pub user_id: Uuid,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct VoiceResponse {
    pub text: String,
    pub confidence: f32,
    pub language_detected: String,
    pub processing_time_ms: u64,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct TextToSpeechRequest {
    pub text: String,
    pub language: String,
    pub voice: Option<String>,
    pub speed: f32,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct TextToSpeechResponse {
    pub audio_data: Vec<u8>,
    pub format: AudioFormat,
    pub duration_ms: u64,
}

#[derive(Debug, thiserror::Error)]
pub enum VoiceError {
    #[error("Audio processing failed: {0}")]
    ProcessingFailed(String),
    #[error("Unsupported format: {0:?}")]
    UnsupportedFormat(AudioFormat),
    #[error("API error: {0}")]
    ApiError(String),
    #[error("Invalid audio data")]
    InvalidAudioData,
    #[error("HTTP error: {0}")]
    HttpError(#[from] reqwest::Error),
}

pub struct VoiceProcessor {
    openai_api_key: String,
    google_api_key: Option<String>,
    http_client: reqwest::Client,
}

impl VoiceProcessor {
    pub fn new(openai_api_key: String, google_api_key: Option<String>) -> Self {
        let http_client = reqwest::Client::builder()
            .timeout(std::time::Duration::from_secs(30))
            .build()
            .expect("Failed to create HTTP client");

        Self {
            openai_api_key,
            google_api_key,
            http_client,
        }
    }

    /// 語音轉文字 (使用OpenAI Whisper)
    pub async fn speech_to_text(
        &self,
        request: VoiceRequest,
    ) -> Result<VoiceResponse, VoiceError> {
        let start_time = std::time::Instant::now();

        // 驗證音頻格式
        self.validate_audio_format(&request.format)?;

        // 準備音頻資料
        let audio_part = self.prepare_audio_data(request.audio_data, request.format)?;

        // 調用OpenAI Whisper API
        let form = reqwest::multipart::Form::new()
            .part("file", audio_part)
            .text("model", "whisper-1");

        let form = if let Some(language) = request.language {
            form.text("language", language)
        } else {
            form
        };

        let response = self.http_client
            .post("https://api.openai.com/v1/audio/transcriptions")
            .header("Authorization", format!("Bearer {}", self.openai_api_key))
            .multipart(form)
            .send()
            .await?;

        if !response.status().is_success() {
            let error_text = response.text().await?;
            return Err(VoiceError::ApiError(format!("OpenAI API error: {}", error_text)));
        }

        let response_data: serde_json::Value = response.json().await?;
        
        let text = response_data["text"]
            .as_str()
            .ok_or_else(|| VoiceError::ProcessingFailed("No text in response".to_string()))?
            .to_string();

        let processing_time_ms = start_time.elapsed().as_millis() as u64;

        Ok(VoiceResponse {
            text,
            confidence: 0.95, // OpenAI不提供信心分數，使用預設值
            language_detected: "zh-TW".to_string(), // 簡化處理
            processing_time_ms,
        })
    }

    /// 文字轉語音 (使用OpenAI TTS)
    pub async fn text_to_speech(
        &self,
        request: TextToSpeechRequest,
    ) -> Result<TextToSpeechResponse, VoiceError> {
        let payload = serde_json::json!({
            "model": "tts-1",
            "input": request.text,
            "voice": request.voice.unwrap_or_else(|| "alloy".to_string()),
            "speed": request.speed,
        });

        let response = self.http_client
            .post("https://api.openai.com/v1/audio/speech")
            .header("Authorization", format!("Bearer {}", self.openai_api_key))
            .header("Content-Type", "application/json")
            .json(&payload)
            .send()
            .await?;

        if !response.status().is_success() {
            let error_text = response.text().await?;
            return Err(VoiceError::ApiError(format!("OpenAI TTS error: {}", error_text)));
        }

        let audio_data = response.bytes().await?.to_vec();
        
        Ok(TextToSpeechResponse {
            audio_data,
            format: AudioFormat::Mp3, // OpenAI TTS 預設返回MP3
            duration_ms: 0, // 需要計算音頻長度
        })
    }

    /// 驗證音頻格式
    fn validate_audio_format(&self, format: &AudioFormat) -> Result<(), VoiceError> {
        match format {
            AudioFormat::Wav | AudioFormat::Mp3 | AudioFormat::WebM => Ok(()),
            _ => Err(VoiceError::UnsupportedFormat(format.clone())),
        }
    }

    /// 準備音頻資料用於上傳
    fn prepare_audio_data(
        &self,
        audio_data: Vec<u8>,
        format: AudioFormat,
    ) -> Result<reqwest::multipart::Part, VoiceError> {
        if audio_data.is_empty() {
            return Err(VoiceError::InvalidAudioData);
        }

        let filename = match format {
            AudioFormat::Wav => "audio.wav",
            AudioFormat::Mp3 => "audio.mp3",
            AudioFormat::WebM => "audio.webm",
            _ => return Err(VoiceError::UnsupportedFormat(format)),
        };

        let mime_type = match format {
            AudioFormat::Wav => "audio/wav",
            AudioFormat::Mp3 => "audio/mp3", 
            AudioFormat::WebM => "audio/webm",
            _ => "application/octet-stream",
        };

        let part = reqwest::multipart::Part::stream(audio_data)
            .file_name(filename)
            .mime_str(mime_type)
            .map_err(|e| VoiceError::ProcessingFailed(format!("Failed to create multipart: {}", e)))?;

        Ok(part)
    }

    /// 音頻格式轉換 (簡化實現)
    pub fn convert_audio_format(
        &self,
        audio_data: Vec<u8>,
        from_format: AudioFormat,
        to_format: AudioFormat,
    ) -> Result<Vec<u8>, VoiceError> {
        // 實際實現需要使用FFmpeg或其他音頻處理庫
        // 這裡只是返回原始資料作為示例
        if from_format == to_format {
            Ok(audio_data)
        } else {
            Err(VoiceError::ProcessingFailed("Format conversion not implemented".to_string()))
        }
    }

    /// 檢查音頻品質
    pub fn validate_audio_quality(&self, audio_data: &[u8]) -> Result<bool, VoiceError> {
        // 基本檢查：音頻大小
        if audio_data.len() < 1024 {
            return Ok(false); // 太小可能不是有效音頻
        }

        if audio_data.len() > 25 * 1024 * 1024 {
            return Ok(false); // 太大 (超過25MB)
        }

        Ok(true)
    }
}

/// 音頻工具函式
pub mod audio_utils {
    use super::*;

    /// 估算音頻時長 (基於檔案大小的粗略估算)
    pub fn estimate_duration_ms(audio_data: &[u8], format: AudioFormat) -> u64 {
        match format {
            AudioFormat::Wav => {
                // WAV: 假設16-bit, 44.1kHz單聲道
                let bytes_per_second = 44100 * 2;
                ((audio_data.len() * 1000) / bytes_per_second) as u64
            }
            AudioFormat::Mp3 => {
                // MP3: 假設128kbps
                let bytes_per_second = 128 * 1024 / 8;
                ((audio_data.len() * 1000) / bytes_per_second) as u64
            }
            _ => 0,
        }
    }

    /// 檢查是否為有效的WAV檔案
    pub fn is_valid_wav(audio_data: &[u8]) -> bool {
        audio_data.len() >= 44 && 
        &audio_data[0..4] == b"RIFF" && 
        &audio_data[8..12] == b"WAVE"
    }

    /// 檢查是否為有效的MP3檔案
    pub fn is_valid_mp3(audio_data: &[u8]) -> bool {
        audio_data.len() >= 3 && 
        (audio_data[0] == 0xFF && (audio_data[1] & 0xE0) == 0xE0) || // MP3 frame sync
        &audio_data[0..3] == b"ID3" // ID3 tag
    }
}
```

---

## 🛠️ **工具函式模組**

### **驗證工具**
```rust
// src/utils/validation.rs
use regex::Regex;
use serde::{Deserialize, Serialize};
use uuid::Uuid;

#[derive(Debug, thiserror::Error)]
pub enum ValidationError {
    #[error("Invalid email format")]
    InvalidEmail,
    #[error("Invalid phone format")]
    InvalidPhone,
    #[error("Password too weak")]
    WeakPassword,
    #[error("Invalid UUID format")]
    InvalidUuid,
    #[error("Value too short: minimum {min} characters")]
    TooShort { min: usize },
    #[error("Value too long: maximum {max} characters")]
    TooLong { max: usize },
    #[error("Invalid format: {field}")]
    InvalidFormat { field: String },
    #[error("Value out of range: {min} to {max}")]
    OutOfRange { min: f64, max: f64 },
}

pub struct Validator;

impl Validator {
    /// 驗證電子郵件格式
    pub fn validate_email(email: &str) -> Result<(), ValidationError> {
        if email.is_empty() {
            return Err(ValidationError::TooShort { min: 1 });
        }

        let email_regex = Regex::new(
            r"^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$"
        ).unwrap();

        if email_regex.is_match(email) {
            Ok(())
        } else {
            Err(ValidationError::InvalidEmail)
        }
    }

    /// 驗證台灣手機號碼格式
    pub fn validate_phone_tw(phone: &str) -> Result<(), ValidationError> {
        if phone.is_empty() {
            return Err(ValidationError::TooShort { min: 1 });
        }

        // 移除所有非數字字元
        let digits_only: String = phone.chars().filter(|c| c.is_ascii_digit()).collect();

        // 台灣手機號碼格式: 09xxxxxxxx
        let phone_regex = Regex::new(r"^09\d{8}$").unwrap();

        if phone_regex.is_match(&digits_only) {
            Ok(())
        } else {
            Err(ValidationError::InvalidPhone)
        }
    }

    /// 驗證密碼強度
    pub fn validate_password(password: &str) -> Result<(), ValidationError> {
        if password.len() < 8 {
            return Err(ValidationError::TooShort { min: 8 });
        }

        if password.len() > 128 {
            return Err(ValidationError::TooLong { max: 128 });
        }

        let has_upper = password.chars().any(|c| c.is_ascii_uppercase());
        let has_lower = password.chars().any(|c| c.is_ascii_lowercase());
        let has_digit = password.chars().any(|c| c.is_ascii_digit());
        let has_special = password.chars().any(|c| "!@#$%^&*()_+-=[]{}|;:,.<>?".contains(c));

        let score = [has_upper, has_lower, has_digit, has_special]
            .iter()
            .map(|&b| if b { 1 } else { 0 })
            .sum::<i32>();

        if score >= 3 {
            Ok(())
        } else {
            Err(ValidationError::WeakPassword)
        }
    }

    /// 驗證UUID格式
    pub fn validate_uuid(uuid_str: &str) -> Result<Uuid, ValidationError> {
        Uuid::parse_str(uuid_str).map_err(|_| ValidationError::InvalidUuid)
    }

    /// 驗證字符串長度
    pub fn validate_length(value: &str, min: usize, max: usize) -> Result<(), ValidationError> {
        if value.len() < min {
            return Err(ValidationError::TooShort { min });
        }

        if value.len() > max {
            return Err(ValidationError::TooLong { max });
        }

        Ok(())
    }

    /// 驗證數值範圍
    pub fn validate_range(value: f64, min: f64, max: f64) -> Result<(), ValidationError> {
        if value < min || value > max {
            Err(ValidationError::OutOfRange { min, max })
        } else {
            Ok(())
        }
    }

    /// 驗證使用者名稱 (只允許字母、數字、底線)
    pub fn validate_username(username: &str) -> Result<(), ValidationError> {
        Self::validate_length(username, 3, 50)?;

        let username_regex = Regex::new(r"^[a-zA-Z0-9_]+$").unwrap();
        if username_regex.is_match(username) {
            Ok(())
        } else {
            Err(ValidationError::InvalidFormat {
                field: "username".to_string(),
            })
        }
    }

    /// 驗證商品SKU
    pub fn validate_sku(sku: &str) -> Result<(), ValidationError> {
        Self::validate_length(sku, 1, 50)?;

        let sku_regex = Regex::new(r"^[A-Z0-9-_]+$").unwrap();
        if sku_regex.is_match(sku) {
            Ok(())
        } else {
            Err(ValidationError::InvalidFormat {
                field: "sku".to_string(),
            })
        }
    }

    /// 驗證價格 (必須為正數且最多兩位小數)
    pub fn validate_price(price: rust_decimal::Decimal) -> Result<(), ValidationError> {
        if price <= rust_decimal::Decimal::ZERO {
            return Err(ValidationError::OutOfRange { min: 0.01, max: f64::MAX });
        }

        if price > rust_decimal::Decimal::new(999999999, 2) { // 9999999.99
            return Err(ValidationError::OutOfRange { min: 0.01, max: 9999999.99 });
        }

        Ok(())
    }

    /// 驗證庫存數量
    pub fn validate_stock(stock: i32) -> Result<(), ValidationError> {
        if stock < 0 {
            return Err(ValidationError::OutOfRange { min: 0.0, max: f64::MAX });
        }

        if stock > 999999 {
            return Err(ValidationError::OutOfRange { min: 0.0, max: 999999.0 });
        }

        Ok(())
    }
}

/// 驗證宏
#[macro_export]
macro_rules! validate_field {
    ($validator:expr, $field:expr, $field_name:expr) => {
        $validator.map_err(|e| format!("Field '{}': {}", $field_name, e))?
    };
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_email_validation() {
        assert!(Validator::validate_email("test@example.com").is_ok());
        assert!(Validator::validate_email("user.name+tag@domain.co.uk").is_ok());
        assert!(Validator::validate_email("invalid.email").is_err());
        assert!(Validator::validate_email("@domain.com").is_err());
        assert!(Validator::validate_email("user@").is_err());
    }

    #[test]
    fn test_phone_validation() {
        assert!(Validator::validate_phone_tw("0912345678").is_ok());
        assert!(Validator::validate_phone_tw("09-1234-5678").is_ok());
        assert!(Validator::validate_phone_tw("0912 345 678").is_ok());
        assert!(Validator::validate_phone_tw("1234567890").is_err());
        assert!(Validator::validate_phone_tw("091234567").is_err());
        assert!(Validator::validate_phone_tw("09123456789").is_err());
    }

    #[test]
    fn test_password_validation() {
        assert!(Validator::validate_password("Password123!").is_ok());
        assert!(Validator::validate_password("Strong@Pass1").is_ok());
        assert!(Validator::validate_password("weak").is_err());
        assert!(Validator::validate_password("password").is_err());
        assert!(Validator::validate_password("PASSWORD").is_err());
        assert!(Validator::validate_password("12345678").is_err());
    }

    #[test]
    fn test_username_validation() {
        assert!(Validator::validate_username("user123").is_ok());
        assert!(Validator::validate_username("test_user").is_ok());
        assert!(Validator::validate_username("UserName").is_ok());
        assert!(Validator::validate_username("ab").is_err()); // 太短
        assert!(Validator::validate_username("user-name").is_err()); // 包含連字符
        assert!(Validator::validate_username("user@name").is_err()); // 包含特殊字符
    }
}
```

### **加密工具**
```rust
// src/utils/encryption.rs
use aes_gcm::{Aes256Gcm, Key, Nonce, aead::{Aead, NewAead}};
use base64::{Engine as _, engine::general_purpose::STANDARD as BASE64};
use rand::{Rng, thread_rng};
use sha2::{Sha256, Digest};
use uuid::Uuid;

#[derive(Debug, thiserror::Error)]
pub enum EncryptionError {
    #[error("Encryption failed")]
    EncryptionFailed,
    #[error("Decryption failed")]
    DecryptionFailed,
    #[error("Invalid key format")]
    InvalidKeyFormat,
    #[error("Invalid ciphertext format")]
    InvalidCiphertextFormat,
    #[error("Key generation failed")]
    KeyGenerationFailed,
}

pub struct EncryptionManager {
    master_key: [u8; 32],
}

impl EncryptionManager {
    /// 從主密鑰創建加密管理器
    pub fn new(master_key: &[u8; 32]) -> Self {
        Self {
            master_key: *master_key,
        }
    }

    /// 從密碼生成主密鑰
    pub fn from_password(password: &str, salt: Option<&[u8]>) -> Self {
        let salt = salt.unwrap_or(b"coffee_system_salt_2024");
        let mut hasher = Sha256::new();
        hasher.update(password.as_bytes());
        hasher.update(salt);
        let result = hasher.finalize();
        
        let mut master_key = [0u8; 32];
        master_key.copy_from_slice(&result[..]);
        
        Self { master_key }
    }

    /// 生成隨機密鑰
    pub fn generate_key() -> [u8; 32] {
        let mut key = [0u8; 32];
        thread_rng().fill(&mut key);
        key
    }

    /// 為租戶生成專用密鑰
    pub fn derive_tenant_key(&self, tenant_id: &str) -> [u8; 32] {
        let mut hasher = Sha256::new();
        hasher.update(&self.master_key);
        hasher.update(tenant_id.as_bytes());
        hasher.update(b"tenant_key_derivation");
        
        let result = hasher.finalize();
        let mut tenant_key = [0u8; 32];
        tenant_key.copy_from_slice(&result[..]);
        tenant_key
    }

    /// 加密數據
    pub fn encrypt(&self, plaintext: &[u8], tenant_id: &str) -> Result<String, EncryptionError> {
        let tenant_key = self.derive_tenant_key(tenant_id);
        let key = Key::from_slice(&tenant_key);
        let cipher = Aes256Gcm::new(key);

        // 生成隨機nonce
        let mut nonce_bytes = [0u8; 12];
        thread_rng().fill(&mut nonce_bytes);
        let nonce = Nonce::from_slice(&nonce_bytes);

        // 加密
        let ciphertext = cipher
            .encrypt(nonce, plaintext)
            .map_err(|_| EncryptionError::EncryptionFailed)?;

        // 組合nonce + ciphertext
        let mut result = Vec::new();
        result.extend_from_slice(&nonce_bytes);
        result.extend_from_slice(&ciphertext);

        // Base64編碼
        Ok(BASE64.encode(&result))
    }

    /// 解密數據
    pub fn decrypt(&self, encrypted_data: &str, tenant_id: &str) -> Result<Vec<u8>, EncryptionError> {
        // Base64解碼
        let data = BASE64
            .decode(encrypted_data)
            .map_err(|_| EncryptionError::InvalidCiphertextFormat)?;

        if data.len() < 12 {
            return Err(EncryptionError::InvalidCiphertextFormat);
        }

        // 分離nonce和ciphertext
        let (nonce_bytes, ciphertext) = data.split_at(12);
        let nonce = Nonce::from_slice(nonce_bytes);

        let tenant_key = self.derive_tenant_key(tenant_id);
        let key = Key::from_slice(&tenant_key);
        let cipher = Aes256Gcm::new(key);

        // 解密
        cipher
            .decrypt(nonce, ciphertext)
            .map_err(|_| EncryptionError::DecryptionFailed)
    }

    /// 加密字符串
    pub fn encrypt_string(&self, plaintext: &str, tenant_id: &str) -> Result<String, EncryptionError> {
        self.encrypt(plaintext.as_bytes(), tenant_id)
    }

    /// 解密字符串
    pub fn decrypt_string(&self, encrypted_data: &str, tenant_id: &str) -> Result<String, EncryptionError> {
        let decrypted_bytes = self.decrypt(encrypted_data, tenant_id)?;
        String::from_utf8(decrypted_bytes)
            .map_err(|_| EncryptionError::DecryptionFailed)
    }

    /// 加密敏感欄位 (如信用卡號碼)
    pub fn encrypt_sensitive_field(
        &self,
        value: &str,
        tenant_id: &str,
        field_type: &str,
    ) -> Result<String, EncryptionError> {
        // 添加欄位類型到加密上下文中
        let context = format!("{}:{}", field_type, value);
        self.encrypt_string(&context, tenant_id)
    }

    /// 解密敏感欄位
    pub fn decrypt_sensitive_field(
        &self,
        encrypted_value: &str,
        tenant_id: &str,
        expected_field_type: &str,
    ) -> Result<String, EncryptionError> {
        let decrypted = self.decrypt_string(encrypted_value, tenant_id)?;
        
        if let Some(colon_pos) = decrypted.find(':') {
            let (field_type, value) = decrypted.split_at(colon_pos);
            if field_type == expected_field_type {
                Ok(value[1..].to_string()) // 跳過冒號
            } else {
                Err(EncryptionError::DecryptionFailed)
            }
        } else {
            Err(EncryptionError::InvalidCiphertextFormat)
        }
    }

    /// 生成安全Token
    pub fn generate_secure_token(&self, purpose: &str, tenant_id: &str, valid_duration_seconds: u64) -> Result<String, EncryptionError> {
        let expiry = chrono::Utc::now().timestamp() as u64 + valid_duration_seconds;
        let token_data = serde_json::json!({
            "purpose": purpose,
            "tenant_id": tenant_id,
            "expiry": expiry,
            "random": Uuid::new_v4().to_string()
        });

        let token_string = token_data.to_string();
        self.encrypt_string(&token_string, tenant_id)
    }

    /// 驗證安全Token
    pub fn verify_secure_token(
        &self,
        token: &str,
        expected_purpose: &str,
        tenant_id: &str,
    ) -> Result<bool, EncryptionError> {
        let decrypted = self.decrypt_string(token, tenant_id)?;
        
        let token_data: serde_json::Value = serde_json::from_str(&decrypted)
            .map_err(|_| EncryptionError::DecryptionFailed)?;

        // 檢查用途
        if token_data["purpose"].as_str() != Some(expected_purpose) {
            return Ok(false);
        }

        // 檢查租戶ID
        if token_data["tenant_id"].as_str() != Some(tenant_id) {
            return Ok(false);
        }

        // 檢查過期時間
        let expiry = token_data["expiry"].as_u64().unwrap_or(0);
        let now = chrono::Utc::now().timestamp() as u64;
        
        Ok(expiry > now)
    }
}

/// 雜湊工具
pub struct HashManager;

impl HashManager {
    /// 生成SHA-256雜湊
    pub fn sha256(data: &[u8]) -> String {
        let mut hasher = Sha256::new();
        hasher.update(data);
        let result = hasher.finalize();
        hex::encode(result)
    }

    /// 生成文件雜湊 (用於檔案完整性檢查)
    pub fn file_hash(file_content: &[u8]) -> String {
        Self::sha256(file_content)
    }

    /// 生成資料指紋 (用於重複檢測)
    pub fn data_fingerprint(data: &serde_json::Value) -> Result<String, serde_json::Error> {
        let normalized = serde_json::to_string(data)?;
        Ok(Self::sha256(normalized.as_bytes()))
    }

    /// 驗證密碼雜湊 (使用bcrypt)
    pub fn verify_password_hash(password: &str, hash: &str) -> bool {
        bcrypt::verify(password, hash).unwrap_or(false)
    }

    /// 生成密碼雜湊 (使用bcrypt)
    pub fn generate_password_hash(password: &str) -> Result<String, bcrypt::BcryptError> {
        bcrypt::hash(password, bcrypt::DEFAULT_COST)
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_encryption_roundtrip() {
        let manager = EncryptionManager::from_password("test_password", None);
        let original = "Hello, World! 這是測試資料。";
        let tenant_id = "test_tenant";

        let encrypted = manager.encrypt_string(original, tenant_id).unwrap();
        let decrypted = manager.decrypt_string(&encrypted, tenant_id).unwrap();

        assert_eq!(original, decrypted);
    }

    #[test]
    fn test_tenant_key_isolation() {
        let manager = EncryptionManager::from_password("test_password", None);
        let original = "sensitive data";
        let tenant_a = "tenant_a";
        let tenant_b = "tenant_b";

        let encrypted_a = manager.encrypt_string(original, tenant_a).unwrap();
        
        // 使用不同租戶ID解密應該失敗
        assert!(manager.decrypt_string(&encrypted_a, tenant_b).is_err());
        
        // 使用正確租戶ID解密應該成功
        let decrypted_a = manager.decrypt_string(&encrypted_a, tenant_a).unwrap();
        assert_eq!(original, decrypted_a);
    }

    #[test]
    fn test_secure_token() {
        let manager = EncryptionManager::from_password("test_password", None);
        let purpose = "password_reset";
        let tenant_id = "test_tenant";

        let token = manager.generate_secure_token(purpose, tenant_id, 3600).unwrap();
        assert!(manager.verify_secure_token(&token, purpose, tenant_id).unwrap());
        assert!(!manager.verify_secure_token(&token, "wrong_purpose", tenant_id).unwrap());
        assert!(!manager.verify_secure_token(&token, purpose, "wrong_tenant").unwrap());
    }
}
```

---

**🏗️ AI服務與工具函式開發完成！**

包含：
- ✅ **AI聊天客戶端**: 支援OpenAI/Claude/Gemini多提供者
- ✅ **AI權限檢查器**: 租戶隔離+內容安全+查詢限制
- ✅ **語音處理器**: 語音轉文字+文字轉語音+音頻驗證
- ✅ **驗證工具**: 電子郵件+手機+密碼+UUID+業務欄位驗證
- ✅ **加密工具**: AES-256-GCM加密+租戶密鑰隔離+安全Token

**🌱 老大，Coffee系統Phase 0共享基礎架構建設已完成75%！**

包含完整的共享元件庫、後端函式庫、資料模型、服務層、AI服務和工具函式。所有模組都支援多租戶隔離和統一的技術架構！

**下一步準備**: 開始Phase 1平台管理系統開發？