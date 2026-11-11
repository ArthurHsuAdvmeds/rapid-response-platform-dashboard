# qa 資料交付規格（給平台專案）

qa 版儀表板（本 repo 的 `qa/`）要拿到什麼資料、放在哪裡、怎麼放上來。qa 驗證通過後，prod 會換成這一版，這份規格也跟著取代 [prod/DATA_CONTRACT.md](../prod/DATA_CONTRACT.md)。
平台專案怎麼撈資料不在這裡。

- 儀表板網址：`https://arthurhsuadvmeds.github.io/rapid-response-platform-dashboard/qa/`
- **兩個檔**，都放在 `qa/`、與 `qa/index.html` 同層：

  | 檔案 | 內容 | 格式版本 | 欄位範本 |
  | --- | --- | --- | --- |
  | `qa/data.json` | 緊急醫療 `emg[]`、優化計畫 `opt[]`，一筆通報一個物件 | `schemaVersion: 2` | [data.template.jsonc](data.template.jsonc) |
  | `qa/data-platform.json` | 醫院內控指標（SNQ）`snq`、平台使用率 `usage` 的彙總數字 | `schemaVersion: 1` | [data-platform.template.jsonc](data-platform.template.jsonc) |

- **qa 目前隱藏醫院內控指標與平台使用率，不讀 `data-platform.json`**（使用者 2026-11-11 指示）：平台專案可以先只產生 `data.json`，`data-platform.json` 的格式（第 4 節）保留，恢復時照用。
- **每個欄位的意義、型別、代碼都寫在兩個範本的註解裡，以範本為準**；這份只寫規則、代碼表、欄位與指標的對應、待確認事項。
- 目前 `qa/` 裡的兩個檔是虛構的範例資料（最外層 `"sample": true`，頁首顯示「範例資料（虛構）」），平台專案產生正式檔後直接覆蓋。
- 標 ★ 的是還沒確認的地方，問題集中在第 6 節。

---

## 0. 開始排程前要先決定：repo 與網站都是公開的

與 prod 相同（見 [prod/DATA_CONTRACT.md](../prod/DATA_CONTRACT.md) 第 0 節）：repo 是 public，Pages 沒有登入機制，推上來的檔案任何人都能下載，也永久留在 git 歷史。
qa 的資料比 prod 多（優化計畫、量表分數、HIS 比對結果），影響更大。明文照推、加密後推、或換有權限控管的託管，**請在 qa 接正式資料前決定**。

---

## 1. 需要平台專案做的事

1. **兩個檔各自是全量快照**：每次都整份重產、整份覆寫，不做增量、不跟舊檔合併。
2. **每次執行都更新兩個檔的 `updatedAt` 並 push**（就算內容沒變）。儀表板以 `data.json` 的 `updatedAt` 當「今天」，距今超過 48 小時頁首會提示「資料已 N 天未更新」。
3. **兩個檔可以分開產生、分開推**（例如 SNQ 由另一支程式每週跑）。`data-platform.json` 讀不到或格式錯，只有醫院內控指標與平台使用率顯示「尚無資料」，頁首提示，其他頁照常。
4. **push 前先檢查，不過就不要推**（保留上一版、發告警）：
   - 標準 JSON（不要註解、不要結尾逗號）；`schemaVersion` 正確；`emg`、`opt` 是陣列；
   - 每筆都有 `reportId`（emg 與 opt 之間也不能重複）與可解析的 `timePoint.reportedAt`；`opt` 每筆都有 `optType`（1／2／3）；
   - 筆數不是 0，也沒有比上一版明顯變少（門檻由平台專案決定）。
5. **只動 `qa/data.json`、`qa/data-platform.json`**（prod 期間另外照 prod 的規格產生 `prod/data.json`，見第 8 節），不改其他檔。
6. **git**：push 前 `git pull --rebase`，被拒絕就重來，不要 force push；commit message 固定格式，例如 `qa data 2026-11-09 06:00（緊急醫療 406、優化計畫 782 筆）`、`qa platform data 2026-11-09 06:00`。
   用只對這個 repo 有寫入權的 deploy key；平台網址、帳密、token、log 一律不進 repo。
7. **推完怎麼確認**：Actions 綠燈後打開 qa 儀表板，頁首的資料更新時間、案件數（緊急醫療／優化計畫）符合預期，沒有黃色提示。

---

## 2. 檔案怎麼放

- UTF-8（不要 BOM）、LF 換行；建議一筆通報一行、依 `timePoint.reportedAt` 排序、欄位順序照範本，git diff 才只會出現有變動的那幾筆。
- 大小：v2 每筆約 0.9 KB，每年約 400 筆 ≈ 0.35 MB，十年內都不用拆。
- **為什麼是這兩個檔**：同一支程式、同一個來源（平台的通報資料）放同一個檔，用 `emg`／`opt` 分類；
  SNQ 與平台使用率另外抓、更新頻率可能不同，所以另開 `data-platform.json`（使用者指定）。不依實體拆（人、量表、時間點、代碼表各一個檔只會多出 join）。
  之後若有第三個來源，比照 `data-platform.json` 另開一個檔，外框同樣有 `schemaVersion`、`updatedAt`。

---

## 3. data.json（schemaVersion 2）

欄位逐一說明見 [data.template.jsonc](data.template.jsonc)。重點：

- **最外層**：`schemaVersion`（2）、`updatedAt`（帶 `+08:00`）、`emg`、`opt`。`asOf`、`sample` 只給測試用，平台不要填。
- **平台起始日固定 2026/10/05**，寫在儀表板裡（不用給 `dataStart`）：日期最早可選這天，本季／本年度與指揮板預設區間從這天起，年度結案率從 2026 年列起。
  早於這天的通報不列入任何頁面，頁首提示件數；正式資料不應該有。
- **emg**：prod 版（schemaVersion 1）的欄位全部沿用，新增 `timePoint.closedAt` 與下面的共用區塊。
- **opt**：共用欄位與 emg 同名同代碼（`reportId`、`reporterDepartment`、`caseInfo`、`assess`、`timePoint.reportedAt`／`assignedAt`／`closedAt`、`status`、`StatisfactionAvg`），
  另有 `optType`、`timePoint.bureauRepliedAt`／`reviewedAt`／`hospitalRepliedAt`／`firstVisitAt`、`bureau`、`review`、`hospital`、`monthlyUse`、`closing`。
- **共用區塊**（emg、opt 都有，還沒有資料的整塊給 `null`）：`scores`（量表原始分數：初評、結案後 3／6 個月）、`followUp`（追蹤量表填寫狀況）、`his`（HIS 比對結果）、`ai`（AI 預測）。
- 時間 `YYYY-MM-DDTHH:mm:ss`，不帶時區＝臺北時間；`bureau.visitDate`、`bureau.dispatchDate` 可以只有日期（只有日期時以通報日起算的天數計，同一天＝0）。

### 3.1 代碼表

| 欄位 | 代碼 |
| --- | --- |
| `reporterDepartment` | ★ 暫定 `0` 社政、`1` 警察、`2` 民政、`3` 衛政、`4` 教育、`5` 消防、`6` 勞政、`8` 其他；`7` 未對應 |
| `caseInfo.gender` | `10` 男、`20` 女，其他算不明 |
| `assess.*` | `0` 輕度／低風險、`1` 中、`2` 重度／高風險、`3` 其他（不列入）。`assess.CTRS` 同時是初評風險 ★ |
| `acceptWay`（emg） | `0` 電話、`1` 電話＋出勤 |
| `status` | `0` 未結案（再依 `assignedAt` 分待派／已派）、`1` 已結案、`2` 未轉介／開案（例如轉介松德院區、暫不轉介、無法收案） |
| `optType` | `1` 第一類、`2` 第二類、`3` 第三類優化計畫 |
| `bureau.result` | `1` 轉介三總北投、`2` 轉介松德院區、`3` 暫不轉介。衛生局不會轉介他院（V9 追加） |
| `review.result` | `1` 同意、`2` 駁回 |
| `hospital.opened` | `true` 開案、`false` 無法收案 |
| `hospital.unableReason` | 回覆單上的選項文字，照原樣顯示。原本的「未轉介 / 轉介他院」請給「未轉介」（V9 追加） |
| `closing.referrals` | 醫療資源轉介勾選的選項代碼陣列：`1` 醫療資源、`2` 社區支持資源、`3` 社區心理衛生中心、`4` EIP計畫；沒有轉介 `[]`。對不到的代碼頁首提示 |
| `followUp.m3`／`m6` | `null` 還沒到應回收時間、`"notReturned"` 應回收未回收、`"trackable"` 回收且有分數、`"unable"` 回收但勾選無法填答 |

量表等級切點（平台只給分數時換算用）：CGI-S 1–3 輕度、4 中度、5–7 重度；BSRS-5 0–9、10–14、15–24；GAF 61–100、41–60、1–40，其他數字給 `3`。★ 與客戶原始清單的切點不同，見第 6 節。

---

## 4. data-platform.json（schemaVersion 1）

欄位見 [data-platform.template.jsonc](data-platform.template.jsonc)。

- **`usage.units[]`**：每個單位的總帳號數 `total`、帳號建立後曾登入過的帳號數 `loggedIn`。統計到 `updatedAt` 為止，不受日期篩選影響；顯示時依使用率排序。只放彙總，不放帳號明細。
- **`snq.months[]`**：每月一列，各指標的分子 `num`／分母 `den`，以及兩類通報的結案後轉介（`den`、`any`、`opd`、`home`、`ipd`）。
  儀表板把日期區間涵蓋到的月份**加總分子、加總分母再相除**，所以兩者都必須是可以相加的人數；「月份」是個案進入該指標母群的月份 ★。只放彙總，不放個案。

---

## 5. 不可放進來的資料

與 prod 相同：姓名、身分證字號、出生日期（只給年齡）、電話、地址文字、病歷號、帳號、通報人與醫護人員姓名、任何自由文字（備註、描述、處置紀錄）都不放。
HIS 的身分證比對在平台專案做完，檔案裡只放比對後的次數；座標建議四捨五入到小數 3 位 ★；測試帳號建立的、已作廢的通報不要放。

---

## 6. 待確認

| # | 問題 | 確認前儀表板的做法 |
| --- | --- | --- |
| 1 | `reporterDepartment` 0–8 各是哪個單位（「0-7 為七大網絡」但只有 7 個網絡） | 暫依舊資料庫規劃的順序，7 未對應 |
| 2 | 初評風險是不是 `assess.CTRS` | 用 CTRS（效率頁總覽、地圖顏色） |
| 3 | 能不能提供 `caseInfo.county` | 沒給時，對得到臺北市 12 區的算臺北市 |
| 4 | 「未完成處遇即失聯／拒訪」怎麼認定 | 平台專案判斷後填 `closing.lostContact`／`refused`，儀表板照旗標計數 |
| 5 | 醫療機構端的轉介率、失聯率、拒訪率，日期區間以收案日、結案日還是通報日為準 | 跟其他頁一樣以通報日；所以剛通報、還沒結案的期間會是 0% |
| 6 | SNQ 各指標的「月份」是初診、出院還是結案的月份；「X 年中每年均穩定」的 X | 照 `snq.months` 的月份加總 |
| 7 | HIS「結案後」次數的統計期間（結案後多久） | 照平台給的次數 |
| 8 | 量表等級的切點：使用者提供的（第 3.1 節）與客戶原始清單不同（GAF 51–60 中度、BSRS-5 6–9 輕度、CGI-S 2–3 輕度） | 照資料給的等級 |
| 9 | 滿意度「應回收」的定義（問卷結案當日發送、3 日內填寫） | 已結案或已有分數＝應回收 |
| 10 | `reportId` 是否唯一、不會變 | 重複照算，頁首提示 |
| 11 | 作廢、測試的通報是否已排除 | 照收 |

已確認（V9 追加，2026-11-09）：醫療資源轉介四個選項的名稱（見 3.1 `closing.referrals`）；衛生局醫療轉介率的分母＝已完成回覆單；衛生局只會轉介三總北投或松德院區；平台起始日 2026/10/05。

---

## 7. 欄位與頁面的對應

平台專案可依此決定先接哪些資料；沒有資料的部分顯示「—」或「尚無資料」，不會算錯。

| 頁面 | 用到的欄位 |
| --- | --- |
| 基本資料（含熱點地圖） | `reportedAt`、`status`、`assignedAt`、`caseInfo`、`assess.CTRS` |
| 通報案件風險分析 | `assess` |
| 案件處理效率指標｜總覽 | `assess.CTRS`；emg `acceptedAt`、opt `firstVisitAt` |
| ｜緊急醫療處置效率 | emg `acceptWay`、`assignedAt`、`acceptedAt`、`outsideAt`、`status` |
| ｜優化計畫醫院端 | opt `bureau.result`、`review.result`、`hospital.opened`、`hospital.contactAt`、`timePoint.bureauRepliedAt`／`reviewedAt`／`assignedAt`／`firstVisitAt`／`hospitalRepliedAt` |
| ｜優化計畫衛生局 | opt `bureau`（`result`、`noReferralReason`、`visitDate`、`dispatchDate`）、`timePoint.bureauRepliedAt` |
| 精神症狀改善成效指標 | `scores`、`followUp`、`his`；opt `monthlyUse` |
| 通報者基本資料分析 | `reporterDepartment`、`status`、`StatisfactionAvg` |
| 醫療機構端處理成效分析 | opt `hospital`、`closing`、`monthlyUse`；`his` |
| 醫院內控指標(SNQ) | `data-platform.json` 的 `snq`（**qa 目前隱藏這頁**，不讀這個檔） |
| 指揮儀表板｜指揮板、結案率 | `reportedAt`、`status`、`assignedAt`、`optType` |
| ｜同期比較 | 效率頁的所有欄位（近三年相同期間；平台 2026/10/05 以前的年份顯示「平台未啟用」，不需要補舊資料） |
| ｜平台使用率 | `data-platform.json` 的 `usage`（**qa 目前隱藏這個子頁籤**，不讀這個檔） |
| AI模型預測準確率 | `ai`（**qa 目前隱藏這頁**，資料可以先給） |

---

## 8. 版本與上線順序

- **qa 驗證期間**：prod 照舊讀 `prod/data.json`（schemaVersion 1，只有 emg）。平台專案產生 v2 後，把 `emg` 的 v1 欄位挑出來、`schemaVersion` 寫 1，就是 prod 要的檔。
- **qa 通過後**：qa 的版面與資料層換到 prod，prod 改讀 v2 的兩個檔；這份規格取代 prod 版。
- 之後改格式：新增選填欄位不用改版號；改名、刪欄位、改代碼意義 → `schemaVersion` 加 1，先讓儀表板同時支援新舊版本並上線，平台再切換。每次都先改範本與這份文件。
- v2 還沒有正式產出過，所以 V9 追加（2026-11-09）直接改在 v2、不加版號：`bureau.result` 的 `2` 改為轉介松德院區、`closing.referrals` 改放代碼、拿掉 `dataStart` 與 `options.referral`。
