# 萬豪追蹤

一個零 build step 的單頁 PWA，用來記錄 `2025` 住宿回顧，同步追蹤 `2026`、`2027` 萬豪保級進度、Q1 品牌獎勵與積分變化。

## Repo 結構

```text
.
├── GUIDELINE_CLOUD.md
├── README.md
├── database.rules.json
├── index.html
├── manifest.json
└── sw.js
```

## Stack

- 原生 `HTML + CSS + JavaScript`
- Firebase Realtime Database
- Firebase Google Auth
- 靜態托管可直接部署
- PWA: `manifest.json` + `sw.js`，提供 `icon-192.png` 與 `apple-touch-icon.png`

## Firebase 資料結構

主要用到以下節點（每個年度一個 `staysYYYY`，現時支援 `2026`、`2027`）：

- `stays2026/{pushId}`、`stays2027/{pushId}`
  - `date`
  - `name`
  - `brand`
  - `nights`
  - `redeem`
  - `redeemPoints`
  - `pointsEarned`
- `config`
  - `pointsBase`
  - `pointsBaseStayIds`（積分餘額 snapshot 當時所有年份嘅住宿 id）
  - `q1/{year}`：`start`、`end`（`YYYY-MM-DD`）、`rule`（`brand` 或 `night`）

## 權限模型

- 前端只會對 `ADMIN_EMAILS` 內的帳號顯示新增、編輯、刪除功能。
- 真正寫入限制要靠 Firebase Realtime Database rules。
- repo 內提供 `database.rules.json` 作為版本化範本，部署前請按需要調整。

## 年度切換

- 分頁：`2027` / `2026` / `2025`。`2026`、`2027` 共用同一個頁面，按年份重新渲染；`2025` 係寫死嘅回顧。
- 預設分頁按手機日期自動選擇當年（`2027-01-01` 起預設 `2027`），過咗年度嘅分頁仍然可以新增、編輯（例如補填積分）。
- 新增住宿按日期嘅年份寫入對應節點；編輯時日期必須留喺同一年。
- 積分餘額係跨年度嘅戶口餘額，`pointsBase` 之後新增嘅所有年份住宿都會計入，所以 `2027` 自動承接 `2026` 年底餘額。
- 新增年份：喺 `index.html` 嘅 `YEARS` 加年份，並喺 `database.rules.json` 加 `stays年份` 節點，再喺 Firebase 控制台套用 rules。

## Q1 規則

- Q1 開始日、結束日同計法按年份設定：`2026` 內置 `2026-02-25` 至 `2026-05-10`，其他年份由 admin 喺 app 內「設定 Q1 日期同計法」輸入（存入 `config/q1/{year}`），未設定前 Q1 區域顯示「未設定」。
- 只計現金入住，`redeem: true` 不計入 Q1 獎勵
- 計法 `brand`：同一品牌第一次符合條件嘅入住，會為保級進度額外加相同房晚數，並額外加 `2,500` 分（`2026` 規則）
- 計法 `night`：窗口內每一晚現金入住都額外加房晚數

## 部署前檢查

- 確認 `index.html` 內 Firebase config 指向正確 project
- 確認 Firebase Auth 已啟用 Google 登入
- 套用 Realtime Database rules（新增 `stays2027` 節點後要重新套用，否則 `2027` 寫入會失敗）
- 如有改動靜態資源，更新 `index.html`、畫面顯示版本與 `sw.js` 版本號

## 備註

- `2025` 資料係寫死喺前端，作為回顧展示
- `2026` 資料由 Firebase Realtime Database 同步
