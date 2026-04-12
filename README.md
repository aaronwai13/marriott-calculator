# Marriott Bonvoy 住宿追蹤

一個零 build step 的單頁 PWA，用來記錄 `2025` 住宿回顧，同步追蹤 `2026` Marriott Bonvoy 保級進度、Q1 品牌獎勵與積分變化。

## Repo 結構

```text
.
├── GUIDELINE_CLOUD.md
├── README.md
├── database.rules.json
├── icon-192.png
├── index.html
├── manifest.json
└── sw.js
```

## Stack

- 原生 `HTML + CSS + JavaScript`
- Firebase Realtime Database
- Firebase Google Auth
- 靜態托管可直接部署
- PWA: `manifest.json` + `sw.js`

## Firebase 資料結構

主要用到兩個節點：

- `stays2026/{pushId}`
  - `date`
  - `name`
  - `brand`
  - `nights`
  - `redeem`
  - `redeemPoints`
  - `pointsEarned`
- `config`
  - `pointsBase`
  - `pointsBaseStayIds`

## 權限模型

- 前端只會對 `ADMIN_EMAILS` 內的帳號顯示新增、編輯、刪除功能。
- 真正寫入限制要靠 Firebase Realtime Database rules。
- repo 內提供 `database.rules.json` 作為版本化範本，部署前請按需要調整。

## Q1 規則

- Q1 視窗：`2026-02-25` 至 `2026-05-10`
- 只計現金入住，`redeem: true` 不計入 Q1 品牌獎勵
- 同一品牌第一次符合條件的入住，會為保級進度額外加相同房晚數
- 同一品牌第一次符合條件的入住，會額外加 `2,500` 分

## 部署前檢查

- 確認 `index.html` 內 Firebase config 指向正確 project
- 確認 Firebase Auth 已啟用 Google 登入
- 套用 Realtime Database rules
- 如有改動靜態資源，更新 `index.html` 與 `sw.js` 版本號

## 備註

- `2025` 資料係寫死喺前端，作為回顧展示
- `2026` 資料由 Firebase Realtime Database 同步
