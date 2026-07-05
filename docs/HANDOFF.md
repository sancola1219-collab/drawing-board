# 自由畫板 Atelier — 交接文件

> 給任何接手的 AI 助手（Claude Code / Codex / 其他）與開發者。
> 一句話：**單檔純前端畫板（index.html），零依賴、零建置，發佈於 GitHub Pages。**
> 本文件是唯一權威來源；`CLAUDE.md` 與 `AGENTS.md` 是它的精簡版入口，三者需同步維護。

---

## 0. 快速上手

| 項目 | 值 |
|---|---|
| 線上網址 | https://sancola1219-collab.github.io/drawing-board/ |
| GitHub repo | `sancola1219-collab/drawing-board`（master 分支根目錄 Pages） |
| 本機執行 | 直接用瀏覽器開 `index.html`；或 `node .claude/serve.js`（讀 `PORT` 環境變數，預設 8124） |
| 原始碼 | 全部在 `index.html` 一個檔案（CSS + HTML + JS） |
| git 身分 | user `tonnychiulab` / `sancola1220@gmail.com`（repo 已設定） |

## 1. 產品定位與設計準則（使用者明確要求，勿違反）

1. **高級質感，不要可愛風**（2026-07-05 定調）：深色主題、金銅色點綴、serif 標題、SVG 線條圖示、文案克制（不用「！」「囉」「吧」等語氣詞）。
2. **永遠單一 `index.html`、零外部依賴**：不可引入 CDN、npm 套件、外部字型檔。
3. 介面文字為繁體中文（台灣用語）。
4. 設計 token 全部集中在 CSS `:root`：`--bg0/--bg1/--bg2/--bg3`（深灰階）、`--line/--line2`（邊框）、`--txt/--txt2`（文字）、`--gold: #c8a468`（唯一點綴色）、`--serif`（標題字型）。新 UI 一律取用這些變數。
5. 目標裝置：桌機 + 手機平板（觸控），支援放大/平移。

## 2. `index.html` 結構地圖

依檔內出現順序（JS 區段可搜尋 `══` 分隔標題）：

**CSS**：token → header → #toolbar/.tool → #stage/#board → #bottombar/調色盤 → .drawer/.panel → .stab 頁籤 → #stampGrid → #pageGrid/#photoPanel → 作品集 → #toast → 手機 media query

**HTML**：`<header>`（7 顆按鈕：圖章/著色本/作品集/復原/重做/清除/存檔）→ `<main>`（`#toolbar` 左欄、`#stage` > `#canvasWrap` > `#board`〔內部解析度固定 1200×800〕）→ `#bottombar`（粗細、26 色調色盤、自訂色、畫紙）→ 三個 `.drawer`（`stampDrawer` / `coloringDrawer` / `galleryDrawer`）→ `#toast`

**JS 區段**：
| 區段 | 重點符號 |
|---|---|
| 基本設定 | `W=1200, H=800`、`state`（tool/color/size/stamp/mirror/paper） |
| 工具列 | `svgIcon()`/`ICONS`（全部圖示的 SVG path）、`TOOLS`（11 工具）、`setTool()`、`mirrorBtn`、`zoomBtn`（`ZOOMS=[1,1.6,2.4]`） |
| 復原/重做 | `snapshotCanvas()`（離屏 canvas 快照）、`pushUndo(snap?)`、同步 `restore()`、`MAX_UNDO=15` |
| 筆刷 | `strokeSegment()` → `drawBrushSegment()`（依 `state.tool` switch；對稱模式畫兩次） |
| 油漆桶 | `floodFill()` 掃描線填色，容差 60²，含**線條吸附**（點到深色線往外螺旋找半徑 8 內最近非線像素） |
| 印章 | `placeStamp()`、`STAMP_SETS`（196 emoji、6 分類） |
| 線稿轉換引擎 | `boxBlur()`、`dilateMask()`、`toLineArt(src, "flat"|"photo", opts)`、`renderEmojiCanvas()`、`emojiColoringPage()` |
| 著色圖庫 | `EMOJI_PAGE_SETS`（7 分類 459 張）+ 執行期 ZWJ/去重過濾、`PAGES`（10 張手繪場景，canvas 程式繪製）、`TOTAL_PAGES=469` |
| 著色分頁 | `renderColoringTab()`、縮圖泵 `queueThumb()`/`thumbChannel`（MessageChannel）、`thumbCache` |
| 照片變線稿 | `photoSrc`（600×400 白底 contain）→ `toLineArt("photo")` → 套用時 ×2 nearest-neighbor 放大到 1200×800 |
| 畫布事件 | pointerdown/move/up：hand=平移 stage、bucket、stamp、筆刷；`endStroke()` |
| 存檔/作品集 | `compositeCanvas()`（合成畫紙底色）、localStorage 鍵 `freeboard.gallery.v1`（JPEG 0.72，空間不足淘汰最舊） |
| 版面 | `fitCanvas()`（含 0 尺寸守衛 + zoomFactor）+ `ResizeObserver` |
| 測試掛鉤 | `window.__board`（見 §4） |

## 3. 核心不變式（改壞會出事，動手前必讀）

1. **線稿必須二值化輸出**：線 = RGB(34,34,34)、底 = 255，皆 alpha 255。油漆桶容差依賴這點才不漏色。改 `toLineArt` 時務必保持二值輸出。
2. **復原快照 = 離屏 canvas + 同步 restore**。不要改回 `toDataURL`（同步 PNG 編碼會讓每次下筆卡 50–250ms），也不要用非同步 `Image.onload` 還原（快按復原會競態）。
3. **油漆桶順序**：先 `snapshotCanvas()` → `floodFill()` → **成功才** `pushUndo(before)`；失敗給 toast。順序反了會把無效點擊塞進復原堆疊並清掉重做紀錄。
4. **`fitCanvas` 開頭的尺寸守衛與 `ResizeObserver` 不能移除**：隱藏視窗載入時容器尺寸可能為 0，算出負縮放設進 CSS 會被靜默忽略、畫面爆版。
5. **縮圖生成只能走 MessageChannel 泵**（`thumbChannel`）：隱藏分頁的 setTimeout 被節流到約 1 次/分，MessageChannel 微任務不受節流。每輪預算 ≤24ms。
6. **縮圖預熱只在第一次打開著色本時啟動**（在 `openDrawer` 內），不要搬回頁面啟動時（會在使用者剛開始畫圖時佔滿主執行緒約 9 秒 + 常駐 40MB）。
7. **分類鍵含 emoji 前綴是資料鍵**（如 `"🐾 動物王國"`），顯示時用 `tabLabel()` 去前綴。改鍵名要同步改 `CLASSIC_TAB`/`PHOTO_TAB`/`currentColorTab` 與所有引用。
8. **彩色 emoji `fillText` 光柵化很貴**（約 15–20ms/字），且成本遞延到第一次 `getImageData` 才爆出來 — profiling 時別誤判成像素迴圈慢。
9. 照片線稿在 600×400 處理後 ×2 無平滑放大 — 是刻意的（預覽即所得 + 粗線利於填色），不是 bug。

## 4. 測試方式

**重要背景：這台機器的預覽/Chrome 視窗常處於 hidden 甚至 0×0**，因此：
- `requestAnimationFrame` 不觸發、鏈式 setTimeout 被節流、截圖常逾時、`innerWidth/Height` 可能為 0。
- 驗證一律用 **DOM / 像素斷言**（`getImageData` 取樣、元素數量、computed style），不要「等它自己跑」、不要依賴截圖。

**測試掛鉤 `window.__board`** 匯出：`state, floodFill, placeStamp, loadColoringPage, loadEmojiPage, emojiColoringPage, toLineArt, renderColoringTab, pushUndo, doUndo, doRedo, canvas, ctx, PAGES, STAMP_SETS, EMOJI_PAGE_SETS, TOTAL_PAGES`。

**常用模擬手法**：
```js
// 筆畫：座標用 getBoundingClientRect 映射（0×0 視窗下 rect 即 1200×800，映射仍正確）
const rect = cv.getBoundingClientRect();
const ev = (t,x,y) => cv.dispatchEvent(new PointerEvent(t,{pointerId:1,
  clientX:rect.left+x/1200*rect.width, clientY:rect.top+y/800*rect.height,
  bubbles:true, isPrimary:true}));
ev("pointerdown",100,700); ev("pointermove",200,720); ev("pointerup",200,720);

// 照片上傳：canvas.toBlob → File → DataTransfer → input.files + change 事件
```

**每次改動後的驗收清單**：
1. 主控台無錯誤；2. 8 種筆刷畫線無例外；3. 載入線稿 + 填色不漏色到線外；4. 直接點輪廓線 → 線保持黑、旁邊區域上色（吸附）；5. 重複點同色 → 有提示且不吃復原紀錄；6. 復原/重做快速連按無錯亂；7. 印章可蓋；8. 照片端對端（選檔→預覽有線條→套用→油漆桶自動選取）；9. 作品集收藏/載回/刪除；10. 放大 ×1.6/×2.4 + 移動工具平移、回 ×1 歸位。

## 5. 發佈流程（全自動，勿要求使用者手動登入）

1. `git add` → `git commit`（訊息用繁中、重點條列）。
2. `git push origin master` — Windows 認證管理員（credential.helper=manager）已存 `sancola1219-collab` 的 PAT，push 自動生效。
3. 輪詢 `https://sancola1219-collab.github.io/drawing-board/` 直到 200 且內容含新版特徵字串（約 30–90 秒）。
4. 若要**建新 repo / 開 Pages**：以 `git credential fill` 取回 PAT 後呼叫 GitHub REST API。
   注意：PowerShell 5.1 用管線餵 stdin 給原生 CLI 會壞 → 用 `cmd /c "git credential fill < in.txt > out.txt"`，取完 password= 後**覆寫並刪除暫存檔**，不要把 token 印到輸出。
5. 這台機器 **`gh` CLI 不在 PATH、`python` 指令無效** — 不要使用；有 Node 24。

## 6. 使用者偏好與協作慣例

- 溝通、commit、文件一律繁體中文（台灣）。
- 筆記/文件規則：一事一檔、開頭一行摘要、記「為什麼重要」、已在 repo/對話中的事不重複存、優先更新既有檔、發現錯誤即刪。
- 改動前先讀本文件；改動後跑 §4 驗收清單；完成後 push 並確認線上更新。
- 大改動先在對話說明方案重點再動手。

## 7. 已知限制與 roadmap 候選

- 照片線稿細節受 600×400 處理解析度限制（換取預覽一致與粗線）。
- 手繪經典線稿在 `PAGES` 陣列（每張一個 `draw(c)` 函式，座標基準 1200×800，線寬 7）；新增場景照同格式加函式即可。
- 候選功能：圖層系統、筆刷壓感（`PointerEvent.pressure` 觸控筆可用）、更多手繪場景、選取/移動局部、匯出 SVG、PWA 離線快取、音效開關。

## 8. 歷史脈絡（為什麼長這樣）

- 2026-07-04 首版：可愛風、10 張手繪著色圖。同日擴充：線稿轉換引擎 + 469 張著色圖 + 照片變線稿；多代理審查修正 10 項問題（油漆桶吸附、快照式復原、預熱時機、觸控目標、手機放大等）。
- 2026-07-05 質感改版：整體改為深色高級質感（使用者不要可愛風），emoji 按鈕全面換成 SVG 線條圖示，文案去語氣詞。**功能與引擎不變，只動皮膚與文案。**
