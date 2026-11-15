# java_account 系統架構

本文描述 `java_account` 三個 Spring Boot 專案的程式邊界、請求流向與可觀察到的基礎設施。文件副本位於 `~/demo/java_account/`；此處沒有服務原始碼或執行環境設定。

## 元件邊界

### `npf-gateway`：API Gateway

- 啟動類別：`com.sp.gateway.GatewayApplication`。
- 使用 Spring Cloud Netflix Zuul，依路由設定將請求轉送至服務 ID。
- Zuul filters 處理認證、IP／黑白名單、維護模式、財務與自動投注相關前置處理、上傳及轉送標頭等流程。
- 專案包含部分 fallback 處理；不能據此假設所有路由均有降級機制。

### `new_tk1907`：平台服務

- Maven artifact：`npf-platform`。
- 啟動類別：`com.sp.platform.PlatformApplication`。
- 提供平台管理、會員／遊戲相關業務、資料存取、報表與排程等功能。
- 常見分層包含 Controller、Service、DAO、DTO、Validator 與 Cache。
- 程式使用負載平衡 `RestTemplate` 進行部分 HTTP 下游呼叫。

### `finace1907`：財務／帳務服務

- Maven artifact：`npf-finance`。
- 啟動類別：`com.sp.financial.PaymentApplication`。
- 涵蓋存提款、交易、帳戶、支付、報表與管理功能。
- 常見分層包含 Controller、Service、DAO、DTO、Validator 與 Cache；另有支付整合、Google Auth、例外處理及背景工作程式。

三個目錄各自擁有 Maven POM 與 Spring Boot 啟動流程。根目錄是 Git monorepo，不是 Maven parent／reactor 專案。

## 請求與資料流

1. 呼叫端將 HTTP 請求送至 Gateway。
2. Gateway 套用 Zuul 路由與 filters，執行對應的請求前處理，再依 service ID 路由至平台或財務服務。
3. Eureka client 提供服務探索能力；Eureka server 不在這三個專案的文件範圍內。
4. 各服務由 Spring MVC Controller 接收請求，Service 執行業務邏輯，DAO／資料工具負責資料存取。
5. 服務使用 MySQL（包含多資料源設定）及 Redis；部分共用資料庫與工具功能由專案內本地 JAR 提供。
6. 平台服務會呼叫部分下游平台、遊戲或通知服務；財務服務會整合支付及其他外部服務。其實際端點由環境設定決定，本文不列出位址。

## 啟動期與執行期能力

- 平台與財務服務可見資料來源、Redis、執行緒池及排程／執行期狀態初始化程式。
- Gateway 可見資料來源與 Redis 初始化程式。
- 三個 POM／程式可見 Eureka discovery 與 Hystrix metrics 相關整合；實際暴露狀態仍取決於環境設定。
- Gateway 的路由、服務位址、資料來源、快取與外部整合參數均依環境設定，不應將某一組設定視為所有環境共用值。

## 建置與依賴關係

- Java 版本：三個 POM 指定 Java 8。
- Spring Boot：Gateway 2.0.2；平台與財務服務 2.1.0。
- 服務分開執行 `mvn -f <專案目錄>/pom.xml clean package`。
- `src/lib` 中有本地 JAR 相依；建置前須備妥對應檔案與其所需的 Maven／JDK 環境。
- 執行還依賴正確的資料庫、Redis、Eureka 及下游服務設定。

## 安全邊界與未確認項目

- 本文件不公開憑證值、正式環境網址或實際連線參數。
- 部分環境設定與可能含敏感值的原始碼不包含在這份文件副本。
- 此架構摘要描述程式中可見的元件，不代表所有路由、API、排程或外部服務都已逐項運行驗證。
- 目前沒有以本文件副本執行建置或整合測試；完整驗證需在具備 JDK 8、本地 JAR 與安全環境設定的環境進行。
