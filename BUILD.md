# BUILD — 施工順序

你是實作者。照階段做，**每階段做完先跑 `ACCEPTANCE.md` 同編號那節**，過了才做下一階段。
契約看 `SPEC.md`，卡住先查 `PITFALLS.md`。

規模參考（我自己那版）：擷取器 ~600 行 Python、外掛 ~600 行 JS（單檔、不用編譯）、樣式 ~130 行 CSS、
擴充背景程式 ~140 行 JS。**寫超過這個量的兩倍就是走偏了**，停下來回看規格。

階段 1–3 蓋完就能用（命令列收藏＋牆）。4 之後是右鍵收藏與自動化。

---

## 階段 0：環境

```
collection-wall/            ← 你的專案
  plugin/     main.js  manifest.json  styles.css
  capture/    capture.py
  extension/  manifest.json  background.js  icon16/48/128.png
```

- **先開一個測試用的空保管庫**，不要一開始就對著你真正的保管庫蓋。驗收全過再換
- `plugin/` 用 symlink 掛進 `<測試保管庫>/.obsidian/plugins/collection-wall/`，改完不用複製
- 確認 `python3 --version` ≥ 3.9、`openssl version` 有輸出；要做影片封面的話 `ffmpeg -version`
- 外掛用純 CommonJS 單檔（`module.exports = class extends obsidian.Plugin`），不用打包工具。
  `manifest.json` 一開始就寫 `"isDesktopOnly": true`

---

## 階段 1：筆記格式與資料夾（先定格式，還不寫程式）

照 `SPEC.md`「筆記格式」**手寫三則**收藏到 `<測試保管庫>/收藏牆/`：

1. 單一分類、有封面、有 `w`/`h`
2. 兩個分類（`kind` 是清單）、有封面、沒有 `w`/`h`
3. 沒封面，標 `needs_cover: true`

封面圖手動放進 `收藏牆/_covers/`。在 Obsidian 的屬性面板確認三則的欄位都被正確讀出（不是一整塊紅字）。

這一步的用意：之後擷取器、外掛、分類器全部圍著這個格式轉。**格式在這裡定死，後面不准各自發明欄位。**

---

## 階段 2：擷取器 CLI

零相依，只用標準函式庫（`urllib`、`json`、`re`、`html`、`hashlib`、`ipaddress`、`socket`）。

1. **設定**：從環境變數讀 `VAULT_PATH`、`WALL_FOLDER`（外掛呼叫時會傳），沒有就報錯退出，不准寫死路徑
2. **抓頁面**：自訂 `User-Agent`；`urllib` 預設不認 308 轉址，要自己補 `http_error_308`
3. **讀 meta**：`og:title`／`twitter:title`／`<title>`、`og:description`、`og:image` 系列。用 `html.unescape`
4. **封面**：照 `SPEC.md`「封面來源的順序」逐一試，**每個候選先過安全規則第 6 條**，存進 `_covers/`
5. **長寬**：讀圖檔標頭拿 `w`/`h`。JPEG（找 SOF 區段）、PNG（IHDR）、WebP、GIF 用標準函式庫幾十行就能讀；
   有 Pillow 就用 Pillow。讀不到就不寫，牆會自己量（只是慢）
6. **去重**：清網址 → 掃 `WALL_FOLDER` 底下每則筆記前 1200 字的 `url:` → 撞到就回 `dup: true`，不建新的
7. **寫筆記**：照 `SPEC.md`「寫 frontmatter 的規則」。依第一個分類歸進子資料夾
8. **`--import`**：吃 JSON 檔，呼叫同一個 `collect()`，最後一行印結果 JSON；出錯也印一行 `ok:false`
9. **`--adopt <資料夾>`**：既有筆記補標籤、分類、封面、長寬。**只 patch，不重寫正文**

這一階段的安全檢查（scheme 白名單、內網位址、每跳轉址重驗、大小上限）就要做，不要留到階段 4——
擷取器在命令列也會替你去抓網址。

選配 `--current-tab`（macOS）：用 `osascript` 對 Chrome 前景分頁 `execute javascript`，跑一段跟擴充 `extract()`
同邏輯的 JS。要使用者開 Chrome「顯示 › 開發人員 › 允許 Apple 事件的 JavaScript」。
有了擴充之後這條就是備援，跳過也行。AppleScript 的跳脫地獄見 `PITFALLS.md`。

---

## 階段 3：外掛——瀑布流面板＋篩選

1. **主入口是面板**（`ItemView`），不是筆記裡的程式碼區塊。左側欄圖示＋指令「打開收藏牆」。
   程式碼區塊（`收藏牆` / `collection-wall`）可以當附加用法，參數 `folder/tag/source/kind/gap/min`
2. **收集**：`app.vault.getMarkdownFiles()` → `metadataCache.getFileCache(f).frontmatter` → 有 `TAG` 的 →
   解析 `cover` 的 wikilink → `metadataCache.getFirstLinkpathDest(link, f.path)` → `vault.getResourcePath()` 當 `<img src>`
3. **排版**：照 `SPEC.md`「瀑布流參數」。關鍵是「放進最矮的那一欄」，以及**先一次寫完所有寬高、再一次讀所有高度**：

   ```js
   for (const it of items) { it.el.style.width = colW + 'px'; it.img.style.height = Math.round(colW * it.ratio) + 'px'; }
   const hs = items.map((it) => it.el.offsetHeight);          // 一次讀完，避免逐張觸發重排
   const colH = new Array(cols).fill(0);
   items.forEach((it, i) => {
     let c = 0; for (let j = 1; j < cols; j++) if (colH[j] < colH[c]) c = j;
     it.el.style.left = Math.round(c * (colW + gap)) + 'px';
     it.el.style.top  = Math.round(colH[c]) + 'px';
     colH[c] += hs[i] + gap;
   });
   ```

   卡片 `position: absolute`，容器高度設成最高那欄。面板剛渲染時寬度可能是 0，用 `requestAnimationFrame` 重試（上限 60 次）
4. **篩選列**：兩排膠囊——「來源」「分類」，各有「全部」。值從目前上牆的筆記動態收集，`kind` 單值清單都吃
5. **卡片互動**：點＝開筆記；⌘／⌥ 點＝開原網址；右鍵選單：開啟筆記／開啟原始網址／複製網址／在系統檔案總管顯示封面／
   刪除（確認對話框，`fileManager.trashFile` 筆記與封面一起丟，可復原）
6. **指令「新增收藏」**：對話框填名稱、點選既有分類（可複選、可打新分類）、備註 → 建一則空殼筆記，提示「把圖片或影片拖進來」
7. **設定頁**：`SPEC.md`「設定值」整張表

樣式：卡片圓角、無框、只用陰影分層；顏色全用 Obsidian 的 CSS 變數（`--background-primary-alt`、`--text-muted`…），深色主題自動跟。

---

## 階段 4：本機中繼＋Chrome 擴充右鍵收藏

### 4a. 外掛裡的伺服器

`require('http').createServer(...)`，`listen(PORT, '127.0.0.1')`。每個請求的處理順序**不准調換**：

```js
const origin = req.headers.origin;                       // undefined＝沒帶；'null' 是字串，要擋
const okHost = [`127.0.0.1:${PORT}`, `localhost:${PORT}`].includes(req.headers.host || '');
if (!okHost || (origin !== undefined && origin !== `chrome-extension://${EXTENSION_ID}`))
  return send(403, { ok: false, msg: '只收收藏牆擴充送來的請求' });
// 之後才看 /ping、/collect
```

然後：讀 body（邊讀邊算長度，超過 1 MB 回 413 並 `req.destroy()`）→ `JSON.parse` → 只取五個欄位驗型別長度 →
寫暫存檔（隨機檔名、`0600`）→ `execFile(PYTHON_PATH, [TOOL_PATH, '--import', tmp], { timeout: 180000 })` →
刪暫存 → 取 stdout 最後一行 `{` 開頭的 JSON 回給擴充。成功時 Obsidian 也跳一個 `Notice`。

`onunload` 一定要 `server.close()`；`listen` 失敗（`EADDRINUSE`）要跳 `Notice` 講「右鍵收藏會接不到」，不能只寫 console。
開門成功時寫一行 log（到外掛資料夾或 `os.tmpdir()`），讓人不開開發者工具也能確認門有沒有開。

**先跑 `ACCEPTANCE.md` 第 4 節的安全測試，全過才接擴充。**

### 4b. 固定擴充 ID

```bash
mkdir -p ~/.config/collection-wall
openssl genrsa 2048 | openssl pkcs8 -topk8 -nocrypt -out ~/.config/collection-wall/ext.pem
openssl rsa -in ~/.config/collection-wall/ext.pem -pubout -outform DER | openssl base64 -A   # → manifest 的 "key"
openssl rsa -in ~/.config/collection-wall/ext.pem -pubout -outform DER | shasum -a 256 | head -c 32 | tr 0-9a-f a-p   # → 擴充 ID
```

把 ID 填進外掛設定頁。到 `chrome://extensions` 載入後，顯示的 ID 要跟算出來的一樣。
**私鑰留在 `~/.config/`，不准放進 `extension/`、不准進 repo。**

### 4c. 擴充

照 `SPEC.md`「擴充（Manifest V3）」：

- `onInstalled` 裡先 `contextMenus.removeAll()` 再 `create`（重載擴充會再進來一次）
- 右鍵或點圖示 → 檢查分頁網址是 http/https → 浮「收藏中…」→ `executeScript(extract)` →
  點在圖上就用 `info.srcUrl` → `fetch('http://127.0.0.1:PORT/collect', { method: 'POST', headers: {'Content-Type':'application/json'}, body })`
- 連不到：浮紅字「接不到 Obsidian。Obsidian 要開著、收藏牆外掛要啟用」
- 頁面浮字：`position:fixed` 置底置中、`z-index` 拉滿、`pointer-events:none`，成功 3.5 秒、失敗 9 秒消失

---

## 階段 5：自動封面（影片抽幀）

兩件不該讓使用者記得做的事：

1. **新筆記自動蓋屬性**：`vault.on('create')` → 延遲約 120 ms（等檔案寫完）→ 只處理 `WALL_FOLDER` 底下、
   非底線開頭、**內容是空的** `.md` → 寫入 frontmatter 骨架，`kind`＝所在子資料夾名
2. **拖了媒體自動生封面**：`metadataCache.on('changed')` → 有 `TAG`、沒 `cover`、正文有 `![[…]]` →
   圖：`cover` 直接指過去，不複製；影片：照 `SPEC.md`「影片抽封面參數」抽一格 →
   量長寬（`new Image()`，5 秒逾時）→ `fileManager.processFrontMatter` 寫 `cover`、`w`、`h`

同一則筆記處理中要上鎖（`Set`），處理完冷卻 5 秒再解鎖——存檔會連發 `changed`，不鎖會同時抽好幾次。

另做指令「補上缺少的封面」：一次掃完所有缺封面的收藏，結束時回報成功幾則、失敗幾則。

擷取器的 `--adopt` 也要能抽影片封面（同一組取樣點），讓命令列一次納入整個資料夾。

---

## 階段 6（選配）：AI 看圖分類

照 `SPEC.md`「分類器介面」與安全規則第 7 條：

1. 設定頁加開關（預設關）＋指令欄位＋那句「會把封面圖、標題、描述送出去」的說明
2. 擷取器 `classify()`：組分類清單 → 組提示 → 子程序（提示走 stdin、補 PATH、清 `CLAUDE*`、唯讀沙盒、120 秒逾時）→
   只留清單裡的名字 → 最多 2 個 → 空的就 `未分類`
3. 只在「使用者沒指定分類」且「設定開著」時才叫
4. 擴充送 `auto: true`，但外掛以自己的設定為準

**分類邏輯只寫在擷取器一份**，外掛、擴充、命令列共用。不要在 JS 裡再寫一次。

先用五張你自己的封面試：每張跑三次，看結果穩不穩。不穩通常是分類說明寫得太像，改 `_categories.md` 比改提示有效。
