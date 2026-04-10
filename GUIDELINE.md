# 單頁 PWA + Firebase 開發指南

適用於以下類型的 app：個人/小團隊工具、即時資料同步、無需複雜後端、直接靜態托管部署。

---

## 1. 技術棧決策

### 單頁 HTML vs 框架

| 情況 | 建議 |
|------|------|
| 小型工具、個人/內部使用、無 SEO 需求 | 單頁 HTML（本指南適用）|
| 超過 ~1500 行、多人協作、複雜狀態管理 | 考慮 Vue / React |

優點：零 build step、直接 GitHub Pages 部署、任何人都能打開 `index.html` 看懂。

### Firebase Realtime Database vs Firestore

- **Realtime Database**：即時同步、結構簡單、資料量小（< 數萬筆）→ 適合此類 app
- **Firestore**：複雜查詢、大資料量、多集合關聯 → 通常是過度設計

### Google Auth：需要 vs 不需要

| 情況 | 建議 |
|------|------|
| 有寫入操作、多用戶區分、需防濫用 | 需要 |
| 純展示、單人本地使用 | 不需要 |

常見模式：前端 `ADMIN_EMAILS` 白名單控制誰可寫入，其餘訪客只讀。

---

## 2. 命名規範

### App 名稱
- `<title>` 及 `manifest.json` 的 `name`：完整名稱（例：「Marriott Bonvoy 住宿追蹤」）
- `manifest.json` 的 `short_name`：2–4 字，主屏顯示（例：「萬豪追蹤」）
- 檔案名：全小寫 kebab-case（`index.html`、`sw.js`、`icon-192.png`）

### CSS
- 組件類：語意命名（`.card`、`.stat-card`、`.stay-item`）
- 狀態類：`.active`、`.show`、`.open`
- 盡量避免 `style="..."` 內聯樣式，動態生成的 HTML 字串例外

### JavaScript
- 供 HTML `onclick` 呼叫的函數：掛在 `window` 上（`window.fnName = function(){}`）
- 內部函數：camelCase，不掛 `window`
- Firebase refs：語意命名（`staysRef`、`configRef`）

---

## 3. 設計系統

### 顏色

用 CSS 變量定義語義色，不在各處直接寫死色碼：

```css
:root {
  /* 背景層次 */
  --bg:       #f5f5f7;              /* 頁面底色 */
  --surface:  #ffffff;              /* 卡片、面板 */
  --surface2: #f5f5f7;             /* 次級面板、input 背景 */

  /* 文字 */
  --text:       #1d1d1f;
  --text-muted: rgba(0, 0, 0, 0.48);

  /* 邊框 */
  --border: rgba(0, 0, 0, 0.08);

  /* 語義色（固定，深淺模式不改） */
  --green:  #34c759;   /* 正數、成功、達標 */
  --red:    #ff3b30;   /* 負數、錯誤、刪除 */
  --accent: #0071e3;   /* 唯一強調色，所有互動元素 */
}

@media (prefers-color-scheme: dark) {
  :root {
    --bg:         #000000;
    --surface:    #1c1c1e;
    --surface2:   #2c2c2e;
    --text:       #ffffff;
    --text-muted: rgba(255, 255, 255, 0.48);
    --border:     rgba(255, 255, 255, 0.1);
    /* --green / --red / --accent 深色模式不需要改 */
  }
}
```

**原則：**
- 強調色只用一個（`--accent`），按鈕、連結、focus ring 全部用它
- 語義色（`--green` / `--red`）固定代表好/壞，不可挪作裝飾用途
- 深色模式只需覆蓋背景與文字系列變量，語義色保持不變

### 字體

中文 app 優先使用系統字體棧，無需引入外部字型：

```css
font-family: -apple-system, 'SF Pro Text', 'Helvetica Neue',
             'PingFang TC', 'Microsoft JhengHei', Arial, sans-serif;
```

字階：

| 用途 | 大小 | 字重 |
|------|------|------|
| 頁面大標題 | 32–48px | 600 |
| 區塊標題 | 20–24px | 600 |
| 卡片標題 | 17–21px | 500–600 |
| 正文 | 14–17px | 400 |
| 輔助文字、標籤 | 11–13px | 400 |
| 微型（時間戳、小 tag）| 10–12px | 400 |

### App 圖示

- **最低要求**：192×192px PNG
- **建議**：同時提供 512×512px
- **Maskable**：`purpose: "any maskable"`，圖案主體需在中心 80% 安全區內，邊緣留白以防系統遮罩裁切
- `manifest.json` 的 `theme_color` 與 `background_color` 使用圖示主色

---

## 4. Firebase 設計規範

### 資料結構

- 保持**扁平**，避免嵌套超過 3 層
- 每筆記錄用 Firebase `push()` 自動生成的 key 作 ID
- 配置/系統資料放在獨立的 `config/` 路徑，與業務資料分離

```
/records          ← 業務資料（用 push key）
  /-Nxxx: { date, name, ... }
/config           ← 系統配置
  /baseline: 1000
  /baselineIds: [...]
```

### 讀寫模式

```js
// 即時監聽（推薦）：每次資料變動自動觸發 render
onValue(ref(db, 'records'), (snapshot) => {
  const data = snapshot.val();
  const items = data ? Object.entries(data).map(([key, val]) => ({ ...val, _key: key })) : [];
  render(items);
});

// 寫入
push(ref(db, 'records'), newItem);           // 新增
update(ref(db, `records/${key}`), changes);  // 修改
remove(ref(db, `records/${key}`));           // 刪除
```

**初始化保護**：用 `dbInitialized` flag，防止空資料庫在首次載入時重複插入預設值：

```js
let dbInitialized = false;
onValue(staysRef, (snapshot) => {
  if (!snapshot.val() && !dbInitialized) {
    dbInitialized = true;
    DEFAULT_DATA.forEach(item => push(staysRef, item));
    return;
  }
  dbInitialized = true;
  render(/* ... */);
});
```

### Security Rules 最低配置

訪客只讀、登入用戶可寫（配合前端白名單）：

```json
{
  "rules": {
    ".read": true,
    ".write": "auth != null"
  }
}
```

---

## 5. Google Auth 整合

```js
import { getAuth, GoogleAuthProvider, signInWithPopup, signOut, onAuthStateChanged }
  from 'firebase/auth';

const ADMIN_EMAILS = ['admin@example.com'];

onAuthStateChanged(auth, (user) => {
  const isAdmin = !!(user && ADMIN_EMAILS.includes(user.email));
  // 根據 isAdmin 顯示/隱藏寫入 UI
});

// 登入
await signInWithPopup(auth, new GoogleAuthProvider());

// 登出
await signOut(auth);
```

**設計模式：**
- 管理員才顯示新增/編輯/刪除按鈕
- 非管理員登入時顯示「無存取權限」提示
- 彈窗被用戶手動關閉（`auth/popup-closed-by-user`）不算錯誤，靜默處理

---

## 6. PWA 配置

### manifest.json

```json
{
  "name": "完整 App 名稱",
  "short_name": "短名",
  "description": "一句話描述",
  "start_url": "./",
  "display": "standalone",
  "background_color": "#主色",
  "theme_color": "#主色",
  "lang": "zh-HK",
  "icons": [
    {
      "src": "icon-192.png",
      "sizes": "192x192",
      "type": "image/png",
      "purpose": "any maskable"
    }
  ]
}
```

### Service Worker 快取策略

```js
const CACHE = 'app-v2026.01.01.0';  // 版本號決定快取更新時機

// index.html → 網絡優先（確保總取到最新版）
// Firebase / googleapis → 直接網絡，不快取
// 其他靜態資源 → 快取優先
```

---

## 7. 版本管理

版本格式：`YYYY.MM.DD.序號`（同日多次更新時序號遞增，如 `2026.04.10.0`、`2026.04.10.1`）

每次更新**必須同步兩處**：

1. `index.html` 頂部：
   ```html
   <!-- VERSION: 2026.04.10.0 -->
   ```

2. `sw.js` 的 cache 名稱：
   ```js
   const CACHE = 'app-v2026.04.10.0';
   ```

更新 cache 名稱會觸發 Service Worker 重新安裝，用戶下次打開 app 時靜默取得新版本。
