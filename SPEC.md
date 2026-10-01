# SPEC — 契約

## 設定值（一律走設定，不准寫死）

外掛要有一個**設定頁**（`PluginSettingTab`）讓使用者填下面這些；存在外掛自己的 `data.json`。
擷取器不自己讀設定：外掛呼叫它時用參數或環境變數把值傳過去；單獨在終端機跑時讀環境變數。

| 名稱 | 預設 | 說明 |
|---|---|---|
| `VAULT_PATH` | 外掛自己取（`app.vault.adapter.getBasePath()`） | 保管庫絕對路徑。只有擷取器需要，外掛傳給它 |
| `WALL_FOLDER` | `收藏牆` | 收藏放哪個資料夾（相對保管庫） |
| `COVERS_FOLDER` | `<WALL_FOLDER>/_covers` | 封面圖 |
| `TAG` | `收藏` | 有這個標籤的筆記才上牆 |
| `TOOL_PATH` | 空（必填） | 擷取器腳本的絕對路徑 |
| `PYTHON_PATH` | `/usr/bin/python3` | 外掛用哪個 python 跑擷取器 |
| `PORT` | `41577`（任一個沒人用的高位埠） | 本機中繼的埠。**擴充那邊要填同一個** |
| `EXTENSION_ID` | 空（必填才開伺服器） | 你那支擴充的固定 ID，32 個 a–p 字母 |
| `FFMPEG_PATH` | 自動找 | 依序試 `/opt/homebrew/bin`、`/usr/local/bin`、`~/.local/bin`、`/usr/bin`，都沒有才請使用者填 |
| `CLASSIFY` | `false` | AI 分類開關（見「分類器介面」） |
| `CLASSIFIER_CMD` | 空 | 分類器指令（參數陣列，不是 shell 字串） |
| `GAP` / `MIN_COL` | `14` / `200` | 瀑布流間距與最小欄寬（px） |

`TOOL_PATH`、`EXTENSION_ID` 沒填時：牆照常顯示，伺服器不開，設定頁上方明講「右鍵收藏還沒接上」。
改了 `PORT` 或 `EXTENSION_ID` 要當場關掉舊伺服器、開新的，不用重開 Obsidian。

外掛 `manifest.json` 必須 `"isDesktopOnly": true`（用到 Node 的 `http`、`child_process`、`fs`）。

## 資料夾結構

```
<VAULT_PATH>/
  <WALL_FOLDER>/
    _covers/                 所有封面圖（下載的、影片抽的）
    _categories.md           選配：分類說明，給分類器看
    排版/                    依「第一個分類」分子資料夾
      某則收藏.md
    配色/
    未分類/                  分類器挑不出來的
```

- 底線開頭的檔案與資料夾是輔助檔，**不是收藏**，掃描時略過
- 一則可以屬於多個分類，但只能待一個資料夾 → **第一個分類決定資料夾**。牆的篩選看屬性不看資料夾
- 牆掃的是「整個保管庫裡有 `TAG` 標籤的筆記」，所以使用者把筆記搬去別處也照樣上牆

## 筆記格式

```yaml
---
tags:
  - 收藏
source: Example
kind:
  - 排版
  - 配色
url: "https://example.com/posts/123"
cover: "[[example-post-a1b2c3.jpg]]"
saved: 2026-10-02
comment:
w: 1080
h: 1350
---

> 頁面的 og:description（有才寫；換行壓平、`<` `>` `[` `]` `!` 跳脫後才寫，見下方規則）

[原始連結](https://example.com/posts/123)
```

| 欄位 | 型別 | 必填 | 說明 |
|---|---|---|---|
| `tags` | 清單 | ✓ | 含 `TAG`。上牆的唯一判準 |
| `source` | 字串 | | 來源名，篩選列第一排。由網域對照表決定，對不到寫 `其他` |
| `kind` | 字串或清單 | ✓ | 分類。單一個寫字串，多個寫清單；**讀的時候兩種都要吃** |
| `url` | 字串 | | 清掉追蹤參數後的網址。**去重的鍵** |
| `cover` | wikilink 字串 | | `"[[檔名]]"`，指向 `_covers/` 或保管庫任何一張圖。沒有就不上牆 |
| `saved` | `YYYY-MM-DD` | ✓ | 牆的排序：新的在前，同日再依標題 |
| `comment` | 字串 | | 備註，卡片 hover 顯示 |
| `w`, `h` | 整數 | | 封面原始像素。**擷取時就寫**，牆才能在圖載入前排版 |
| `needs_cover` | `true` | | 抓不到封面時標記，方便之後補 |

### 寫 frontmatter 的規則

- 字串值一律經過跳脫：網址、標題、備註**可能含引號、冒號、換行**。換行壓成空白，再用 JSON 字串格式包雙引號（YAML 吃得下）
- 只准新增或改自己管的欄位，**不准重寫整則筆記**。補 `w`/`h` 前先刪掉舊的那兩行，否則 YAML 出現重複鍵整則失效
- 用正規表示式比對屬性行時，空白只准 `[ \t]*`，不准 `\s`（見 `PITFALLS.md`）
- 檔名＝標題去掉 `/ \ : * ? " < > | # ^ [ ]`、去掉開頭的 `.`（不然變隱藏檔，Obsidian 看不到）、壓空白、截 60 字；同名加 ` (2)`、` (3)`
- 封面檔名＝`<標題先過上一條同一套去字元，再截 28 字、空白換連字號>-<圖片網址 sha1 前 6 碼>.<副檔名>`；副檔名**只准白名單**對照 Content-Type：`image/jpeg→.jpg`、`image/png→.png`、`image/webp→.webp`、`image/gif→.gif`、`image/avif→.avif`，其他一律跳過這個候選
- 正文裡的 og:description 是網頁作者寫的：壓成一行，`<` `>` 轉成 `&lt;` `&gt;`，`[` `]` `!` 前面加反斜線——否則對方能塞進遠端圖片（你打開筆記就通知對方）、`![[…]]` 嵌入保管庫裡別的檔案
- **所有寫入（筆記、封面、影片抽幀）在寫之前都要 realpath，確認落在 `WALL_FOLDER`／封面資料夾底下**；不在就中止

### 來源對照表

一張「網域片段 → 來源名, 預設分類」的表，放在擷取器頂端讓使用者自己改：

```
("example.com",   "Example", "參考"),
("video.example", "ExampleVideo", "攝影"),
```

對不到的：來源 `其他`、分類 `參考`。

### 網址清理（去重前做）

刪掉 `utm_*` 與常見追蹤參數（`fbclid`、`gclid`、`si`、`ref`、`ref_src`，以及你常收的網站自己的分享參數）、刪掉 `#片段`。
不清的話，同一頁從不同地方點進來就變兩張卡。

## 擷取器命令列

| 用法 | 行為 |
|---|---|
| `capture.py <網址> [網址…]` | 直接收。自己抓頁面讀 og 標籤 |
| `capture.py --kind 排版,配色 --comment 備註 <網址>` | 指定分類（逗號分隔，多值）與備註 |
| `capture.py --import <json 檔>` | 給外掛用。吃擴充送來的 JSON，**最後一行印結果 JSON** |
| `capture.py --adopt <資料夾>` | 把既有筆記納入：補標籤、分類＝資料夾名、從正文第一個嵌入的圖或影片生封面 |
| `capture.py --auto …` | 沒指定分類時叫分類器（`CLASSIFY` 關著時忽略） |
| `capture.py --current-tab`（選配，macOS） | 用 AppleScript 讀 Chrome 前景分頁的 DOM。要開 Chrome「允許 Apple 事件的 JavaScript」 |

`--import` 的最後一行：

```json
{"ok": true, "file": "某則收藏.md", "kinds": ["排版"], "cover": true, "dup": false, "msg": "收進「排版」：某則收藏"}
```

失敗也要印一行 `{"ok": false, "msg": "<錯誤類型>: <訊息>"}` 再以 exit 1 結束。外掛**先看 stdout 最後一行 JSON，再看 exit code**。

旗標名稱可以自取（也可以給中文別名），但 `--import` 的輸入輸出格式照這份。

## 封面來源的順序

1. 擴充送來的 `image`（右鍵點在圖上就是那張）
2. 頁面的 `og:image:secure_url` → `og:image` → `twitter:image`（相對網址要對最終網址 `urljoin`）
3. 公開 oEmbed（影片站常有）：例如 Vimeo 要帶 `width=1280` 才給大圖；YouTube 先試 `maxresdefault` 再退回 `hqdefault`
4. 圖床若有「縮小版／原尺寸」兩種路徑，先試原尺寸，失敗退回原本那張
5. 都沒有 → 筆記照建，`needs_cover: true`。**不做網頁截圖補封面**

每個候選都要先過「安全規則」第 6 條；下載後小於 2000 bytes 的當成佔位圖跳過。

## HTTP 契約（外掛內的本機中繼）

只綁 `127.0.0.1:<PORT>`。所有請求**先過安全檢查**（下一節），才看路徑。
**會做事的端點只准 POST**：網頁用 `<img>`、導頁、`<link>` 發出的 GET 不帶 Origin，Origin 檢查會放行它們，所以 GET 只能是 `/ping` 這種沒有副作用的。

| 方法 | 路徑 | 進 | 出 |
|---|---|---|---|
| GET | `/ping` | — | `200 {"ok":true,"plugin":"collection-wall"}` |
| POST | `/collect` | 擴充 JSON（下表） | `200` ＋ 擷取器回的那行 JSON |
| 其他 | | | `404 {"ok":false,"msg":"沒有這個路徑"}` |

| 狀況 | 狀態碼 |
|---|---|
| 安全檢查沒過 | `403 {"ok":false,"msg":"只收收藏牆擴充送來的請求"}` |
| body 超過上限 | `413`，並中斷連線 |
| JSON 壞掉／欄位不合 | `400` |
| 擷取器沒回 JSON、逾時 | `500 {"ok":false,"msg":"<stderr 最後 300 字>"}` |

回應一律 `Content-Type: application/json; charset=utf-8`。
同一時間只跑一支擷取器，其餘排隊（右鍵連點時不會同時開十支 python）。

## 擴充送的 JSON

```json
{"url": "https://example.com/posts/123", "title": "頁面標題", "image": "https://cdn.example.com/a.jpg",
 "desc": "og:description", "auto": true}
```

| 欄位 | 上限 | 說明 |
|---|---|---|
| `url` | 2048 字 | 分頁網址。必須 http/https |
| `title` | 110 字 | 已壓平換行 |
| `image` | 2048 字 | 封面候選，可空 |
| `desc` | 400 字 | 可空 |
| `auto` | bool | 擴充要求分類。**伺服器以設定頁的 `CLASSIFY` 為準**，關著就當 false |

外掛只把**這五個欄位**轉給擷取器，其他欄位（例如 `kind`、`comment`）一律丟掉——分類只能由使用者或分類器決定，不能由請求決定。

## 擴充（Manifest V3）

```json
{
  "manifest_version": 3,
  "key": "<你自己的公鑰 base64>",
  "permissions": ["contextMenus", "scripting", "activeTab", "notifications"],
  "host_permissions": ["http://127.0.0.1/*"],
  "background": {"service_worker": "background.js"},
  "action": {"default_title": "收藏這頁"}
}
```

- 右鍵選單 `contexts: ["all"]`；工具列圖示點一下也等於收藏這頁
- 在頁面裡執行 `extract()`（`chrome.scripting.executeScript`），回 `{title, img, desc}`；注入失敗（`chrome://`、PDF、商店頁）就只送網址
- `info.mediaType === "image"` 時用 `info.srcUrl` 當封面
- 請求由**背景程式（service worker）**發，不經頁面
- 回報三管齊下：頁面底部浮一行字（主要，右鍵完一定看得到）、圖示角標 `…`/`✓`/`!`、系統通知（可能被擋，只當備援）
- 送出前在頁面浮「收藏中…」，因為開了分類一張要十幾秒

`extract()` 的判斷順序：`og:image` → 頁面上最大的圖（寬 > 240）→ `video` 的 `poster` → 任何面積 > 10000 的圖。
有些社群網站同一頁會有大量推薦內容，**最大的圖常常不是這一則** → 以 `og:image` 為準；要高解析就拿 `og:image` 檔名裡的媒體 ID
回頭在頁面找同一則的大圖，找不到就用 `og:image` 本身。**正確優先於解析度。**

## 安全規則（全部必做，對應 `ACCEPTANCE.md` 第 4 節）

本機伺服器開著，等於電腦上多了一扇門。瀏覽器裡任何一個分頁都敲得到 `127.0.0.1`，所以門要自己顧。

### 1. 只聽 127.0.0.1，不准 `Access-Control-Allow-Origin: *`

`listen(PORT, "127.0.0.1")`，不准 `0.0.0.0` 或省略。CORS 標頭只回擴充自己的 origin：
`Access-Control-Allow-Origin: chrome-extension://<EXTENSION_ID>`。

### 2. 驗 Origin

| 請求帶的 Origin | 動作 |
|---|---|
| 沒帶 | 放行（本機 curl、腳本——它們本來就能直接跑擷取器） |
| `chrome-extension://<EXTENSION_ID>` | 放行 |
| 其他任何值，**包括字面的 `null`** | `403` |

為什麼這樣就夠、而且非做不可：
- 網頁發出的跨來源請求，瀏覽器**一定會帶 Origin，而且網頁改不了它**。所以「有 Origin 但不是我的擴充」＝某個網頁
- **光靠 CORS 擋不住**。CORS 只阻止網頁「讀回應」，不阻止請求送達。`Content-Type: text/plain` 的 POST 是「簡單請求」，
  **不會觸發 preflight**，body 照樣送到你的伺服器、你照樣 `JSON.parse`、照樣建筆記
- 沙盒 iframe 與 `file://` 頁面送的是 `Origin: null`，它是字串、不是「沒帶」，要擋
- 瀏覽器正在推的「私有網路存取」保護還沒全面生效，**不要依賴它**

### 3. 驗 Host

`Host` 標頭必須**完全等於** `127.0.0.1:<PORT>` 或 `localhost:<PORT>`，否則 `403`。沒帶 Host 也 `403`。

這條擋 DNS rebinding：惡意網域先載入頁面，再把自己的 DNS 改成指向 `127.0.0.1`。
之後那個頁面打「自己的網域」其實打到你的伺服器，而且是**同來源 GET——瀏覽器不帶 Origin**，會穿過第 2 條。
這時唯一露餡的是 `Host: evil.example:<PORT>`。

### 4. 擴充 ID 固定，私鑰不進 repo

擴充 manifest 用 `key` 固定 ID（不然 ID 跟著資料夾路徑變，伺服器的白名單就對不上）。
私鑰 `.pem` 放**擴充資料夾外面、repo 外面**（例如 `~/.config/collection-wall/ext.pem`），`.gitignore` 擋 `*.pem`。
`key` 欄位是公鑰，可以留在 manifest；ID 由它算出來，產法見 `BUILD.md` 階段 4。

### 5. 輸入與子程序

- request body 上限 1 MB，超過回 `413` 並中斷，不要先讀完再判斷
- 只接受擴充 JSON 的五個欄位，逐欄驗型別與長度（上表）
- 子程序一律 `execFile(PYTHON_PATH, [TOOL_PATH, "--import", tmp])`——**參數陣列，不經 shell**。標題、網址不准拼進指令字串
- 擷取器 timeout 180 秒；分類器 timeout 120 秒；ffmpeg 每格 25 秒
- 暫存 JSON 放 `os.tmpdir()`，檔名不可預測（`crypto.randomUUID()`），權限 `0600`，**成功失敗都要刪**
- `kind` 值要能當資料夾名：拒絕含 `/`、`\`、`..`、開頭是 `.` 或 `_` 的值，寫入前確認目標路徑在 `WALL_FOLDER` 底下
- log 只記方法、路徑、Origin、Host、結果，**不記標題、網址、描述**
- **牆上顯示的字（標題、來源、備註）一律用 `setText`／`createEl({ text })`，不准 `innerHTML`**：外掛有 Node 權限，牆上的 XSS 等於能在你電腦上跑任何指令

### 6. 封面與頁面網址：只抓公開的 http/https

擷取器會替你去抓兩種網址：封面圖、（擴充沒給標題或圖時）頁面本身。兩者都要先過這個檢查：

| 檢查 | 不合就 |
|---|---|
| scheme 只准 `http`、`https`（擋 `file:`、`ftp:`、`data:`、其他） | 跳過這個候選 |
| 主機名解析出的**每一個** IP 都不能是：loopback（`127.0.0.0/8`、`::1`）、私有（`10/8`、`172.16/12`、`192.168/16`、`fc00::/7`）、link-local（`169.254/16`、`fe80::/10`）、CGNAT（`100.64/10`）、`0.0.0.0/8`、multicast、保留段 | 跳過 |
| 每一跳轉址都重新檢查上面兩條 | 中止 |
| 回應 Content-Type 要是 `image/*`（抓封面時） | 跳過 |
| 下載上限 20 MB，邊讀邊算 | 中止 |

不做的後果：Python 的 `urllib` 預設會開 `file://`——一個 `"image": "file:///Users/你/.ssh/…"` 就會把本機檔案複製進保管庫，
分類器開著的話還會被當成封面送出去。內網位址則會讓你的電腦替別人去敲路由器、NAS 的管理頁。

⚠️ 用 `ipaddress` 模組判斷：條件寫成 **`not ip.is_global or ip.is_multicast`**；IPv4-mapped IPv6（`::ffff:127.0.0.1`）先取 `ip.ipv4_mapped` 再判斷。
**不要**只用 `is_private`／`is_loopback`／`is_link_local`／`is_reserved` 拼起來——`100.64.0.1`（CGNAT，也是 Tailscale 的網段＝你自己的私人機器）在這四個全是 False，會漏掉。也不要自己寫字串前綴比對。
進階：檢查完到真正連線之間 DNS 會再查一次（DNS rebinding 的空檔）；要補就連線後用 socket 的 `getpeername()` 再驗一次對方 IP，或直接連驗過的那個 IP 並帶 Host 標頭。

### 7. AI 分類要讓使用者知情

- 預設關。設定頁開關旁邊要寫清楚：「開啟後，每收一則會把封面圖、標題、頁面描述送給你設定的模型服務」
- 頁面標題與描述是**不可信的輸入**（別人寫的），會原封不動進提示 → 分類器只准在唯讀沙盒、不給工具權限下跑；
  輸出**只接受分類清單裡有的名字**，其他一律丟掉

## 分類器介面（可換）

規格只要求：

```
輸入：封面圖檔路徑（可能沒有）、標題、描述、來源、分類清單（含每類一句說明）
輸出：0–2 個分類名，最貼切的在前；都不像就空
```

實作是一個子程序：

| 項目 | 規定 |
|---|---|
| 指令 | `CLASSIFIER_CMD`，參數陣列；圖檔路徑以參數傳（或寫進提示請它讀） |
| 提示 | **走 stdin**，不要當成指令參數（見 `PITFALLS.md`） |
| 環境 | 補 `PATH`（GUI 程序拿到的很短）；清掉 `CLAUDE*` 開頭的環境變數 |
| 沙盒 | 唯讀、不給寫檔與網路工具 |
| 輸出解析 | 取最後一則回答，依 `[,，、\s/／]+` 切開，**只留清單裡有的**，最多 2 個 |
| 失敗 | 逾時、非零結束、空回答 → 回空 → 筆記進 `未分類` |

分類清單＝`_categories.md` 列的（順序優先）＋ `WALL_FOLDER` 底下現有子資料夾名（略過 `_`、`.` 開頭與 `未分類`）。

`_categories.md` 格式，只認這種行：

```
- 排版＝版面、字體、網格
- 配色=色彩搭配、色票
- 攝影＝構圖、光線
```

分隔符**只認 `＝` 或 `=`**。冒號留給說明內文用，不能當分隔符。

提示底稿：

```
你在幫一面「設計靈感收藏牆」把剛收進來的一張卡分類。
請直接看附上的封面圖，再判斷。（沒圖時改成：這則沒有封面圖，只能靠文字判斷。）
來源：{source}
標題：{title}
說明：{desc 前 300 字}

可用分類（只能從這裡選）：
- 排版：版面、字體、網格
- …

規則：選 1 到 2 個最貼切的，最貼切的放前面；都不像就輸出「未分類」。
只輸出分類名稱，用逗號分隔，不要解釋、不要任何其他文字。
```

可選的後端：

| 來源 | 備註 |
|---|---|
| 能看圖的 LLM CLI（例如 Codex CLI、Claude Code） | 用你已經登入的帳號，不用另管金鑰。單張約 7–20 秒 |
| 你自己的 API 金鑰 | 金鑰放環境變數或系統鑰匙圈，不准寫進設定檔或 repo |
| 本機視覺模型 | 圖不離開本機；分得比較粗 |
| 不分類 | 用來源對照表的預設分類 |

## 瀑布流參數

| 參數 | 值 |
|---|---|
| 欄數 | `max(1, floor((寬 + GAP) / (MIN_COL + GAP)))` |
| 欄寬 | `(寬 − GAP × (欄數 − 1)) / 欄數` |
| 卡片圖高 | `欄寬 × h / w`；沒有 `w/h` 就先載圖量，4 秒量不到用 `1.25` |
| 擺放 | 依排序逐張放進目前最矮的那一欄 |
| 重算時機 | 面板寬度變（`ResizeObserver`）、保管庫檔案新增／刪除／改名、metadata 變動、切回這個分頁（debounce 300 ms） |

## 影片抽封面參數

| 參數 | 值 | 為什麼 |
|---|---|---|
| 取樣點 | 2、6、12、20、32、48、70、100、140 秒 | 不靠 ffprobe 讀片長；超過片長的點自然失敗、略過 |
| 每格 | `ffmpeg -y -ss <t> -i <src> -frames:v 1 -q:v 3 <tmp>`，timeout 25 秒 | |
| 挑哪格 | 檔案最大的那格（黑幀、全白壓得特別小）；有 Pillow 時可改用灰階標準差 | |
| 有效門檻 | > 2000 bytes | |
| 執行 | **非同步、九格並行**；沒選上的暫存檔刪掉 | 同步會凍住 Obsidian |
| 支援副檔名 | `mp4 mov m4v webm avi` | |
| 檔名 | `<影片檔名前 40 字>-frame.jpg` | 用影片名不用筆記名：新筆記剛建時還叫「未命名」 |
