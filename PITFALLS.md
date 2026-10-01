# PITFALLS — 靜默地雷

共同點：**它們幾乎都不會噴錯**。先看這份，不要自己重推一遍。

## 架構與權限

### 1. 瀏覽器起的程序碰不到你的保管庫（macOS TCC，完全靜默）

這條決定了整個架構，所以寫完整。

**直覺做法**：Chrome 擴充 → 原生訊息主機（native messaging host）→ 一支 Python 直接寫進保管庫。第一版就是這樣蓋的。

**症狀**：右鍵之後 Chrome 只回一句「Native host has exited」。主機腳本自己的 log 一行都沒有，像是根本沒被叫起來。

**怎麼查到的**：
1. 主機腳本放在受保護的資料夾裡（例如 `~/Downloads/`）——它連被讀進來都失敗，所以什麼 log 也寫不出來
2. 在受保護位置**之外**（例如 `~/.local/bin/`）放一支小起跑器，把 stderr 導到 `/tmp`，才終於看到：`Operation not permitted`
3. 換成用 launchd 起同一支程式，症狀一模一樣。所以不是 Chrome 的問題，是「誰起的程序」的問題
4. `tccutil reset` 重設權限：沒用。系統**從頭到尾沒跳過任何詢問視窗**

**原因**：macOS 的 TCC 把 `~/Downloads`、`~/Desktop`、`~/Documents`，以及 iCloud 雲碟（`~/Library/Mobile Documents/`，
iCloud 同步的 Obsidian 保管庫就在這底下）列為受保護位置。存取時系統要問「某個 App」要不要允許——
但由 Chrome 起的原生訊息主機、由 launchd 起的背景程序，**沒有一個可以被問的身分**，系統就直接拒絕，不問、不記。

**推理**：那誰碰得到？使用者自己打開的 App 有身分，第一次存取會跳詢問，按允許之後就記住。
Obsidian 本來就要讀寫保管庫，權限早就有了；而它用 `child_process` 起的子程序跟著它的身分走——
在 Obsidian 裡 `execFile('python3', [...])` 去寫保管庫，一次就過。

**結論**：把「寫檔的那一端」放進 Obsidian 外掛。外掛開一個只聽 `127.0.0.1` 的小伺服器，擴充的背景程式 POST 給它，
它再叫擷取器。代價是 **Obsidian 要開著**才收得到，所以擴充連不上時要明講「接不到 Obsidian」。

**通則**：以後任何由瀏覽器、launchd、cron 起的東西要碰受保護資料夾，直接找一個使用者會開的 App 當中繼，
別在權限設定上耗時間。

### 2. 書籤小工具（bookmarklet）那條路是死的

在頁面裡 `fetch('http://127.0.0.1:…')`：很多網站的 CSP（`connect-src`）會擋掉頁面內所有連本機的請求，
`fetch`、`XMLHttpRequest` 都被拒。改用 `window.open` 導頁繞過，又撞彈出視窗封鎖。
→ 請求要由**擴充的背景程式**發。它不在頁面的 CSP 管轄範圍內，頁面只負責被讀 DOM。

### 3. CORS 設成 `*` ＝ 任何網頁都能往你的保管庫塞東西

最早的版本回 `Access-Control-Allow-Origin: *`、不驗來源。實測：任何網頁送一個 `Content-Type: text/plain` 的 POST，
**不觸發 preflight**，伺服器照樣解析、照樣建筆記、照樣叫電腦去抓指定網址、照樣花分類器額度。網頁讀不到回應，但事情已經做完了。
→ `SPEC.md` 安全規則 1–3。驗 Origin＋驗 Host，缺一不可。

### 4. 私鑰放在擴充資料夾裡，Chrome 會警告

載入未封裝擴充時 Chrome 會抱怨資料夾裡有私鑰。而且那個資料夾很容易整包進 repo。
→ 私鑰放 `~/.config/` 之類的地方，`.gitignore` 擋 `*.pem`。

### 5. 從 GUI 起的子程序 PATH 很短

Obsidian、Chrome 起的子程序拿到的 `PATH` 通常只有 `/usr/bin:/bin`，Homebrew 裝的 `ffmpeg`、各家 CLI 全部「找不到」。
→ 用絕對路徑（設定頁可填），或在子程序環境前面補 `/opt/homebrew/bin:/usr/local/bin`。

### 6. 開兩個保管庫都啟用外掛，第二個搶不到埠

埠是固定的（擴充要知道打哪裡），第二個 `listen` 會 `EADDRINUSE`。
→ 失敗要跳 `Notice`，不能只寫 console；右鍵收藏的筆記會進「先開的那個」保管庫。

## 擷取

### 7. 頁面上最大的圖，常常不是這一則的圖

有些社群網站單篇頁面下方掛滿推薦內容，「頁面上面積最大的 `<img>`」常是別人的貼文——曾因此整批抓錯封面。
→ 以 `og:image` 認定是哪一則。想要高解析，就拿 `og:image` 檔名裡的媒體 ID 回頭在頁面找同一則的大圖；找不到就用 `og:image` 本身。
**正確優先於解析度。**

### 8. `urllib` 不認 308 轉址

只處理 301/302/303/307。很多短網址服務用 308 → 直接 `HTTPError`。
→ 自訂 `HTTPRedirectHandler.http_error_308`，轉給 301 的處理。（轉址的每一跳仍要過安全檢查。）

### 9. `urllib` 預設會開 `file://`

`build_opener()` 內建 `FileHandler`。不檢查 scheme 的話，一個 `file://` 的「封面網址」就會把本機檔案讀進來寫到保管庫。
→ 安全規則第 6 條：只准 http/https，內網位址也擋。

### 10. 圖床網址會過期

很多圖床網址帶簽章與到期時間，牆上直接引用網址，幾天後整面開天窗。→ 封面一律下載到 `_covers/`。

### 11. 追蹤參數讓同一頁變兩張卡

從 App 分享出來的網址常帶 `utm_*`、`si`、`igsh` 之類的參數。→ 去重前先清網址，清完的版本才寫進 `url`。

### 12. 影片站的公開縮圖很小

Vimeo oEmbed 不帶 `width` 只給約 295px；YouTube oEmbed 給的是 `hqdefault`（480px）。
→ Vimeo 帶 `width=1280`；YouTube 先試 `maxresdefault`，失敗再退回。

## AppleScript 讀分頁（選配的 `--current-tab`）

### 13. 在 `tell application "Google Chrome"` 裡，`tab` 被 Chrome 的分頁類別搶走

拿 `tab` 當分隔符，輸出的是字面的 `tab`。→ 用 `|@|` 這種不會撞的字串。

### 14. 注入的 JS 裡不能寫 `\n`、`\s`

會先被 AppleScript 的字串跳脫吃掉。→ 換行用 `String.fromCharCode(10)`，正規表示式避開反斜線。
另外有些網站的 `og:title` 是整段含換行的貼文，**換行要在 JS 端先壓平**，不然逐行解析會把圖片欄位切掉。

### 15. 背景分頁 Chrome 不載入圖片

讀到的 `naturalWidth` 全是 0，挑不出大圖。→ 讀之前先 `activate`，等半秒。

## 筆記與 YAML

### 16. Python 的 `\s` 會吃掉換行

`^comment:\s*(.*)$` 在備註是空的時候，會一路吃到下一行，抓到 `w: 1216` 當備註，寫出無效 YAML——
**十幾張卡一起從牆上消失**，Obsidian 不報錯，只是那些筆記突然沒有屬性了。
→ 屬性行的空白一律 `[ \t]*`。

### 17. 改既有筆記只能 patch

整則重寫會蓋掉使用者寫的心得。補 `w`/`h` 前沒先刪舊的那兩行，會出現重複鍵，整則 frontmatter 失效。
→ 只動自己管的那幾行；外掛端用 `fileManager.processFrontMatter`。

### 18. 沒有 `w`/`h`，瀑布流要等每張圖載完才能排

畫面先疊在一起再跳開。→ 擷取時就讀圖檔標頭寫進 frontmatter；量不到才退回載圖量，並設 4 秒上限。

### 19. 一則多分類，只能待一個資料夾

→ 第一個分類決定資料夾，篩選只看屬性。使用者改了分類不會自動搬家，要搬另做指令。

## 影片封面

### 20. 沒有 ffprobe，讀片長會靜靜失敗

很多人只裝了 ffmpeg。靠 ffprobe 讀片長的寫法會失敗後退回「抽第 1 秒」——常常是黑幀。
→ 固定時間梯九個點（2…140 秒），超過片長的點自然沒產出、略過；挑檔案最大的那格。

### 21. 同步呼叫 ffmpeg 會把 Obsidian 凍住

`execFileSync` 抽九格，介面卡五到十秒，像當機。→ 一律非同步 `execFile`，九格並行。

## Obsidian 開發

### 22. 改了 `main.js` 沒生效

Obsidian 只在外掛載入時讀程式碼。→ 設定 › 社群外掛，把它關掉再打開。或者整個 Obsidian 結束再開。

### 23. 不要用 AppleScript 模擬按鍵來重載 Obsidian

想用 `osascript` 送 ⌘P、打「重新載入」、按 Enter——**會出事**：
- 輸入法是中文時，`keystroke` 打出去的是注音符號，**直接打進你當時開著的筆記**
- `activate` 之後焦點不一定真的到 Obsidian。另一個 App 搶了前景時，⌘A ⌘V 會貼進**那個 App 的文件**
- 開著兩個保管庫視窗時，按鍵可能落在另一個
- Obsidian 是 Electron，輔助使用 API 讀不到它的元件，沒辦法改用「點按鈕」

→ 要重載就 `osascript -e 'tell application "Obsidian" to quit'`，再 `open -a Obsidian`。全程不碰鍵盤。

### 24. 指令面板吃介面語言

中文介面下指令叫中文名，打英文找不到；面板還會記住上次查詢。又一個不該用按鍵自動化的理由。

### 25. 牆做成筆記裡的程式碼區塊，看起來會像壞掉

即時預覽模式下，游標停在程式碼區塊裡就顯示原始碼。→ 主入口做成獨立面板（`ItemView`），程式碼區塊只當附加用法。

## 擴充

### 26. 重載擴充後右鍵選單報「id 重複」

`onInstalled` 每次重載都會再跑。→ 先 `contextMenus.removeAll()` 再 `create`。

### 27. 有些頁面注入不了腳本

`chrome://`、PDF 檢視器、擴充商店頁會讓 `executeScript` 丟例外。→ 接住，只送網址，讓擷取器自己去探。

### 28. 系統通知可能被擋

使用者關了 Chrome 通知，右鍵完什麼都沒看到，以為沒反應又按一次。→ 主要回報走**頁面底部浮字**，通知只是備援。

## 分類器

### 29. `claude -p --allowedTools Read "提示"` 會把提示吞掉

`--allowedTools` 吃不定數量的參數，後面的提示被當成工具名，回「Input must be provided」。
→ 提示一律走 stdin。其他 CLI 也照這樣做，順便避開參數長度與跳脫問題。

### 30. 在 agent session 裡呼叫另一個 agent CLI

從 Claude Code 裡（或測試時在它的終端機裡）起 `claude` 子程序，會繼承一堆 `CLAUDE*` 環境變數，行為跟你手動跑的不一樣。
→ 起子程序前把 `CLAUDE` 開頭的環境變數清掉。

### 31. 分類說明檔的分隔符只能用等號

說明檔裡寫了一行規則「- 新增分類：加一行就好」，冒號被當分隔符，「新增分類」就變成一個分類，分類器開始選它。
→ 只認 `＝`／`=`。冒號留給說明內文。

### 32. 模型會回解釋，不只回名字

「我認為這是排版，因為……」。→ 切開之後只留清單裡有的名字，其他全丟；空了就 `未分類`。
