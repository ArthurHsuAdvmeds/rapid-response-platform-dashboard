# prod — 正式版儀表板（真實資料）

三總北投精神健康個案通報管理儀表板的正式版。版面沿用原型 v7，只保留緊急醫療（基本資料／風險／效率／通報者／指揮儀表板），
畫面上的每個數字都由 `data/data.json` 即時計算；HTML 裡原型留下的數字只是佔位，載入完成前整頁隱藏。
前身是 advmeds-prototype 的 `tsghb-dashboard/real-data/20261103/`，2026-11 搬來這裡，資料檔改放 `data/`。

- **`data/data.json` 的格式、排程要遵守的事、待確認問題，全部以 [DATA_CONTRACT.md](DATA_CONTRACT.md) 為準。** 本文件只記儀表板端。
- `data/` 屬於排程主機（另一個專案），它定期全量產生 data.json 並 push。**不要手動改、不要 commit 測試資料進去**；測別的資料見第 4 節。
- repo 與 Pages 都是公開的，上線前要先決定資料怎麼保護：見 DATA_CONTRACT.md 第 0 節。
- 目前的 `data/data.json` 是使用者提供的 14 筆（通報日 2026/10/05–11/03），還是舊格式（沒有 `schemaVersion`、`updatedAt` 只有日期）；儀表板相容，排程第一次執行就會換成新格式。

---

## 1. 程式在哪

全部集中在 `index.html` 檔尾的 `<style id="real-data">` 與 `<script id="real-data">`（`window.RD`）。前面兩萬行是 v7 原型的版面與互動，數字不從那裡來。

- **載入**：`loadFromServer()` 以 `cache:'no-store'` fetch `DATA_FILE`（`data/data.json`）→ `parseJsonLoose()`（標準 JSON 失敗時去掉註解與結尾逗號再試）→ `start()`。
  `file://` 下 fetch 一定被擋，直接顯示「選擇 data.json」（`showError(null)` → `loadFromPicker()`）。
- **`build()`**：`schemaVersion` 不等於 `SCHEMA_VERSION`（目前 1，沒有這個鍵視為 1）就丟錯、整頁不畫 → 每筆 `normalizeCase()` →
  資料品質計數 `RD.stats`（`badTime` 排除、`badStatus`、`unmappedDept`、`dupId`）→ 決定 `updatedAt`／`asOf`／`dataStart`。
- **`normalizeCase()`** 把一筆通報轉成畫面共用的案件物件，各頁繪製函式只讀這個物件。
  **data.json 改欄位名或代碼時，只改 `normalizeCase()` 與檔頭的代碼表**（`DEPARTMENT`、`NETWORKS`、`SCALE_ROW`、`GENDER`、`ACCEPT_WAY`、`COUNTIES`）。
  案件物件：`id`、`reported`（毫秒）、`day`（臺北日期）、`year`、`age`、`gender`（`M`／`F`／`U`）、`status`（`pending_assign`／`assigned`／`closed`／`not_referred`）、
  `risk`（CTRS 等級）、`network`、`county`、`district`、`lat`／`lng`、`assigned`、`intakeMethod`、`intake`、`dispatched`、`levels`、`surveys`。
- **日期篩選**：`PAGES` 是五個分頁篩選器的設定；`setRange()` → `casesIn()` → 各頁 `render*()`。結案率 `renderClosure()` 不受篩選影響。
- **熱點地圖**在主腳本的地圖模組（`GEO_*`），用 `window.RD.basicCases()` 取基本資料頁目前區間的案件（`geoCases()`），
  以案件的 `county`／`district` 對圖資的 `COUNTYNAME`／`TOWNNAME`（台→臺）。圖資 `taiwan-atlas` 從 jsDelivr 載入。
- **頁首狀態列** `#rdStatus`（`renderStatus()`）：資料更新時間、總案件數，加上黃色提示（已排除、status 無法對應、reporterDepartment 未對應、
  reportId 重複、`updatedAt` 距今超過 `STALE_AFTER_HOURS`＝48 小時）。≥900px 絕對定位在頁首右下角、不佔高度；
  ≤899px 在版面流裡，多一行提示會把下面內容往下推一行（正常更新時只有一行）。
- `<style id="real-data">` 另外修了 v7 原有的幾個版面問題：效率頁流程的「小時」與數字重疊、通報者頁長條數值壓到標題、
  結案率圓環數字超出內圈、通報類別篩選只剩一顆鈕時的寬度。改版面前先讀 [../prototype/CLAUDE.md](../prototype/CLAUDE.md) 的 v5–v7 注意事項與根目錄 CLAUDE.md 的 CSS 權重規則。

---

## 2. 推導規則（★ 待確認，問題清單在 DATA_CONTRACT.md 第 5 節）

- **待派／已派**：`status = 0` 且 `assignedAt` 為 null＝待派，有值＝已派；`1`＝結案；`2`＝未轉介／開案。
- ★ **初評風險**取 `assess.CTRS`：效率頁總覽「不同風險個案處理時間」、地圖點位顏色與初評風險篩選都用它。
- ★ **reporterDepartment**：暫定對照見 DATA_CONTRACT.md 3.3。`8` 其他固定列在通報者頁四張圖、排在七大網絡之後；
  對不到的代碼或空值列為灰色的「代碼 N（未對應）」「未填」，排最後。
- **縣市／行政區**：`caseInfo.county` ＞ `dict` 開頭的縣市（用 `COUNTIES` 整張表比對，竹北市這類縣轄市不會被當成縣市）＞
  沒有縣市時，`dict` 對得到臺北市 12 區的算臺北市（短名補「區」），其餘有行政區的算外縣市。
  地圖全臺層級只有帶縣市的案件有數字；排程補上 `county` 後其他縣市也會出現。
- ★ **滿意度**：問卷於結案時發出，所以「應回收」＝已結案（或已有分數）的案件，「已回收」＝`StatisfactionAvg` 有值；分數是各案件平均分再取平均。
- **今天**：`asOf` → `updatedAt` 的臺北日期 → 最晚一筆通報的日期。
- `assess` 代碼（0 輕度／1 中度／2 重度／3 其他）與 CGI-S、BSRS-5、GAF 的等級切點（使用者提供）在 DATA_CONTRACT.md 3.3。

---

## 3. 指標怎麼算

所有分頁都先依該頁的日期區間（`reportedAt` 的臺北日期，含頭尾）篩出案件；只有「結案率」不受日期篩選影響。

| 分頁 | 指標 | 算法 |
| --- | --- | --- |
| 基本資料 | 五張數值卡 | 案件數；待派／已派／結案／未轉介 依上方推導規則計數 |
| | 臺北市各行政區 | 臺北市 12 區（0 件也列出）由多到少；有外縣市、不詳的案件時多列「外縣市」「不詳」，總和＝總收案件數 |
| | 年齡 | 0-10 表示 0–9 歲…80+；空值另列「不詳」 |
| | 性別 | 男／女／不明（不明 > 0 才出現在圖例） |
| | 熱點地圖 | 縣市內各區件數＋初評風險組成；點位只畫有座標的案件，沒有座標的件數寫在圖例 |
| 風險分析 | 四種量表 | 該區間案件的 `assess` 各等級人次占比（三級合計 100%） |
| 效率：處置結果 | 結案率／電話處置率／出勤處置率 | 分母＝`acceptWay` 為 0 或 1 的案件；分子分別為已結案、0、1 |
| 效率：處理流程 | 各段平均（小時） | 電話流程用 `acceptWay = 0`、出勤流程用 `1` 的案件，**只取該流程每個時間點都有、且先後順序正確的案件**，所以各段平均相加＝總反應時間；樣本數在流程區塊的滑鼠提示 |
| 效率：總覽 | 高／中／低風險處理時間 | 依 CTRS 等級分組，`acceptedAt − reportedAt` 平均（小時），N＝兩個時間都有的案件數 |
| | 摘要卡 | 結案率、出勤總反應時間（同上） |
| 通報者 | 占比、轉介數量 | 依 `reporterDepartment` 計數 |
| | 回收率 | 已回收 ÷ 應回收（見上方滿意度規則） |
| | 滿意度分數 | 已回收案件的 `StatisfactionAvg` 平均 |
| 指揮板 | 各卡片 | 只有緊急醫療一類，左側總數＝緊急醫療列；預設區間為 `dataStart`～`asOf` |
| 結案率 | 各年度 | 依通報年度：已結案件數 ÷ 該年度通報件數，從 `dataStart` 的年度列到 `asOf` 的年度 |

時間差為負（資料錯誤）或缺任一端時不列入平均；沒有樣本時顯示「—」。

---

## 4. 開啟、驗證、測別的資料

- **file:// 直接開檔**：頁首出現「選擇 data.json」，選了就載入（只在瀏覽器內讀，不上傳；檔名不限）。
- **本機伺服器**：見根目錄 CLAUDE.md「本機驗證」，開 `http://127.0.0.1:8771/rapid-response-platform-dashboard/prod/`。
- **測別的資料不要動 `data/`**：Playwright 用 `page.route('**/data/data.json', …)` 攔截換成測試檔，或 file:// 選檔。
- 改程式或換資料後檢查：頁首「資料更新」、總案件數、黃色提示是否符合預期；console 只能有本機伺服器的 favicon 404。
- **要改格式（schemaVersion 加 1）時**：先改 DATA_CONTRACT.md，再讓 `build()` 同時接受新舊版本並上線，最後才讓排程切換。

## 5. 之後擴充

- **優化計畫**：`build()` 只讀 `emg`；指揮板 `renderCommand()` 只有緊急醫療一列。被移除的頁面與頁籤可以從 `../prototype/v7/` 對照取回。
- **另一個資料檔**（例如 `data/his.json`）：`loadFromServer()` 改成一起載入，缺檔或失敗的區塊顯示「尚無資料」，各檔的 `updatedAt` 分別顯示。
