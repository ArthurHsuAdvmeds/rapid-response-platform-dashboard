# rapid-response-platform-dashboard

三軍總醫院北投分院「精神健康個案通報管理儀表板」。2026-11 從 advmeds-prototype 的 `tsghb-dashboard/` 獨立出來：
`prod/` 是接平台真實資料的正式版，`prototype/` 是 v1–v8 原型（模擬資料）。

全部是靜態檔，push 到 `main` 後由 GitHub Actions（`.github/workflows/pages.yml`）把整個 repo 原樣部署到 GitHub Pages，**沒有 build、套件管理或測試流程**。

- 網址：`https://arthurhsuadvmeds.github.io/rapid-response-platform-dashboard/`（根目錄轉址到 `prod/`）
- 原型：`…/prototype/`（轉址到最新版）、版本說明 `…/prototype/release-note/`

> **repo 是公開的（2026-11-09 確認），Pages 網站也沒有登入機制。** 推上來的東西任何人都看得到，也永久留在 git 歷史。
> API Key、Token、帳密、內網 IP 一律不可進 repo。`prod/data.json` 的真實資料要怎麼保護，上線前要先決定：見 [prod/DATA_CONTRACT.md](prod/DATA_CONTRACT.md) 第 0 節。

---

## 1. 資料夾地圖

| 路徑 | 內容 | 進版控 | 文件 |
| --- | --- | --- | --- |
| `index.html` | 轉址到 `prod/` | ✓ | — |
| `prod/` | 正式版：`index.html`（以原型 v7 為基礎，只保留緊急醫療，數字由資料即時計算）＋ `data.json`（排程主機定期 push） | ✓ | [prod/CLAUDE.md](prod/CLAUDE.md)、[prod/DATA_CONTRACT.md](prod/DATA_CONTRACT.md) |
| `prototype/` | `v1/`–`v9/` 原型（最新 `v9/`）、`release-note/`、轉址頁 `index.html` | ✓ | [prototype/CLAUDE.md](prototype/CLAUDE.md) |
| `V9調整.md` | 客戶對 v9「醫療機構端處理成效分析」的調整清單（v9 的需求來源） | 使用者決定 | — |
| `kpi/` | `kpi.csv` 指標清單、`original_list.xlsx` 客戶原始清單。**只留本機**，不要 `git add -f` | ✗（`.gitignore`） | 第 6 節 |
| `.github/workflows/pages.yml` | Pages 部署 | ✓ | **不要改**，除非使用者明確要求 |

**動手前先讀該資料夾的 CLAUDE.md。** advmeds-prototype 的資料庫規劃 `tsghb-dashboard/db_desc/` 沒有搬過來，需要時到那個 repo 看。

---

## 2. 資料與排程（誰動哪些檔）

- `prod/data.json` 屬於排程主機上的另一個專案：它定期從平台撈全部緊急醫療通報，**整份覆寫**這個檔後 push 到 `main`。
  排程要拿到什麼、怎麼放，全部寫在 [prod/DATA_CONTRACT.md](prod/DATA_CONTRACT.md)；本 repo 不記排程的實作。
- 開發時**不要手動改 `prod/data.json`、不要把測試資料 commit 進去**。測別的資料用 file:// 選檔或 Playwright 攔截（見 prod/CLAUDE.md 第 4 節）。
- 格式要改：先改 DATA_CONTRACT.md，儀表板先支援新格式並上線，排程再切換。
- 排程會一直推 `main`，所以**每次 push 前先 `git pull --rebase`**。

---

## 3. 開發規則（沿用 advmeds-prototype）

- **不能用需要編譯的寫法**（JSX、TypeScript、SCSS、npm 套件…），瀏覽器直接開得起來才算完成。
- **單檔優先**：CSS／JS 內嵌在 `index.html`；外部套件只走 `https://` CDN（目前 jsDelivr：d3、topojson-client、taiwan-atlas 圖資）。
- **一律用相對路徑**，不要寫 `/xxx/`，否則在 `/rapid-response-platform-dashboard/` 子路徑下會壞；也要讓 file:// 直接開檔能運作。每層網址都要有 `index.html`。
- 改完用瀏覽器實際開啟確認畫面，console 無錯誤。

### 大檔與疊層（prod 與 prototype 都適用）

- `prod/index.html` 約 0.77 MB、`prototype/v*/index.html` 約 0.8–2.7 MB，都是兩萬多行。**不要整份 Read**：
  先 Grep 區段（`<style id="v…">`、`<script id="v…">`、`<script id="real-data">`、常數 `GEO_*`／`EFF_*`／`OUTCOME_*`／`BASIC_*`）再局部讀。
  `prod/index.html` 的 `<header class="brand">`（約第 11680 行）有一行內嵌 base64 院徽的超長行，讀那附近一定要指定範圍。
- 檔案是逐版疊加的：後段的 `<style id="vNN-…">`／`<script id="vNN-…">` 覆寫前面的行為（`v41`～`v87` 是開發流水號，跟資料夾版本無關）。
  同一個選擇器常被五、六層 `!important` 覆寫，**最後一層才算數**，改之前先確認最後生效的是哪一段。
- **CSS 權重**：舊層最高到 `(2,4,0)`＋`!important`，新規則要壓得過才有效：
  - ≤899px 的規則一律以 `html body .dashboard #頁面.page …` 起頭（舊層是 `.dashboard .page:is(#…) :is(.…)`）。
  - 過濾器：`html body .dashboard .page.active:is(頁面 id)` 再接容器 id（如 `#riskTimeFilter`、`#reporterTypeFilter`）。
  - 彈窗在 `.dashboard` 外，用 `html body #彈窗id #容器`。
  - 指揮儀表板子頁籤：`html body .dashboard #command #檢視id …`。
- **各尺寸基準**：
  - ≥1200px：多數頁面高度鎖在視窗內（`calc(100vh - Npx)`、`overflow:hidden`），內容要縮得進去；以 125% 縮放的 **1536px** 視窗為主要驗證尺寸。
  - 900–1199px：平板，頁面依內容撐高。
  - ≤899px：手機版，主選單變黏頂下拉。
- 各版補丁（手機 RWD、過濾器字級…）的注意事項在 [prototype/CLAUDE.md](prototype/CLAUDE.md)，prod 包含 v7 以前的所有層，一樣適用。

---

## 4. Git 與部署

- 直接在 `main` 上工作；push 即部署（約 1 分鐘，`cancel-in-progress` 只保留最後一次），Actions →「Deploy GitHub Pages」綠燈才算上線。
- Commit message 用繁體中文，開頭標示資料夾，例如 `prod 頁首顯示資料更新時間`、`prototype v9 …`。排程的資料 commit 固定以 `prod data` 開頭。
- 環境是 Windows；沒有 `zip` 指令，打包用 PowerShell `System.IO.Compression`（zip 內路徑用 `/`）。
- 上線後看到舊畫面時先 `Ctrl` + `F5` 排除快取。

---

## 5. 本機驗證

- 環境沒有 Node；`python` 是 WindowsApps 的空殼。一律用 uv：
  - **本機伺服器**（在 repo 根目錄執行，伺服 repo 的上一層，網址才會跟 Pages 一樣帶 `/rapid-response-platform-dashboard/` 子路徑）：
    `uv run --no-project python -I -m http.server 8771 --bind 127.0.0.1 --directory ..`，開 `http://127.0.0.1:8771/rapid-response-platform-dashboard/prod/`。
    不用 8765：advmeds-prototype 常留著舊伺服器佔用它，Windows 允許兩個程序同時 listen 同一個 port，請求會跑到舊的那個。
  - **Playwright**（呼叫系統 Chrome，不用下載瀏覽器）：`uv run --no-project --with playwright python -I 腳本.py`，腳本裡 `p.chromium.launch(channel='chrome')`。
    腳本、截圖放 `.tool/`（已 gitignore）或系統暫存資料夾，不要放進 repo。
  - 終端機是 cp950，Python 印中文會亂碼；結果寫成 UTF-8 檔案再讀。
- ≤899px 主選單按鈕被藏起來，Playwright 要用 `page.evaluate(() => el.click())` 切頁，不能用 `locator.click()`。
- 常用尺寸：桌機 2560×1440、1920×1080、1536×864／730、1366×657、1280×720；平板 1024×768、820×1180；手機 390×844、360×740、320×640。
- 判斷爆版：文字超出 `overflow:hidden` 的祖先＝被切掉；內容底部超出視窗、頁面又不能捲動＝下緣被切；
  跟上一版比對時逐一比較各元素的 `getBoundingClientRect`（原型的 AI 頁有動畫，截圖 bytes 比對會有假差異）。
- 唯一可接受的 console 錯誤是本機伺服器的 `/favicon.ico` 404。

---

## 6. 指標清單 `kpi/`（只在使用者本機）

- `kpi.csv` 依 v6 畫面整理，150–153 是 v8 新增的平台使用率。UTF-8 含 BOM、CRLF（給 Excel 直接開）；檔尾有一列 `,,,,,` 空白列，新增的列插在它前面。
- 欄位：項次、主分頁、次分頁、指標名稱、圖表種類、重複項次、計算公式、是否依賴HIS資料。
- 「計算公式」以客戶原始清單 `original_list.xlsx` 為準，清單沒有的才依儀表板定義補。
  句尾【原清單無】＝儀表板有、原清單沒有；【差異】＝兩邊定義不一致，兩種都要跟客戶確認。xlsx 可用 Python 標準庫 `zipfile` 讀 `xl/sharedStrings.xml`。
- 「重複項次」由使用者維護；「是否依賴HIS資料」不依賴就留空。

---

## 7. 已知狀況

- **2026-11-05 實測**：從使用者目前的網路連 GitHub Pages，`185.199.109.153`、`185.199.110.153` 的 HTTPS 在 TLS 握手時被重設，`.108`、`.111` 正常。
  所有 `*.github.io` 都受影響，跟 repo 內容無關；瀏覽器出現「連線被重設」時先用 `curl --resolve 主機:443:IP` 逐個 IP 測。暫時解法：hosts 固定 `.108`，或改用手機熱點。
- advmeds-prototype 的 `tsghb-dashboard/` 還留著舊的各版與 `real-data/20261103/`（含 14 筆真實資料，已公開在那個 repo 的 Pages）；要不要移除由使用者決定。
