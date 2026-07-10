# 自由畫板 Atelier — 給 AI 助手的專案說明

**動手前必讀 [docs/HANDOFF.md](docs/HANDOFF.md)**（完整架構地圖、不變式、測試與發佈流程）。本檔只列最關鍵的規則；`AGENTS.md` 是本檔的鏡像（給 Codex），兩檔內容需同步修改。

## 專案本質
- 單檔純前端畫板：全部程式在 `index.html`（CSS+HTML+JS），**零依賴、零建置**，不可引入 CDN/npm。
- 線上：https://sancola1219-collab.github.io/drawing-board/（repo `sancola1219-collab/drawing-board`，master 根目錄 Pages）。
- 本機執行：直接開 `index.html`，或 `node .claude/serve.js`（讀 PORT 環境變數）。

## 絕對規則
1. **高級質感、不要可愛風**：深色主題、金銅色 `--gold:#c8a468` 唯一點綴、SVG 線條圖示（`ICONS` 物件）、文案不用「！囉吧」語氣詞。設計 token 都在 CSS `:root`。
2. 介面/commit/文件一律繁體中文（台灣）。
3. 線稿轉換（`toLineArt`/`photoToColoring`）必須輸出二值圖（線 34、底 255），油漆桶填色依賴這點；照片走「區域分割」不是邊緣偵測（開放輪廓會漏色）。手繪閉合形狀用 `P.blob`。
3b. **公開網站不放官方版權角色**（三麗鷗/任天堂/迪士尼…）：主題著色系列一律原創致敬風格；官方角色引導使用者用「照片變線稿」自行轉換。
4. 復原系統用離屏 canvas 快照 + 同步 restore（`snapshotCanvas`/`pushUndo`）；勿改回 toDataURL 或非同步 Image 還原。
4b. 畫布是雙層：`#refBoard`（素描參考層，在下）+ `#board`（作畫層，背景透明，在上）；畫紙底色設在 `#canvasWrap`。素描參考層不進復原、不被 floodFill 讀取。
5. 油漆桶：先快照 → floodFill → 成功才 pushUndo；失敗給 toast。
6. `fitCanvas` 的 0 尺寸守衛與 `ResizeObserver` 不可移除（隱藏視窗載入時容器尺寸可能是 0）。
7. 縮圖生成只能用 MessageChannel 泵（隱藏分頁 setTimeout 會被節流）；預熱只在第一次開著色本時啟動。

## 測試（這台機器瀏覽器常 hidden／0×0）
- 用 `window.__board` 掛鉤 + DOM/像素斷言驗證；不要依賴截圖、rAF 或計時器。
- 筆畫用 `PointerEvent` 模擬（座標按 getBoundingClientRect 映射）；照片流程用 DataTransfer 塞 `input.files`。
- 驗收清單見 HANDOFF §4（10 項）。

## 發佈
`git push origin master` 即可（Windows 認證管理員已存 sancola1219-collab 的 PAT，全自動、勿要求使用者登入）；push 後輪詢線上網址確認更新。`gh` CLI 與 `python` 不可用；建新 repo 用 `git credential fill` 取 PAT 呼叫 GitHub API（PS5.1 要用 `cmd /c "... < in > out"` 重導向）。
