# FastIM 快速部署腳本

## 自動化部署腳本
本腳本將自動執行 FastIM 後端的完整部署流程。

### 使用方式
```bash
cd /root/.openclaw/workspace/fastIM
chmod +x scripts/deploy.sh
./scripts/deploy.sh
```

### 參數說明
- `--build-only`: 僅編譯，不部署
- `--deploy-only`: 僅部署，不重新編譯  
- `--clean`: 清理現有部署後重新部署
- `--help`: 顯示說明

### 範例
```bash
# 完整部署（預設）
./scripts/deploy.sh

# 僅重新編譯
./scripts/deploy.sh --build-only

# 清理後重新部署
./scripts/deploy.sh --clean
```