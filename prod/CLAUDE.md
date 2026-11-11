# prod — 正式版儀表板（真實資料）

三總北投精神健康個案通報管理儀表板的正式版。**2026-11-11 由 qa 同步**：版面是原型 v9 的全部頁面，畫面上的每個數字都由同資料夾的 `data.json`（schemaVersion 2）即時計算；
HTML 裡原型留下的數字只是佔位，載入完成前整頁隱藏。前一版（以 v7 為基礎、只有緊急醫療、schemaVersion 1）在 git 歷史裡。

- **資料格式與平台專案要做的事以 [DATA_CONTRACT.md](DATA_CONTRACT.md) 為準，欄位以 [data.template.jsonc](data.template.jsonc) 為準。** 本文件只記儀表板端。
- `data.json` 是平台的正式資料（2026-11-11 使用者從平台取得，與當時的 `qa/data.json` 相同），之後由平台專案整份覆寫並 push。**不要手動改、不要 commit 測試資料進去**；測別的資料見第 5 節。
- repo 與 Pages 都是公開的，資料怎麼保護還沒決定：見 DATA_CONTRACT.md 第 0 節。
- **`index.html` 與 `qa/index.html` 只差檔頭兩行**（`<title>` 沒有「｜QA」、第 8 行的說明註解）。資料層沿用 qa 的名稱 `<style id="qa-data">`／`<script id="qa-data">`、`.qa-*` class，
  下一輪從 qa 同步時直接複製再改這兩行即可（第 6 節）。
- **不用 `data-platform.json`**（使用者 2026-11-11 指示），`prod/` 沒有放這個檔：AI模型預測準確率（`SHOW_AI=false`）、醫院內控指標(SNQ) 與指揮儀表板「平台使用率」（`SHOW_PLATFORM=false`）隱藏中，
  兩個都是 `false` 就不抓這個檔（不會 404、頁首不提示）。程式、HTML 都保留；恢復的作法見 [../qa/CLAUDE.md](../qa/CLAUDE.md) 最上方，先在 qa 恢復、驗證，再同步過來，並把 `data-platform.json` 的規格與範本一起搬到 prod。
- **平台起始日固定 2026/10/05**（`PLATFORM_START`），**指揮儀表板「同期比較」保留三年欄位，舊年份標示「平台未啟用」**：見第 3 節。
- 版面、疊層 CSS、各版補丁的注意事項見根目錄與 [../prototype/CLAUDE.md](../prototype/CLAUDE.md)；prod 包含 v9 以前的所有層。

---

## 1. 程式在哪

全部在檔尾 `<style id="qa-data">` 與 `<script id="qa-data">`（`window.RD`）。前面兩萬行是 v9 原型的版面與互動，數字不從那裡來。
從 v9 原型怎麼改出來（刪掉哪些模擬程式、補了哪些 HTML 掛點）見 [../qa/CLAUDE.md](../qa/CLAUDE.md) 第 1 節。

- **載入**：`loadFromServer()` 以 `cache:'no-store'` fetch `data.json`（`LOAD_PLATFORM` 為 `true` 時另外抓 `data-platform.json`，目前不抓）→ 標準 JSON 失敗時去掉註解與結尾逗號再試 → `build()`。
  `file://` 下 fetch 一定被擋，頁首顯示「選擇 data.json」改用選檔（只在瀏覽器內讀，不上傳；檔名不限）。
- **`build()`**：接受 `schemaVersion` 1（舊格式，只有 emg）與 2，其他版本整頁顯示錯誤；`normalizeCase(r,'emg'|'opt')` 轉成共用的案件物件；通報日早於 `PLATFORM_START` 的排除（計入 `RD.stats.beforeStart`）。
  **data.json 改欄位名或代碼時，只改 `normalizeCase()` 與檔頭代碼表**（`DEPARTMENT`、`NETWORKS`、`SCALE_ROW`、`GENDER`、`ACCEPT_WAY`、`COUNTIES`、`BUREAU_RESULT`：1 北投／2 松德／3 暫不轉介，`REFERRAL`：醫療資源轉介四項；`referralLabel()` 也接受直接寫選項名稱）。
- **案件物件**：`id`、`reported`（毫秒）、`day`（臺北日期）、`year`、`age`、`gender`（`M`／`F`／`U`）、`status`（`pending_assign`／`assigned`／`closed`／`not_referred`）、
  `risk`（CTRS 等級）、`network`、`county`、`district`、`lat`／`lng`、`assigned`、`levels`、`surveys`，另有
  `kind`（emg／opt）、`category`（emg／opt1／opt2／opt3）、`closed`、`scores`、`followUp`、`his`；
  emg：`intakeMethod`、`intake`、`dispatched`；opt：`bureauReplied`、`reviewed`、`hospitalReplied`、`firstVisit`、`bureauResult`、`noReferralReason`、`bureauVisitDays`、`bureauDecisionDays`、`reviewResult`、`opened`、`unableReason`、`contact`、`monthly`、`closing`。
- **篩選**：`PAGES` 設定各分頁的日期快速鈕、套用鈕、通報類別鈕；`render(ctx)` 拿到 `{ cases（日期內全部）, filtered（再依通報類別）, range, type: all／emg／opt }`。
  `presetRange()`、`setRange()`、`initDateInputs()` 都以 `PLATFORM_START` 為下限。同期比較另有 `compareState`（模式、月份、類別），不受指揮板日期影響。
- **`effMetrics(cases)`**：效率頁四個頁籤與同期比較共用，回傳的鍵名就是 HTML 的 `data-eff-value`。
- **各頁**：`renderBasic`、`renderRisk`、`renderEfficiency`、`renderOutcome`（改善情況、評估表填寫狀況、後續醫療資源、個案明細 dialog）、`renderReporter`、
  `renderHospital`（V9 兩區、無法收案 dialog、指標計算說明 dialog）、`renderCommandFlow`、`renderClosure`、`renderComparison`；隱藏中的 `renderSnq`、`renderUsage` 照常執行（畫成「尚無資料」）。
- **熱點地圖**在主腳本的地圖模組（`GEO_*`），用 `window.RD.basicCases()` 取基本資料頁目前區間的案件（`geoCases()`），以案件的 `county`／`district` 對圖資的 `COUNTYNAME`／`TOWNNAME`（台→臺）。圖資 `taiwan-atlas` 從 jsDelivr 載入。
- **頁首狀態列** `#rdStatus`（`renderStatus()`）：資料更新時間、總案件數（緊急醫療／優化計畫，「2026/10/05 起」），加上黃色提示（`updatedAt` 距今超過 48 小時、早於起始日已排除、代碼對不到（含醫療資源轉介）、reportId 重複…）。
  ≥900px 絕對定位在頁首右下角、不佔高度；≤899px 在版面流裡，多一行提示會把下面內容往下推一行。

---

## 2. 推導規則（★ 待確認，問題清單在 DATA_CONTRACT.md 第 6 節）

- **待派／已派**：`status = 0` 且 `assignedAt` 為 null＝待派，有值＝已派；`1`＝結案；`2`＝未轉介／開案。
- ★ **初評風險**取 `assess.CTRS`：效率頁總覽、地圖點位顏色與初評風險篩選都用它。
- ★ **reporterDepartment**：暫定對照見 DATA_CONTRACT.md 3.1。`8` 其他固定列在通報者頁四張圖、排在七大網絡之後；
  對不到的代碼或空值列為灰色的「代碼 N（未對應）」「未填」，排最後。
- **縣市／行政區**：`caseInfo.county` ＞ `dict` 開頭的縣市（用 `COUNTIES` 整張表比對，竹北市這類縣轄市不會被當成縣市）＞
  沒有縣市時，`dict` 對得到臺北市 12 區的算臺北市（短名補「區」），其餘有行政區的算外縣市；都沒有算「不詳」，地圖上也沒有點。
- ★ **滿意度**：問卷於結案時發出，所以「應回收」＝已結案（或已有分數）的案件，「已回收」＝`StatisfactionAvg` 有值；分數是各案件平均分再取平均。
- **今天**：`asOf` → `updatedAt` 的臺北日期 → 最晚一筆通報的日期。
- `assess` 代碼（0 輕度／1 中度／2 重度／3 其他）與 CGI-S、BSRS-5、GAF 的等級切點在 DATA_CONTRACT.md 3.1。

---

## 3. 指標怎麼算

所有分頁都先依該頁的日期區間（`reportedAt` 的臺北日期，含頭尾）篩出案件，再依通報類別；只有「結案率」不受日期篩選影響。
時間差為負（資料錯誤）或缺任一端時不列入平均；沒有樣本時顯示「—」。

| 範圍 | 算法 |
| --- | --- |
| 平台起始日 | 固定 2026/10/05（不讀 data.json 的 `dataStart`）：日期欄位最早可選、快速鈕的下限（2026 年的本季、本年度都是 10/05 起）、手動輸入更早的日期拉回這天、指揮板預設區間起點、年度結案率從 2026 年列起；早於這天的通報排除 |
| 基本資料 | 五張數值卡依第 2 節計數；臺北市 12 區（0 件也列出）由多到少，有外縣市、不詳時多列，總和＝總收案件數；年齡 0-10 表示 0–9 歲…80+，空值另列「不詳」；性別不明 > 0 才出現在圖例；地圖點位只畫有座標的案件 |
| 風險分析 | 該區間案件的 `assess` 各等級人次占比（三級合計 100%） |
| 效率頁總覽 | 依初評風險：緊醫 `acceptedAt`、優化計畫 `firstVisitAt` 減 `reportedAt`，以「天」計，兩類一起平均 |
| 緊急醫療處置效率 | 結案率／電話處置率／出勤處置率的分母＝`acceptWay` 為 0 或 1 的案件；處理流程各段平均（小時）只取該流程每個時間點都有、且先後順序正確的案件，所以各段相加＝總反應時間 |
| 優化計畫醫院端 | 開案率＝轉介三總北投中 `opened=true` ÷ 轉介三總北投（回覆單還沒填的也在分母）；覆核同意／駁回率、醫療介入率的分母＝暫不轉介；覆核時間＝`reviewedAt − bureauRepliedAt`（天）；派案、首次聯繫以小時，首次專業評估、回覆衛生局以天；轉介案件首次聯繫＝`hospital.contactAt − reportedAt`、派案後首次聯繫＝`contactAt − assignedAt`，第一～三類都算（`contactAt`＝第一份評估訪視紀錄單的填寫時間，使用者 2026-11-11 定義）；轉介案件派案時間、覆核時間等用衛生局回覆單的只有第一／二類 |
| 優化計畫衛生局 | 分母＝有 `bureau.result` 的案件（已完成回覆單）；醫療轉介率＝(北投＋松德)÷已完成回覆單；本院占率＝北投÷(北投＋松德)；訪視／分流決策時間用回覆單日期與通報日相差的天數 |
| 成效頁改善情況 | 改善率＝(初評−追蹤)÷初評（GAF 反過來），> +1% 改善、< −1% 變差；N＝初評與該期追蹤分數都有的個案 |
| 成效頁評估表 | 應回收＝`followUp` 有值；已回收＝trackable＋unable；可追蹤率＝trackable÷已回收；回收率＝已回收÷應回收；無法填答率＝unable÷已回收 |
| 次數變化 | 每月門診／急診＝`monthlyUse.closing − first`（只有優化計畫）；再住院＝`ipdAfter − ipdBefore`；身心科＝`psyOpdAfter − opdBefore`；居家照護科＝`homeOpdAfter − opdBefore`；都是各個案相減後平均 |
| 通報者 | 占比、轉介數量依 `reporterDepartment` 計數；回收率＝已回收÷應回收；滿意度＝已回收案件的 `StatisfactionAvg` 平均 |
| 醫療機構端（V9） | 專屬區分母＝優化計畫中 `opened=true`（收案）；轉介率分子＝`closing.referrals` 非空，四項（固定：醫療資源、社區支持資源、社區心理衛生中心、EIP計畫，對不到的代碼接在後面）各自÷同一分母；失聯／拒訪＝`closing.lostContact`／`refused`；無法收案＝`opened=false` 依原因文字分組。共同區依通報類別 |
| 指揮板 | 四類（緊急醫療、第一～三類）各自的狀態件數；類別件數占總通報、各狀態占該類別；預設區間 `PLATFORM_START`～`asOf` |
| 結案率 | 依通報年度：已結案件數 ÷ 該年度通報件數，從 2026 年列到 `asOf` 的年度 |
| 同期比較 | 近三年（`asOf` 的年度往回兩年）相同期間：同期累計 1/1～asOf 月日、同月比較本月 1 日～asOf 日、其他月份整月（本月到 asOf 日，未來月份本年度「尚未發生」）；每年各算一次 `effMetrics`。`comparePeriods()` 依 `PLATFORM_START` 標記：期間結束日早於起始日＝`disabled`（不計算，表頭「平台未啟用」、數字與較去年「—」、底色灰）；開始日早於、結束日不早於＝`partial`（照算，表頭「10/5 起」）。有任一欄是這兩種時，表格上方註明「平台 2026/10/05 啟用，之前沒有資料」 |

---

## 4. 版面修正（v9 原型沒有的）

都在 `<style id="qa-data">`，跟 qa 相同（詳細的尺寸與原因見 [../qa/CLAUDE.md](../qa/CLAUDE.md) 第 4 節）：

- 通報者頁：8 個單位時 ≥1200px 圓餅圖例改兩欄；900–1199px 長條間距與最小寬度縮小，第 8 根才放得下。
- 成效頁「改善情況」卡片：內容高度 < 230px 時以 `@container outcome-card` 改橫式（圓環在左、三項數字在右）。
- 效率頁「優化計畫衛生局處置效率」：下方「暫不轉介原因分布／衛生局處理時間」至少 236px，放不下時整個面板內捲動。
- 沿用舊 prod 的修正（v8／v9 原型也有這兩個問題）：效率頁流程的「小時」與數字重疊、通報者頁長條數值壓到軸標題。

---

## 5. 開啟、驗證、測別的資料

- **本機伺服器**：見根目錄 CLAUDE.md「本機驗證」，開 `http://127.0.0.1:8771/rapid-response-platform-dashboard/prod/`。
- **測別的資料不要動 `data.json`**：Playwright 用 `page.route('**/prod/data.json', …)` 攔截換成測試檔，或 file:// 選檔。
- 改程式或換資料後檢查：頁首「資料更新」、總案件數、黃色提示是否符合預期；console 只能有本機伺服器的 favicon 404；矮視窗刻意捲動的區塊見 qa/CLAUDE.md 第 5 節。
- 2026-11-11 同步後驗過：正式資料（緊急醫療 16、優化計畫 2 筆）在 1536×730、1920×1080、390×844 切換全部 14 個頁面與子頁籤，沒有 console 錯誤、沒有 NaN／undefined，沒有請求 `data-platform.json`。
- **要改格式（schemaVersion 加 1）時**：先改 DATA_CONTRACT.md 與範本，再讓 `build()` 同時接受新舊版本並上線，最後才讓平台切換。

---

## 6. 從 qa 同步

prod 不直接開發新功能：先在 `qa/` 改、驗證，再照這裡同步（2026-11-11 第一次）。

1. `qa/index.html` 複製成 `prod/index.html`，改回檔頭兩行（`<title>` 拿掉「｜QA」、第 8 行換成 prod 的說明）；兩檔其餘部分應該完全相同（`diff <(sed 1,8d qa/index.html) <(sed 1,8d prod/index.html)`）。
2. 格式有變時：`qa/data.template.jsonc` 複製成 `prod/data.template.jsonc`（檔頭改 prod 路徑），`DATA_CONTRACT.md` 對應的段落一起更新；平台專案改產生新格式前，儀表板要先上線。
3. `data.json` 由平台專案產生，同步時只有在使用者指示時才從 qa 複製（2026-11-11 是使用者指示複製的）。
4. 隱藏的 AI 頁、醫院內控指標、平台使用率要恢復時，`data-platform.json` 的規格（qa/DATA_CONTRACT.md 第 4 節）、範本與資料檔一起搬過來。
