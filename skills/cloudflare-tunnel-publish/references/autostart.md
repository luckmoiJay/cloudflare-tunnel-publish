# 自動啟動的詳細做法

主流程第 12 步問過使用者、取得同意後，依對方的選擇照下面做。代號的意思和主流程相同，給指令前一樣要先換成實際的值。

## 方式一：登入後自動啟動（建議）

做法是寫一個啟動腳本，再把它的捷徑放進 Windows 的「啟動」資料夾。

**1. 建立啟動腳本**（視窗 B，位置在 `<PROJECT_DIR>`）：

```powershell
$ps = @'
Set-Location '<PROJECT_DIR>'
# 1) Start the website or tool. Delete this line if it already starts by itself.
Start-Process powershell -ArgumentList '-NoExit', '-Command', '<START_COMMAND>' -WorkingDirectory '<APP_DIR>' -WindowStyle Minimized
# 2) Wait until the network is ready (up to 2 minutes)
for ($i = 0; $i -lt 24; $i++) { if (Resolve-DnsName region1.v2.argotunnel.com -ErrorAction SilentlyContinue) { break }; Start-Sleep 5 }
# 3) Start the tunnel
cloudflared tunnel --config tunnel\config.yml run <TUNNEL_NAME>
'@
[IO.File]::WriteAllText("$PWD\start-tunnel.ps1", $ps, [Text.UTF8Encoding]::new($true))
```

- `<START_COMMAND>` 裡如果有單引號 `'`，要寫成兩個 `''`。
- 結尾的 `'@` 必須頂格。
- 工具本身已經會自動啟動（例如設定了自動啟動的 Docker 容器）時，刪掉第一個 `Start-Process` 那一行。

**2. 把捷徑放進「啟動」資料夾：**

```powershell
$startup = [Environment]::GetFolderPath('Startup')
$lnk = (New-Object -ComObject WScript.Shell).CreateShortcut("$startup\Cloudflare Tunnel - <TUNNEL_NAME>.lnk")
$lnk.TargetPath = "$env:WINDIR\System32\WindowsPowerShell\v1.0\powershell.exe"
$lnk.Arguments = '-NoProfile -ExecutionPolicy Bypass -File "<PROJECT_DIR>\start-tunnel.ps1"'
$lnk.WorkingDirectory = '<PROJECT_DIR>'
$lnk.WindowStyle = 7
$lnk.Save()
```

- `-ExecutionPolicy Bypass` 只讓這個腳本略過 Windows 的腳本執行限制，不會改變整台電腦的設定。
- `WindowStyle = 7` 讓視窗縮小在工作列上，不會擋畫面。

**3. 測試：** 先在視窗 A 和視窗 B 按 `Ctrl + C`，停止原本的網站和 Tunnel。發布自己的工具時，如果腳本會啟動它，也要先把原本開著的工具關掉，否則 port 會衝突。然後執行：

```powershell
Start-Process "$([Environment]::GetFolderPath('Startup'))\Cloudflare Tunnel - <TUNNEL_NAME>.lnk"
```

等半分鐘，重做第 11 步的 `curl.exe -I`，看到 `200` 就成功了。工作列上會多出縮小的 PowerShell 視窗，這是正常的；關掉它們，網站就會停止。最後建議使用者重新開機一次，登入後確認網址打得開。

**取消：**

```powershell
Remove-Item "$([Environment]::GetFolderPath('Startup'))\Cloudflare Tunnel - <TUNNEL_NAME>.lnk"
```

## 方式二：開機就啟動（Windows 服務，進階）

只有同時符合這三個條件才建議：掃描時沒有 cloudflared 服務、工具本身在沒登入時也會啟動（例如它已經是 Windows 服務）、使用者有系統管理員權限。Python 範例網站不符合，請改用方式一。

服務以系統帳號執行，讀不到使用者資料夾裡的檔案，所以依 Cloudflare 官方文件的做法，把 Tunnel 的憑證複製到系統帳號的資料夾，再讓服務指定設定檔。只複製 `<TUNNEL_ID>.json` 就夠了，它只能執行這一個 Tunnel；`cert.pem` 有整個網域的管理權限，執行 Tunnel 用不到，所以不要複製。

先在視窗 B 按 `Ctrl + C` 停止 Tunnel，再請使用者在開始功能表的 PowerShell 上按右鍵，選**以系統管理員身分執行**，然後：

```powershell
cd <PROJECT_DIR>
$src = "$env:USERPROFILE\.cloudflared"
$dst = "$env:WINDIR\System32\config\systemprofile\.cloudflared"
New-Item -ItemType Directory -Force $dst
Copy-Item "$src\<TUNNEL_ID>.json" $dst
$yaml = @"
tunnel: <TUNNEL_ID>
credentials-file: $dst\<TUNNEL_ID>.json
logfile: <PROJECT_DIR>\tunnel\cloudflared.log

ingress:
  - hostname: <HOSTNAME>
    service: <ORIGIN>
  - service: http_status:404
"@
[IO.File]::WriteAllText("$dst\config.yml", $yaml)
cloudflared tunnel --config "$dst\config.yml" ingress validate
```

如果系統管理員視窗是用另一個帳號開的，`$env:USERPROFILE` 會指到那個帳號，要把 `$src` 改成原本使用者的 `.cloudflared` 資料夾。顯示 `OK` 後安裝服務：

```powershell
cloudflared service install
$exe = (Get-Command cloudflared).Source
Set-ItemProperty HKLM:\SYSTEM\CurrentControlSet\Services\Cloudflared -Name ImagePath -Value "`"$exe`" --config `"$dst\config.yml`" tunnel run"
Restart-Service cloudflared
Get-Service cloudflared | Select-Object Status, StartType
```

要看到 `Running` 和 `Automatic`。重做第 11 步驗證；有問題時查看 `<PROJECT_DIR>\tunnel\cloudflared.log`。之後 Tunnel 由服務負責，不需要再開視窗 B。

**取消**（系統管理員 PowerShell）：

```powershell
cloudflared service uninstall
Remove-Item "$env:WINDIR\System32\config\systemprofile\.cloudflared\<TUNNEL_ID>.json", "$env:WINDIR\System32\config\systemprofile\.cloudflared\config.yml"
```

`service uninstall` 失敗時，改用 `Stop-Service cloudflared` 再 `sc.exe delete cloudflared`。
