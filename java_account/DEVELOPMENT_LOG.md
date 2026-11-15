# 開發紀錄

## 2026-11-16

- 新增 `java_account` 文件彙整目錄，包含專案說明與系統架構。
- 整理三個獨立 Maven／Spring Boot 服務的職責、資料流、技術基線與建置前提。
- 此目錄只保存說明文件，不複製程式碼、環境設定或憑證。
- 本次未執行 Maven 建置或服務整合測試；文件不能視為部署驗證結果。
- 文件與索引以 commit `c035a3d` 推送至公開遠端 `lucas152112/demo` 的 `main`；推送後已讀回確認遠端 commit 與本機一致。
- 依使用者確認，將 `javatest`、`javatrst` 設為彙整例外：自索引移除兩項，且不建立其 repo 說明內容。
- 例外設定與索引調整以 commit `55802cd` 推送至公開遠端 `lucas152112/demo` 的 `main`，並讀回確認遠端版本。
- 經授權移除遠端 `javatest/README.md`、`javatrst/README.md`，以 commit `224d03e` 推送並確認遠端樹中已不存在；其他既有刪除未納入。
- 依要求同步唯一剩餘的本機刪除 `private-repos-documentation-README.md`；並移除根 README 中指向該檔的失效連結，其餘待提交刪除不納入。
- 以 commit `f315e2a` 推送至公開遠端 `lucas152112/demo` 的 `main`；讀回確認該檔及兩個例外 README 均已從遠端版本樹移除，根 README 連結檢查通過。
