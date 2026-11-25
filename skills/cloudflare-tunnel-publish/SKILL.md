---
name: cloudflare-tunnel-publish
description: 用繁體中文、對話式地帶使用者用 Cloudflare Tunnel（cloudflared）把自己電腦上的網站、工具或本機服務發布成公開的 HTTPS 網址，例如 demo.example.com。會先掃描電腦上已有的 Tunnel、正在執行的服務與網路狀況並整理成報告，再詢問網域與子網域，接著完成安裝 cloudflared、登入授權、建立 Tunnel、設定 DNS、啟動與驗證，最後詢問要不要開機自動啟動；也涵蓋關閉、移除，以及 502、Error 1033 等排錯。網頁版 AI 不能操作使用者的電腦時，會一步步帶使用者自己執行；能執行本機指令的 AI 則可以代為操作。只要使用者提到 Cloudflare Tunnel、cloudflared、開通道、開 tunnel、內網穿透、把本機網站或工具打出去、對外發布、讓外面連到我的電腦、公開 localhost、用自有網域或子網域架站、開機自動開通道，即使沒有明說 Tunnel，也請使用這個 Skill。Also use for exposing a local website, tool, or localhost service to the internet on a custom domain through Cloudflare Tunnel.
---

# 用 Cloudflare Tunnel 發布本機網站或工具

帶使用者把自己電腦上的網站或工具發布成公開的 HTTPS 網址，不需要固定 IP，也不用在路由器上開 port。

網站仍然跑在使用者的電腦上，電腦關機、睡眠、斷線或程式被關掉時，網站就會連不上。這個做法適合測試、Demo 和短期展示。如果使用者要的是長期運作的正式網站，先說明這個限制，再讓對方決定是否繼續。

## 語言與說話方式

- 預設用繁體中文（台灣用語）回覆；使用者明確改用其他語言時才跟著切換。
- 用對話的方式進行，口語、白話，一次只推進一件事。
- 專有名詞第一次出現時用一句話解釋，例如「Tunnel（通道）：讓外面的人透過 Cloudflare 連到你電腦上的網站，不用開 port」。

## 先確認你能做到什麼

這決定整個流程怎麼進行，也是最容易出錯的地方。

- **不能在使用者的電腦上執行指令（大多數網頁版 AI）：** 用**引導模式**。一次只給一個步驟：說明這步在做什麼、給可以直接複製的指令、說明成功時會看到什麼，再請使用者貼回結果。確認結果後才進下一步，因為前面的輸出（例如 Tunnel ID）後面會用到。
- **能在使用者的電腦上執行指令（例如 Claude Code）：** 用**代操作模式**。自己執行並檢查結果，但安裝軟體、修改系統設定（例如第 12 步的自動啟動）前要先徵得同意，第 6 步的登入授權也要由使用者本人完成。

網頁版 AI 的程式碼沙盒不是使用者的電腦。在沙盒裡安裝 cloudflared 或啟動網站，使用者的網站並不會上線，所以不要在那裡跑這些指令。只有看過實際輸出的事才算確認，不要把還沒發生的步驟說成已完成。

## 流程總覽

1. 掃描電腦現況，整理成報告
2. 一次問完需要的資訊：要發布什麼、網域、子網域
3. 檢查網域
4. 準備要發布的內容
5. 安裝 cloudflared
6. 登入 Cloudflare 並授權
7. 建立 Tunnel
8. 把網址指向 Tunnel
9. 寫設定檔
10. 啟動 Tunnel
11. 驗證
12. 詢問是否自動啟動

掃描顯示已經安裝、登入過，或已有使用者自己的 Tunnel 時，對應的步驟可以跳過，或改走「加到既有的 Tunnel」。

過程中會用到兩個 PowerShell 視窗：**視窗 A** 跑網站或工具，**視窗 B** 跑 Tunnel。

## 代號

給使用者指令前，先把代號換成實際的值，讓對方可以直接複製。`example.com` 及其子網域是保留的示意網址，不是預設發布網域；未取得使用者的實際網域前，不要產生或執行發布指令。

| 代號 | 意思 | 範例 |
| --- | --- | --- |
| `<DOMAIN>` | 主網域 | `example.com` |
| `<SUBDOMAIN>` | 子網域 | `demo` |
| `<HOSTNAME>` | 完整網址，等於 `<SUBDOMAIN>.<DOMAIN>` | `demo.example.com` |
| `<TUNNEL_NAME>` | Tunnel 名稱 | `site-demo` |
| `<PROJECT_DIR>` | 放 Tunnel 設定檔（和範例網站）的資料夾 | `C:\projects\my-site` |
| `<PORT>` | 本機網站或工具的 port | `8080` |
| `<ORIGIN>` | Tunnel 要轉送到的本機位址 | `http://127.0.0.1:8080` |
| `<APP_DIR>` | 啟動網站或工具時所在的資料夾 | `C:\projects\my-site` |
| `<START_COMMAND>` | 啟動網站或工具的指令 | `python -m http.server 8080 --bind 127.0.0.1 --directory site` |
| `<TUNNEL_ID>` | 第 7 步才會產生 | `xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx` |

## 1. 掃描電腦現況（一定先做）

開始任何設定前，先了解這台電腦的狀況，避免重複建立 Tunnel、搶到正在使用的 port，或在擋住 Cloudflare 的網路上白忙一場。即使使用者第一句話就給了網域和子網域，也先掃描，再一起確認。

下面這段只會讀取資料，不會修改任何設定，也不會輸出密碼或金鑰，大約需要半分鐘。引導模式請使用者整段複製到 PowerShell 執行，再把結果全部貼回來；代操作模式自己執行。使用者用 macOS 或 Linux 時，改用文末「其他作業系統」的版本。

```powershell
"== 1. cloudflared =="
$cf = Get-Command cloudflared -ErrorAction SilentlyContinue
if ($cf) { $cf.Source; cloudflared --version } else { "not installed" }

"== 2. Local Cloudflare files =="
$cfDir = "$env:USERPROFILE\.cloudflared"
if (Test-Path $cfDir) { Get-ChildItem $cfDir -Force | Select-Object Name, LastWriteTime | Format-Table -AutoSize } else { "no .cloudflared folder" }

"== 3. Tunnels in the account =="
if ($cf -and (Test-Path "$cfDir\cert.pem")) { cloudflared tunnel list } else { "not logged in, skipped" }

"== 4. Running cloudflared =="
$procs = Get-CimInstance Win32_Process -Filter "Name='cloudflared.exe'"
if ($procs) { $procs | ForEach-Object { "PID $($_.ProcessId): $($_.CommandLine -replace 'eyJ[\w\-.=]+', 'eyJ***')" } } else { "none" }

"== 5. Auto-start =="
$svc = Get-Service cloudflared -ErrorAction SilentlyContinue
if ($svc) { "service: $($svc.Status), $($svc.StartType)"; (Get-ItemProperty HKLM:\SYSTEM\CurrentControlSet\Services\Cloudflared).ImagePath -replace 'eyJ[\w\-.=]+', 'eyJ***' } else { "no cloudflared service" }
Get-ChildItem ([Environment]::GetFolderPath('Startup')) -Filter '*cloudflare*' -ErrorAction SilentlyContinue | Select-Object -ExpandProperty Name
Get-ScheduledTask -ErrorAction SilentlyContinue | Where-Object TaskName -like '*cloudflare*' | Select-Object TaskName, State

"== 6. Listening ports =="
$skip = 'System', 'Idle', 'svchost', 'lsass', 'wininit', 'services', 'spoolsv'
Get-NetTCPConnection -State Listen -ErrorAction SilentlyContinue |
  ForEach-Object { [pscustomobject]@{ Port = $_.LocalPort; Address = $_.LocalAddress; Program = (Get-Process -Id $_.OwningProcess -ErrorAction SilentlyContinue).ProcessName } } |
  Where-Object { $_.Program -notin $skip } | Sort-Object Port, Address -Unique | Format-Table -AutoSize

"== 7. Network =="
foreach ($t in @(@('region1.v2.argotunnel.com', 7844), @('api.cloudflare.com', 443))) {
  $ok = Test-NetConnection $t[0] -Port $t[1] -InformationLevel Quiet -WarningAction SilentlyContinue
  "$($t[0]):$($t[1]) = $(if ($ok) { 'OK' } else { 'BLOCKED' })"
}
```

### 判讀並整理成報告

把結果整理成一段簡短的「電腦現況」給使用者看，只列結論，不要把原始輸出整段貼回去。例如：

> **你的電腦現況**
> - cloudflared：已安裝
> - Cloudflare 登入：已登入過
> - 已有的 Tunnel：`site-blog`（這台電腦上正在執行）
> - 自動啟動：沒有設定
> - 可能要發布的工具：`3000` port（node，只接受本機連線）
> - 網路：可以連到 Cloudflare Tunnel 的伺服器 ✅

判讀重點：

- **1、2：** 已安裝就跳過第 5 步。有 `cert.pem` 代表這台電腦登入過，可以跳過第 6 步；但它只對當初授權的那個網域有效，第 8 步要特別核對 CNAME。出現 `Your version ... is outdated` 的警告可以先忽略，之後再用 `winget upgrade --id Cloudflare.cloudflared` 更新。
- **3、4：** `tunnel list` 列的是整個帳號的 Tunnel，不只這台電腦，`CONNECTIONS` 有值只代表它在某台電腦上執行中。清單可能包含同帳號其他人的 Tunnel 名稱，使用者不想貼出來時，請對方只告訴你有沒有 Tunnel、有沒有和預定名稱重複即可。有 Tunnel 時，先問使用者那是不是自己建立的、還在不在用。是自己在這台電腦建立、而且正在用的，就問要「把新網址加到既有的 Tunnel」還是另建一個。別人的或不確定的，一律不要動，另建新的，名稱也不要和清單上的重複。
- **4、5 出現 `eyJ***`：** 那是被遮蔽的 Tunnel token，代表有用 token 執行的 Tunnel，通常是在 Cloudflare 後台建立的。
- **5：** 已經有 cloudflared 服務時，第 12 步不能再用「開機就啟動」，因為一台 Windows 只能有一個 cloudflared 服務。
- **6：** 找出可能是使用者工具的程式，例如 `node`、`python`、`java`、`dotnet`、`php`、`ruby`、`deno`、`bun`、`com.docker.backend`；瀏覽器、通訊軟體、遊戲平台這類通常不是。再看 `Address`：
  - `127.0.0.1` 或 `::1`：只接受本機連線，最安全。只有 `::1` 時，第 9 步的 `<ORIGIN>` 要用 `http://[::1]:<PORT>`。
  - `0.0.0.0` 或 `::`：同網路的其他電腦也能直接連到。提醒使用者，如果工具可以設定，改成只接受 `127.0.0.1`。
  - 範例網站預設的 `8080` 已被占用時，換一個不在清單上的 port。
- **7：** `region1.v2.argotunnel.com:7844` 是 `BLOCKED`，代表這個網路擋住了 Tunnel 需要的連線，公司或學校網路常見，之後很可能失敗。先說明原因，建議換網路（例如手機熱點）或請網管開放 TCP 和 UDP 的 7844 port，再讓使用者決定要不要繼續。`api.cloudflare.com:443` 也是 `BLOCKED` 時，通常是整台電腦沒有網路。

## 2. 一次問完需要的資訊

根據掃描結果，把問題一次問完，不要拆成好幾輪；對方已經提供的資訊就不用再問。電腦要保持開機、不能睡眠的限制，也放在同一則訊息裡說明。

1. **要發布什麼？** 掃描找到可能的工具時直接確認，例如「是 3000 port 的 node 程式嗎？」。找不到時請對方選：
   - 已經在跑的工具或本機服務：在哪個 port、平常在哪個資料夾用什麼指令啟動（第 12 步設定自動啟動會用到）。
   - 現成的靜態網頁檔案，只有 HTML、CSS、JS 和圖片等。
   - 先用範例頁面測試。
2. **網域和子網域是什麼？** 主網域必須已經託管在 Cloudflare，而且使用者的帳號有管理權限。例如網域 `example.com`、子網域 `demo`，合起來就是 `demo.example.com`（僅為示意，不可直接使用）。對方直接給完整網址也可以，自己拆成網域和子網域。

其餘的先用預設值，並告訴使用者可以更改：專案資料夾 `C:\projects\my-site`、Tunnel 名稱 `site-<SUBDOMAIN>`、範例網站的 port `8080`。

選網址時要注意：

- 先確認欲使用的子網域是否已有 DNS 紀錄或服務；若已被使用，請選擇其他子網域，或向網域管理者確認，不要覆蓋現有設定。
- 只用一層子網域。`demo.example.com` 可以；`a.b.example.com` 不在 Cloudflare 免費憑證的範圍內，HTTPS 會出錯。

## 3. 檢查網域

```powershell
Resolve-DnsName <DOMAIN> -Type NS
Resolve-DnsName <HOSTNAME>
```

- 第一行的結果要以 `ns.cloudflare.com` 結尾。不是的話，代表網域不在 Cloudflare，這個流程走不通，請對方找網域管理者。
- 第二行**應該報錯**，顯示 DNS 名稱不存在，代表這個網址還沒被使用。先告訴使用者這個紅字是好消息。如果查得到 IP，代表網址已被使用，請換一個，或確認那是對方自己的。

要發布範例頁面或靜態網頁時，再確認有沒有 Python：

```powershell
python --version
```

沒有 Python 時，請使用者到 python.org 下載安裝，安裝時勾選 **Add python.exe to PATH**，裝完後重開 PowerShell。

## 4. 準備要發布的內容（視窗 A）

先建立放設定檔的資料夾：

```powershell
New-Item -ItemType Directory -Force <PROJECT_DIR>\tunnel
cd <PROJECT_DIR>
```

設定檔放在 `tunnel`，不會被當成網站內容公開。

### A. 發布已經在跑的工具

- 請使用者在瀏覽器打開 `http://localhost:<PORT>`，確認工具正常。
- `<ORIGIN>` 用 `http://127.0.0.1:<PORT>`；掃描顯示這個 port 只有 `::1` 時，改用 `http://[::1]:<PORT>`。
- 記下 `<APP_DIR>` 和 `<START_COMMAND>`，第 12 步會用到。
- 提醒使用者，這個工具的所有頁面都會公開，請確認它適合對外，例如有沒有登入保護、有沒有管理頁面。
- 開發用的伺服器（例如 Vite）可能會擋掉不認得的網址，處理方式見「排錯」。

工具維持原本的執行方式即可，不需要重新啟動。

### B. 發布靜態網頁或範例頁面

```powershell
New-Item -ItemType Directory -Force site
```

`site` 是唯一會公開的資料夾，和 `tunnel` 分開，設定檔才不會被一起發布。

**沒有現成網站時**，建立範例頁面：

```powershell
$html = @'
<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>Demo site</title>
</head>
<body>
  <h1>Hello, world!</h1>
  <p>Published with Cloudflare Tunnel.</p>
</body>
</html>
'@
[IO.File]::WriteAllText("$PWD\site\index.html", $html)
```

**有現成網站時**，先確認它是不是純靜態網頁。如果它需要後端程式或資料庫才能運作，複製檔案沒有用，請先在本機把它跑起來，改走上面的「A. 發布已經在跑的工具」。

純靜態網頁就只把要公開的檔案複製到 `site`，不要直接發布整個專案資料夾。複製後列出全部內容，請使用者逐一確認：

```powershell
Get-ChildItem site -Recurse -Force | Select-Object -ExpandProperty FullName
```

這裡的每個檔案發布後都能被任何人下載，所以要特別留意 `.env`、金鑰、資料庫和未公開的資料。發現可疑的檔案就先指出來，建議移出 `site`，確認後才繼續。

啟動網站：

```powershell
python -m http.server <PORT> --bind 127.0.0.1 --directory site
```

- `--bind 127.0.0.1`：只接受本機連線，同網路的其他電腦不能繞過 Tunnel 直接連。
- `--directory site`：只開放 `site` 資料夾。

這時 `<ORIGIN>` 是 `http://127.0.0.1:<PORT>`，`<APP_DIR>` 是 `<PROJECT_DIR>`，`<START_COMMAND>` 就是上面這行指令。

請使用者打開 `http://localhost:<PORT>` 確認能看到網頁。視窗 A 會停在執行中，這是正常的，**請保持開啟**，後面的步驟改用視窗 B。

## 5. 安裝 cloudflared（視窗 B）

從這一步開始改用另一個 PowerShell 視窗（視窗 B），先移到專案資料夾：

```powershell
cd <PROJECT_DIR>
```

掃描顯示已安裝就跳到第 6 步。否則先確認有沒有 winget：

```powershell
winget --version
```

有版本號就安裝：

```powershell
winget install --id Cloudflare.cloudflared --exact --source winget
```

沒有 winget 時，請使用者到 [cloudflared GitHub](https://github.com/cloudflare/cloudflared/releases) 下載 Windows 安裝檔（`cloudflared-windows-amd64.msi`）並安裝。

裝完後**關掉視窗 B，重開一個**，新視窗才找得到這個指令。然後：

```powershell
cd <PROJECT_DIR>
cloudflared --version
```

## 6. 登入 Cloudflare 並授權（使用者本人操作）

掃描時已經有 `cert.pem` 就跳到第 7 步。否則執行：

```powershell
cloudflared tunnel login
```

瀏覽器會打開 Cloudflare 頁面。請使用者登入、點選 `<DOMAIN>`，再按 **Authorize**。這是授權這台電腦管理網域的動作，所以要由本人完成，也不要向對方索取 Cloudflare 密碼。

指令結束後確認：

```powershell
Test-Path "$env:USERPROFILE\.cloudflared\cert.pem"
```

顯示 `True` 就成功了。

## 7. 建立 Tunnel

```powershell
cloudflared tunnel create <TUNNEL_NAME>
```

成功時會顯示 `Created tunnel <TUNNEL_NAME> with id <TUNNEL_ID>`。請使用者貼回**這一行**就好，記下 ID。

如果名稱已存在，代表掃描時的清單裡就有它。確認那是對方自己在這台電腦建立的，才沿用；否則換一個名稱。

## 8. 把網址指向 Tunnel

```powershell
cloudflared tunnel route dns <TUNNEL_NAME> <HOSTNAME>
```

成功時會顯示 `Added CNAME <HOSTNAME> which will route to this tunnel`。**請逐字核對 CNAME 後面的網址。** 如果後面多接了別的網域，代表登入時授權的是別的網域，處理方式見「排錯」。

## 9. 寫設定檔

```powershell
$yaml = @"
tunnel: <TUNNEL_ID>
credentials-file: $env:USERPROFILE\.cloudflared\<TUNNEL_ID>.json

ingress:
  - hostname: <HOSTNAME>
    service: <ORIGIN>
  - service: http_status:404
"@
[IO.File]::WriteAllText("$PWD\tunnel\config.yml", $yaml)
cloudflared tunnel --config tunnel\config.yml ingress validate
```

最後要顯示 `OK`。結尾的 `"@` 必須頂格，不能有空白，否則 PowerShell 會報錯。最後一條 `http_status:404` 規則是必要的，用來處理其他沒有列出的網址。

## 10. 啟動 Tunnel

```powershell
cloudflared tunnel --config tunnel\config.yml run <TUNNEL_NAME>
```

看到 `Registered tunnel connection` 就是連上了。視窗 B 也要**保持開啟**。

## 11. 驗證

請使用者另外開一個 PowerShell 視窗：

```powershell
curl.exe -I https://<HOSTNAME>
curl.exe -s -o NUL -w "%{http_code}" https://<HOSTNAME>/tunnel/config.yml
```

- 第一行要看到 `200` 和 `Server: cloudflare`。新建立的網址如果還打不開，等幾分鐘再試。
- 第二行要看到 `404`，代表設定檔沒有被公開。發布的是自己的工具時跳過這一行，改請對方確認沒有不該公開的頁面。

最後請使用者用瀏覽器打開網址，再用手機關掉 Wi-Fi 開一次，確認其他網路也連得到。

## 12. 詢問是否自動啟動

驗證都通過後，主動問使用者：「要不要讓電腦開機後，自動把網站和通道打開？」並說明三種選擇：

| 選擇 | 什麼時候啟動 | 需要系統管理員 | 適合 |
| --- | --- | --- | --- |
| **登入後自動啟動**（建議） | 開機並登入 Windows 後 | 不用 | 大多數情況 |
| **開機就啟動**（進階） | 開機後，不用登入 | 要 | 工具本身已經是 Windows 服務，而且掃描時沒有 cloudflared 服務 |
| **不用** | 自己手動開 | 不用 | 偶爾才展示 |

不管選哪一種，網站或工具本身也要能自動啟動，否則通道開了，訪客只會看到 502。另外提醒：電腦睡眠時網站也會斷線，需要長時間在線的話，請使用者到 Windows 設定的「電源」頁面，把接上電源時的睡眠時間改成「永不」。

代操作模式下，這會修改系統設定，使用者明確同意後才做，做完要告訴對方怎麼取消。

使用者選了「登入後自動啟動」或「開機就啟動」後，讀取 `references/autostart.md`，照裡面對應的方式操作；裡面也有測試方法和取消方式。

## 加到既有的 Tunnel

掃描發現使用者自己在這台電腦建立、而且正在使用的 Tunnel 時，可以把新網址加進去，不用再開一個。原本的網址在重新啟動時會短暫中斷，先告訴使用者。

**用設定檔執行的 Tunnel**（掃描 4 或 5 的指令裡有 `--config`，或設定檔在 `%USERPROFILE%\.cloudflared\config.yml`）：

1. 請使用者貼上那個 `config.yml` 的內容。它只有 Tunnel ID 和檔案路徑，沒有密碼，可以貼。
2. 在 `ingress` 裡、最後一條 `http_status:404` 的**上面**加入新規則，給對方完整的新內容覆蓋原檔：

   ```yaml
     - hostname: <HOSTNAME>
       service: <ORIGIN>
   ```

3. 執行 `cloudflared tunnel route dns <既有的 Tunnel 名稱> <HOSTNAME>`，核對方式同第 8 步。
4. 執行 `cloudflared tunnel --config <設定檔路徑> ingress validate`，要顯示 `OK`。
5. 重新啟動那個 Tunnel：在它的視窗按 `Ctrl + C` 再執行一次；如果它是服務，用系統管理員 PowerShell 執行 `Restart-Service cloudflared`。
6. 做第 11 步驗證新網址，也確認原本的網址還正常。

**用 token 執行的 Tunnel**（掃描看到 `eyJ***`）：它的網址是在 Cloudflare 後台設定的。請使用者到後台的 Tunnel 頁面選這個 Tunnel，新增一個公開網址（Public Hostname，新版後台可能叫 Published application routes）：子網域填 `<SUBDOMAIN>`、網域選 `<DOMAIN>`、服務類型選 HTTP、網址填 `<ORIGIN>` 去掉 `http://` 的部分。

既有的 Tunnel 已經會自動啟動時，第 12 步只需要處理網站或工具本身。

## 完成時要告訴使用者

- 公開網址是什麼。
- 哪些檢查確實通過，哪些沒有確認。
- 有沒有設定自動啟動，用的是哪一種。
- 網站要在線上需要什麼：電腦開機、不睡眠；手動啟動時，視窗 A 和視窗 B 都要開著。
- 怎麼關閉、下次怎麼啟動、怎麼取消自動啟動（見下一節和 `references/autostart.md`）。
- 發布靜態網頁時，要修改網頁就直接編輯 `site` 裡的檔案，存檔後重新整理即可。
- 不再使用時，請回來做「徹底移除」。

## 關閉與重新啟動

**關閉：**

- 手動啟動的：在視窗 A 和視窗 B 分別按 `Ctrl + C`。
- 登入後自動啟動的：關掉工作列上縮小的 PowerShell 視窗。
- 服務：用系統管理員 PowerShell 執行 `Stop-Service cloudflared`。

這只會停止程式，Cloudflare 上的 Tunnel 和 DNS 紀錄都還在，訪客會看到錯誤頁面。

**重新啟動：** 第 3 到第 9 步不用重做。設定了登入後自動啟動的，登出再登入，或執行一次啟動捷徑即可。手動啟動時開兩個視窗，各執行一段：

```powershell
cd <APP_DIR>
<START_COMMAND>
```

```powershell
cd <PROJECT_DIR>
cloudflared tunnel --config tunnel\config.yml run <TUNNEL_NAME>
```

## 徹底移除

只在使用者要求時進行。先列出要刪除的 Tunnel、網址和自動啟動設定，請對方確認都是自己建立的，因為刪除後無法復原。

收到確認後才給下面的指令（代操作模式則是到這時才執行）：

1. 有設定自動啟動的，先依 `references/autostart.md` 的「取消」移除。
2. 關閉網站和 Tunnel，然後：

   ```powershell
   cloudflared tunnel cleanup <TUNNEL_NAME>
   cloudflared tunnel delete <TUNNEL_NAME>
   ```

3. 刪除 Tunnel **不會**刪除 DNS 紀錄。請使用者到 [Cloudflare 後台](https://dash.cloudflare.com)，選擇網域，進入 **DNS → Records**，刪除對應的 CNAME。網域可能有其他人在用，刪除前請看清楚名稱。
4. 刪除 `<PROJECT_DIR>` 裡的 `tunnel` 資料夾和 `start-tunnel.ps1`。`site` 是使用者的網頁檔案，問過再刪。
5. 如果這台電腦以後都不再使用 Tunnel，再刪除 `%USERPROFILE%\.cloudflared\` 裡的 `cert.pem` 和對應的 `.json` 檔。

## 排錯

| 狀況 | 原因 | 處理方式 |
| --- | --- | --- |
| 找不到 `cloudflared` 指令 | 視窗是在安裝前開的 | 重開 PowerShell；VS Code 內的終端機要重開 VS Code。也可以用完整路徑：`& "C:\Program Files (x86)\cloudflared\cloudflared.exe"` |
| 出現找不到 `cert.pem` 的錯誤 | 這台電腦還沒登入 | 做第 6 步 |
| 掃描時 7844 是 `BLOCKED`，或 Tunnel 一直連不上 | 網路擋住了 Tunnel 需要的連線 | 換網路（例如手機熱點），或請網管開放 TCP 和 UDP 7844 到 `region1.v2.argotunnel.com`、`region2.v2.argotunnel.com` |
| 授權頁面找不到網域 | 帳號沒有權限，或登入了別的帳號 | 確認登入的帳號；向網域管理者申請權限 |
| `tunnel login` 說憑證已存在 | 這台電腦已經登入過 | 網域相同就不用重新登入，直接做第 7 步 |
| CNAME 後面多接了別的網域 | 登入時授權的是別的網域 | 到後台刪除建錯的紀錄，把 `cert.pem` 移到別處，重做第 6 步和第 8 步 |
| `route dns` 說紀錄已存在 | 這個網址已被使用 | 改用別的網址，第 9 步的 `hostname` 也要跟著改。如果紀錄是對方自己之前建立的，請對方到後台確認後刪除，再重做第 8 步。`--overwrite-dns` 會覆蓋現有紀錄，可能讓別人的網站中斷，不要主動建議 |
| 網頁顯示 502 | Tunnel 正常，但本機網站或工具沒有啟動；Tunnel 被強制結束後的一段時間內也可能出現 | 檢查視窗 A 是否在執行、port 是否和 `config.yml` 一致 |
| 本機 `localhost` 打得開，公開網址卻是 502 | 工具只接受 IPv6 的本機連線（`::1`） | 把 `config.yml` 的 `service` 改成 `http://[::1]:<PORT>`，重新啟動 Tunnel |
| 顯示 `Blocked request. This host is not allowed` | 開發用伺服器（例如 Vite）擋掉不認得的網址 | 在工具的設定把 `<HOSTNAME>` 加進允許清單，例如 Vite 的 `server.allowedHosts`，再重新啟動工具 |
| 網頁顯示 Error 1033（HTTP 530） | Tunnel 沒有在執行 | 檢查視窗 B 或服務是否在執行；用 `cloudflared tunnel info <TUNNEL_NAME>` 查看連線 |
| 啟動網站時說 port 被占用 | 其他程式正在使用這個 port | 換一個 port，啟動指令和 `config.yml` 都要改 |
| 登入後沒有自動啟動 | 捷徑不見、腳本出錯，或網路太晚就緒 | 確認「啟動」資料夾裡有捷徑；手動執行 `powershell -ExecutionPolicy Bypass -File <PROJECT_DIR>\start-tunnel.ps1`，看錯誤訊息 |
| 服務無法啟動或一直停止 | 設定檔路徑或 ImagePath 有誤 | 查看 `<PROJECT_DIR>\tunnel\cloudflared.log`；確認 ImagePath 沒有多餘的空白或字元 |
| 改了內容卻沒有更新 | Cloudflare 或瀏覽器快取了舊檔 | 按 `Ctrl + F5`；仍未更新就等一下，或到後台清除快取 |
| HTTPS 憑證錯誤 | 用了兩層以上的子網域 | 改用一層，例如 `demo.example.com` |
| `http://` 不會自動轉 `https://` | Cloudflare 預設沒有強制轉址 | 請網域管理者開啟 **SSL/TLS → Edge Certificates → Always Use HTTPS**。這會影響整個網域 |

遇到表中沒有的錯誤時，請使用者貼上完整訊息，根據實際內容判斷，不要憑印象猜測。

## 安全界線

這些界線的原因都一樣：Tunnel 會把東西公開到整個網際網路，而憑證等於這個網域或 Tunnel 的管理權限。

- **不要索取憑證、金鑰或密碼。** 流程不需要 Cloudflare 密碼，也不需要 `cert.pem`、`<TUNNEL_ID>.json` 或 Tunnel token 的內容。如果使用者貼了這些內容，立刻提醒那是機密，不要複述，再依外洩的東西處理：
  - `<TUNNEL_ID>.json` 外洩：請對方移除這個 Tunnel 並建立新的，到後台刪除舊的 CNAME 後重做第 7 到第 9 步。
  - Tunnel token（`eyJ` 開頭的長字串）外洩：請對方到 Cloudflare 後台重新產生這個 Tunnel 的 token。
  - `cert.pem` 外洩：請對方立刻通知網域管理者。
- **掃描只讀需要的資訊。** 只看程式名稱、port 和檔名，不要讀取憑證或其他設定檔的內容；唯一的例外是「加到既有的 Tunnel」時的 `config.yml`，它沒有密碼。掃描結果只拿來判斷流程，不要做其他用途。
- **不要讓敏感資料上線。** 未公開的資料、個資、密碼和 API 金鑰都不該放進 `site`，也不該出現在要發布的工具裡。內容如果只想給特定的人看，不要用這個流程直接公開，改建議對方研究 Cloudflare Access。
- **只開放需要公開的內容。** 不要把 `--directory` 指到專案根目錄或使用者資料夾，也要保留 `--bind 127.0.0.1`。如果子資料夾沒有 `index.html`，訪客會看到檔案清單。
- **不要動到別人的東西。** 不要主動覆蓋 DNS 紀錄，也不要修改或刪除不是使用者自己建立的 Tunnel、服務或紀錄。
- **自動啟動要經過同意。** 這會讓電腦每次開機或登入都自動對外公開，設定前先確認，設定後說明怎麼取消。
- **遵守網路規定。** Tunnel 可以從防火牆內部對外發布服務。在公司、學校或實驗室的電腦或網路上使用前，請對方確認符合網管規定。
- **說明這不是正式環境的做法。** Python 內建的伺服器只適合測試和簡單的靜態網頁，Python 官方也不建議用在正式環境。

## 其他作業系統

這個流程只在 Windows 11 上實測過。遇到 macOS 或 Linux 時，先告訴使用者這點，再依下面調整：

- 第 1 步的掃描改用：

  ```bash
  command -v cloudflared && cloudflared --version
  ls -la ~/.cloudflared 2>/dev/null
  [ -f ~/.cloudflared/cert.pem ] && cloudflared tunnel list
  ps -axo pid,command | grep '[c]loudflared' | sed -E 's/eyJ[A-Za-z0-9_.=-]+/eyJ***/g'
  lsof -nP -iTCP -sTCP:LISTEN
  nc -vz -w 5 region1.v2.argotunnel.com 7844
  ```

- 用系統的方式安裝 cloudflared，例如 macOS 用 `brew install cloudflared`；其他系統請看 [cloudflared GitHub](https://github.com/cloudflare/cloudflared) 或 Cloudflare 官方文件。
- 通常要用 `python3`，不是 `python`。
- 憑證放在 `~/.cloudflared/`，`config.yml` 的 `credentials-file` 也要改成這個路徑。
- 用 `dig NS <DOMAIN>` 取代 `Resolve-DnsName`，用一般的 shell 語法建立檔案。
- 自動啟動可以用 `cloudflared service install`（Linux 要加 `sudo`），但設定檔的位置和 Windows 不同，請依 Cloudflare 官方文件操作，並同樣確認網站或工具本身也會自動啟動。
