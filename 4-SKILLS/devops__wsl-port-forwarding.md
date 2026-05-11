---
name: wsl-port-forwarding
description: 为 WSL Web 服务配置 Windows 端口转发 + 防火墙规则 + 开机自启持久化，使局域网其他设备可通过 Windows 宿主的 LAN IP 访问。
---

# WSL 端口转发配置（完整版）

在 WSL 中新部署 Web 服务后，使其可从局域网永久访问 `http://<Windows-LAN-IP>:<PORT>/`。

## 方案选择：Mirrored 模式 vs PortProxy

**推荐方案：Mirrored 网络模式（免配置，一劳永逸）**

在 Windows 11 22H2+ 中，WSL 支持 `networkingMode=mirrored`。启用后 WSL 与 Windows 共享同一 IP，端口绑定在 `0.0.0.0` 的服务**自动**从局域网可见，无需任何 portproxy 或防火墙规则。

| 维度 | Mirrored 模式 ✅ | PortProxy 方案 ❌ |
|------|------------------|-------------------|
| 配置复杂度 | 加一行 `.wslconfig` | 多步 + 管理员权限 |
| UAC 提权 | 不需要 | **必须**（从 WSL 无法绕过） |
| WSL IP 变化 | 不适用（共享 IP） | 需动态更新脚本 |
| 持久性 | 永久有效 | 依赖 schtasks 自启 |
| 适用场景 | Windows 11 22H2+ | 旧版 Windows 或仍需独立网络 |

### ⚠️ 关键陷阱：Mirrored 模式下 PortProxy 规则会破坏访问

如果 WSL 已在 Mirrored 模式下运行，旧的 PortProxy 规则（指向已不存在的 WSL 虚拟 IP，如 `172.19.x.x`）**会主动拦截流量**，导致局域网设备访问超时。这是因为 portproxy 接收到请求后尝试转发到不存在的目标 IP，而非让 WSL 服务直接在共享网络栈上处理。

**发现此问题的典型过程：**
1. 用户部署了新 Web 服务，`localhost` 可访问
2. 添加了 portproxy 规则（指向旧 WSL 虚拟 IP，如 172.19.156.154）
3. 局域网依然无法访问
4. 排查发现 WSL 实际已是 Mirrored 模式（WSL IP = Windows LAN IP），旧虚拟 IP 已消失

**排查方法：** 见下方「诊断」章节的 WSL 网络模式检测。

**修复：** 用管理员身份运行以下命令清除所有 portproxy 规则：

```batch
netsh interface portproxy delete v4tov4 listenaddress=<LAN_IP> listenport=<PORT>
```

> **从 WSL 无法程序化绕过 Windows UAC** — 所有 `Start-Process -Verb RunAs`、`schtasks /ru SYSTEM`、VBScript `ShellExecute runas` 等方法均会触发桌面 UAC 弹窗并挂起等待用户确认，无法静默执行。如果必须用 PortProxy，需要用户手动右键 → 以管理员身份运行脚本。

**何时选 PortProxy：** 使用旧版 Windows（< 22H2）、需要 WSL 独立网络栈（如 VPN 分流），或无法重启 WSL。

## 触发条件
- 部署了新的 Web 服务在 WSL 中，需要局域网访问
- 用户反馈"其他设备无法打开"时排查

## ⚠️ 关键陷阱：Windows 批处理文件的换行符与编码

**从 WSL 写入 Windows 的 `.cmd` 文件有两个陷阱需要同时检查：**

1. **换行符**：必须 CRLF (`\r\n`)，不能用 LF (`\n`)
2. **编码**：`write_file` 输出 UTF-8，而 Windows cmd.exe 中文版使用 GBK → 中文字符乱码

**必做预检：创建文件前执行 `file` 命令确认编码和换行符**

### 正确写入法（用 Python 控制 CRLF + 编码，推荐）

### 备选：从 WSL 弹出 UAC 窗口（需用户手动确认）

无需用户手动找脚本右键管理员运行，直接从 WSL 弹出 UAC 授权弹窗：

```bash
powershell.exe -Command "Start-Process powershell -Verb RunAs -ArgumentList '-NoProfile -Command \"netsh advfirewall firewall add rule name=''WSL Port <PORT>'' dir=in action=allow protocol=TCP localport=<PORT>; netsh interface portproxy add v4tov4 listenaddress=<LAN_IP> listenport=<PORT> connectaddress=<WSL_IP> connectport=<PORT>; Write-Host ''Done!''; pause\"'"
```

Windows 会弹出 UAC 窗口等待用户点「是」。**注意：此命令从 WSL 执行后会挂起直到用户在桌面确认 UAC 弹窗**，如果超时设置过短会导致命令失败。建议 timeout 设为 60s+。对于多端口场景，用户可以一次性在管理员 PowerShell 中手动执行命令，或切换至 Mirrored 网络模式彻底避免 UAC。

## 三步安装法

### 步骤 1：创建持久化 PowerShell 脚本

在 WSL 中用 Python 生成脚本到 Windows 桌面（注意 CRLF 换行符）：

```bash
LAN_IP="192.168.3.102"
PORT=5000
WIN_USER="yeyu_"

python3 << PYEOF
lan_ip = "$LAN_IP"
port = $PORT
content = f'''# WSL Port Forwarding - 开机自动配置
$logFile = "$env:USERPROFILE\\.wsl-port-forwarding.log"
$timestamp = Get-Date -Format "yyyy-MM-dd HH:mm:ss"
function Write-Log {{ param([string]\\$msg) "$timestamp \\$msg" | Out-File -Append -Encoding utf8 \\$logFile }}

Write-Log "=== Start ==="

for (\\$i = 0; \\$i -lt 30; \\$i++) {{
    \\$test = wsl hostname -I 2>\\$null
    if (\\$test -and \\$test.Trim() -ne "") {{ break }}
    Start-Sleep -Seconds 1
}}

\\$wslIP = (wsl hostname -I 2>\\$null).Trim().Split(' ')[0]
Write-Log "WSL IP: \\$wslIP"
if (-not \\$wslIP) {{ Write-Log "FAIL"; exit 1 }}

netsh interface portproxy delete v4tov4 listenaddress={lan_ip} listenport={port} 2>\\$null
netsh interface portproxy add v4tov4 listenaddress={lan_ip} listenport={port} connectaddress=\\$wslIP connectport={port}
Write-Log "Proxy added: {lan_ip}:{port} -> \\$wslIP:{port}"

\\$fwCheck = netsh advfirewall firewall show rule name="WSL Port {port}" 2>&1
if (\\$fwCheck -match "没有匹配") {{
    netsh advfirewall firewall add rule name="WSL Port {port}" dir=in action=allow protocol=TCP localport={port}
    Write-Log "Firewall rule created"
}}
Write-Log "=== Complete ==="
'''
with open(f'/mnt/c/Users/$WIN_USER/Desktop/update-port-forward.ps1', 'w', newline='') as f:
    f.write(content.replace('\\$', '$'))
PYEOF
```

### 步骤 2：创建一键安装 CMD 脚本

```bash
cat > /mnt/c/Users/<username>/Desktop/install-lan-access.cmd << 'CMDEOF'
@echo off
title WSL 局域网访问安装

echo === 1. 端口转发 ===
netsh interface portproxy add v4tov4 listenaddress=<LAN_IP> listenport=<PORT> connectaddress=<WSL_IP> connectport=<PORT>

echo === 2. 防火墙规则 ===
netsh advfirewall firewall add rule name="WSL Port <PORT>" dir=in action=allow protocol=TCP localport=<PORT>

echo === 3. 开机自启任务 ===
schtasks /create /tn "WSL-Port-<PORT>" /tr "powershell.exe -ExecutionPolicy Bypass -File \"%%USERPROFILE%%\Desktop\update-port-forward.ps1\"" /sc onstart /delay 0000:30 /ru SYSTEM /rl highest /f

echo === 4. 立即执行 ===
powershell.exe -ExecutionPolicy Bypass -File "%%USERPROFILE%%\Desktop\update-port-forward.ps1"

echo === 5. 验证 ===
netsh interface portproxy show all
pause
CMDEOF
```

替换占位符后，脚本生成完成。

### 步骤 3：用户执行

告诉用户在 Windows 桌面 **右键 → 以管理员身份运行** `install-lan-access.cmd`。

## 关联文件

| 文件 | 说明 |
|:----|:-----|
| `scripts/check-and-report-ports.sh` | 多端口审计脚本 — 一键检查 service/proxy/firewall/LAN |
| `references/multi-port-batch-setup.md` | 批量补全缺失端口的实战方案（含 Step-by-Step + 代码模板） |

## 快速审计脚本

skill 内置了一个自动化审计脚本，对一组端口逐一检查服务状态、portproxy 规则、防火墙规则和局域网可达性：

```bash
# 默认检查 5000 5001 5002
bash ~/.hermes/skills/devops/wsl-port-forwarding/scripts/check-and-report-ports.sh

# 指定任意端口
bash ~/.hermes/skills/devops/wsl-port-forwarding/scripts/check-and-report-ports.sh 8080 3000 5173
```

输出表格一目了然，每列直接指出问题所在（MISSING / NO / YES）。适合新服务部署后的快速验证。

## Hyper-V 防火墙 — Mirrored 模式的第二层拦截

在 WSL Mirrored 网络模式下，**有两个独立的防火墙层**都需要放行端口：

| 层 | 管理命令 | 说明 |
|----|---------|------|
| **Windows 防火墙** | `netsh advfirewall firewall add rule ...` | 传统 Windows 防火墙 |
| **Hyper-V 防火墙** | `New-NetFirewallHyperVRule ...` | WSL 虚拟交换机专用防火墙，**独立于 Windows 防火墙** |

端口必须在**两层同时放行**，缺一不可。从 Windows 本机 `localhost` 访问不受这两层影响，但从 LAN 设备访问时两层的规则都会检查。

### 排查方法

```powershell
# 写 PS1 文件执行，避免 $_ 转义问题
Get-NetFirewallHyperVRule | Where-Object { $_.DisplayName -like '*WSL*' } | Format-Table DisplayName,Enabled,LocalPort,Action
```

**典型失败模式：** 端口 5000 有 Windows 防火墙规则但无 Hyper-V 防火墙规则 → 手机能访问 5002（两层都有规则）但无法访问 5000（缺 Hyper-V 层规则）。这是最常见且最隐蔽的失败模式。

### 添加规则（需管理员权限）

```powershell
New-NetFirewallHyperVRule -DisplayName "WSL_Port_<PORT>" -Direction Inbound -Protocol TCP -LocalPort <PORT> -Action Allow
```

从 WSL 触发 UAC 提权执行（需用户在 Windows 上点「是」）：

```powershell
$psi = New-Object System.Diagnostics.ProcessStartInfo
$psi.FileName = "powershell.exe"
$psi.Arguments = "-NoProfile -Command `"New-NetFirewallHyperVRule -DisplayName 'WSL_Port_<PORT>' -Direction Inbound -Protocol TCP -LocalPort <PORT> -Action Allow`""
$psi.Verb = "runas"
$psi.UseShellExecute = $true
[System.Diagnostics.Process]::Start($psi) | Out-Null
```

## Windows 回环保护 — 本机无法通过公网 IP 访问

从 Windows **本机** 浏览器访问 `http://192.168.x.x:<PORT>/` 会失败（HTTP 超时），但从手机/平板访问正常。

**根因：** Windows 默认启用回环保护（Loopback Protection），禁止从自身连接到自己的公网 IP。这是 Windows 系统行为，**不是配置故障**，无需修复。

**解决方案：**
- **本机**用 `http://localhost:<PORT>/` 访问
- **局域网设备**用 `http://<LAN-IP>:<PORT>/` 访问

## 诊断（当用户说"还是打不开"）

### 0. 检测 WSL 网络模式（必须先做！）

在排查前，先判断 WSL 是哪个网络模式，因为这决定了是否需要 portproxy：

```bash
# 检测方法 1：检查 WSL 是否有独立虚拟 IP
wsl_ip=$(ip addr show 2>/dev/null | grep "inet " | grep -v "127.0.0.1" | grep -v "10\.255" | awk '{print $2}' | cut -d/ -f1)
# 检测方法 2：检查 Windows 侧 LAN IP
win_lan_ip=$(powershell.exe -NoProfile -Command "Get-NetIPAddress -AddressFamily IPv4 | Where-Object {\$_.PrefixOrigin -eq 'Dhcp'} | Select-Object -First 1 -ExpandProperty IPAddress" 2>/dev/null | tr -d '\r\n')

echo "WSL IP: $wsl_ip"
echo "Windows LAN IP: $win_lan_ip"

if [ "$wsl_ip" = "$win_lan_ip" ]; then
    echo "→ Mirrored 模式（共享 IP）— 不需 portproxy"
else
    echo "→ 传统 NAT 模式（独立虚拟 IP）— 需要 portproxy"
fi
```

**加粗告诫：** 如果检测到是 Mirrored 模式，**不要添加 portproxy 规则**。已有旧规则应删除（见上文「关键陷阱」）。

### 0a. 审计现有规则（新服务部署后的第一步）

新部署一个 Web 服务后，先检查现有 portproxy 规则，而非假设已全部配置好：

```bash
powershell.exe -ExecutionPolicy Bypass -NoProfile -Command "netsh interface portproxy show all"
```

输出示例：
```
侦听 ipv4:                 连接到 ipv4:
地址            端口        地址            端口
--------------- ----------  --------------- ----------
192.168.3.102   5000        172.19.156.154  5000
```

**如果新服务端口未出现** → 补充该端口（见下方的 .bat 方案或三步安装法）。

### 0b. WSL IP vs Windows LAN IP — 关键区别

WSL 运行在 Hyper-V 虚拟机中，有自己的虚拟网卡 IP（`172.x.x.x` 网段），而 Windows 宿主有独立的 LAN IP（`192.168.x.x` 或 `10.x.x.x`）。

```bash
# 查看 WSL 内部 IP
ip addr show eth0 | grep inet     # → 172.19.156.154

# 查看 Windows 宿主的 LAN IP（写 PS1 文件防转义）
echo '$adapter = Get-NetIPAddress -AddressFamily IPv4 | Where-Object { $_.PrefixOrigin -eq "Dhcp" }; foreach ($a in $adapter) { Write-Host ($a.InterfaceAlias + ": " + $a.IPAddress) }' | powershell.exe -ExecutionPolicy Bypass -NoProfile -
```

**端口转发本质**：将 `Windows-LAN-IP:PORT` → `WSL-内部-IP:PORT`。两个 IP 不同网段，必须显式映射。

**常见误区**：认为 `localhost:PORT` 可访问 = 局域网也可访问。实际上 `localhost` 走的是 WSL 内部回环，而局域网访问走的是 Windows 网卡，需要 portproxy 桥接。

### 0c. 一键生成 .bat 方案（针对缺少单个端口）

当端口转发不完整、且不想跑完整安装流程时，从 WSL 生成 .bat 文件到 Windows 桌面，让用户右键→以管理员身份运行：

```bash
WIN_USER="yeyu_"
PORT=5002
LAN_IP="192.168.3.102"
WSL_IP="172.19.156.154"

python3 << PYEOF
user = "$WIN_USER"
lan_ip = "$LAN_IP"
wsl_ip = "$WSL_IP"
port = int("$PORT")

content = f"""@echo off
chcp 65001 >nul
echo [1/2] Adding portproxy for port {port}...
netsh interface portproxy add v4tov4 listenaddress={lan_ip} listenport={port} connectaddress={wsl_ip} connectport={port}
echo [2/2] Current rules:
netsh interface portproxy show all
echo.
echo Complete! Press any key to exit.
pause >nul
"""

path = f'/mnt/c/Users/{user}/Desktop/add_port_{port}.bat'
with open(path, 'w', newline='\r\n') as f:
    f.write(content)
print(f"Created: {path}")
print("Tell user: Right-click the file -> Run as administrator")
PYEOF
```

这个模式适用于：仅缺少数个端口、新服务部署后的快速修复、或用户不想运行完整安装脚本的场景。

### 三层测试法（从 WSL 内部诊断）

```bash
PORT=5000
LAN_IP="192.168.3.102"

# Test 1: localhost 直连 — 验证 Flask 服务本身
curl -s --connect-timeout 3 http://127.0.0.1:$PORT/v2/health

# Test 2: 通过端口转发回环 — 验证 portproxy 是否工作
# （从 WSL → Windows LAN IP → portproxy → WSL）
curl -s --connect-timeout 5 http://$LAN_IP:$PORT/v2/health

# Test 3: 从 Windows 本机验证（在 powershell 中）
powershell.exe -NoProfile -Command "curl.exe -s --connect-timeout 5 http://$LAN_IP:$PORT/v2/health"
```

| 测试结果 | 含义 |
|:---------|:-----|
| T1 ✅ / T2 ✅ / T3 ✅ | 全部正常，检查用户设备网络 |
| T1 ✅ / T2 ❌ / T3 ❌ | **防火墙拦截** → 检查 `netsh advfirewall firewall show rule name=...` |
| T1 ✅ / T2 ✅ / T3 ❌ | Windows 本机问题，少见 |
| T1 ❌ | **服务未运行** → `ps aux \| grep app.py` 检查进程 |

### 防火墙规则检查

```bash
powershell.exe -NoProfile -Command "netsh advfirewall firewall show rule name='WSL Port <PORT>' dir=in verbose"
```

成功输出应包含：
```
规则名称:        WSL Port <PORT>
已启用:          是
操作:            允许
配置文件:        域,专用,公用
本地端口:        <PORT>
```

如果显示"没有与指定标准相匹配的规则"，说明**规则未创建成功**（常见原因：之前运行的 .cmd 文件换行符是 LF 而非 CRLF，导致命令实际未执行）。需要用 Python 重新生成带 CRLF 的 .cmd 文件。

### CMD 输出乱码 / 编码踩坑

**根本原因：** WSL 用 `write_file` 创建的 `.cmd` 文件编码为 UTF-8，而中文 Windows 的 `cmd.exe` 默认使用 GBK（CP936）解析，中文字符显示为乱码；更糟的是，乱码可能导致 `netsh` 命令的参数被截断或解析错误，造成规则实际上**未被执行**。

**必检：创建 .cmd 文件前用 `file` 命令检查编码**

```bash
file /path/to/script.cmd
# 如果是 "UTF-8 Unicode text" → 在 cmd.exe 中中文会乱码
```

**两种修复方案（二选一）：**

1. **纯 ASCII（推荐，零依赖）** — 脚本中所有提示信息用英文，彻底避免编码问题。
2. **代码页切换** — 在 `.cmd` 文件开头加 `chcp 65001 >nul` 将控制台切换到 UTF-8。

**换行符陷阱：** Hermes 的 `write_file` 默认写入 LF 换行符。Windows CMD 需要 CRLF，否则命令可能粘行报错。正确创建方式见下方 Python 写入法。

**完整参考：** 跨平台编码检查与处理详见 `wsl-windows-interop` 技能的 `references/cross-platform-encoding.md`。

### 常见失败原因

| 现象 | 根因 | 解决 |
|:----|:-----|:-----|
| `localhost` 可访问，`<LAN_IP>` 不行 | **防火墙未开放端口** | 以管理员运行脚本，或手动执行 `netsh advfirewall firewall add rule name="WSL Port <PORT>" dir=in action=allow protocol=TCP localport=<PORT>` |
| 之前能连，WSL重启后断了 | **WSL IP 变化** | 开机自启任务会自动更新；可手动运行 `update-port-forward.ps1` |
| 从 WSL 内 curl `<LAN_IP>` 超时 | **Windows 禁止 WSL→宿主回环** | 这是正常的。正确测试方式：从另一台 LAN 设备访问，或在 Windows CMD 中用 `curl.exe` |
| CMD 显示 `'xxx' 不是内部或外部命令` | **.cmd 文件换行符为 LF 而非 CRLF** | 用 Python 重写：`with open(path, 'w', newline='')` + 内容用 `\r\n` 分隔 |
| CMD 出现乱码字符 | **UTF-8 中文在 GBK 控制台显示** | 脚本开头加 `chcp 65001 >nul`，或所有输出用英文 |

## 持久化原理

| 组件 | 持久性 | 说明 |
|:----|:------|:-----|
| `netsh portproxy` 规则 | Windows 重启不丢失 | 但 WSL IP 变化后目标地址失效 |
| `netsh advfirewall` 规则 | Windows 重启不丢失 | 一次配置永久有效 |
| `schtasks` 开机自启 | 永久有效 | 以 SYSTEM 身份运行，无弹窗、无 UAC |
| `.ps1` 脚本 | 检测启动时 WSL IP | 更新 portproxy 目标地址 |

## 清理
```powershell
netsh interface portproxy delete v4tov4 listenaddress=<LAN_IP> listenport=<PORT>
netsh advfirewall firewall delete rule name="WSL Port <PORT>"
schtasks /delete /tn "WSL-Port-<PORT>" /f
```
