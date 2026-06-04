# 2026-06-03 工作記錄（bike）— 97spicyhotpot 後台

## 06:47~07:08 / 為 74 個商品批次建立賣場（分類掛「【玖食柒食堂】> 單品」）

### 需求

- 每個商品建立一個對應賣場
- 分類 = 【玖食柒食堂】> 單品
- 賣場的「商品頁內容」(description) = 商品備註 (backendMemo)
- 賣場的「圖片管理」(marketImages) = 商品的產品圖

### B 後台賣場新增 API（透過試錯摸清楚）

`PATCH /api/market`（create + update 都用同一個 endpoint，PATCH 也用於 create）

**必填欄位**：
- `uid`: 空字串 = 新建（伺服器自產 UID）；傳已存在 UID = 更新；傳不存在 UID = 404 拒絕
- `name`, `urlCode`, `startAt`, `endAt`：基本資料
- `isPrivate`: `'N'`/`'Y'`（不是 bool，bool 會被「輸入值不在白名單裡」擋下）
- `marketTags`, `marketImages`, `marketProducts`, `marketCategories`：四個都必填，可為空陣列

**子陣列格式**：
- `marketCategories: ['<category uid>']` — 純字串陣列
- `marketProducts: [{ productUid, salePrice, sort }]` — 物件陣列
  - `salePrice` 必填，沒給會 `請填寫所有必填欄位`
  - 商品的 minSalePrice 必須**已審核**，否則回 `請先設定商品的最低售價`
- `marketImages: [{ marketUid, imageUrl, originalImageUrl, sort }]` — 物件陣列
  - `marketUid` 必填（建立當下 uid 還沒生成 → 無法在新建同步加圖 → 必須兩段 PATCH）

### 商品售價審核

`PATCH /api/product/approve-sale-price?productUid=<uid>` — 把 `temporaryMinSalePrice`/`temporaryMaxSalePrice` 提升為已審核的 `minSalePrice`/`maxSalePrice`。沒審核的商品不能掛到賣場。

### 最終流程（每筆 3 calls）

1. `PATCH /api/product/approve-sale-price?productUid=<p.uid>`
2. `PATCH /api/market` 帶空殼（uid='', 全部子陣列空） → 拿回 `marketUid`
3. `PATCH /api/market` 帶 `uid=marketUid` + 完整資料（含 marketImages、marketProducts、marketCategories）

74 筆全部在一個 evaluate 內用 async for-loop 跑完，0 錯誤。

### 賣場欄位填法

| 賣場欄位 | 來源 |
|----------|------|
| name | product.name |
| urlCode | product.uid（保證唯一）|
| startAt / endAt | 2026-06-03 ~ 2099-12-31（固定）|
| description | `<p>` + 換行→`<br>` 的 product.backendMemo + `</p>` |
| isPrivate | 'N' |
| marketCategories | `['5746393c-09ab-41c2-867e-7b4839288c42']`（單品 uid）|
| marketProducts | `[{ productUid, salePrice: product.minSalePrice ?? product.temporaryMinSalePrice, sort: 1 }]` |
| marketImages | `[{ marketUid, imageUrl: product.firstImageUrl, originalImageUrl: product.originalFirstImageUrl, sort: 1 }]` 或空陣列（3 筆無圖）|

### 清理

刪除 B 原有 2 筆測試賣場「好吃的鍋底」、「賣場一」（`DELETE /api/market?uid=<uid>`，回 200 "true"）。最終賣場列表剛好 74 筆。

### 踩雷

- `marketCategories` 是 **string array**，但 `marketProducts` / `marketImages` 是 **object array**。第一次以為都一樣→ JSON parse 錯誤揭示 dto 名稱 `UpsertMarketVsProduct`。
- 客戶端自產 UID 想一次 PATCH 完成 → 404「查無賣場資料」。確認 PATCH 嚴格區分新建（uid=''）vs 更新（uid=已存在）。
- 商品需先審核售價：B 後台商品有 `temporaryMinSalePrice`（未審核）和 `minSalePrice`（已審核）兩套價格，賣場吃已審核的。
