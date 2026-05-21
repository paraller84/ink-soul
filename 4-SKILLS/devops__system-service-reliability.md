---
name: system-service-reliability
description: >-
  Ensure Hermes Agent's dependant services survive OS reboots and restart safely.
  Covers WSL-specific limitations, Windows Scheduled Task auto-start, external
  watchdog pattern, pre-flight restart checks, and the STOPPED.flag mechanism.
  Complements disaster-recovery (data backup) — this is the "stays running"
  layer, not the "restore from backup" layer.
version: 2.1.0
tags: [devops, reliability, wsl, systemd, restart, monitoring, watchdog]
---

# System Service Reliability

Keep Hermes Agent's supporting services alive through OS reboots, process
crashes, and self-triggered restarts — without the agent going permanently
offline.

## ⚠️ Critical WSL Limitation (Must Know)

**systemd user services do NOT auto-start at WSL boot.** They start only when
the user logs into WSL (e.g., opens a terminal). This is because `systemctl
--user` runs inside a user session, which is created on login.

Even with `loginctl enable-linger openclaw` (linger=yes), the WSL VM itself is
started lazily — it doesn't boot until something accesses it (terminal open,
`wsl` command from Windows).

**Consequence**: After an OS reboot, you must log into Windows + open WSL (or
use a Windows Scheduled Task) before any services start. This is NOT a bug —
it's a WSL2 design constraint.

## Correct Architecture (Windows + WSL)

There are two viable approaches. **Approach A (Registry Run key)** is simpler and
requires zero admin privileges. **Approach B (Windows Scheduled Task)** is more
featureful but requires user action at the Windows desktop.

### Preferred: Approach A — Registry Run Key + WSL Cron

```
Windows Boot
  ↓
Windows Registry Run key fires (as part of user login process)
  ↓  Key: HKCU\...\Run\Hermes WSL Boot
  ↓  Value: wscript.exe C:\Users\<USER>\hermes-scripts\hermes-boot.vbs
  ↓  (VBScript runs invisibly — no console window)
WSL VM starts → services launch
  ↓
Edu-Hub starts (python3 app.py)
Hermes Gateway starts (venv gateway run)
  ↓  (Gateway auto-launches MCP servers: token-savior, getnote)
  ↓
User opens Feishu → agent is online
  ↓
┌────────────────────────────────────────────────┐
│ WSL Cron Job "*/5 * * * * watchdog.sh"         │
│                                                 │
│  → Checks: pgrep -f 'hermes_cli.*gateway'       │
│  → If dead AND no ~/.hermes/STOPPED.flag        │
│  → Auto-restart: cd ~/.hermes/hermes-agent      │
│  → nohup ./venv/bin/python -m hermes_cli.main   │
│      gateway run --replace                      │
│  → Wait 45s, verify, log to                     │
│      ~/.hermes/logs/watchdog-restarts.log        │
└────────────────────────────────────────────────┘
```

**Advantages**: 
- Zero admin rights needed (Registry Run key is user-writable from WSL via
  `Set-ItemProperty -Path "HKCU:\..."`)
- All scripts manageable from WSL side (no Windows desktop access needed)
- Watchdog runs in same environment as the agent (no cross-boundary calls)

### Alternative: Approach B — Windows Scheduled Task

```
Windows Boot
  ↓
Windows Scheduled Task "Hermes WSL Boot" (trigger: AT LOGON)
  ↓  Action: cmd.exe → wsl bash -c "..."
  ↓
WSL VM starts → systemd boots
  ↓
Edu-Hub starts (python3 app.py)
Hermes Gateway starts (venv gateway run)
  ↓  (Gateway auto-launches MCP servers: token-savior, getnote)
  ↓
User opens Feishu → agent is online
  ↓
┌──────────────────────────────────────────────┐
│ Windows Scheduled Task "Hermes Watchdog"      │
│ (trigger: AT LOGON, repeat every 5 minutes)   │
│                                               │
│  → Checks: Is Gateway PID alive?              │
│  → If dead AND no STOPPED.flag → auto-restart │
│  → Logs all restarts for later review         │
└──────────────────────────────────────────────┘
```

**The watchdog runs on Windows, not in WSL.** This means it survives Hermes
Gateway restarts, agent crashes, and even WSL restarts (as long as Windows is
up).

**Disadvantages**:
- Creating Scheduled Tasks requires either admin rights or the user to be at the
  Windows desktop to double-click an installer script
- Cannot be configured remotely from WSL (Access Denied when calling
  `schtasks /Create` from WSL's PowerShell context)

## Service Inventory & Current State

Refer to `references/service-inventory.md` for the full live audit of what's
running, what auto-starts, and what gaps exist.

| Service | Type | Auto-start Mechanism | Reliability |
|---------|------|---------------------|-------------|
| Hermes Gateway | systemd --user | Windows Task → wsl → systemd | ✅ (with Windows Task) |
| Ollama | Windows app | Windows auto-start | ✅ |
| Edu-Hub (app.py) | — | Windows Task → wsl → nohup python3 | ⚠️ needs Windows Task |
| Token Savior MCP | gateway-launched | Auto-launched by Gateway | ✅ (if Gateway is up) |
| GetNote MCP | gateway-launched | Auto-launched by Gateway | ✅ (if Gateway is up) |
| KB pipeline | cron | crontab persistent | ✅ |
| Feishu sync | cron | crontab persistent | ✅ |

## Plan A: OS Reboot Auto-Recovery

### Preferred: Registry Run Key + VBScript

This approach requires zero admin privileges and can be configured entirely from
WSL. It's the recommended default.

#### Step 1: Create Boot VBScript (Windows side)

```vbscript
' C:\Users\<USER>\hermes-scripts\hermes-boot.vbs
' Runs invisibly (hidden window) — no console popup at login
CreateObject("WScript.Shell").Run "wsl -d Ubuntu bash -c ""if ! pgrep -f 'hermes_cli.*gateway' > /dev/null 2>&1; then cd ~/.hermes/hermes-agent && nohup ./venv/bin/python -m hermes_cli.main gateway run --replace > ~/.hermes/logs/gateway-auto.log 2>&1; fi && if ! pgrep -f 'edu-hub/app.py' > /dev/null 2>&1; then cd ~/edu-hub && nohup python3 app.py > ~/edu-hub/edu-hub.log 2>&1; fi""", 0, False
```

#### Step 2: Register Run Key (from WSL)

```powershell
# From WSL — no admin needed, writes to HKCU (current user only)
powershell.exe -Command "
Set-ItemProperty -Path 'HKCU:\\Software\\Microsoft\\Windows\\CurrentVersion\\Run' \
    -Name 'Hermes WSL Boot' \
    -Value 'wscript.exe C:\\Users\\<USER>\\hermes-scripts\\hermes-boot.vbs'
"
```

Verification:
```powershell
powershell.exe -Command "
Get-ItemProperty -Path 'HKCU:\\Software\\Microsoft\\Windows\\CurrentVersion\\Run' \
    -Name 'Hermes WSL Boot'
"
```

### Alternative: Windows Scheduled Task + Batch

Use this if you have Windows desktop access and prefer task scheduler
management (e.g., want to see the task in Task Scheduler UI).

## Plan B: External Watchdog

### The Pattern

A watchdog is a separate process that runs at a **different level** than the
agent. When the agent restarts (and its process disappears), the watchdog keeps
running and can detect failures.

**This is the ONLY reliable way to handle agent restart failures.** The agent
cannot monitor its own restart because its process is gone during the restart
window.

### Preferred: WSL Cron Watchdog

Simpler than the Windows-level approach — no cross-WSL/Windows calls needed.
The watchdog lives entirely inside WSL, triggered by cron.

#### Step 1: Create watchdog script (WSL side)

```bash
#!/bin/bash
# ~/.hermes/scripts/watchdog.sh
LOG=~/.hermes/logs/watchdog-restarts.log
GW_LOG=~/.hermes/logs/gateway-auto.log
GW_DIR=~/.hermes/hermes-agent

# Check if gateway is alive
if pgrep -f 'hermes_cli.*gateway' > /dev/null 2>&1; then
    exit 0  # healthy — quiet exit
fi

# Check if intentionally stopped
if [ -f ~/.hermes/STOPPED.flag ]; then
    exit 0
fi

# Gateway dead — restart
echo "$(date '+%Y-%m-%d %H:%M:%S') | Watchdog: restarting..." >> "$LOG"
cd "$GW_DIR" || exit 1
nohup ./venv/bin/python -m hermes_cli.main gateway run --replace >> "$GW_LOG" 2>&1 &
GWPID=$!

# Verify after cooldown
sleep 45
if kill -0 "$GWPID" 2>/dev/null; then
    echo "$(date '+%Y-%m-%d %H:%M:%S') | Watchdog: SUCCESS (PID $GWPID)" >> "$LOG"
else
    echo "$(date '+%Y-%m-%d %H:%M:%S') | Watchdog: FAILED" >> "$LOG"
    tail -10 "$GW_LOG" >> "$LOG"
fi
```

#### Step 2: Register cron job

```bash
(crontab -l 2>/dev/null | grep -v 'watchdog.sh'; \
 echo "*/5 * * * * /home/openclaw/.hermes/scripts/watchdog.sh > /dev/null 2>&1") \
 | crontab -
```

**Advantages**: No Windows cross-boundary calls, no admin rights, all
manageable from WSL.

**Disadvantage**: If WSL itself crashes, the watchdog goes down with it. For
most users this is acceptable (WSL is more stable than individual services).

### Alternative: Windows-Level Watchdog

| Component | Detail |
|-----------|--------|
| Runs at | **Windows level** (Scheduled Task or Windows Service) |
| Trigger | AT LOGON, repeat every 5 minutes |
| Check method | `wsl -d Ubuntu bash -c "pgrep -f 'hermes_cli.*gateway'"` |
| Restart action | `wsl -d Ubuntu bash -c "cd ~/.hermes/hermes-agent && nohup ./venv/bin/python -m hermes_cli.main gateway run --replace ..."` |
| Distinguishes crash vs intentional stop | Checks `~/.hermes/STOPPED.flag` |
| Logging | Writes to `C:\Users\<USER>\hermes-scripts\watchdog.log` |

### STOPPED.flag Mechanism

The watchdog must distinguish between "agent crashed" and "user intentionally
stopped the agent." Use a flag file for this:

```bash
# WSL side — stop agent (watchdog will skip auto-restart)
touch ~/.hermes/STOPPED.flag
echo "Watchdog will NOT auto-restart"

# WSL side — resume agent (watchdog will monitor again)
rm -f ~/.hermes/STOPPED.flag
~/.hermes/scripts/hermes-start.sh

# Windows side (in watchdog script)
$stopped = wsl -d Ubuntu bash -c "test -f ~/.hermes/STOPPED.flag && echo YES || echo NO"
if ($stopped -eq "YES") {
    exit 0  # Skip restart — intentionally stopped
}
```

### Watchdog PowerShell Script

```powershell
# C:\Users\<USER>\hermes-scripts\hermes-watchdog.ps1
$wslAlive = wsl -d Ubuntu echo "OK" 2>&1
if ($LASTEXITCODE -ne 0) { exit 0 }

$gatewayAlive = wsl -d Ubuntu bash -c "pgrep -f 'hermes_cli.*gateway' > /dev/null 2>&1 && echo YES || echo NO"
if ($gatewayAlive -eq "YES") { exit 0 }

$stopped = wsl -d Ubuntu bash -c "test -f ~/.hermes/STOPPED.flag && echo YES || echo NO"
if ($stopped -eq "YES") { Write-Log "Skipping restart (STOPPED.flag)"; exit 0 }

# Attempt restart
wsl -d Ubuntu bash -c "cd ~/.hermes/hermes-agent && nohup ./venv/bin/python -m hermes_cli.main gateway run --replace > ~/.hermes/logs/gateway-auto.log 2>&1 &"
Start-Sleep -Seconds 45

$verify = wsl -d Ubuntu bash -c "pgrep -f 'hermes_cli.*gateway' > /dev/null 2>&1 && echo YES || echo NO"
if ($verify -eq "YES") { Write-Log "SUCCESS: restarted" }
else { Write-Log "FAILURE: still down — check logs" }
```

## Pre-Flight Restart Checks

These run BEFORE the agent invokes a restart, while the agent is still online.
They are the only checks the agent can perform before going offline.

Always-run checklist:

```bash
# 1. Config syntax
python3 -c "import yaml; yaml.safe_load(open('~/.hermes/config.yaml'))" || exit 1

# 2. Port not in use by unrelated process
ss -tlnp | grep -q ":8080 " && echo "WARN: port in use"

# 3. Dependencies reachable (non-fatal)
curl -s -o /dev/null http://localhost:11434/api/tags 2>/dev/null || echo "WARN: Ollama unreachable"

# 4. Last-good config saved
cp ~/.hermes/config.yaml ~/.hermes/config.yaml.last-good

# 5. Record current config hash for rollback
sha256sum ~/.hermes/config.yaml > ~/.hermes/config.hash
```

## Safe Restart Protocol (Corrected)

**Wrong approach** (v1.0 — don't do this): Assume agent can monitor restart,
verify health, and rollback on failure. The agent's process is gone during
restart — it CANNOT do these things.

**Correct approach:**

```
PHASE 1 — Pre-flight (agent still alive)
  ├── Validate config.yaml syntax
  ├── Check port availability
  ├── Check dependency health (Ollama, etc.)
  ├── Save last-good config
  └── Ensure watchdog is registered and active

PHASE 2 — Restart (agent goes offline)
  ├── Process terminates
  ├── systemd Restart=always triggers new process
  └── OR: watchdog detects dead process + triggers restart

PHASE 3 — Post-restart recovery (watchdog handles it)
  ├── Watchdog detects process is alive (PID check)
  ├── OR: detects process is dead → attempts restart
  ├── Logs outcome for agent to review on next startup
  └── If restart fails repeatedly → watchdog logs details
      → agent reviews logs on next manual restart
```

## Flask Dev Server Lifecycle & Dual-Process Pitfall

When serving Flask applications in development mode (`app.run()`), be aware of a
critical deployment trap: **two processes binding to the same port.**

### The Trap

```bash
# Server A started (20:00 — old code)
$ python app.py &   # PID 95616, listens on 0.0.0.0:5003

# Server B started later (21:30 — new code), WITHOUT killing A
$ python app.py &   # PID 100901, also listens on 0.0.0.0:5003

# Now BOTH servers share port 5003
# Kernel round-robins incoming connections between them
# ~50% of requests → old server, ~50% → new server
```

The user sees an intermittent "old page" — every other refresh may show stale
content. This is especially insidious because the new template files are read
from disk by both servers (Jinja2 renders fresh each request), so the **HTML
looks correct**, but **Python route handlers and in-memory data structures**
differ between the old and new process.

### Diagnosis

```bash
# Check all processes on target port
ss -tlnp | grep 5003
# Output shows MULTIPLE PIDs for the same port:
# users:(("python",pid=100901,...),("python",pid=95616,...))
```

### Fix

```bash
# 1. Kill ALL processes on the port
fuser -k 5003/tcp
# or: kill each PID individually

# 2. Wait for port release
sleep 2
ss -tlnp | grep 5003 || echo "Port free"

# 3. Start single fresh server (use Hermes terminal background=true)
# so the process lifecycle is tracked
terminal(background=true, command="cd ~/project && python app.py")

# 4. Verify single process
ss -tlnp | grep 5003
# Expect: exactly one PID
```

### Prevention

- **Always kill existing processes** before starting a new dev server: check
  the port first with `ss -tlnp | grep <port>`.
- **Use Hermes `terminal(background=true)`** for Flask servers so the gateway
  tracks the process lifecycle and kills it cleanly on restart.
- **Never start a Flask dev server with `nohup`/`&` in foreground terminal** —
  this bypasses process tracking and is the root cause of orphaned servers.
- After modifying route handlers or services, **restart the server** to ensure
  in-memory Python code is fresh (Jinja2 templates are auto-read from disk,
  but route code is loaded at startup).

## Pitfalls

- **systemd --user ≠ system boot**: WSL user services require login. Do NOT
  assume `systemctl --user enable` is sufficient for auto-start after reboot.
  Always pair with Windows Scheduled Task.
- **Watchdog must run at a different level**: If watchdog runs inside WSL as a
  systemd service, it dies when WSL restarts. Run it on Windows.
- **STOPPED.flag is one-way**: Touching it stops auto-restarts. To resume, must
  explicitly remove the flag AND start the service. `hermes-start.sh` does both.
- **Watchdog check interval vs restart delay**: If gateway takes >5 min to
  start, watchdog will fire repeatedly. Set a cooldown: only restart if
  gateway has been down for > 2 consecutive checks (10+ minutes).
- **@reboot cron unreliable in WSL**: cron may start before network/PATH is
  ready, or WSL may not trigger @reboot at all. Never rely on @reboot for
  critical services.
- **schtasks /Create from WSL fails "Access Denied"**: Even when running as the
  logged-in user, calling `schtasks /Create` from WSL's PowerShell context
  fails with permission errors. This is a WSL/Windows interop limitation —
  creating scheduled tasks requires execution in the Windows desktop session.
  Use Registry Run key instead (no admin needed), or have the user double-click
  an installer script from Windows Explorer.
- **`.ps1` scripts "flash and close" on double-click**: By default, Windows
  restricts PowerShell script execution. Double-clicking a `.ps1` file opens
  a console window that closes immediately (usually with an execution-policy
  error), giving the user no feedback. **Always use `.bat` (batch) files**
  for user-facing installer/deployment scripts on Windows. `.bat` files run
  directly via `cmd.exe` with no execution policy restriction and visible
  console output. If a `.ps1` script must be used, instruct the user to
  right-click → "Run with PowerShell" or pass `-ExecutionPolicy Bypass`.
- **User may not be at computer**: When deploying reliability infrastructure,
  the user may be remote. Design for zero-desktop-interaction deployment. Use
  Registry Run key (settable from WSL via `Set-ItemProperty`) and WSL cron
  (settable from WSL via `crontab`). Do NOT create scripts that require the
  user to double-click anything.
- **Registry Run key vs Scheduled Task**: The Run key fires slightly later in
  the login sequence (after explorer.exe initializes) vs Scheduled Task which
  fires at logon trigger. In practice the difference is negligible (~2-3 sec).
- **VBScript hidden window**: Use `0` as the second arg to `WScript.Shell.Run`
  to create an invisible window. Without this, a console window pops up at
  every login.
- **Cron environment may lack PATH**: WSL cron jobs run with a minimal PATH.
  Always use full paths in cron scripts (`/usr/bin/pgrep`, not `pgrep`).
  Alternatively, source the user's profile first.
- **Gateway PID file goes stale on crash**: The `--replace` flag checks the PID
  file but may hang on stale PIDs. Always verify with `kill -0 PID` first.
- **MCP servers launched twice**: If gateway already manages MCP servers,
  adding separate systemd services for them will cause duplication.
- **Port forwarding lost on WSL restart**: WSL IP changes after reboot. Must
  re-apply Windows port forwarding rules (see `wsl-port-forwarding` skill).
- **Diagnostic trap: systemctl shows inactive but gateway is running under watchdog**:
  When the gateway is started by the watchdog script (not by systemd), `systemctl
  --user status hermes-gateway.service` shows `inactive (dead)`, but the gateway
  process may actually be alive under a PID managed by the watchdog. This happens
  because the watchdog starts the gateway directly via `nohup` rather than through
  systemd. **Always cross-check with `ps aux | grep 'hermes_cli.*gateway'` (or
  `pgrep -f 'hermes_cli.*gateway'`) before concluding the gateway is down.**
  Systemctl status alone is not authoritative when the watchdog pattern is in use.
- **Self-restart pre-flight is the ONLY check you can run**: Once restart
  begins, your process is gone. Do not attempt post-restart verification from
  the agent itself — delegate to the watchdog.

## Data-Flow Dependency Topology

Service reliability isn't just about processes staying running — it's about whether
those processes can **produce output** when they run. A cron job that runs successfully
but finds nothing to process is an "empty run" (空跑). The root cause is often a
broken infrastructure-level dependency that no process monitor catches.

### Methodology: Mapping Hidden Dependencies

When auditing task batches for "空跑" risk, do NOT stop at cron→cron chains. Trace
the full data flow from source to consumer, including infrastructure layers:

```
[Source] → [Sync/Transport] → [Mount] → [Script reads] → [Script writes]
```

For each task batch, ask:
1. What **infrastructure** must be available for this task to produce real output?
   (Filesystem mount? Cloud sync client? Network volume?)
2. What **upstream system** must have run before this task? Is there a cron for it?
3. What happens when that upstream produces **zero new data**? (Legitimate empty run
   vs broken pipeline?)
4. Is there a **single point of failure** that multiple pipelines share?

### Canonical Example: WPS Cloud Drive Dependency

This is the largest hidden dependency in the current system — discovered during a
session audit when the user corrected surface-level cron analysis.

```
Windows端 (公司电脑)
  │
  ├─ Foxmail → foxmail-auto-export.py (Task Scheduler 30min)
  │     → .eml 写入 G:\WPSDocument\WPS云盘\邮件\自动导出\
  │
  └─ WPS云盘客户端（后台同步）
        → G:\WPSDocument\ 本地同步目录
        │
WSL (DrvFs 直挂)
  /mnt/g/WPSDocument/  (9p filesystem mount, real-time access)
  │
  ├─ email_classifier.py      → 邮件分类    (依赖: /mnt/g/WPSDocument/WPS云盘/邮件/自动导出/)
  ├─ kb_organize.py            → 筛选入知识库 (依赖: /mnt/g/WPSDocument/)
  ├─ kb_pipeline.py            → 增量+7级去重 (依赖: /mnt/g/WPSDocument/)
  ├─ kb_vectorize.py           → 向量化      (依赖: /mnt/g/知识库文档/)
  ├─ kb_health.py              → 健康检查     (依赖: /mnt/g/WPSDocument/)
  └─ kb-wiki-sync.py           → Wiki 同步   (依赖: /mnt/g/WPSDocument/)
```

**6 scripts, 2 pipelines** depend on a single mount point. The failure modes:

| Failure Point | Effect | Detection |
|---------------|--------|-----------|
| WPS云盘客户端未启动 (Windows reboot) | G:盘文件不是最新 | Scripts run, find nothing new → silent 空跑 |
| DrvFs 挂载失败 (WSL restart issue) | /mnt/g/ inaccessible | kb_pipeline.py reports "目录不存在" |
| Foxmail导出脚本停运 | 无新 .eml 进入管道 | email_classifier stats 停滞 |
| 路径变更 | 硬编码路径失效 | Scripts crash with FileNotFoundError |

**Key insight**: DrvFs gives real-time read access to the Windows G: drive from WSL.
No WPS cloud sync delay exists for WSL reads — the mount is direct filesystem access.
The risk is that the Windows-side **WPS cloud sync client** must be running for the
G: drive directory to contain the latest synced files.

### Pitfall: Infrastructure Dependency Blindness

- **Cron status "ok" ≠ pipeline producing output**: A cron job may exit 0 but
  process zero new items because its upstream data source is stale. This is the
  most common "空跑" pattern — the script is correct, but the infrastructure layer
  that feeds it is broken.
- **/mnt/g/ is not "the cloud"**: It's a direct DrvFs mount of the Windows G: drive.
  Don't confuse filesystem mount semantics with cloud sync semantics. The Windows
  WPS cloud sync client is the actual sync layer — if it's down, the G: drive
  directory is stale even though it's accessible.
- **WPS云盘 is not the only mount**: Any pipeline that reads from `/mnt/c/`, `/mnt/d/`,
  or `/mnt/g/` inherits WSL's DrvFs reliability characteristics. If DrvFs ever fails
  to mount a drive letter, ALL pipelines reading from that drive go silent.
- **Data-flow audit before feature work**: Before assuming a cron job is "working,"
  check whether its upstream data source is producing new data. A healthy pipeline
  with no input is not the same as a broken pipeline.

## Related Skills

- `hermes-disaster-recovery` — Data backup & restore (complementary)
- `wsl-port-forwarding` — Re-apply Windows port forwarding after WSL IP change
- `system-health-audit` — Broader health audit checklist
- `agent-memory-management` — Memory pruning to avoid runaway state
- `email-to-knowledge-pipeline` — Email pipeline that depends on WPS云盘 mount
- `personal-knowledge-base-management` — KB pipeline that depends on WPS云盘 mount
