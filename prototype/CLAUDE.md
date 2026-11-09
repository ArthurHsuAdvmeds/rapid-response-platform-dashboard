# prototype — 儀表板原型 v1–v8（模擬資料）

每一版都是一份可獨立開啟的單檔 HTML（D3／topojson 走 CDN），數字是寫死的模擬資料。
2026-11 從 advmeds-prototype 的 `tsghb-dashboard/` 原樣搬來，網址改為 `…/rapid-response-platform-dashboard/prototype/`。
通用規則（不可編譯、相對路徑、大檔怎麼讀、CSS 權重、本機驗證）見根目錄 [CLAUDE.md](../CLAUDE.md)。

---

## 1. 資料夾地圖

| 路徑 | 內容 |
| --- | --- |
| `index.html` | 轉址頁：導到最新版（`LATEST = 8`），沒改也會往後探測 3 版 |
| `v1/`–`v8/` | 各版原型，全部並存、可個別開啟；**最新是 `v8/`** |
| `release-note/` | V2 以後每版相對前一版的差異與點選路徑（給客戶會議對照） |

正式版 `../prod/` 以 v7 為基礎改接真實資料，跟這裡的版號分開走。

---

## 2. 版本沿革

| 版本 | 日期 | 相對前一版 |
| --- | --- | --- |
| v1 | — | 最早的原型（約 2.5 MB），版本說明不涵蓋 |
| v2 | 2026-09-06 | **基準版本**：九個頁面的架構到此定型，之後沒有再增刪主頁面（約 2.7 MB） |
| v3 | 2026-09-06～08 | 改動最大：效率頁整頁重做成四頁籤、全站補上 RWD、區域分布圖由面量圖改為個案 GPS 點位；單檔壓到約 0.8 MB |
| v4 | 2026-09-25 | 會議修改意見：緊醫處理流程依受理方式拆兩張、效率頁籤與圖表位置對調、指標定義對齊官方清單、區域分布圖加初評風險過濾 |
| v5 | 2026-09-25 | 點位加大並合併同址、字級以 1920×1080 為基準放大、模擬資料補到 2026/09/25、SNQ 轉介比例 dialog、醫療機構端無法收案原因收成卡片＋dialog、同期比較改比效率指標並可篩通報類別 |
| v6 | 2026-09-29 | 手機 RWD（≤899px）：主選單改下拉、子頁籤換行、長條圖改橫向、指揮板直式、熱點地圖觸控；可點擊卡片的提示改按鈕 |
| v7 | 2026-10-16 | 各頁過濾器按鈕文字放大（版面不位移），修兩處手機過濾器排版 |
| v8 | 2026-11-05 | 指揮儀表板新增「平台使用率」子頁籤；修正結案率圓環數字超出內圈 |

---

## 3. 頁面結構（v8）

主選單九頁（`.tabs` 內的 `.tab[data-page]`，對應 `<section class="page" id="…">`）：

| 頁面 id | 名稱 | 子頁籤／重點 |
| --- | --- | --- |
| `basic` | 基本資料 | 五張數值卡、行政區、年齡、性別、案件熱點地圖（彈窗 `#districtMapDialog`） |
| `risk` | 通報案件風險分析 | 四種量表風險占比 |
| `efficiency` | 案件處理效率指標 | `data-eff-view`：總覽／緊急醫療處置效率／優化計畫醫院端處置效率／優化計畫衛生局處置效率 |
| `outcome` | 精神症狀改善成效指標 | `data-outcome-tab`：改善情況／評估表填寫狀況／後續醫療資源使用狀況（明細彈窗 `#outcomeDetailDialog`） |
| `ai` | AI模型預測準確率 | 唯一沒有過濾器的頁 |
| `reporter` | 通報者基本資料分析 | 轉介單位占比、七大網絡轉介數、滿意度回收率／分數 |
| `hospital` | 醫療機構端處理成效分析 | 無法收案原因卡片＋dialog |
| `snq` | 醫院內控指標(SNQ) | 結案後轉介比例 dialog |
| `command` | 指揮儀表板 | `data-command-view`：指揮板（`flow`）／結案率（`closure`）／同期比較（`comparison`）／平台使用率（`usage`，v8） |

- 各頁大多有時間篩選（開始／結束日期、本日／本月／本季／本年度）與通報類別（全部／緊急醫療／優化計畫）。
- 模擬資料的「今天」是 **2026-09-25**，本年度總數 420 件；數字多半直接寫在 HTML 或各段腳本的常數裡，**沒有共用資料層**，改一個數字不會連動其他頁，要一致得自己對齊。
- 七大網絡的名稱與順序沿用 `reporterNetworkData`：民政、社政、消防、教育、勞政、警察、衛政。

---

## 4. 各版補丁的注意事項（v5–v7 的層 prod 也有）

**v5**
- 修改都在檔尾 `<style id="v5-…">` 與地圖、SNQ 的 JS。字級層 `v5-typography-1920` 只在 ≥1200px 生效。

**v6（手機 RWD）**
- 主選單下拉的選項由 `.tabs` 內按鈕自動產生，切頁一律觸發對應 `.tab` 的 click。**新增的控制項不可帶 `tab` class**，否則會被 `querySelectorAll('.tab')` 當成頁籤。
- 長條圖在手機上改橫向：圖表只寫行內 `height:%`，由 `v6-mobile-rwd-behavior` 同步成 `--v6-bar`（MutationObserver 監看 `#basic`、`#reporter`）。**新增直條圖要把元素加進 `BAR_SELECTOR`**，否則手機上長條寬度是 0。
- 熱點地圖：`geoCanvasBox()` 在 ≤899px 依地圖框比例決定畫布高度；未放大時單指觸控不交給 d3（`geoZoomBehavior.filter`），讓手指能捲動彈窗。
- `v84-responsive-release` 在 ≤1199px 用 `[class*="-shell"]` 把頁面內所有 `*-shell` 的 `max-height` 清成 none，會波及放在頁面裡的彈窗外框；新彈窗的外框避免用 `-shell` 結尾，或比照補回。

**v7（過濾器字級）**
- 只有檔尾 `<style id="v7-filter-text">`。字級集中在 `:root` 的 `--v7-filter-btn`／`-input`／`-label`／`-cap`／`--v7-geo-btn`：
  ≥1200px 為 `clamp(15px,1.05vw,16px)`；900–1199px 與 ≤899px 為 15px；≤899px 的日期欄位維持 16px，避免 iOS 聚焦時放大整頁。
- **新增過濾器時要把容器 id 加進該層對應的清單**。
- 熱點地圖工具列在 >820px 是固定高度，換行會被切掉；初評風險鈕在 821–1920px 必須維持一列。

**v8（平台使用率）**
- 子頁籤 `data-command-view="usage"`，內容是 `#commandUsageView`，程式在檔尾 `<style id="v8-platform-usage">`／`<script id="v8-platform-usage-behavior">`。
- 指標定義：平台使用率＝帳號建立後曾登入過的帳號數 ÷ 平台總帳號數（從未登入的帳號算在分母）。統計至 2026/09/25，不受日期篩選影響。
- 數字由腳本裡的 `USAGE_UNITS` 算出（模擬資料，七大網絡＋醫療機構共 8 個單位，依使用率高到低排序）。單位分類待確認。
- 版面直接套用「結案率」的 `closure-v74-*` class，只補了舊層以 `#commandClosureView` 指名的容器規則，並把清單列數改由 `--usage-rows` 決定（結案率各層都寫死 4 列）。≤899px 每列改成兩行（單位＋使用率／長條）。
- 圓環中間的數字改依圓環寬度縮放（`container-type:inline-size` ＋ `19cqi`），結案率與平台使用率共用。

**還沒帶進 v8 的修正**（prod 的 `<style id="real-data">` 有）：效率頁處理流程的「小時」和數字重疊（`.eff-v79-flow-seg`）；≥900px 通報者頁長條數值壓到上緣（`.reporter-bar-chart` 上方留白）。

---

## 5. 發佈新版的檢查清單

1. `cp vN/index.html vN+1/index.html`，改 `<title>` 與 `<!-- Prototype Version: vN -->`，**新的修改集中在檔尾的 `<style id="vN-…">`／`<script id="vN-…">`**，不回頭改前面的層。
2. 轉址頁 `index.html`：`LATEST`、canonical、`<noscript>`、手動連結、檔頭註解的版本列表都改成新版（沒改也會自動往後探測 3 版，但多一次請求；file:// 下要連到 `vN/index.html`）。
3. `release-note/index.html`：頁首版本入口加一筆並把「目前版本」移過去，再新增一個 `<section class="release" id="vN">`。
   每項變更寫說明與點選路徑（`crumb nav` 是主選單，`crumb tab` 是子頁籤）；標籤用 `tag new`／`fix`／`move`／`rename`／`pending`。
4. 增刪指標時同步 `../kpi/kpi.csv`（見根目錄 CLAUDE.md）。
5. 更新本文件的版本沿革與注意事項。
6. 驗證：新功能在各尺寸都不爆版；其他頁和上一版逐一比對元素位置相同；console 無錯誤。
