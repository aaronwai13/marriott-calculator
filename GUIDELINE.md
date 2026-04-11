# 單頁 PWA + Firebase 開發指南（雲端同步版）

適用場景：個人/小團隊工具、**需要即時資料同步**、**Google 帳號登入**控制寫入權限、直接靜態托管部署。

---

## 1. 技術棧決策

### 單頁 HTML vs 框架

| 情況 | 建議 |
|------|------|
| 小型工具、個人/內部使用、無 SEO 需求 | 單頁 HTML（本指南適用）|
| 超過 ~1500 行、多人協作、複雜狀態管理 | 考慮 Vue / React |

優點：零 build step、直接 GitHub Pages 部署、任何人都能打開 `index.html` 睇明。

### Firebase Realtime Database vs Firestore

| | Realtime DB | Firestore |
|---|---|---|
| 即時同步 | 天然支援 | 需額外配置 |
| 資料量 | 適合小量（< 數萬筆） | 適合大量 |
| 查詢能力 | 簡單 | 複雜查詢 |
| **建議** | **本指南選用** | 通常係過度設計 |

### Google Auth：需要 vs 不需要

| 情況 | 建議 |
|------|------|
| 有寫入操作、多用戶區分、需防濫用 | 需要 |
| 純展示、單人本地使用 | 不需要（用純本地版指南） |

常見模式：前端 `ADMIN_EMAILS` 白名單控制誰可寫入，其餘訪客只讀。

---

## 2. 檔案結構

```
project/
├── index.html      ← 全部邏輯（HTML + CSS + JS + Firebase SDK via CDN）
├── manifest.json   ← PWA 設定
├── sw.js           ← Service Worker
└── icon-192.png    ← 主畫面 icon（192×192）
```

Firebase SDK 直接從 CDN 引入，唔需要 `npm install`。

---

## 3. 命名規範

### 檔案
- 全部小寫 kebab-case：`index.html`、`sw.js`、`icon-192.png`

### CSS
- 組件類：語意命名（`.card`、`.stat-card`、`.stay-item`）
- 狀態類：`.active`、`.show`、`.open`
- 盡量避免 `style="..."` 內聯樣式，動態生成 HTML 字串例外

### JavaScript
- 供 HTML `onclick` 呼叫的函數：掛在 `window` 上（`window.fnName = function(){}`）
- 內部函數：camelCase，唔需要掛 `window`
- Firebase refs：語意命名（`staysRef`、`configRef`）

### App 名稱
- `<title>` 及 `manifest.json` 的 `name`：完整名稱（例：「Marriott Bonvoy 住宿追蹤」）
- `manifest.json` 的 `short_name`：2–4 字，主屏顯示（例：「萬豪追蹤」）

---

## 4. Icon 生成規則

- **背景色**：固定深藍 `#1A3C5E`（唔跟 app 主題色，永遠用呢個）
- **文字**：預設用 app 名稱第一個字，除非另有指示
- **文字色**：固定白色
- **字體**：固定 PingFang SC（蘋方），路徑 `/System/Library/Fonts/PingFang.ttc`
- **圓角**：約 20–22% border-radius（192px icon → 約 40px）
- **尺寸**：192×192px
- **Maskable**：`purpose: "any maskable"`，圖案主體需在中心 80% 安全區內

用以下 Python 腳本生成（需安裝 Pillow）：

```python
from PIL import Image, ImageDraw, ImageFont

SIZE = 192
RADIUS = 40
BG = "#1A3C5E"   # 固定深藍，唔需要改
TEXT = "萬"       # 換成 app 名稱第一個字
FONT_SIZE = 96
FONT_PATH = "/System/Library/Fonts/PingFang.ttc"  # 固定 PingFang，唔需要改

img = Image.new("RGBA", (SIZE, SIZE), (0, 0, 0, 0))
draw = ImageDraw.Draw(img)

def rounded_rect(draw, xy, radius, fill):
    x0, y0, x1, y1 = xy
    draw.rectangle([x0+radius, y0, x1-radius, y1], fill=fill)
    draw.rectangle([x0, y0+radius, x1, y1-radius], fill=fill)
    for cx, cy in [(x0+radius, y0+radius), (x1-radius, y0+radius),
                   (x0+radius, y1-radius), (x1-radius, y1-radius)]:
        draw.ellipse([cx-radius, cy-radius, cx+radius, cy+radius], fill=fill)

rounded_rect(draw, [0, 0, SIZE, SIZE], RADIUS, BG)

font = ImageFont.truetype(FONT_PATH, FONT_SIZE)
bbox = draw.textbbox((0, 0), TEXT, font=font)
w, h = bbox[2]-bbox[0], bbox[3]-bbox[1]
draw.text(((SIZE-w)/2 - bbox[0], (SIZE-h)/2 - bbox[1]), TEXT, font=font, fill="white")

img.save("icon-192.png")
print("Done: icon-192.png")
```

---

## 5. Design 決策原則

根據 app 類型揀最合適嘅 design 方向：

| App 類型 | 建議 Design 方向 |
|---|---|
| 查詢 / 目錄 | 卡片列表 + 搜索框 + filter chips + expand/collapse |
| 比較 / 評分 | 表格或橫向 scroll，highlight 最優選項 |
| 工具 / 計算機 | 輸入為主，結果即時更新，減少層次 |
| 記錄 / Tracker | 列表 + 狀態標記，支援新增／刪除（Firebase 最適合） |
| 指南 / 教學 | 長篇內容，分 section，加 numbered steps |

設計時考慮：
- Tab bar 只在有 **3 個或以上獨立頁面** 時才需要
- 搜索框只在條目 **超過 8–10 個** 時才有意義

---

## 6. CSS 設計系統

### 顏色變數（light/dark 自動切換）

```css
:root {
  /* 背景層次 */
  --bg:       #F7F7F5;
  --white:    #FFFFFF;
  --surface2: #F0F0EE;

  /* 文字 */
  --text:       #111110;
  --text-muted: #888884;
  --text-light: #BBBBB8;

  /* 邊框 */
  --border:        #E5E5E2;
  --border-strong: #D0D0CC;

  /* 強調色（唯一，全 app 統一） */
  --accent:       #1A3C5E;
  --accent-light: #E8EFF5;

  /* 語義色（固定代表好/警告/壞，不可挪作裝飾） */
  --green:       #1A6B45;
  --green-light: #E8F5EE;
  --amber:       #8B5E00;
  --amber-light: #FEF7E8;
  --red:         #C0392B;
  --red-light:   #FDF0EE;

  --shadow: 0 2px 12px rgba(0,0,0,0.07);
}

@media (prefers-color-scheme: dark) {
  :root {
    --bg:         #111112;
    --white:      #1C1C1E;
    --surface2:   #252527;
    --text:       #F2F2F0;
    --text-muted: #888884;
    --text-light: #48484A;
    --border:        #2A2A2C;
    --border-strong: #3A3A3C;
    --accent:       #5B9BD5;  /* dark mode 要比 light mode 調淺 */
    --accent-light: #1A2A3A;
    --green:       #4ADE80;
    --green-light: #0A2016;
    --amber:       #FCD34D;
    --amber-light: #251A00;
    --red:         #F87171;
    --red-light:   #2A1010;
    --shadow: 0 2px 16px rgba(0,0,0,0.4);
  }
}
```

**原則：**
- `--accent` 係唯一強調色，按鈕、連結、focus ring 全部用它
- `--green` / `--amber` / `--red` 固定代表好 / 警告 / 壞，唔可以挪作裝飾
- dark mode 只需覆蓋背景同文字系列變數，語義色保持不變

### 字型

中文 app 優先使用系統字體棧，無需引入外部字型：

```css
font-family: -apple-system, 'SF Pro Text', 'Helvetica Neue',
             'PingFang TC', 'Microsoft JhengHei', Arial, sans-serif;
```

如需要更強烈嘅設計感，可選用 Google Fonts（DM Serif Display / DM Mono / Figtree）。

### 字階

| 用途 | 大小 | 字重 |
|------|------|------|
| 頁面大標題 | 32–48px | 600 |
| 區塊標題 | 20–24px | 600 |
| 卡片標題 | 17–21px | 500–600 |
| 正文 | 14–17px | 400 |
| 輔助文字、標籤 | 11–13px | 400 |
| 微型（時間戳、小 tag）| 10–12px | 400 |

### 佈局基礎

```css
* {
  margin: 0; padding: 0; box-sizing: border-box;
  -webkit-tap-highlight-color: transparent;
}
body {
  background: var(--bg); color: var(--text);
  font-family: -apple-system, 'PingFang TC', sans-serif;
  min-height: 100vh;
  padding-bottom: calc(60px + env(safe-area-inset-bottom, 0px));
}
#app { max-width: 480px; margin: 0 auto; }
```

### 常用 UI 元件

**Sticky Header：**
```css
.header {
  background: var(--white);
  padding: 20px 16px 0;
  border-bottom: 1px solid var(--border);
  position: sticky; top: 0; z-index: 20;
}
```

**Tab Bar（底部固定，3 個頁面或以上才用）：**
```css
.tab-bar {
  position: fixed; bottom: 0;
  left: 50%; transform: translateX(-50%);
  width: 100%; max-width: 480px;
  height: 60px; background: var(--white);
  border-top: 1px solid var(--border);
  display: flex; z-index: 100;
  padding-bottom: env(safe-area-inset-bottom, 0px);
}
.tab-bar::after {
  content: ''; position: absolute;
  bottom: calc(-1 * env(safe-area-inset-bottom, 0px));
  left: 0; right: 0;
  height: env(safe-area-inset-bottom, 0px);
  background: var(--white);
}
```

**Badge（狀態標籤）：**
```css
.badge {
  font-size: 10px; font-weight: 600;
  padding: 3px 8px; border-radius: 6px;
}
.badge-green { background: var(--green-light); color: var(--green); }
.badge-amber { background: var(--amber-light); color: var(--amber); }
.badge-red   { background: var(--red-light);   color: var(--red); }
.badge-blue  { background: var(--accent-light); color: var(--accent); }
```

**Filter Chips：**
```css
.filter-row { display: flex; gap: 6px; padding-bottom: 10px; }
.chip {
  flex: 1; text-align: center;
  padding: 6px 4px; border-radius: 99px;
  font-size: 12px; font-weight: 600;
  border: 1.5px solid var(--border-strong);
  background: transparent; color: var(--text-muted);
  cursor: pointer; transition: all 0.15s;
}
.chip.active { background: var(--accent); color: white; border-color: var(--accent); }
```

---

## 7. Firebase 設計規範

### 資料結構

- 保持**扁平**，避免嵌套超過 3 層
- 每筆記錄用 Firebase `push()` 自動生成的 key 作 ID
- 業務資料同系統配置分離

```
/records          ← 業務資料（用 push key）
  /-Nxxx: { date, name, nights, ... }
/config           ← 系統配置（唔係業務資料）
  /baseline: 1000
  /lastUpdated: "2026.04.10"
```

### 讀寫模式

```javascript
import { getDatabase, ref, onValue, push, update, remove }
  from 'https://www.gstatic.com/firebasejs/10.x.x/firebase-database.js';

// 即時監聽（推薦）：每次資料變動自動觸發 render
onValue(ref(db, 'records'), (snapshot) => {
  const data = snapshot.val();
  const items = data
    ? Object.entries(data).map(([key, val]) => ({ ...val, _key: key }))
    : [];
  render(items);
});

// 新增
push(ref(db, 'records'), newItem);

// 修改
update(ref(db, `records/${key}`), changes);

// 刪除
remove(ref(db, `records/${key}`));
```

### 初始化保護

防止空資料庫首次載入時重複插入預設值：

```javascript
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

## 8. Google Auth 整合

```javascript
import { getAuth, GoogleAuthProvider, signInWithPopup, signOut, onAuthStateChanged }
  from 'https://www.gstatic.com/firebasejs/10.x.x/firebase-auth.js';

const ADMIN_EMAILS = ['your@email.com']; // 換成你的 Gmail

onAuthStateChanged(auth, (user) => {
  const isAdmin = !!(user && ADMIN_EMAILS.includes(user.email));
  // 根據 isAdmin 顯示/隱藏寫入 UI
  document.querySelectorAll('.admin-only').forEach(el => {
    el.style.display = isAdmin ? '' : 'none';
  });
});

// 登入
window.login = async function() {
  try {
    await signInWithPopup(auth, new GoogleAuthProvider());
  } catch (e) {
    if (e.code !== 'auth/popup-closed-by-user') {
      console.error(e); // 手動關閉彈窗唔算錯誤，靜默處理
    }
  }
};

// 登出
window.logout = async function() {
  await signOut(auth);
};
```

**設計模式：**
- 管理員才顯示新增/編輯/刪除按鈕（用 `.admin-only` class 控制）
- 非管理員用 Google 帳號登入時顯示「無存取權限」提示
- 彈窗被用戶手動關閉（`auth/popup-closed-by-user`）唔算錯誤，靜默處理

---

## 9. PWA 配置

### manifest.json

```json
{
  "name": "完整 App 名稱",
  "short_name": "短名",
  "description": "一句話描述",
  "start_url": "./",
  "display": "standalone",
  "background_color": "#1A3C5E",
  "theme_color": "#1A3C5E",
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

### sw.js（Firebase 請求唔快取）

```javascript
const CACHE = 'app-name-v2026.01.01.0'; // 版本號同 index.html 頂部保持一致

self.addEventListener('install', e => {
  e.waitUntil(
    caches.open(CACHE).then(c => c.addAll(['./', './index.html', './manifest.json']))
  );
  self.skipWaiting();
});

self.addEventListener('activate', e => {
  e.waitUntil(
    caches.keys().then(keys =>
      Promise.all(keys.filter(k => k !== CACHE).map(k => caches.delete(k)))
    )
  );
  self.clients.claim();
});

self.addEventListener('fetch', e => {
  // Firebase 同 googleapis 請求：直接網絡，唔快取
  if (e.request.url.includes('firebaseio.com') ||
      e.request.url.includes('googleapis.com') ||
      e.request.url.includes('gstatic.com')) {
    return;
  }
  if (e.request.mode === 'navigate') {
    // 主頁面：網絡優先（確保拿到最新版）
    e.respondWith(
      fetch(e.request).then(res => {
        caches.open(CACHE).then(c => c.put(e.request, res.clone()));
        return res;
      }).catch(() => caches.match(e.request))
    );
    return;
  }
  // 其他靜態資源：緩存優先
  e.respondWith(
    caches.match(e.request).then(cached =>
      cached || fetch(e.request).then(res => {
        caches.open(CACHE).then(c => c.put(e.request, res.clone()));
        return res;
      })
    )
  );
});
```

---

## 10. 版本管理

版本格式：`YYYY.MM.DD.序號`（同日多次更新時序號遞增，例：`2026.04.10.0`、`2026.04.10.1`）

每次更新**必須同步兩處**：

1. `index.html` 頂部 comment：
   ```html
   <!-- VERSION: 2026.04.10.0 -->
   ```

2. `sw.js` 的 cache 名稱：
   ```javascript
   const CACHE = 'app-name-v2026.04.10.0';
   ```

更新 cache 名稱會觸發 Service Worker 重新安裝，用戶下次打開 app 時靜默取得新版本。

---

## 11. 部署（GitHub Pages）

1. Repo 設定 → Pages → Source 選 `main` branch，根目錄 `/`
2. 直接 push 到 `main` → 自動 deploy（約 1–2 分鐘）
3. Firebase Console → Authentication → 加入你嘅 Google 帳號
4. Firebase Console → Realtime Database → Rules → 貼入上方 Security Rules
5. 更新內容後記得同步版本號，確保用戶清除舊緩存

---

## 12. 快速開始 Checklist

- [ ] 根據 app 類型決定 design 方向（參考上方表格）
- [ ] 建立 Firebase 項目，啟用 Realtime Database 同 Google Auth
- [ ] 複製 HTML 骨架，引入 Firebase SDK（CDN），填入 firebaseConfig
- [ ] 設定 `ADMIN_EMAILS` 白名單
- [ ] 決定主題色，更新 `--accent` 及 dark mode 對應色
- [ ] 用 Python 腳本生成 `icon-192.png`（背景固定 `#1A3C5E`，白色文字，app 名第一個字）
- [ ] 定義 Firebase 資料結構（扁平、最多 3 層）
- [ ] 實作 `onValue()` 監聽 + render 函數
- [ ] 更新 `manifest.json` 名稱、描述、顏色
- [ ] 改 `sw.js` 的 `CACHE` 版本號，與 `index.html` 頂部 comment 一致
- [ ] Firebase Console 設定 Security Rules
- [ ] 設定 GitHub Pages，測試 PWA 安裝同 Firebase 同步
