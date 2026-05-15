---
name: wsl-port-forwarding
description: 为 WSL Web 服务配置 Windows 端口转发 + 防火墙规则 + 开机自启持久化，使局域网其他设备可通过 Windows 宿主的 LAN IP 访问。
---

# WSL 端口转发配置（完整版）

在 WSL 中新部署 Web 服务后，使其可从局域网永久访问 `http://<Windows-LAN-IP>:<PORT>/`。

## 方案选择：Mirrored 模式 vs PortProxy

## 方案选择：Mirrored 模式 vs PortProxy

**Mirrored 模式**（`networkingMode=mirrored`） 在 Windows 11 22H2+ 中可用，WSL 与 Windows 共享同一 IP。但**实测在 Insider/Preview 版中存在不可靠现象**——端口未必真实暴露到 Windows 网络栈。

| 维度 | Mirrored 模式 | PortProxy 方案 |
|------|---------------|----------------|
| 配置复杂度 | 加一行 `.wslconfig` | 多步 + 管理员权限 |
| UAC 提权 | 不需要 | **必须** |
| WSL IP 变化 | 不适用（共享 IP） | 需动态更新（NAT 模式）|
| 持久性 | 依赖 Insider Build 稳定性 | 规则永久有效 |
| Insider Build 兼容性 | ❌ **已知不可靠**（port 可能不真实暴露） | ✅ 可靠 |
| 推荐度 | 仅适用于非 Insider 稳定版 | 综合推荐 |

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
| `references/comprehensive-7-step-diagnosis.md` | 7 步诊断流程 + 快速判读表 + 实战案例（2026-05-15） |

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

### 前置检查：服务绑定地址（Mirrored 模式下最易忽略）

在 Mirrored 模式下，防火墙和网络全部正确，但服务绑在了 `127.0.0.1` 而非 `0.0.0.0`——从局域网依然无法访问。

```bash
ss -tlnp | grep <PORT>
# 期望输出：0.0.0.0:<PORT>  （接受外部连接）
# 错误输出：127.0.0.1:<PORT> （仅本地可访问）
```

Flask app 的 `app.run(host='0.0.0.0', port=5002)` 或 `--host=0.0.0.0` 参数确保绑定所有接口。

**诊断顺序（从服务侧开始，而非从网络侧）：**

1. **服务本身运行？** → `curl -s --connect-timeout 3 http://127.0.0.1:$PORT/`
2. **绑定 0.0.0.0？** → `ss -tlnp | grep $PORT`
3. **WSL 网络模式？** → 对照 `cat /mnt/c/Users/<user>/.wslconfig` 的 `networkingMode`
4. **防火墙允许？** → `powershell.exe ... netsh advfirewall firewall show rule ...`
5. **LAN 可达？** → `curl -s http://<LAN_IP>:$PORT/`

这个顺序确保先排除最靠近服务的问题，再向外排查网络层。

### 三层测试法（从 WSL 内部诊断）

```bash
PORT=5000
LAN_IP="192.168.3.102"

# Test 1: localhost 直连 — 验证 Flask 服务本身
curl -s --connect-timeout 3 http://127.0.0.1:$PORT/v2/health

# Test 2: 通过 LAN IP 直连（Mirrored 模式下不走 portproxy）
curl -s --connect-timeout 5 http://$LAN_IP:$PORT/v2/health

# Test 3: 从 Windows 本机验证（在 powershell 中）
powershell.exe -NoProfile -Command "curl.exe -s --connect-timeout 5 http://$LAN_IP:$PORT/v2/health"
```

| 测试结果 | 含义 |
|:---------|:-----|
### ⚠️ 重要：WSL 内部 curl LAN IP ≠ 真实 LAN 可达

在 Mirrored 模式下从 WSL 内部执行 `curl http://$LAN_IP:$PORT/` 返回 200 **不代表**局域网其他设备能访问。WSL 可能通过自己的网络栈直接响应，而 Windows 网络栈实际上未暴露该端口。

**诊断金标准：从 Windows 侧检查端口是否真实暴露**

```powershell
# 从 Windows 检查 5002 端口是否真的有 LISTENING 状态
# 在 WSL 中执行：
powershell.exe -NoProfile -Command "netstat -ano | findstr ':5002'" 2>&1

# 期望输出（端口真实暴露）：
#   TCP    0.0.0.0:5002    0.0.0.0:0    LISTENING    12345

# 异常输出（端口未暴露，即使 WSL 内部能访问）：
#   空输出 或 只有 TIME_WAIT 等非 LISTENING 状态
```

### 🔍 进阶诊断：内核级端口捕获（`netstat` 空 + 绑定失败）

**场景：** `netstat -ano` 完全找不到端口的 LISTENING 记录，但 Windows 上的 Python 或其他进程也无法绑定到该端口（报 `Cannot bind` / `Address already in use`）。

**根因：** WSL Mirrored 模式在 Windows 内核层面捕获了该端口，但**没有在用户态 `netstat` 中注册为 LISTENING 状态**。端口处于一种「半透明」状态：从 WSL 内部可用，从 Windows 用户态进程可用（有绑定冲突），但从物理网卡不可见。

**完整诊断链（4 步确认）：**

| 步骤 | 测试 | 预期（异常） | 含义 |
|:----|:-----|:------------|:-----|
| S1 | `ss -tlnp \| grep <PORT>`（WSL 内） | ✅ LISTEN | 服务在 WSL 内正常运行 |
| S2 | `netstat -ano \| findstr :<PORT>`（Win） | ❌ 空输出 | 端口未在 Windows netstat 注册 |
| S3 | Python/localhost curl 能否连 127.0.0.1:<PORT> | ✅ 能连 | portproxy loopback 正常 |
| S4 | Windows Python 能否绑定 `<PORT>` | ❌ Cannot bind | 端口在内核被 WSL 锁定 |

**当 S1✅ + S2❌ + S3✅ + S4❌ → 确认内核级端口捕获。修复方法见下方「PortProxy `0.0.0.0` 不捕获外部接口流量」。**

**修复方案（当 Windows `netstat` 无 LISTENING 端口时）：**

在 Mirrored 模式下，WSL 没有独立虚拟 IP，portproxy 的目标地址用 `127.0.0.1`。但必须**指定具体 LAN IP 而非 `0.0.0.0`** 才能捕获物理网卡流量：

```batch
:: ❌ 错误：0.0.0.0 无法捕获 WLAN 物理网卡流量（Insider Build）
netsh interface portproxy add v4tov4 listenaddress=0.0.0.0 listenport=<PORT> connectaddress=127.0.0.1 connectport=<PORT>

:: ✅ 正确：指定具体 LAN IP，强制绑定到物理网卡接口
netsh interface portproxy add v4tov4 listenaddress=<LAN_IP> listenport=<PORT> connectaddress=127.0.0.1 connectport=<PORT>
```

这与传统 NAT 模式（目标地址是 WSL 虚拟 IP 如 `172.19.x.x`）不同，注意区分。

### ⚠️ PortProxy `0.0.0.0` 不捕获外部接口流量（Mirrored 模式深坑）

**场景：** 已添加 portproxy `listenaddress=0.0.0.0` 且正确，`127.0.0.1:<PORT>` 可访问，但 `192.168.x.x:<PORT>` 从外部设备访问不通。

**根因：** 在某些 Windows 构建版本（尤其是 Insider/Preview 版本）中，portproxy 绑定的 `0.0.0.0` 仅捕获 loopback 接口流量，不会捕获到达物理网卡（如 WLAN）的入站流量。这是 Windows 网络栈底层行为，非配置错误。

**诊断三步法：**

| 测试 | 命令 | 预期结果 |
|------|------|----------|
| T1: localhost | `curl http://127.0.0.1:<PORT>/` | ✅ 正常 |
| T2: 从 WSL 用 LAN IP | `curl http://<LAN_IP>:<PORT>/` | ✅ 正常（WSL 内部网络栈） |
| T3: 从 Windows 用 LAN IP | `powershell.exe curl.exe http://<LAN_IP>:<PORT>/` | ❌ 超时 |

如果 T1✅, T2✅, T3❌，且 portproxy 规则为 `0.0.0.0:<PORT> → 127.0.0.1:<PORT>` → 确认此问题。

**修复方案（二选一）：**

**方案 A：重新创建 portproxy 指定具体 LAN IP（需管理员权限）**

```batch
netsh interface portproxy delete v4tov4 listenport=<PORT> listenaddress=0.0.0.0
netsh interface portproxy add v4tov4 listenaddress=<LAN_IP> listenport=<PORT> connectaddress=127.0.0.1 connectport=<PORT>
```

指定具体 LAN IP 而非 `0.0.0.0`，强制 portproxy 绑定到物理网卡接口。

**方案 B：Python TCP 转发器（无需管理员权限）**

当无法获取管理员权限，或方案 A 仍然失效时，在 Windows 侧运行 Python TCP 转发器：

```powershell
# 在 Windows CMD 或 PowerShell 中运行
python C:\path\to\forward.py
```

转发器脚本见 `references/python-tcp-forwarder.md`，原理是在 Windows 应用层监听 `0.0.0.0:<PORT>`，将每个连接转发到 `127.0.0.1:<PORT>`。端口 > 1024 时无需提权，Windows 防火墙首次运行时弹窗询问，点击「允许」即可。

| 测试结果 | 含义 |
|:---------|:-----|
| T1 ✅ / Windows netstat 无 LISTENING | **Mirrored 模式端口未真实暴露** → 加 portproxy (connectaddress=127.0.0.1) |
| T1 ✅ / T2 ✅ / portproxy(0.0.0.0) 加后 127.0.0.1 通但 LAN IP 不通 | **portproxy 0.0.0.0 不捕获物理网卡流量** → 改用 `listenaddress=<LAN_IP>` 或 Python TCP 转发器 |
| T1 ✅ / T2 ✅ / T3 ✅ | 全部正常，检查用户设备网络 |
| T1 ✅ / T2 ❌ / T3 ❌ | **防火墙拦截** → 检查 `netsh advfirewall firewall show rule name=...` |
| T1 ✅ / T2 ✅ / T3 ❌ | Windows 本机问题，少见 |
| T1 ❌ | **服务未运行或绑定地址不对** → `ss -tlnp` 检查进程和绑定地址 |
| T1 ✅ / T2 ❌ / 绑定 127.0.0.1 | **服务绑在 127.0.0.1 而非 0.0.0.0** → 改 `host='0.0.0.0'` |

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

**完整参考：** 跨平台编码检查与处理详见 `wsl-windows-interop` 的 `references/cross-platform-encoding.md`。PortProxy 失效时的 Python TCP 转发器备选方案详见 `references/python-tcp-forwarder.md`。

### `.bat` / `.cmd` 文件创建方法优先级

创建供用户右键→管理员运行的 Windows 批处理文件时，按以下优先级选择：

| 方法 | 适用场景 | 编码控制 | 换行符控制 | 推荐度 |
|------|----------|----------|-----------|:------:|
| Python `execute_code` + `encoding='gbk'` | 含中文提示的 .bat | ✅ GBK | ✅ CRLF | ⭐⭐⭐ |
| Python `execute_code` + `encoding='ascii'` | 纯英文 .bat（零编码问题） | ✅ ASCII | ✅ CRLF | ⭐⭐⭐ |
| Hermes `write_file`（默认） | Linux 脚本 | ❌ UTF-8 | ❌ LF | ⭐ |
| Hermes `write_file` + 手动转换 | 无 Python 环境时 | 需额外步骤 | 需额外步骤 | ⭐ |

**推荐方案**：用 Python 创建，显式指定编码和换行符：

```python
content = """@echo off
echo [OK] Port forwarding rule added!
pause
"""
with open('/mnt/c/Users/yeyu_/Desktop/add_port.bat', 'w', encoding='gbk') as f:
    f.write(content)
```

或用纯 ASCII 彻底避开编码问题。

### 常见失败原因

| 现象 | 根因 | 解决 |
|:----|:-----|:-----|
| `localhost` 可访问，`<LAN_IP>` 不行 | **防火墙未开放端口** | 以管理员运行脚本，或手动执行 `netsh advfirewall firewall add rule name="WSL Port <PORT>" dir=in action=allow protocol=TCP localport=<PORT>` |
| 之前能连，WSL重启后断了 | **WSL IP 变化** | 开机自启任务会自动更新；可手动运行 `update-port-forward.ps1` |
| 从 WSL 内 curl `<LAN_IP>` 超时 | **Windows 禁止 WSL→宿主回环** | 这是正常的。正确测试方式：从另一台 LAN 设备访问，或在 Windows CMD 中用 `curl.exe` |
| CMD 显示 `'xxx' 不是内部或外部命令` | **.cmd 文件换行符为 LF 而非 CRLF** | 用 Python 重写：`with open(path, 'w', newline='')` + 内容用 `\r\n` 分隔 |
| CMD 出现乱码字符 | **UTF-8 中文在 GBK 控制台显示** | 脚本开头加 `chcp 65001 >nul`，或所有输出用英文 |

### 服务启动失败排查（步骤 1 失败时先查这个）

当 `curl 127.0.0.1:<PORT>` 超时，说明**服务本身未运行**。不要直接跳到网络层，先诊断服务为啥没起来：

1. **检查进程** → `ps aux | grep -E 'app.py|python.*app'` — 看服务进程是否存在
2. **检查后台日志** → 如果服务是用 `terminal(background=true)` 启动的，用 `process(action='log')` 查看完整错误栈
3. **常见失败模式**：
   - Python import error（module not found、blueprint name mismatch 等）
   - Flask Blueprint 注册名不匹配（如 `__init__.py` 中叫 `chinese_bp`，`app.py` 却 import `chinese_v3_bp`）
   - 端口被占用 → `ss -tlnp | grep <PORT>` 确认
   - SQLite 数据库创建失败（权限或路径问题）
4. **修复后重启** → 先修复错误，再启动服务，重新执行步骤 1 确认通过

> 经验教训：用户说「手机打不开」时，直觉反应是网络问题，但至少 1/3 的故障是服务根本没启动。先查服务，再查网络。

### 常见失败原因表（补充）

| 现象 | 根因 | 解决 |
|:----|:-----|:-----|
| `curl 127.0.0.1` 超时，进程不存在 | 服务启动失败（import error 等） | 查后台日志 `process(action='log')` → 定位具体错误 → 修复后重启 |

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

## 复发模式识别（重复出问题时的应对）

当 WSL 端口转发问题**反复出现**（3+ 次），说明每次的"修复"没有触及根因。按以下步骤打破复发循环：

### 步骤 1：重建完整时间线

回溯每次复发的时间、触发条件、解决方案。典型模式包括：

| 模式 | 特征 | 对策 |
|:----|:-----|:-----|
| IP 漂移 | NAT 模式下 WSL 重启 IP 变 | 切换 Mirrored 模式，或计划任务动态更新 |
| 模式切换不清 | NAT→Mirrored 后旧规则残留 | 一步到位：先检测当前模式再决策 |
| Insider Build 不兼容 | Mirrored 下端口半透明 | Portproxy 用具体 LAN IP 而非 0.0.0.0 |
| 第二层防火墙 | WinFW 有规则但 Hyper-V FW 无 | 添加 New-NetFirewallHyperVRule |

### 步骤 2：完整诊断（不得跳过任何一步）

按此顺序执行：

```
1. 服务运行？   → curl 127.0.0.1:<PORT>
2. 绑定地址？   → ss -tlnp | grep <PORT> → 必须 0.0.0.0
3. WSL 网络模式？→ cat /mnt/c/Users/<user>/.wslconfig + 比对 WSL IP vs LAN IP
4. Windows netstat？→ netstat -ano | findstr :<PORT> → 看有无 LISTENING
5. Windows Python 能否绑定？→ Python socket.bind → 看有无内核占用
6. Portproxy 规则？→ netsh interface portproxy show all
7. 防火墙两层？→ WinFW + Hyper-V FW 都要检查
8. 从本机 Windows 侧？→ PowerShell Test-NetConnection 127.0.0.1 <PORT>
```

### 步骤 3：一次性根因修复

| 诊断结论 | 根因 | 修复动作 |
|:---------|:-----|:---------|
| netstat 无 LISTENING + Python 绑定冲突 | WSL 内核级端口捕获 | portproxy 用 listenaddress=<LAN_IP> 而非 0.0.0.0 |
| netstat 有 LISTENING 但 LAN 不通 | 第二层防火墙缺失 | 加 Hyper-V Firewall rule |
| 旧 portproxy 指向死 IP | 模式切换后规则残留 | 清理所有旧规则再重新配置 |
| portproxy 规则存在但 0.0.0.0 不变 | Insider Build bug | 改 listenaddress 为具体 LAN IP |

**Hyper-V 防火墙规则最易遗漏** — Mirrored 模式下有两层防火墙，加完 WinFW 务必再加 Hyper-V FW。
