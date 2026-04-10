# Marriott Bonvoy 住宿追蹤 — 開發指南

## 概覽

一個單一頁面 PWA，追蹤 Marriott Bonvoy 年度住宿進度。支援兩個年度（2025 靜態回顧、2026 即時追蹤），透過 Firebase Realtime Database 同步資料，Google 帳號驗證管理員權限。

---

## 架構

### 檔案結構

```
index.html      — 整個 app（CSS + HTML + JS 全部內聯）
sw.js           — Service Worker（PWA 離線支援）
manifest.json   — PWA manifest
icon-192.png    — App 圖示
CLAUDE.md       — Git workflow 指引
GUIDELINE.md    — 本文件
```

### 為什麼全部放在 `index.html`？

App 設計為零構建工具，直接 GitHub Pages 部署。所有樣式和邏輯都內聯，避免需要 bundler 或 build step。

---

## 資料模型

### Firebase 路徑

| 路徑 | 內容 |
|------|------|
| `stays2026` | 2026 年住宿記錄（object，key 由 Firebase push 生成）|
| `config/pointsBase` | 積分基準值（number）|
| `config/pointsBaseStayIds` | 計入基準時已存在的住宿 key 陣列 |

### 住宿記錄結構

```js
{
  date: "2026-04-02",          // YYYY-MM-DD
  name: "广州天河合景福朋喜来登酒店", // 酒店全名
  brand: "Four Points",        // 必須符合 BRAND_KEYWORDS 內的品牌名
  nights: 1,                   // 1–30
  redeem: false,               // true = 積分兌換住宿
  redeemPoints: 0,             // 兌換消耗積分（redeem=true 時填）
  pointsEarned: 0              // 實際賺取積分（退房後填寫）
}
```

### 2025 資料

2025 年住宿係靜態硬編碼在 `index.html` 的 `STAYS_2025` 陣列，唔會寫入 Firebase，只有 App 更新先會改。

---

## 核心業務邏輯

### Q1 獎勵計算

- **活動期間**：`Q1_START = 2026-02-25`，`Q1_END = 2026-05-10`
- **條件**：非積分兌換（`redeem === false`）+ 在 Q1 期間內
- **獎勵**：每個品牌第一次 Q1 合資格住宿，獲 **+2,500 積分** bonus（等同該住宿的房晚數）
- **房晚計算**：`total = actualNights + q1Bonus`，用於計算達標進度

```js
function isQ1Eligible(stay) {
  if (stay.redeem) return false;
  const d = new Date(stay.date);
  return d >= Q1_START && d <= Q1_END;
}
```

`calcQ1(stays)` 按日期排序，每個品牌只計第一次，返回 `{ bonus, usedBrands }`。

### 積分計算

```
目前積分 = pointsBase + netChange

netChange = Σ 新住宿的 pointsEarned
           + Σ Q1 品牌首次住宿各 +2,500
           - Σ 兌換住宿的 redeemPoints
```

「新住宿」= 不在 `pointsBaseStayIds` 內的記錄。

### 品牌自動匹配

`BRAND_KEYWORDS` 陣列包含每個品牌的關鍵字（中英文），用戶輸入酒店名稱時自動匹配品牌。如需新增品牌關鍵字，在此陣列加入即可。

---

## 顏色語義

| 顏色 | CSS 變量 / 值 | 用途 |
|------|-------------|------|
| 金色 | `var(--gold)` | 主題色（標題、進度條、里程碑）|
| 綠色 | `var(--green)` | 所有「賺積分」正數、已達標狀態 |
| 紅色 | `var(--red)` | 所有「扣積分」負數 |
| 藍色 | `#0071e3` | 所有 Q1 相關標籤和說明 |
| 紫色 | `var(--purple)` 或 `#C084FC` | 目前積分餘額 |

住宿列表積分顯示格式：
- 普通住宿：`+4,069`（綠色）
- Q1 首次品牌（已填）：`+6,569`（綠色）`住宿 +4,069`（綠色小字）`· Q1 積分 +2,500`（藍色小字）
- Q1 首次品牌（未填）：`+?`（綠色）`+ Q1 積分 +2,500`（藍色小字）
- 兌換住宿：`−15,000`（紅色）`+500`（綠色）

---

## 權限控制

```js
const ADMIN_EMAILS = ['nentivericfv@gmail.com', '1252719751@qq.com'];
```

- 管理員：可見新增表單，可編輯/刪除任何記錄
- 訪客：只讀，無法修改資料
- 新增管理員：在 `ADMIN_EMAILS` 陣列加入 email 即可

---

## 版本管理

每次更新 `index.html` 時，需同步更新兩個地方：

1. **`index.html`** 頂部的版本註解：
   ```html
   <!-- VERSION: 2026.04.10.0 -->
   ```

2. **`sw.js`** 的 cache 名稱（觸發 Service Worker 更新，強制用戶取得新版）：
   ```js
   const CACHE = 'marriott-v2026.04.10.0';
   ```

版本格式：`YYYY.MM.DD.序號`（同日多次更新時序號遞增）。

**注意**：`sw.js` 改動後，用戶下次打開 app 時會自動靜默更新（見 `controllerchange` 監聽器）。

---

## Service Worker 快取策略

| 請求類型 | 策略 |
|----------|------|
| `index.html` / navigate | 網絡優先，失敗回落快取 |
| Firebase / googleapis | 永遠直接網絡（不快取）|
| 其他靜態資源 | 快取優先 |

---

## 新增年度（例如 2027）

1. 在 `index.html` 新增一個 `<div class="page" id="page2027">` 區塊
2. 複製 2026 的業務邏輯，更新 `TARGET`、`Q1_START`、`Q1_END`、`staysRef` 路徑
3. 在 tab bar 加入對應按鈕
4. `switchTab()` 函數加入 2027 邏輯
5. Firebase 建立 `stays2027` 路徑（第一次 push 自動創建）

---

## 常見改動位置

| 改動 | 位置 |
|------|------|
| 年度目標晚數 | `const TARGET = 75` |
| Q1 活動日期 | `Q1_START` / `Q1_END` |
| Q1 獎勵積分金額 | `+2500`（在 `calcQ1`、`render2026`、`q1Hint` 共三處）|
| 品牌關鍵字 | `BRAND_KEYWORDS` 陣列 |
| 2025 靜態住宿 | `STAYS_2025` 陣列 |
| 管理員帳號 | `ADMIN_EMAILS` 陣列 |
| 積分基準 | Firebase `config/pointsBase`（透過 UI 更新） |
