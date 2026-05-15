# 咖啡館連鎖管理系統 - 文件索引

## 專案概述
本系統為咖啡館連鎖店管理系統，包含門店管理（CRM）和總部管理（ERP）兩大模組。

**技術棧:**
- 後端：Go + Gin 微服務架構
- 前端：Vue 3 + Nuxt 3 + TypeScript
- 資料庫：MySQL 8.0+
- 快取：Redis
- 訊息佇列：NATS
- 容器化：Docker + Docker Compose

---

## 📚 文件結構

### 01. 系統分析 (SA - System Analysis)
- [需求分析](./01-SA-SystemAnalysis/01-Requirements.md)
- [使用案例圖](./01-SA-SystemAnalysis/02-UseCaseDiagram.md)
- [使用案例規格](./01-SA-SystemAnalysis/03-UseCaseSpecifications.md)
- [領域模型](./01-SA-SystemAnalysis/04-DomainModel.md)
- [業務流程](./01-SA-SystemAnalysis/05-BusinessProcesses.md)
- [系統範圍與限制](./01-SA-SystemAnalysis/06-ScopeAndConstraints.md)

### 02. 系統設計 (SD - System Design)
- [架構設計](./02-SD-SystemDesign/01-ArchitectureDesign.md)
- [類別圖](./02-SD-SystemDesign/02-ClassDiagrams.md)
- [序列圖](./02-SD-SystemDesign/03-SequenceDiagrams.md)
- [元件圖](./02-SD-SystemDesign/04-ComponentDiagram.md)
- [部署圖](./02-SD-SystemDesign/05-DeploymentDiagram.md)
- [狀態圖](./02-SD-SystemDesign/06-StateDiagrams.md)

### 03. 資料庫設計 (Database Design)
- [資料庫 ERD](./03-Database/01-ERD.md)
- [資料字典](./03-Database/02-DataDictionary.md)
- [索引設計](./03-Database/03-IndexDesign.md)
- [資料庫規範](./03-Database/04-DatabaseStandards.md)

### 04. API 規格 (API Specifications)
- [API 總覽](./04-API-Specs/00-API-Overview.md) ✅
- [Auth API](./04-API-Specs/01-Auth-API.md) ✅
- [Customers API](./04-API-Specs/02-Customers-API.md) ✅
- [Orders API](./04-API-Specs/03-Orders-API.md) ✅
- [Inventory API](./04-API-Specs/04-Inventory-API.md) 🔄
- [Loyalty API](./04-API-Specs/05-Loyalty-API.md)
- [Notifications API](./04-API-Specs/06-Notifications-API.md)
- [Reports API](./04-API-Specs/07-Reports-API.md)

### 05. 系統原型 (Prototypes)
- [UI/UX 設計指南](./05-Prototypes/01-UI-UX-Guidelines.md)
- [CRM 前端原型](./05-Prototypes/02-CRM-Wireframes.md)
- [ERP 前端原型](./05-Prototypes/03-ERP-Wireframes.md)
- [使用者流程](./05-Prototypes/04-UserFlows.md)

### 06. 技術架構 (Architecture)
- [技術選型說明](./06-Architecture/01-TechStack.md)
- [微服務架構](./06-Architecture/02-MicroservicesArchitecture.md)
- [安全設計](./06-Architecture/03-SecurityDesign.md)
- [效能考量](./06-Architecture/04-PerformanceConsiderations.md)
- [監控與日誌](./06-Architecture/05-MonitoringAndLogging.md)

### 07. 開發規範 (Development Standards)
- [編碼規範](./07-Development/01-CodingStandards.md)
- [Git 工作流程](./07-Development/02-GitWorkflow.md)
- [測試策略](./07-Development/03-TestingStrategy.md)
- [CI/CD 流程](./07-Development/04-CICD.md)
- [程式碼審查指南](./07-Development/05-CodeReview.md)

### 08. 專案管理 (Project Management)
- [專案時程](./08-ProjectManagement/01-Timeline.md)
- [里程碑](./08-ProjectManagement/02-Milestones.md)
- [風險管理](./08-ProjectManagement/03-RiskManagement.md)
- [團隊分工](./08-ProjectManagement/04-TeamStructure.md)
- [開發前檢查清單](./08-ProjectManagement/05-PreDevelopmentChecklist.md)

---

## 📋 文件狀態

| 類別 | 狀態 | 完成日期 |
|------|------|----------|
| 系統分析 (SA) | ✅ 已完成 | 2025-11-16 |
| 系統設計 (SD) | ✅ 已完成 | 2025-11-16 |
| 資料庫設計 | ✅ 已完成 | 2025-11-16 |
| API 規格 | 🟡 進行中 (57%) | - |
| 系統原型 | ⚪ 未開始 | - |
| 技術架構 | ✅ 已完成 | 2025-11-16 |
| 開發規範 | ⚪ 未開始 | - |
| 專案管理 | 🟡 進行中 | - |

---

## 📝 版本歷程

| 版本 | 日期 | 作者 | 說明 |
|------|------|------|------|
| 0.1 | 2025-11-16 | System | 初始文件架構建立 |
| 0.2 | 2025-11-16 | System | 完成 SA/SD/資料庫設計文件 |
| 0.3 | 2025-11-16 | System | 新增 API 規格文件 (4/7 完成) |

---

## ✅ 開發前確認事項

在開始開發前，請確認以下文件已完成且經過審查：

- [x] 所有 SA 文件已完成並經過審查
- [x] 所有 SD 文件已完成並經過審查
- [x] 資料庫設計已完成並經過審查
- [x] API 總覽規格已完成並經過審查
- [x] Auth API 規格已完成並經過審查
- [x] Customers API 規格已完成並經過審查
- [x] Orders API 規格已完成並經過審查
- [ ] Inventory API 規格已完成並經過審查 (進行中)
- [ ] Loyalty API 規格已完成並經過審查
- [ ] Notifications API 規格已完成並經過審查
- [ ] Reports API 規格已完成並經過審查
- [ ] 系統原型已完成並經過審查
- [ ] 技術架構已確定並經過審查
- [ ] 開發規範已制定並經過審查
- [ ] 專案時程已確定並經過審查
- [ ] 團隊成員已完成培訓
- [ ] 開發環境已準備就緒

**⚠️ 警告：所有文件必須經過完整審查和確認後，才可開始實際開發工作。**
