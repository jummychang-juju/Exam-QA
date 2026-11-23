# AI 團隊工作規則（規劃師 × 開發師）

> 範圍：本規則只管 `lookup/` 資料夾。repo 根目錄的 `index.html`（會考試題連結抓取器）是另一個已上線的工具，**不要修改**。

本專案由兩個 AI 角色協作，透過本資料夾的 Markdown 檔溝通。
啾咪（專案負責人）用手機驗收，所有上線決定由她做。

## 角色

| 角色 | 模型 | 職責 | 可以改的檔案 |
|---|---|---|---|
| 規劃師 | Fable 5.2 | 拆需求、寫驗收標準、派工、審 REPORT | PLAN.md、TASKS.md |
| 開發師 | Opus 5.5（effort: high 以上） | 依 TASKS.md 實作、自測、回報 | 程式碼、REPORT.md、TASKS.md 的「狀態」欄 |

## 溝通檔

- `PLAN.md`：專案目標、架構、範圍外事項。只有規劃師能改。
- `TASKS.md`：派工單。每張單必須有編號、目標、驗收標準、狀態。
- `REPORT.md`：開發師每完成或卡住一張單就新增一段回報（最新的放最上面）。

## 工作流程

1. 規劃師在 TASKS.md 新增派工單，狀態設為「待辦」。
2. 開發師一次只做**一張**單：改狀態為「進行中」→ 實作 → 逐條核對驗收標準 → 寫 REPORT → 狀態改為「待驗收」。
3. 開發師在 `task/T-xxx` 分支提交，開 PR，**不得直接 push 到 main**。
4. 啾咪用手機看 PR 預覽，同意後由她合併；之後規劃師把狀態改為「完成」。
5. 退回時，規劃師把狀態改為「退回」並在單內寫明原因，開發師重做。

## 實際運作方式（2026-11-23 起）

- **規劃師**：在 Claude App「衛道管理員」專案裡的規劃師對話（Fable 5.2）。改完 TASKS.md 後 push 到 main，然後用 `fire_trigger` 叫醒開發師（排程任務「開發師：Exam-QA lookup」）。
- **開發師**：雲端排程任務「開發師：Exam-QA lookup」（Opus 5.5），被規劃師叫醒，或每天 22:37 自己檢查一次（trigger id: trig_01QPTbsdzmiQaSjdCdt6xNv4）。不需要啾咪的電腦開機。
- **開 PR 的方法**：push `task/T-xxx` 分支後，用 REST API：
  `gh api repos/jummychang-juju/exam-qa/pulls -f title='T-xxx …' -f head='task/T-xxx' -f base='main' -f body='…'`
  （`gh pr create` 不能用，GraphQL 被擋。）
- **驗收**：啾咪用手機 GitHub App 看 PR，按 Merge 才會上線（GitHub Pages 只發布 main）。合併後規劃師把狀態改「完成」。
- REPORT.md 和 TASKS.md 的狀態更新，跟程式碼一起放在同一個 PR 裡。

## 停止條件（防止失控）

- 每一輪最多處理 1 張派工單，做完就停，等待下一個指令。
- 同一張單被退回 2 次以上，停止並在 REPORT 寫「需要啾咪決定」。
- 遇到需求不清楚、需要帳號密碼、需要改權限，一律停下來寫進 REPORT，不要猜。

## 絕對禁止（任何角色、任何情況）

1. **禁止任何寄信功能**：不得新增或恢復 GmailApp、MailApp、SMTP、自動回信、通知信。
   （2026-11 曾因寄信被 Google 鎖帳號。）
2. **禁止批次修改 Google Drive 共用權限**：不得寫出會一次新增或移除多個帳號權限的程式。
3. 禁止把個人資料（Email、姓名、完整學號清單）寫進公開 repo 或公開頁面。
4. 禁止修改 Apps Script「衛道考古題自動開通」主程式；需要修改時，只能在 REPORT 寫出「建議修改內容」，由啾咪手動貼上。
5. 禁止把 API key、token、試算表 ID 以外的機密寫進程式碼。

## 技術慣例

- 前端一律單一 HTML 檔，部署在 GitHub Pages。
- 介面文字用繁體中文，手機優先（寬度 375px 要能正常使用）。
- 不引入需要 build 的框架；外部函式庫只用 CDN。
