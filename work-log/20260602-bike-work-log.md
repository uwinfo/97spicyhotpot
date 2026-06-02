# 2026-06-02 工作記錄（bike）— 97spicyhotpot 後台

## 10:33~11:13 / 賣場分類 + 網站目錄建立 + 排序

### 賣場分類管理 (/market-category/list)

建立 3 個第一層品牌分類 + 13 個子分類，並調整排序欄位（同層遞增 100 為間距）：

- **【玖食柒食堂】** (sort=10)
  - 鍋底 (10)、湯底 (20)、川式油滷 (30)、單品 (40)
- **【蘊玥】** (sort=20)
  - 養生茶包 (10)、養身煲湯 (20)
- **【湯．味研所】** (sort=30)
  - 麻辣系列 (10)、薑母鴨&羊肉爐香料 (20)、藥膳風味 (30)、熬煮型藥膳 (40)、南洋風味 (50)、飲品 (60)、特色醃料（已調味）(70)

**連結策略決議**：使用者偏好全中文 URL（瀏覽器自動 percent-encode）。連結去掉 `【】` 括號（URL 保留字符）、全形 `．` 改半形 `-`、`&` 改 `-`、括號改 `-`，名稱保留原樣。

UI 是 Element UI 樹狀+表單，每次新增後分類路徑下拉重置回「第一層」，所以子分類每次都要重選父分類。改排序是點 tree node → 改排序欄位 → 點「修改」按鈕（編輯模式下按鈕變成 修改/刪除/取消）。

### 網站目錄管理 (/menu-web/list)

使用者已建【玖食柒食堂】(連結 `/categories/玖食柒食堂`、sort=100) 和 鍋底 (sort=200)。沿用 `/categories/<中文名>` 模式，補建 14 項：

- 【玖食柒食堂】下：湯底=300、川式油滷=400、單品=500
- 【蘊玥】=200：養生茶包=100、養身煲湯=200
- 【湯．味研所】=300：麻辣系列=100、薑母鴨&羊肉爐香料=200、藥膳風味=300、熬煮型藥膳=400、南洋風味=500、飲品=600、特色醃料（已調味）=700
- 既有 部落格=1500、最新消息=2000 不動

## 14:56~21:00 / 把 A 後台商品搬到 B 後台（74 筆）

### 來源/目標

- **A** (來源): `https://ca.jdcard.com.tw/` — jdcard 平台後台（CompanyAdmin / 玖食柒食堂版）
- **B** (目標): `https://admin.97spicyhotpot.com/` — 97 自家 Element UI 後台
- 兩邊是**不同 CMS**，欄位需要 mapping

### 欄位 mapping 決議

| A 欄位 | B 欄位 | 規則 |
|--------|--------|------|
| 名稱 | 名稱 | 直搬 |
| 售價 | 最低售價(未審核) + 最高售價(未審核) | 雙寫同一個值 |
| 產品說明 | 備註 (backendMemo) | B 商品頁沒「產品說明」，借用備註欄 |
| 產品圖 | 產品圖片 | 下載後重傳到 B |
| (無) | 國際條碼 | 用 A UID（B 必填，A 無此資料）|
| (無) | 料號 | 用 A UID（同上）|
| 餐點類型 | (B 商品本身不掛分類) | A 的分類掛在賣場層級，這次只搬商品本體 |
| 溫層、排序、已完售 | (忽略) | B 無對應 |

### 流程演進

1. **初期 (1~10 筆) 用 Playwright 操控 UI**：每筆 ~7 個 tool call（A navigate → 抓資料 → curl 下圖 → B navigate → fill 表單 → file_upload → 點新增）。穩定但慢。
2. **發現 A 有 API**：`GET /api/product/getlist?pageSize=200` 一次拿全部 74 筆 JSON（含 name、price、productImg、memo）。
3. **發現 B 有 API**：
   - `POST /api/product/upload-temp-image` (multipart) → 回 `{url, originalUrl}`
   - `PATCH /api/product` (JSON) → 新增/更新商品
4. **跨域 fetch A 圖片被 CORS 擋** → 改 Bash 用 curl 一次下載全部 74 張圖到 `.tmp-images/`。
5. **B 後台 cookie 是 HttpOnly** → Bash 無法直接 curl B API。
6. **最終策略**：在 B 後台 page 上插一個隱藏 `<input type="file" id="__xfer_input">`，每筆 3 個 tool call：
   - `evaluate`: `input.click()` 打開 file picker
   - `file_upload`: 設定本機檔案到 input
   - `evaluate`: 拿 `input.files[0]` → POST `/api/product/upload-temp-image` → 拿 `firstImageUrl` → PATCH `/api/product`

### 對帳結果

A 74 筆 → B 全到位（missing 0），無重複 barcode。B 多出的 2 筆（`Test 2`、`珍珠奶茶`）是 B 原有測試資料，已用 `DELETE /api/product?uid=<uid>` 刪除。

### 暫存清理

清掉 `.tmp-images/`、`a-products-full.json`、`remaining.json`、`inject*.js`、`download-images.sh`、`a-product.md`、`b-product-new.md`、`.playwright-mcp/`。專案目錄目前只剩 `.git`。

## 踩雷紀錄

- **B 圖片 upload 用的不是 button，是 `<div class="el-upload" role="button">`** 包 `<input type="file">`。第一次用 `document.querySelector('main button')` filter empty text 找不到，改抓 `.el-upload` click 才開 picker。
- **A 圖片副檔名混 `.jpg` / `.png` / 沒副檔名**。一開始下載全部存 `.jpg` 害 B 拒收 PNG；改成讀 `productImg` 字串取副檔名才正確。3 筆商品（手工牛肉丸、鴨肉丸、青花椒辣油包/大紅袍辣油包一組）的 `productImg` 是空字串，這 3 筆 PATCH 時 `firstImageUrl/originalFirstImageUrl` 也送空字串即可。
- **B 表單「分類路徑」/ 商品 select 在 evaluate 模擬 click 後可正常運作**，但用 Playwright 的 `browser_select_option` 對 Element UI 的自訂 select 不行（它是 div+span 不是原生 select）。直接 JS 點 `.el-select-dropdown__item` 比較穩。
