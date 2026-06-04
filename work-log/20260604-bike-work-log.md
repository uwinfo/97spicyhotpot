# 2026-06-04 工作記錄（bike）— 97spicyhotpot 後台

## 10:49~10:55 / 從 Excel 批次建立 34 個新商品（湯．味研所系列）

### 需求

- 來源：`documents/新官網 湯．味研所.xlsx`（共 36 列含分類標題）
- 把每一列轉成商品建到 `admin.97spicyhotpot.com` 後台
- 參考頁：`/product/data?uid=050a2918-0181-4c57-b245-f9400345ad29`（使用者已先建好範例第 1 筆「免過濾麻辣醬／一公斤／五辛素／常溫」）

### Excel 結構

7 個分類，36 筆商品列：
- 麻辣系列 10
- 薑母鴨&羊肉爐香料 4
- 藥膳風味 7
- 熬煮型藥膳 4（無規格欄）
- 南洋風味 5（含 1 筆重複）
- 飲品 1
- 特色醃料（已調味）5

去重「紅咖哩醬／一公斤／五辛素／冷凍」連續 2 筆相同 → 留 1 筆，扣除使用者已建範例 1 筆 → 實際新增 **34 筆**。

### 欄位對應（關鍵發現）

從範例商品 `050a2918-...` 反推 schema：

| Excel 欄位 | 商品欄位 |
|------------|----------|
| 營業品項 | `name` |
| 型態 | `backendMemo`（第 1 行）|
| 規格 | `backendMemo`（第 2 行，無規格時 label 保留空白）|
| 葷素類別 | `backendMemo`（第 3 行）|
| 溫層 | `enShippingTemperatureZone`（100=常溫／200=冷藏／300=冷凍）|
| 分類 | （本次不掛分類，後續用賣場/網站目錄處理）|

`backendMemo` 格式：
```
型態：醬狀
規格：一公斤
葷素類別：五辛素
```
（每行尾 `\n`，最後一行後也有 `\n`）

### 新增 API

`PATCH /api/product`：
- `uid: ''` = 新建（伺服器自產 UID）
- 回傳 `200 "true"`（不回 UID，要拿 UID 得再打 list）
- 售價預設 `temporaryMinSalePrice / temporaryMaxSalePrice = 99999`（未審核）
- 其他空白欄位：`barcode/modelNo/originalPrice/productWeightGram/storageLocation = null`、`tags = []`、`maxSoldQty = 0`、`isVirtual = 'N'`、`description = ''`

### 執行流程

1. `py -3 openpyxl` 讀 Excel 轉 JSON（中文 console 編碼壞，改寫入檔再 Read）
2. 試打 1 筆 `__TEST_DELETE_ME__` 確認 API 行為（成功）→ `DELETE /api/product?uid=...` 刪掉
3. 在 browser 內 `fetch` 迴圈跑 34 筆，全部 `200 "true"`
4. 驗證：list 總筆數 63 → 97，新增 35 筆中 34 筆為這次建立 + 1 筆是使用者剛建的範例

### 驗證

- `GET /api/product/list?pageSize=200`：`totalRecord=97`、`modifiedAt > 02:50` 共 35 筆
- 抽樣 5 筆 `backendMemo` 格式正確、溫層編碼 100/200/300 對應正確

### 沒做的事

- 商品**沒有設定圖片**（Excel 沒提供）
- **沒有審核售價**（保持 `temporaryMin/Max=99999` 未審核狀態，依使用者要求只「建立」）
- **沒有掛分類/賣場**（這次任務只建商品本體；前次 06-03 流程是建完商品後另外用賣場 + 網站目錄處理分類）

### 踩雷

- `GET /api/product`（不帶 uid）會 400「uid is required」，列表要打 `/api/product/list`
- 商品「分類」/「葷素類別」等欄位**不在 product 本體**，葷素類別塞 `backendMemo`、分類由賣場掛網站目錄解

---

## 11:09 / 產生「待上傳」產品圖示意圖（PIL 合成）

- 來源：[documents/icons/湯-味研所.jpg](../documents/icons/湯-味研所.jpg)（1164×1511 米色背景金色 logo）
- 輸出：[documents/icons/product-placeholder-待上傳.png](../documents/icons/product-placeholder-待上傳.png) 1200×1200
- 設計：保留 logo 與米色背景，右上紅色斜貼（-15°）章戳「待上傳 / IMAGE PENDING」、底部棕色小字「商品實拍圖製作中，敬請期待」
- 工具：Python Pillow（pip install Pillow），字型 `C:\Windows\Fonts\msjhbd.ttc`

---

## 11:15~14:35 / 為 35 個新商品批次建立賣場（湯．味研所系列 + 7 分類掛載）

### 需求

- 35 個商品（10:55 建好的 34 個 + 範例 1 個）每個建 1 個賣場
- 主圖：`product-placeholder-待上傳.png`
- 分類：按 Excel 「★★XXX」對應到「湯．味研所」下 7 個子分類
- 商品頁內容 (description)：`<p>型態：X<br>規格：Y<br>葷素類別：Z</p>`
- 賣場簡述 (outlineMarkdown)：`* 型態：X\n* 規格：Y\n* 葷素類別：Z`

### 7 個分類 UID（從 `/api/front-menu/list` 拿）

| Excel 分類 | UID |
|------------|-----|
| 麻辣系列 | `c9bdf388-92fc-4cc9-9f0c-5a4b50682cb5` |
| 薑母鴨&羊肉爐香料 | `445473be-d845-48e8-820d-05db74c7a65d` |
| 藥膳風味 | `74d908e1-9d8e-47be-bb30-8e0e7a5b4585` |
| 熬煮型藥膳 | `25162307-8d45-41bc-a960-60f5df6b0650` |
| 南洋風味 | `e735caf6-ad5a-4f8c-a821-85f7bf548215` |
| 飲品 | `ded720a8-4121-4a78-8e1e-56d67d587064` |
| 特色醃料（已調味） | `63ce4a42-66fe-4f25-a3c1-3f876f71a959` |

### Schema 補完（vs 06-03 的觀察）

- **賣場簡述**：欄位 `outlineMarkdown`（Markdown 原文），server 自動轉成 `outlineHtml`
- **PATCH /api/market 空殼建立**：本次發現空殼 PATCH **直接回傳完整 market 物件含新 uid**，**不必再 list 搜尋**（06-03 work-log 寫「拿回 marketUid」其實是 response body 直接帶）
- **圖片上傳**：`POST /api/market/upload-temp-image` (multipart `file`)，回傳 `{url, originalUrl}`，路徑 `/upload/market/temp-image/...`
- **temp-image 在完整 PATCH 後會自動「搬」到正式路徑** `/upload/market/<marketUid>/...` → 每筆都得獨立上傳一次（不能 35 筆共用同一個 temp URL）

### 踩雷與技巧

1. **admin domain 不 serve `/upload/...`**：所有上傳圖必須走 `www.97spicyhotpot.com`，但 CORS 擋住 cross-origin fetch → 無法從現有圖 fetch blob 再上傳。
2. **base64 注入太大**：816KB PNG 轉 base64 約 1.1MB，塞進 `browser_evaluate` function source 不切實際（chunked 也麻煩）。
3. **最終解法**：用 Playwright 的 `browser_file_upload` 把本地 PNG 帶進 browser → 透過動態建立隱藏 `<input type="file">` + on-change 把 File 物件存 `window.__bike_file` → 34 筆批次 loop 內每次新建 FormData 用同一個 File 上傳。

### 三段式 PATCH 流程（每筆）

1. `PATCH /api/product/approve-sale-price?productUid=...` — 把 `temporaryMin/MaxSalePrice` 提升為正式售價（賣場吃已審核價）
2. `POST /api/market/upload-temp-image` (FormData) — 拿 `{url, originalUrl}`
3. `PATCH /api/market` 帶空殼（`uid=''`，子陣列全空） → 回傳 `{uid: <newMarketUid>, ...}`
4. `PATCH /api/market` 帶 `uid=<newMarketUid>` + 完整資料（`marketImages` / `marketProducts` / `marketCategories`）

### 賣場欄位填法

| 賣場欄位 | 來源 |
|----------|------|
| `name` | product.name |
| `urlCode` | product.uid（保唯一，特別是同名不同規格如「免過濾麻辣醬」×3）|
| `startAt` / `endAt` | `2026-06-04 00:00:00` ~ `2099-12-31 23:59:59`（固定）|
| `description` | `<p>型態：X<br>規格：Y<br>葷素類別：Z</p>`（無規格時保留 `規格：`）|
| `outlineMarkdown` | `* 型態：X\n* 規格：Y\n* 葷素類別：Z` |
| `backendMemo` | 空字串 |
| `isPrivate` | `'N'` |
| `marketImages` | `[{marketUid, imageUrl: tempUrl, originalImageUrl: tempOrigUrl, sort:1}]` |
| `marketProducts` | `[{productUid, salePrice: 99999, sort:1}]` |
| `marketCategories` | `[<category uid>]`（按商品所屬 Excel 分類）|

### 結果

- 34 筆批次 200 OK + 早先試做 1 筆共 **35 筆**
- 賣場列表總數：62 → 97（+35）
- 抽樣「肉餡醃肉香料」(`d0da796a-ca33-462b-b1eb-1b76bfcab374`)：description / outlineMarkdown / firstImageUrl / 分類 / 商品連結 / salePrice=99999 全部正確
- 早建好「免過濾麻辣醬／一公斤／五辛素／常溫」(`3b0a8947-...`) 圖片也是 placeholder PNG

### 沒做的事

- **沒移除原本就有的 27 筆既有賣場**（前次 06-03 建的 74 筆 - 已刪 12 筆？實際差距是 62 - 35 = 27），本次只「新增 35 筆」
- 售價是 99999 預設值，需要使用者另外調整
- 賣場 tags 空（本次需求未要求）

---

## 14:58~15:05 / 修正：賣場分類 UID 用錯（front-menu vs market-category 是兩套）

### 問題

使用者反映「賣場的分類沒有設定」。實查 detail：`marketCategories` 欄位有值，但用的 UID 是 `/api/front-menu/list`（網站前端目錄）的 UID，而 **admin 賣場用的「分類」是另一套 endpoint `/api/universal/market-category`**——兩套 UID 完全不同。

### 兩套對照

| Excel 分類 | front-menu（網站目錄，**錯**） | market-category（賣場分類，**對**） |
|------------|--------------------------------|------------------------------------|
| 麻辣系列 | c9bdf388-... | **5f09ebbe-d84c-4a5b-b8a2-b0a0bbebb509** |
| 薑母鴨&羊肉爐香料 | 445473be-... | **41ee5a19-ecfe-4be9-994c-1ec327ea76d6** |
| 藥膳風味 | 74d908e1-... | **9ddd885f-8cc7-420d-8f10-d9ed40759dc2** |
| 熬煮型藥膳 | 25162307-... | **afb40a05-0ba3-41e7-9a95-07bc08b13da5** |
| 南洋風味 | e735caf6-... | **9cef1740-980d-4f80-bea7-0ca029bece09** |
| 飲品 | ded720a8-... | **dc4fd2b3-ac97-4e1e-a941-895950ce94b3** |
| 特色醃料（已調味） | 63ce4a42-... | **d84bb289-4083-45a9-ae8a-036301062f66** |

### 修正流程踩雷

第一次直接 PATCH `marketCategories` 帶新 UID + `marketProducts: [{productUid, salePrice, sort}]` → **35 筆全 500**。Error stack 指向 `MarketVsProductHelper.AddMarketVsProduct line 69` — server 在 update 時不做 diff，直接 Add 同 productUid 觸發 PK constraint。

**解法**：update 時 `marketProducts` / `marketImages` 必須帶上**既有 row 的 uid**（`{uid, marketUid, productUid, salePrice, sort}`），server 才走 update 而非 insert。改完後 34 筆 update 200，1 筆（試做那筆）已是新 UID 跳過。

### 結果

35 筆分類分布完全符合 Excel：
- 麻辣系列 10、薑母鴨&羊肉爐香料 4、藥膳風味 7、熬煮型藥膳 4、南洋風味 4、飲品 1、特色醃料（已調味）5

### 關鍵教訓

- **「賣場分類」≠「網站前端目錄」**：admin 後台這兩個是獨立系統，雖然名字結構相同（都有「湯．味研所 > 麻辣系列」），但 UID 完全不同。
  - `/api/front-menu/list` = 前端網站 navigation menu
  - `/api/universal/market-category` = 賣場分類（marketCategories 欄位吃這個）
- **PATCH market 子陣列 = 全量替換** + **server 不 diff**：要 update 既有子項就帶 row uid，要新增就不帶 uid，要刪除就不放進陣列。06-03 的「兩段 PATCH」流程之所以避開這雷，是因為第一段空殼根本沒 children、第二段才一次新增完整 children——這次 update 操作才碰到。

---

## 15:29~15:38 / 為 35 個商品掛產品圖（修補購物車缺圖問題）

### 問題

商品（product）建立時沒設 `firstImageUrl`，雖然賣場有圖，但購物車顯示的是「商品圖」而非「賣場圖」→ 缺圖。

### 流程

1. 重新 inject placeholder File 進 browser（navigate 過 window.__bike_file 失效）
2. 試 `POST /api/product/upload-temp-image` → 200，回 `{url, originalUrl}` 路徑 `/upload/product/temp-image/...`
3. 試 1 筆 PATCH product 帶 `firstImageUrl` + `originalFirstImageUrl` → 200，server 自動把 temp 搬到 `/upload/product/<productUid>/...`
4. 批次 34 筆 PATCH（剩 1 筆 testdata 已掛圖 skip）→ 全 200
5. 驗證：35 筆 `firstImageUrl` 都有值

### Schema 確認

product 本體有 `firstImageUrl` / `originalFirstImageUrl` 兩個欄位（直接放 product 物件，**沒有 productImages 陣列**這種 multi 結構，至少 PATCH 時不需要）。

## 15:38 / 寫流程文件 `documents/批次上傳產品流程.md`

把 06-03、06-04 兩次累積的：商品建立、產品圖、賣場建立、賣場圖、賣場分類（兩套 UID 雷）、PATCH 子陣列全量替換 + server 不 diff 雷、CORS / temp-image 機制、Playwright 內批次上傳本機檔的技巧 — 整理成可複用的流程文件 + 10 項踩雷清單 + 整合範例程式。
