---
name: wsl-windows-interop
title: WSL-Windows 交叉诊断与系统管理
description: 从 WSL 内部查询 Windows 系统信息、事件日志、系统状态的技术集合。涵盖 PowerShell/wevtutil 在 WSL bash 中的执行技巧、shell 转义陷阱、Windows 事件 ID 参考。
tags: [wsl, windows, interop, diagnostics, event-log, powershell, wevtutil]
---

# WSL-Windows 交叉诊断

从 WSL 内部查询 Windows 宿主机的系统状态（事件日志、启动原因、蓝屏记录、电源事件等），不依赖第三方工具或远程连接。

## 触发条件

- 用户问"Windows 上次为什么重启/关机"
- 需要排查 Windows 端问题（蓝屏、卡顿、驱动异常）
- 从 WSL 获取 Windows 系统信息（启动时间、内存、磁盘等）
- 需要跨 WSL/Windows 边界的系统诊断

## 核心原则

### 0. ⚠️ 编码预检（跨平台文件创建的第一准则）

**从 WSL 为 Windows 创建任何文件前，必须先检查目标环境的默认编码。**

WSL 文件系统默认 **UTF-8**，而 Windows 宿主中文环境下：
- **cmd.exe** → GBK/CP936（.bat/.cmd 文件）
- **PowerShell 5.x** → 兼容 UTF-8（.ps1 文件）
- **PowerShell 7+/Windows Terminal** → UTF-8

后果：WSL 的 `write_file` 创建的 `.cmd`/`.bat` 文件含中文字符时，在 Windows cmd.exe 中会显示为乱码，严重的会导致 `netsh` 等命令被错误解析。

**必做检查流程：**

```bash
# [步骤1] 识别目标环境的编码
# cmd.exe 在中文 Windows 的编码 = GBK/CP936
# 可通过以下命令验证 Windows 当前代码页
powershell.exe -NoProfile -Command "[System.Console]::OutputEncoding.CodePage"

# [步骤2] 用 file 命令检查待写入文件的编码
file /path/to/your-file.cmd

# [步骤3] 如果文件编码是 UTF-8 且将运行在 cmd.exe 中，
# 要么全用 ASCII（去掉中文），要么加 chcp 65001 切换代码页
```

**具体对策：**

| 目标环境 | 默认编码 | 对策 |
|----------|----------|------|
| cmd.exe (.bat/.cmd) | GBK/CP936 | ① 纯 ASCII（推荐，零坑）；② 或加 `chcp 65001 >nul` + UTF-8 内容 |
| PowerShell (.ps1) | UTF-8（兼容好） | 可含中文，但注意 PS1 脚本的语法解析器也受 BOM 影响 |
| Windows Terminal / PowerShell 7 | UTF-8 | 兼容性最好 |

**文件创建模式：**

```bash
# 正确：用 Python 显式控制换行符（CRLF for .cmd）
python3 << 'PYEOF'
content = "@echo off\r\nchcp 65001 >nul\r\necho Done\r\npause\r\n"
with open('/mnt/c/Users/yeyu_/Desktop/script.cmd', 'w', newline='') as f:
    f.write(content)
PYEOF

# 验证
file /mnt/c/Users/yeyu_/Desktop/script.cmd
# 期望输出: ... with CRLF line terminators
```

**陷阱提示：** `write_file` 工具写入的文件默认 UTF-8 + LF。这对 Linux 环境友好，对 Windows 环境可能不兼容。创建跨平台文件前务必做编码预检。

### ⚠️ 2026-05-19 真实踩坑：纯英文才是最稳的方案

给用户生成 `c_drive_analyzer.bat` 时，第一版含中文 echo 和注释，UTF-8 编码。用户在中文 Windows CMD 中运行，出现典型症状：

```
'垚鏃堕棿锛?DATETIMEOUTPUT' 不是内部或外部命令
'OTAL_BYTES' 不是内部或外部命令
命令语法不正确。
```

中文字符被 GBK 错误解析为乱码后，CMD 把拼凑后的字符串当作命令尝试执行。**根因**：`write_file` 写入 UTF-8 文件，CMD 用 GBK 读取，中文文本被错误分割。

**修复方案**（已沉淀为 `templates/c-drive-analyzer.bat`）：全量使用英文文本，0 中文。不需要 `chcp` 切换代码页，零坑。

**教训**：对 CMD 目标，纯 ASCII/纯英文输出是最稳的方案。加 `chcp 65001` 虽然可切换 UTF-8，但需要等宽字体支持中文显示（CMD 默认点阵字体常缺），反而不如纯英文可靠。

### ⚠️ 2026-05-20 追加：模板→部署同步约束

**修复模板不等于修复问题。** 2026-05-19 已将 `templates/c-drive-analyzer.bat` 改为纯 ASCII，但飞书 日常运营 文件夹中仍存有旧版 UTF-8 文件。用户下载旧版运行，重复遭遇同一编码错误。

**规则：修复技能模板后，必须检查是否有已部署的副本（飞书云盘、用户桌面、共享目录等），一并更新或替换。** 否则"已修复"只存在于技能内部，用户依然拿到旧版。

具体做法：
1. 模板修复完成后 → 检查飞书对应文件夹中是否有同名/同用途文件
2. 若有，重新生成并上传新版本（建议加版本号，如 `_v1.1`）
3. 告知用户旧版已废弃，提供新版下载链接

### 1. 工具路径

Windows 可执行文件位于 `/mnt/c/Windows/System32/`：

| 工具 | 路径 | 用途 |
|------|------|------|
| **systeminfo** | `/mnt/c/Windows/System32/systeminfo.exe` | 系统启动时间、基本配置 |
| **wevtutil** | `/mnt/c/Windows/System32/wevtutil.exe` | 查询事件日志（文本输出，防转义） |
| **powershell** | `/mnt/c/Windows/System32/WindowsPowerShell/v1.0/powershell.exe` | 复杂查询（事件过滤、CIM 调用） |

### 2. ⚠️ Shell 转义陷阱（高优先级）

**从 WSL bash 中传递内联 PowerShell 脚本时，`$_` 会被 bash 展开为 WSL 环境变量**，导致 PowerShell 语法错误。这是一个极其隐蔽的坑——错误输出是乱码中文，容易误以为是权限问题。

**解法一（推荐）：写 PS1 脚本文件**

```bash
write_file content="Get-WinEvent -FilterHashtable @{LogName='System';Id=1074} -MaxEvents 5 | Format-Table TimeCreated, Id, LevelDisplayName" path="/tmp/query.ps1"

powershell.exe -ExecutionPolicy Bypass -File /tmp/query.ps1
```

**解法二（简单查询用 wevtutil）：**

```bash
# wevtutil 不需要 PowerShell，无转义问题
# 按事件 ID 过滤
wevtutil.exe qe System "/q:*[System [(EventID=1074)]]" /c:5 /f:text /rd:true

# 按时间范围过滤（7天内）
wevtutil.exe qe System "/q:*[System [TimeCreated[timediff(@SystemTime) <= 604800000]]]" /c:10 /f:text /rd:true
```

### 3. 输出编码

`wevtutil` 输出包含中文和特殊字符，`grep` 等工具会报 "binary file matches"。**解决**：通过 `strings` 管道过滤，或直接全文输出查看。

### 4. ⚠️ PS1 脚本中的中文字符编码陷阱

**当从 WSL `write_file` 写入的 PS1 文件包含中文字符串时，PowerShell 会报解析错误**（通常显示为乱码的语法错误信息）。

**错误写法**（中文嵌入字符串中，引发解析错误）：
```powershell
Write-Host "错误: $_"          # 编码错乱 + $_ 转义双重问题
```

**正确写法**（使用字符串拼接分离中文字符）：
```powershell
Write-Host ("Error: " + $_.Exception.Message)   # 全英文字符串拼接
```

**临时目录方案**：PS1 文件写入 `/tmp/`，执行后自动清理，避免污染 Windows 文件系统。

```bash
wevtutil.exe qe System "/q:*[System [(EventID=1074)]]" /c:5 /f:text /rd:true 2>&1 | strings | grep -E "Date|Event ID|更新|重启"
```

### 4. 权限边界

从 WSL 调用 Windows 工具运行在普通用户权限下。部分事件 ID 在受限访问时可能返回空结果：

| 事件 ID | 非管理员可用 | 说明 |
|---------|:-----------:|------|
| **1074**（计划关机/重启） | ✅ | 可用，含发起进程、原因代码 |
| **1001**（崩溃/蓝屏） | ✅ | 可用，含 BugCheck 代码和 minidump 路径 |
| **506/507**（睡眠/唤醒） | ✅ | 可用 |
| **12**（启动时间戳） | ⚠️ | 可能返回重复数据或空 |
| **6005**（事件日志启动） | ❌ | 通常返回空 |
| **6006/6008**（关机相关） | ❌ | 通常返回空 |
| **41**（意外断电） | ❌ | 通常返回空 |

**推荐策略**：优先使用 `wevtutil` 查询（它通常绕过权限限制获得更多数据），或者用 `systeminfo` 获取基本启动时间。复杂查询用 PS1 脚本文件方式执行，但注意上述权限限制。

## 关键事件 ID 参考

### 系统重启/关机

| Event ID | 来源 | 含义 |
|----------|------|------|
| **1074** | User32 | 有计划的关机/重启（含发起进程、原因、用户） |
| **41** | Kernel-Power | 意外断电/崩溃后系统未经正常关机直接启动 |
| **6008** | EventLog | 上次关闭是意外的（前一次正常关机未记录） |
| **6005** | EventLog | 事件日志服务启动（= 系统正常启动的标志） |
| **6006** | EventLog | 事件日志服务停止（= 系统正常关机的标志） |

### 崩溃/蓝屏

| Event ID | 来源 | 含义 |
|----------|------|------|
| **1001** | Windows Error Reporting | BugCheck 蓝屏/崩溃报告（含 BugCheck 代码、minidump 路径） |
| **1002** | WER | 应用程序 Hang |
| **1000** | WER | 应用程序崩溃 |

### 电源管理

| Event ID | 来源 | 含义 |
|----------|------|------|
| **506** | Kernel-Power | 系统进入睡眠 |
| **507** | Kernel-Power | 系统从睡眠唤醒 |
| **1** | Kernel-General | 系统时间变更（通常伴随唤醒/电源事件） |
| **42** | Kernel-Power | 系统正在进入睡眠（休眠准备） |
| **12** | Kernel-General | 操作系统启动时间戳 |

### Windows Update

| Event ID | 来源 | 含义 |
|----------|------|------|
| **19** | WindowsUpdateClient | 安装完成 — 重新启动挂起 |
| **43** | WindowsUpdateClient | 更新已下载 |
| **44** | WindowsUpdateClient | 更新正在安装 |
| **24** | Kernel-General | 系统时间更新（有时关联更新后重启） |

## 可复用模板

该技能下提供以下可复用的 Windows 管理批处理模板（`templates/` 目录）：

| 模板 | 用途 | 
|------|------|
| `c-drive-analyzer.bat` | C盘空间分析 — 6项检查（磁盘/Temp/下载/缓存/更新/Win.old），输出到桌面txt报告 |
| `c-drive-cleanup.bat` | C盘安全清理 — 清Temp+浏览器缓存+回收站+桌面安装包 |

使用方式：从模板目录复制内容后，通过 `write_file` 写入 `/mnt/c/Users/Public/` 或用户桌面等 Windows 可访问路径，然后告知用户在 CMD（管理员）中运行。

## 查询策略

### 查询上次重启原因

```bash
# 方法 1：wevtutil（最可靠，无转义问题）
wevtutil.exe qe System "/q:*[System [(EventID=1074)]]" /c:5 /f:text /rd:true 2>&1

# 方法 2：PowerShell 脚本文件（复杂查询）
# 写入 /tmp/query.ps1，然后用 -File 执行
```

1074 事件的关键属性（由 Properties 数组提供）：
- **Properties[0]**: 发起进程路径 (如 `C:\WINDOWS\servicing\TrustedInstaller.exe`)
- **Properties[2]**: 关机类型 (如 "操作系统: 更新(计划内)")
- **Properties[4]**: 原因代码
- **Properties[6]**: 用户信息

### 检查蓝屏记录

```bash
wevtutil.exe qe System "/q:*[System [(EventID=1001)]]" /c:3 /f:text /rd:true 2>&1 | strings | grep -E "Date|BugCheck|Minidump|0x[0-9A-Fa-f]{8}"
```

输出中的 `0x00000116` 等是 BugCheck 代码，对应 Windows 蓝屏错误码。

### 检查电源/睡眠历史

```bash
wevtutil.exe qe System "/q:*[System [(EventID=506 or EventID=507)]]" /c:20 /f:text /rd:true 2>&1 | strings | grep -E "Date|Event ID"
```

### 获取系统启动时间

```bash
/mnt/c/Windows/System32/systeminfo.exe 2>/dev/null | strings | grep -i "启动时间\|Boot Time"
```

或者用 PowerShell：
```
powershell.exe -ExecutionPolicy Bypass -Command "Get-CimInstance Win32_OperatingSystem | Select-Object LastBootUpTime"
```

## 常见 BugCheck 代码参考

| BugCheck | 名称 | 常见原因 |
|----------|------|----------|
| `0x00000116` | VIDEO_TDR_FAILURE | 显卡超时/无响应，驱动问题 |
| `0x00000050` | PAGE_FAULT_IN_NONPAGED_AREA | 内存访问错误，驱动或硬件问题 |
| `0x0000019` | BAD_POOL_HEADER | 驱动程序内存池损坏 |
| `0x0000007e` | SYSTEM_THREAD_EXCEPTION_NOT_HANDLED | 驱动或系统服务异常 |
| `0x000000d1` | DRIVER_IRQL_NOT_LESS_OR_EQUAL | 驱动程序试图访问不可访问的内存 |
| `0x00000133` | DPC_WATCHDOG_VIOLATION | 驱动或硬件未在时限内响应 |

## 流程

### 一般诊断流程

```
1. 获取系统启动时间       → systeminfo
2. 查询 Event ID 1074    → 上次关机/重启原因
3. 查询 Event ID 1001    → 蓝屏记录（如有）
4. 查询 Event ID 506/507 → 睡眠/唤醒历史
5. 分析时间线，给用户结论
```

### 示例：完整重启原因分析（参考此会话中的输出结构）

| 时间 | 事件 | 说明 |
|------|------|------|
| T1 | MoUsoCoreWorker 发起计划重启 | Windows Update 触发 |
| T2 | TrustedInstaller 执行重启 | 更新安装后重启 |
| T3 | 系统启动 | 用户开机或唤醒 |
| T3 + 11s | 蓝屏崩溃 | BugCheck 代码... |
| T3 + 24s | 系统正常进入桌面 | 自动重启恢复 |

## 注意事项

### ⚠️ WSL 基础工具缺失

此 WSL 环境中 `cat`、`wc`、`find`、`head` 等基础工具可能缺失，导致 `write_file`、`search_files` 等 Hermes Agent 工具失败。**不要重试失败的命令**，改用 Python stdlib 替代。

参见 `references/wsl-basic-tools-workarounds.md` 获取完整替代方案清单（文件搜索、文件创建、语法检查、行数统计等场景）。

也见下方「WSL Cron 环境工具可用性」了解 cron 模式的特殊情况。

### ⚠️ WSL Cron 环境工具可用性

在 cron 模式下（非交互式 shell），`terminal` 和 `search_files` 工具调用的 bash 可能有极简 PATH，**`grep`、`head`、`find` 等标准命令可能报 "command not found"**。参见 `references/wsl-cron-tool-quirks.md` 了解详细症状和解决方法。

解决方法速览：
1. **用 Python stdlib 替代 bash** — `os.walk()` 代替 `find + grep`，`os.listdir()` 代替 `ls`（推荐）
2. 在 terminal 调用前 `export PATH=...` 显式设置
3. 用 `/usr/bin/grep` 等完整路径调用

- **总是写 PS1 文件，不要用内联 PowerShell 命令** — Shell 转义问题几乎不可避免
- **wevtutil 优先于 Get-WinEvent** — 对于简单查询更可靠，且无转义问题
- **wevtutil 输出含非 ASCII 字符** — 用 `strings` 配合 `grep` 处理
- **脚本文件放 `/tmp/`** — 临时文件，执行后自动清理（WSL 重启后消失）
- **避免写入 Windows 文件系统** — 除非有持久化需求
- **`wevtutil` 事件数量用 `/c:N` 参数控制** — 默认可能返回成千上万条
- **`/rd:true` 倒序排列** — 最新的在前
