# Windows 登录与安装排障

这份文档记录 Windows 上安装和登录 `boss-agent-cli` 时最常见的失败链路。它面向用户主动登录、只读辅助和本地凭据恢复场景；不要把 Cookie、CDP、patchright 或其他浏览器通道用于规避平台风控。

## 1. `uv` 无法识别

现象：

```text
uv: 无法将 "uv" 项识别为 cmdlet、函数、脚本文件或可运行程序的名称
```

处理：

```powershell
powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"
$env:Path = "$env:USERPROFILE\.local\bin;$env:Path"
uv --version
```

如果重新打开 PowerShell 后仍不可用，先临时补充 PATH：

```powershell
$env:Path = "$env:USERPROFILE\.local\bin;$env:Path"
```

## 2. `patchright` 无法识别

现象：

```text
patchright: 无法将 "patchright" 项识别为 cmdlet、函数、脚本文件或可运行程序的名称
```

这通常表示 `patchright` 没有作为全局命令暴露。使用 `uvx` 运行：

```powershell
uvx patchright install chromium
```

如果需要确认已安装的浏览器缓存：

```powershell
uvx --from patchright patchright install --list
```

看到类似下面的目录即表示 Chromium 缓存存在：

```text
C:\Users\<user>\AppData\Local\ms-playwright\chromium-1223
C:\Users\<user>\AppData\Local\ms-playwright\chromium_headless_shell-1223
```

## 3. Chromium 下载慢或失败

默认下载源是 Playwright 官方 CDN。国内网络可能出现 `ECONNRESET`、`ENOTFOUND cdn.playwright.dev` 或下载长时间不动。

如果之前设置过镜像源，先清掉：

```powershell
Remove-Item Env:PLAYWRIGHT_DOWNLOAD_HOST -ErrorAction SilentlyContinue
uvx patchright install chromium
```

如果使用代理，优先使用默认官方源。若必须尝试镜像，可以临时设置：

```powershell
$env:PLAYWRIGHT_DOWNLOAD_HOST="https://playwright.azureedge.net"
uvx patchright install chromium
```

或：

```powershell
$env:PLAYWRIGHT_DOWNLOAD_HOST="https://npmmirror.com/mirrors/playwright"
uvx patchright install chromium
```

注意：镜像可能缺少指定版本，出现 `NoSuchKey` 时不要反复重试同一个镜像，改回默认源或换网络。

## 4. `boss login` 扫码成功后仍失败

现象：

```text
检测到登录成功，正在提取凭证 ...
登录失败：Timeout 30000ms exceeded.
"load" event fired
```

这表示浏览器里已经完成扫码，但 CLI 在提取凭证或等待页面加载时超时。`login_timeout` 只控制扫码等待时间，不一定能解决这个 30 秒页面加载超时。

优先改用 CDP 登录。

## 5. Edge/Chrome CDP 登录

启动一个带远程调试端口的 Edge：

```powershell
& "${env:ProgramFiles(x86)}\Microsoft\Edge\Application\msedge.exe" --remote-debugging-port=9222 --remote-allow-origins=* --user-data-dir="$env:LOCALAPPDATA\boss-agent-edge-cdp"
```

如果 Edge 安装在另一路径，改用：

```powershell
& "$env:ProgramFiles\Microsoft\Edge\Application\msedge.exe" --remote-debugging-port=9222 --remote-allow-origins=* --user-data-dir="$env:LOCALAPPDATA\boss-agent-edge-cdp"
```

在打开的 Edge 窗口里登录 BOSS，保持窗口不关闭。然后打开新的 PowerShell：

```powershell
boss --cdp-url http://localhost:9222 login --cdp
boss --cdp-url http://localhost:9222 status
```

后续命令可继续带上同一个 CDP URL：

```powershell
boss --cdp-url http://localhost:9222 search "AI产品经理" --city 上海
```

## 6. `--cookie-source edge/chrome` 仍然打开 Chromium

现象：

```powershell
boss login --cookie-source edge
```

但仍然打开 patchright Chromium。

原因是 Cookie 提取失败后，CLI 会自动降级到 CDP、QR httpx、patchright 浏览器链路。常见原因：

- 浏览器 Cookie 数据库被锁定。
- Windows Cookie 解密需要管理员权限。
- Chrome/Edge Cookie 解密失败。
- 实际登录的是 patchright Chromium，而不是普通 Chrome/Edge。
- 当前浏览器 profile 没有 `zhipin.com` 的有效 `wt2` Cookie。

如果反复出现这种情况，不要继续重试 `--cookie-source`。改用 CDP 登录：

```powershell
boss --cdp-url http://localhost:9222 login --cdp
```

## 7. 验证顺序

推荐按以下顺序验证：

```powershell
uv --version
boss doctor
uvx --from patchright patchright install --list
boss --cdp-url http://localhost:9222 doctor
boss --cdp-url http://localhost:9222 login --cdp
boss status
```

`auth_session`、`auth_token_quality`、`cookie_extract` 在未登录前显示 `warn` 是正常的；安装阶段重点看 `python`、`patchright`、`patchright_chromium`、`network`。
