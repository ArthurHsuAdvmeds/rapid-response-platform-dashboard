# qa — V9 版面接真實資料（驗證版）

把原型 v9（全部頁面、V9 的醫療機構端）改成由資料計算，給使用者驗證；驗證通過後再更新到 prod（第 6 節）。

- **資料格式與平台專案要做的事以 [DATA_CONTRACT.md](DATA_CONTRACT.md) 為準，欄位以 [data.template.jsonc](data.template.jsonc)、[data-platform.template.jsonc](data-platform.template.jsonc) 為準。**
- `data.json`、`data-platform.json` 目前是虛構範例（`"sample": true`），由 `.tool/qa_sample_data.py` 產生（固定種子，重跑結果相同）；平台專案的正式檔會直接覆蓋。**不要手動改這兩個檔**。
  範例的通報日都在 2026/10/05–11/09（約 210 件）；為了讓要等結案、追蹤的頁面也有數字可驗，優化計畫結案時間壓短，3／6 個月追蹤與 HIS 次數不檢查時間是否已到（正式資料剛上線時這些頁面會是「—」）。
- **AI模型預測準確率先隱藏**（使用者 2026-11-09 指示）：主選單按鈕註解掉、`#ai` 加 `hidden`、≥1200px 主選單改 8 欄（`<style id="qa-data">`）。程式、HTML、`ai` 欄位都保留，恢復時照按鈕旁的註解還原。
- **指揮儀表板「同期比較」先隱藏**（V9 追加，比照 prod：平台 2026/10/05 起才有資料，沒有往年可比）：子頁籤按鈕註解掉，`#commandComparisonView` 與 `renderComparison()` 都保留，恢復時拿掉按鈕的註解即可。
- **平台起始日固定 2026/10/05**（V9 追加）：`PLATFORM_START`，詳見第 3 節。只套用在 qa，原型的模擬日期不動（使用者決定）。
- 版面、疊層 CSS、各版補丁的注意事項見根目錄與 [../prototype/CLAUDE.md](../prototype/CLAUDE.md)；qa 包含 v9 以前的所有層。

---

## 1. 怎麼從 v9 做出來的

- 複製 `prototype/v9/index.html`，套用 prod 當初對 v7 做的 JS 手術（`v7→prod` 的 diff 只挑 JS 的 hunk 套用，主腳本內容 v7 與 v9 相同），**HTML 一律保留**：prod 刪掉的成效、AI、醫療機構端、SNQ 頁，三顆通報類別按鈕、同期比較、第一～三類優化計畫的流程列都在。
- **刪除**的原型程式：主腳本的風險頁模擬資料、基本資料頁模擬、效率頁～指揮板 V4 的模擬區塊（含成效頁、通報者頁、效率頁風險下探 dialog）、同期比較模擬、醫療機構端／通報者頁的 `DOMContentLoaded` 區塊、年齡長條的 JS tooltip；
  整段移除 `v68`（通報者篩選）、`v73`（醫療機構端／SNQ 篩選）、`v5-command-compare-efficiency`、`v8-platform-usage-behavior`、`v9-hospital-indicators-behavior`（後兩段留一行註解標示）。
- **改寫**：`v80` 只留指標說明 dialog（定義含優化計畫指標）、子頁籤切換、主選單保險，移除模擬值與日期倍率；AI 腳本的 `buildAiPairs()`／`getAiHistoryRows()` 改讀 `RD.aiPairs()`／`RD.aiHistory()`，資料不足時畫「尚無…」。
- **保留**：分頁切換、地圖模組（已是 prod 的真實案件版）、指揮板連線幾何（V7）、`v84` 響應式、`v86` 效率頁跳轉卡、`v5-dialog-behavior`、`v6` 手機下拉與長條橫向、`v7` 字級。
- **HTML 補的掛點**：效率頁總覽風險列 `data-eff-n`／`data-eff-bar`、衛生局分流件數 `data-eff-count="replied|tsgh|songde|hold"`（松德院區的比率是 `data-eff-computed="songdeReferral"`）、暫不轉介原因 `#effBureauReasonBody`、SNQ 卡片 `data-snq-metric`、平台使用率 `#usageAsOf`、醫療機構端轉介選項 `#hospitalReferralSplit`；頁首加 `#rdStatus`。
- **V9 追加**跟 v9 原型一起改的部分（文字、四項名稱、轉介率卡片一列一項不畫進度條、拿掉「定義待確認」）見 [../prototype/CLAUDE.md](../prototype/CLAUDE.md) 的 v9 注意事項；qa 另外改了代碼表、起始日、同期比較隱藏、範本與規格。

---

## 2. 程式在哪

全部在檔尾 `<style id="qa-data">` 與 `<script id="qa-data">`（`window.RD`）。

- **載入**：`loadFromServer()` 同時抓 `data.json` 與 `data-platform.json`；後者失敗只記在 `RD.platformError`，不中斷。file:// 用選檔，可以一次選兩個檔（依內容判斷是哪一個）。
- **`build()`**：接受 `schemaVersion` 1（prod 舊格式，只有 emg，方便拿正式資料對照）與 2；`normalizeCase(r,'emg'|'opt')` 轉成共用的案件物件；通報日早於 `PLATFORM_START` 的排除（計入 `RD.stats.beforeStart`）。
  **data.json 改欄位名或代碼時，只改 `normalizeCase()` 與檔頭代碼表**（含 `BUREAU_RESULT`：1 北投／2 松德／3 暫不轉介，`REFERRAL`：醫療資源轉介四項；`referralLabel()` 也接受直接寫選項名稱）。
- **案件物件**：prod 的欄位全部沿用（`id`、`reported`、`day`、`status`、`risk`、`network`、`county`、`district`、`lat`／`lng`、`levels`、`assigned`、`surveys`…），另加
  `kind`（emg／opt）、`category`（emg／opt1／opt2／opt3）、`closed`、`scores`、`followUp`、`his`、`ai`；
  emg：`intakeMethod`、`intake`、`dispatched`；opt：`bureauReplied`、`reviewed`、`hospitalReplied`、`firstVisit`、`bureauResult`、`noReferralReason`、`bureauVisitDays`、`bureauDecisionDays`、`reviewResult`、`opened`、`unableReason`、`contact`、`monthly`、`closing`。
- **篩選**：`PAGES` 設定 8 個分頁的日期快速鈕、套用鈕、通報類別鈕；`render(ctx)` 拿到 `{ cases（日期內全部）, filtered（再依通報類別）, range, type: all／emg／opt }`。
  `presetRange()`、`setRange()`、`initDateInputs()` 都以 `PLATFORM_START` 為下限。同期比較（目前隱藏）另有 `compareState`（模式、月份、類別），不受指揮板日期影響。
- **`effMetrics(cases)`**：效率頁四個頁籤與同期比較共用，回傳的鍵名就是 HTML 的 `data-eff-value`。
- **各頁**：`renderBasic`、`renderRisk`、`renderEfficiency`、`renderOutcome`（改善情況、評估表填寫狀況、後續醫療資源、個案明細 dialog）、`renderReporter`、
  `renderHospital`（V9 兩區、無法收案 dialog、指標計算說明 dialog）、`renderSnq`（月彙總）、`renderCommandFlow`、`renderClosure`、`renderComparison`、`renderUsage`。
  成效頁子頁籤、個案明細的篩選與排序、醫療機構端與 SNQ 的 dialog 開關也在這裡（原本的程式隨模擬資料一起刪了）。
- **頁首狀態列** `renderStatus()`：資料更新時間、總案件數（緊急醫療／優化計畫，「2026/10/05 起」）、範例資料標籤、黃色提示（超過 48 小時未更新、平台資料未載入、早於起始日已排除、代碼對不到（含醫療資源轉介）、reportId 重複…）。

---

## 3. 推導規則與指標（prod 沒有的部分）

prod/CLAUDE.md 第 2、3 節的規則（待派／已派、初評風險取 CTRS、縣市推論、滿意度應回收、各頁算法）qa 全部沿用。所有分頁都以 `reportedAt` 的臺北日期篩選。

| 範圍 | 算法 |
| --- | --- |
| 平台起始日 | 固定 2026/10/05（不讀 data.json 的 `dataStart`）：日期欄位最早可選、快速鈕的下限（2026 年的本季、本年度都是 10/05 起）、手動輸入更早的日期拉回這天、指揮板預設區間起點、年度結案率從 2026 年列起；早於這天的通報排除 |
| 效率頁總覽 | 依初評風險：緊醫 `acceptedAt`、優化計畫 `firstVisitAt` 減 `reportedAt`，以「天」計，兩類一起平均 |
| 優化計畫醫院端 | 開案率＝轉介三總北投中 `opened=true` ÷ 轉介三總北投；覆核同意／駁回率、醫療介入率的分母＝暫不轉介；覆核時間＝`reviewedAt − bureauRepliedAt`（天）；派案、首次聯繫以小時，首次專業評估、回覆衛生局以天 |
| 優化計畫衛生局 | 分母＝有 `bureau.result` 的案件（已完成回覆單）；醫療轉介率＝(北投＋松德)÷已完成回覆單（V9 追加確認分母）；本院占率＝北投÷(北投＋松德)；訪視／分流決策時間用回覆單日期與通報日相差的天數 |
| 成效頁改善情況 | 改善率＝(初評−追蹤)÷初評（GAF 反過來），> +1% 改善、< −1% 變差；N＝初評與該期追蹤分數都有的個案 |
| 成效頁評估表 | 應回收＝`followUp` 有值；已回收＝trackable＋unable；可追蹤率＝trackable÷已回收；回收率＝已回收÷應回收；無法填答率＝unable÷已回收 |
| 次數變化 | 每月門診／急診＝`monthlyUse.closing − first`（只有優化計畫）；再住院＝`ipdAfter − ipdBefore`；身心科＝`psyOpdAfter − opdBefore`；居家照護科＝`homeOpdAfter − opdBefore`；都是各個案相減後平均 |
| 醫療機構端（V9） | 專屬區分母＝優化計畫中 `opened=true`（收案）；轉介率分子＝`closing.referrals` 非空，四項（固定：醫療資源、社區支持資源、社區心理衛生中心、EIP計畫，對不到的代碼接在後面）各自÷同一分母；失聯／拒訪＝`closing.lostContact`／`refused`；無法收案＝`opened=false` 依原因文字分組。共同區依通報類別 |
| SNQ | 日期區間涵蓋到的月份（`YYYY-MM` 比較）加總分子、分母再相除；轉介卡依類別顯示 |
| 同期比較（隱藏中） | 近三年（`asOf` 的年度往回兩年）相同期間：同期累計 1/1～asOf 月日、同月比較本月 1 日～asOf 日、其他月份整月（本月到 asOf 日，未來月份本年度「尚未發生」）；每年各算一次 `effMetrics`。恢復時要另外處理起始日以前的年份（目前會算成 0 件） |
| 指揮板 | 四類（緊急醫療、第一～三類）各自的狀態件數；類別件數占總通報、各狀態占該類別 |

---

## 4. 版面修正（prod 沒有的）

都在 `<style id="qa-data">`。後兩項是 v9 原型就有的問題（prod 沒有這兩頁），更新到 prod 時跟著 `index.html` 一起過去。

- 通報者頁：真實資料是七大網絡＋「其他」共 8 個單位。≥1200px 圓餅圖例 8 項以上改兩欄（原本單欄會撐高整列、連圓餅底部的百分比都切掉）；900–1199px 長條間距與最小寬度縮小，第 8 根才放得下。**prod 也有這兩個問題，還沒修。**
- 成效頁「改善情況」卡片：≥1200px 卡片高度約＝(視窗高 − 324px) ÷ 2，原型的直式排列（圓環在上、改善／持平／變差在下）在卡片內容高度約 220px 以下時，圓環會蓋住「持平」，再矮連中心百分比都被切（1536×730、1366×657、1280×720）。
  卡片內容高度 < 230px 時以 `@container outcome-card` 改橫式：圓環在左、三項數字在右直排。container query 改不到卡片本身，所以卡片格線在 ≥1200px 一律兩欄（第 2 欄 `auto`），直式時三個子元素都橫跨兩欄、版面跟原本相同。
  橫式時圓環按鈕自己是 size container（`outcome-donut`），圓環上限 150px，切換點前後大小差不多（150 → 143px）。
- 效率頁「優化計畫衛生局處置效率」：原型讓下方「暫不轉介原因分布／衛生局處理時間」只吃剩餘高度，視窗矮於約 834px 就被壓扁（1536×730 原因表只露出 1 列、1366×657 整列看不到）。
  改成下方至少 236px，放不下時整個面板內捲動；1536×864 以上版面不變。

---

## 5. 驗證

- 本機伺服器見根目錄 CLAUDE.md，開 `http://127.0.0.1:8771/rapid-response-platform-dashboard/qa/`。
- 範例資料：`uv run --no-project python -I .tool/qa_sample_data.py`（repo 根目錄執行，覆寫 `qa/data.json`、`qa/data-platform.json`）。
- 測別的資料不要改 `qa/` 的兩個檔：Playwright `page.route('**/qa/data.json', …)`／`'**/qa/data-platform.json'` 攔截。
  2026-11-09 驗過：範例資料、`data.template.jsonc`（含註解）、prod 的正式資料（schemaVersion 1）、`data-platform.json` 404。
- 檢查：頁首狀態列、各頁數字、通報類別與日期切換、各 dialog；8 個尺寸（1920×1080 … 320×640）沒有比 v9 多出的文字裁切；console 只能有本機的 favicon 404。
- 刻意用捲動的區塊：矮視窗的衛生局原因表、醫院端「本院處理時效」、成效頁評估表面板、指揮板同期比較表，以及手機上的衛生局原因表（表格最小寬度 540px，左右捲動）。
  它們一個方向是 `hidden`、另一個方向是 `auto`，只看祖先有沒有 `hidden` 的裁切檢查會把捲動範圍外的內容誤判成被切掉；要另外比對 `scrollHeight`／`scrollWidth` 與 `clientHeight`／`clientWidth`，確認捲得到。
  2026-11-09 驗過：扣掉這些捲動區，只剩 320×640 基本資料的性別圖例（「177 (46.83%)」）與 SNQ 卡片的人數各被切 12–19px（v9 相同，還沒修）。

---

## 6. 更新到 prod 的檢查清單

1. 平台專案能穩定產生 v2 的 `data.json` 與 `data-platform.json`（先推到 `qa/` 驗證過）。
2. `qa/index.html` 複製成 `prod/index.html`：拿掉標題的「｜QA」與檔頭 qa 註解；檔名不用改（同樣讀同資料夾的 `data.json`、`data-platform.json`）。
3. `qa/DATA_CONTRACT.md` 與兩個範本搬到 `prod/`，路徑 `qa/` 改 `prod/`，取代 prod 舊的規格；`prod/CLAUDE.md` 改寫成這份的內容。
4. 平台專案改成把 v2 推到 `prod/`；`qa/` 的範例檔視需要保留作為下一輪驗證用。
5. 若 AI 頁、同期比較仍要隱藏，照最上方的作法保留。
