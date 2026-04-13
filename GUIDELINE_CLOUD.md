# My App Recipe for AI

呢份文件係我私人用嘅 app 開發 recipe。
目的唔係追求通用工程最佳實踐，而係令 AI 用接近我現有 app 嘅方式去修改舊 app / 建新 app。

除非我明確要求，否則優先跟呢份 recipe，而唔好自行升級去 React、Next.js、TypeScript、bundler、Firestore 或其他更重方案。

---

## 1. 核心原則

- 優先做出可直接使用、可直接部署、可直接睇明嘅 app
- 優先簡單、直接、可維護，而唔係為將來假設需求預先過度設計
- 保持零 build step 為預設
- 小型工具優先原生 HTML + CSS + JavaScript
- 設計要有審美，但唔好變成花巧 demo
- 修改現有 app 時，優先延續原有結構同視覺語言

---

## 2. Default Stack

新 app 預設使用：

- 單頁 `index.html`
- 原生 HTML + CSS + JavaScript
- 無 build step
- 可直接靜態托管
- 如需同步，用 Firebase CDN SDK
- 如需登入，用 Google Auth
- 如需手機主畫面使用，加 `manifest.json` + `sw.js`

預設檔案結構：

```text
project/
├── index.html
├── manifest.json
└── sw.js
```

Firebase SDK 直接從 CDN 引入，唔需要 `npm install`。

---

## 3. Do Not Overengineer

除非我明確要求，否則不要：

- 引入 npm / pnpm / yarn
- 加 bundler
- 改用 TypeScript
- 改做 React / Vue / Next.js
- 引入複雜 component architecture
- 改用 Firestore 取代 Realtime Database
- 為咗所謂可擴展性而大改原本簡單結構

如果依家個 app 已經可以用，而且只係做小幅功能修改，應優先小改而唔係重構。

---

## 4. 什麼情況適合這套 Recipe

呢套 recipe 特別適合：

- 個人工具
- 小型 tracker / calculator / directory / guide
- 手機優先使用
- 無 SEO 需求
- 想直接部署到靜態 hosting
- 想用最少工程成本換最高完成度

如果係以下情況，先可以提出偏離呢套 recipe：

- 我明確要求用框架
- 畫面 / 狀態已經複雜到單檔明顯難維護
- 需要多人長期協作
- 有大量重用元件
- 查詢需求已經超出 Realtime Database 嘅舒適範圍

即使要升級，都要先講清楚原因同代價。

---

## 5. When Editing Existing Apps

如果係修改現有 app：

- 先理解現有結構、命名、資料流、UI 風格
- 優先延續原本做法
- 唔好為咗整齊而硬拆成框架式架構
- 唔好隨便改動已經運作良好嘅 interaction pattern
- 改動應盡量細、準、可預測
- 只喺維護成本已經明顯太高時先建議拆檔或重構

如果要偏離原本方向，先解釋點解原做法已經唔夠用。

---

## 6. Escalation Rules

預設單檔。

只有當出現以下情況，先建議拆成 `styles.css` / `app.js`：

- `index.html` 已經太長，改動風險高
- CSS 已經大到難以閱讀
- JavaScript 邏輯已經難追
- 同類功能會反覆新增
- 我明確表示想提升可維護性

即使拆檔，都仍然優先保持：

- 無 build step
- 原生 JavaScript
- 可直接靜態部署

拆檔順序優先：

1. 先拆 `app.js`
2. 再拆 `styles.css`
3. 最後先考慮更重方案

---

## 7. Firebase Defaults

如果 app 需要即時同步，預設使用 Firebase Realtime Database。

原因：

- 對小工具夠用
- 配置直接
- 即時同步自然
- 適合簡單資料結構

如果 app 需要登入，預設用 Google Auth。

常見模式：

- 前端用 `ADMIN_EMAILS` 控制可寫入用戶
- 未登入或非白名單用戶只讀
- Firebase Database 存主資料
- `config` 節點存全局設定

安全底線：

- 前端白名單只係 UI / 權限提示
- 真正寫入限制要靠 Firebase security rules

---

## 8. Naming Conventions

### 檔案

- 全部小寫 kebab-case
- 預設檔名：`index.html`、`sw.js`、`manifest.json`

### CSS

- 組件類用語意命名：`.card`、`.stat-card`、`.stay-item`
- 狀態類用：`.active`、`.show`、`.open`
- 除非動態生成 HTML，否則盡量避免大量 inline style

### JavaScript

- HTML `onclick` 直接用到嘅函數掛 `window`
- 內部函數用 camelCase
- Firebase refs 用語意命名：`staysRef`、`configRef`

### App 名稱

- `<title>` 同 `manifest.json.name` 用完整名稱
- `manifest.json.short_name` 用 2 至 4 字

---

## 9. Design Taste

我偏好嘅 app 風格：

- 精緻、克制、有設計感
- 唔似企業 dashboard
- 唔似 generic SaaS template
- 唔似 AI 自動生成 landing page
- 手機優先，但 desktop 都要順眼
- 一頁內盡量完成主要任務
- 內容層次清楚，但唔好過度堆疊

視覺方向可以參考：

- premium tracker
- boutique hotel / editorial feeling
- calm, polished, slightly luxurious

避免：

- 紫白預設配色
- 過重 dashboard 感
- 太多冇意義卡片
- 過多動畫
- 看起來很「模板」

---

## 10. Design Decision Rules

根據 app 類型，優先使用以下方向：

| App 類型 | 建議 Design 方向 |
|---|---|
| 查詢 / 目錄 | 卡片列表 + 搜索框 + filter chips + expand/collapse |
| 比較 / 評分 | 表格或橫向 scroll，highlight 最優選項 |
| 工具 / 計算機 | 輸入為主，結果即時更新，減少層次 |
| 記錄 / Tracker | 列表 + 狀態標記，支援新增 / 刪除 |
| 指南 / 教學 | 長篇內容，分 section，加明確步驟 |

設計時考慮：

- Tab bar 只在有 3 個或以上獨立頁面時才用
- 搜索框只在條目超過 8 至 10 個時先值得加
- 主操作應該一眼睇到
- 螢幕細時，優先保留層次而唔好塞太多資訊

---

## 11. CSS System Defaults

顏色原則：

- 全 app 只用一個主要 accent color
- `green / amber / red` 只用作語義色
- dark mode 可以調背景同文字，但唔好破壞語義色邏輯

中文字體預設：

```css
font-family: -apple-system, 'SF Pro Text', 'Helvetica Neue',
             'PingFang TC', 'Microsoft JhengHei', Arial, sans-serif;
```

如果要更有設計感，可以用 Google Fonts，但只在真有幫助時加入。

字階原則：

- 頁面大標題：32 至 48px
- 區塊標題：20 至 24px
- 卡片標題：17 至 21px
- 正文：14 至 17px
- 輔助文字：11 至 13px
- 微型資訊：10 至 12px

---

## 12. PWA Defaults

如果 app 適合放上手機主畫面，預設加：

- `manifest.json`
- `sw.js`

PWA 原則：

- app 名稱清楚
- 預設唔指定 custom icon，交由系統按網站捷徑方式處理
- 可離線讀基本頁面
- 更新策略簡單直接

---

## 13. Icon Defaults

預設唔另外提供 custom PWA icon。

如果我明確要求，先再為個別 app 製作 icon。

---

## 14. How AI Should Respond

當我要求修改 app 或建立新 app 時，AI 應：

- 優先直接實作，而唔係先講長篇方案
- 如有明顯 tradeoff，先簡短講原因
- 完成後用簡潔方式總結改咗咩
- 如果偏離呢份 recipe，要明確講偏離原因
- 如果現有 app 已經有明顯風格，優先跟返個風格

我唔需要 generic best practices；我需要貼近我工作方式、審美同部署習慣嘅結果。

---

## 15. Simple Heuristic

可以用以下順序判斷：

1. 先問：呢個 app 可唔可以繼續保持單頁、原生、零 build step？
2. 如果可以，就唔好升級架構
3. 先做可用版本，再考慮整理
4. 如果係改舊 app，先跟舊 pattern
5. 只有原方案明顯唔夠用，先提出更重方案

呢份 recipe 係偏好聲明。
除非我明確要求，否則 AI 應將它視為預設做法，而唔係其中一個可選方案。
