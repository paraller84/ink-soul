---
title: Hermes Disaster Recovery
name: hermes-disaster-recovery
description: Preserve Hermes Agent across hardware failures — memories, skills, session history, config, and environment. Multi-layer backup architecture with GitHub, external drives, NAS, and cloud storage. Full recovery procedure for new machines.
---

# Hermes Disaster Recovery

Preserve the agent's persistent state — its **soul** (memories + skills + config), **experience** (session history + search index), and **infrastructure** (scripts + cron + knowledge base) — so it survives hard drive failure, WSL corruption, or total machine loss.

## Architecture Overview

### The Four-Layer Model

```
Layer 1 🫀 Soul        — memories, skills, config        → Frequency: every change
Layer 2 🧠 Experience  — sessions, state.db              → Frequency: daily
Layer 3 🏗️ Infra       — scripts, cron, knowledge base   → Frequency: weekly
Layer 4 🏠 Environment — full WSL export, port forwarding → Frequency: monthly
```

### Target Storage Destinations

| Layer | Primary | Secondary | Tertiary |
|-------|---------|-----------|----------|
| Soul | GitHub private repo | Cloud docs (Feishu) | External drive |
| Experience | External drive | NAS | Cloud storage |
| Infra | GitHub private repo | NAS | — |
| Environment | External drive | NAS | — |

Three physically isolated storage classes:
- **Online cloud**: GitHub repositories, cloud document platforms
- **Local physical**: External SSD, internal data partition (D:)
- **Off-machine**: Home NAS, cloud drive sync folders

### Typical Data Scale

| Asset | Size | Notes |
|-------|------|-------|
| MEMORY.md + USER.md | ~7KB | Irreplaceable. Back up every change. |
| Session history | ~270MB (775 sessions) | JSONL format. Compress well. |
| state.db (SQLite index) | ~320MB | Full-text search + memory index. |
| Skills directory | ~1MB | 30-50 skill directories. |
| Scripts directory | ~1MB | Custom automation tools. |
| Cron definitions | ~24KB | JSON job definitions. |
| Config + .env | ~5KB | Provider configs, API tokens. |
| **Total** | **~600MB** | Easily fits any backup target. |

## Backup Pipeline Setup

### Step 1: Initialize Git Repository for Soul Data

```bash
cd ~/.hermes
git init
echo 'sessions/' > .gitignore
echo 'state.db' >> .gitignore
echo 'state.db-wal' >> .gitignore
echo 'state.db-shm' >> .gitignore
echo 'cache/' >> .gitignore
echo 'pastes/' >> .gitignore
echo 'logs/' >> .gitignore
echo '__pycache__/' >> .gitignore
echo '*.lock' >> .gitignore
echo 'node_modules/' >> .gitignore
git add -A
git commit -m "init: soul backup"
git remote add origin git@github.com:<user>/<repo-name>.git
git push -u origin main
```

**Tracked** (committed on every change):
- MEMORY.md, USER.md
- config.yaml, .env
- `skills/` directory
- `scripts/` directory
- `cron/jobs.json`

**Excluded** (backed up separately):
- `sessions/` (too large, handled by daily pipeline)
- `state.db` (too large, handled by daily pipeline)
- `cache/`, `pastes/`, `logs/` (recreatable)

### Step 2: Daily Experience Backup (cron job)

```python
# ~/.hermes/scripts/daily-backup.py
# Schedule: 0 23 * * *
import sqlite3, shutil, gzip, json, os
from datetime import datetime

BACKUP_DIR = "/path/to/backup/dir"  # external drive or NAS mount
RETENTION_DAYS = 30
HERMES_HOME = os.path.expanduser("~/.hermes")
today = datetime.now().strftime("%Y%m%d")

def backup_state_db():
    """SQLite dump → gzip → rotate"""
    db_path = os.path.join(HERMES_HOME, "state.db")
    dest = os.path.join(BACKUP_DIR, f"hermes_state_{today}.sql.gz")
    with sqlite3.connect(db_path) as conn:
        with gzip.open(dest, "wt") as f:
            for line in conn.iterdump():
                f.write(line + "\n")

def backup_recent_sessions(days=7):
    """Package last N days of sessions"""
    sessions_dir = os.path.join(HERMES_HOME, "sessions")
    cutoff = datetime.now().timestamp() - (days * 86400)
    recent = []
    for f in os.listdir(sessions_dir):
        fpath = os.path.join(sessions_dir, f)
        if os.path.isfile(fpath) and os.path.getmtime(fpath) > cutoff:
            recent.append(fpath)
    dest = os.path.join(BACKUP_DIR, f"hermes_sessions_{today}.tar")
    shutil.make_archive(dest.replace(".tar", ""), "tar", root_dir=sessions_dir,
                        base_dir=".", dry_run=False)
    
def rotate_old(prefix, days=RETENTION_DAYS):
    """Remove backups older than retention"""
    cutoff = datetime.now().timestamp() - (days * 86400)
    for f in os.listdir(BACKUP_DIR):
        if f.startswith(prefix) and os.path.getmtime(os.path.join(BACKUP_DIR, f)) < cutoff:
            os.remove(os.path.join(BACKUP_DIR, f))
```

### Step 3: Weekly Infra Backup (cron or manual)

```bash
# Schedule: 0 6 * * 0
cd ~/.hermes
git add -A
git commit -m "weekly snapshot $(date +%Y-%m-%d)"
git push
```

### Step 4: Monthly WSL Export

```powershell
# Windows side (run as admin via scheduled task)
$date = Get-Date -Format "yyyyMM"
$dest = "G:\Backup\wsl\hermes_$date.tar"
wsl --export Ubuntu $dest
```

## Disaster Recovery Procedure

### Full Recovery (new machine)

```bash
# 1. Install WSL + Ubuntu on new machine
# 2. Restore Hermes Agent
cd ~
git clone https://github.com/NousResearch/hermes-agent.git
cd hermes-agent
make install

# 3. Restore soul data
cd ~/.hermes
git clone git@github.com:<user>/<repo-name>.git .
git checkout HEAD -- MEMORY.md USER.md config.yaml .env skills/ scripts/ cron/

# 4. Restore session data from external drive
cp /path/to/backup/*.sql.gz ~/.hermes/
gunzip -c latest_state.sql.gz | sqlite3 ~/.hermes/state.db
tar -xzf latest_sessions.tar.gz -C ~/.hermes/sessions/

# 5. Re-register cron jobs
python3 ~/.hermes/scripts/register-cron.py

# 6. Start Hermes
hermes start
```

### Partial Recovery (WSL repair only)

```bash
cd ~/.hermes
git checkout HEAD -- MEMORY.md USER.md config.yaml .env skills/ scripts/ cron/
gunzip -c /path/to/latest_state.sql.gz | sqlite3 state.db
hermes start
```

## Risk Management

| Risk | Mitigation |
|------|-----------|
| Git push fails | Multiple backup targets provide redundancy |
| External drive failure | NAS + cloud as secondary |
| GitHub auth lost | SSH keys backed up elsewhere; NAS has full copy |
| NAS offline | Local G: drive + cloud as fallback |
| Corruption propagates | Retention window allows rollback to good snapshot |

## Verification Checklist

After backup setup:
- [ ] `git push` succeeds for soul data
- [ ] Daily backup script runs and produces .sql.gz + .tar.gz
- [ ] Backup files readable (`gunzip -c | head`, `tar -tzf`)
- [ ] WSL export exists and is recent
- [ ] NAS/cloud sync folder contains backup copies
- [ ] Rollback verified: can `git checkout` old memory state

## Pitfalls

- **.gitignore mistakes**: Accidentally ignoring MEMORY.md = complete memory loss on git-based recovery. Always double-check what's tracked.
- **OneDrive assumed as default**: Many backup scripts assume OneDrive. Verify the user's actual cloud provider before writing paths.
- **NAS auto-mount**: If NAS isn't in `/etc/fstab`, backup scripts fail silently on reboot. Test the mount after a simulated reboot.
- **Token exposure**: `.env` contains API tokens. Ensure git repo is **private**.
- **state.db consistency**: SQLite dump while agent is running may produce inconsistent snapshots. Consider running backup during idle time (e.g., 3 AM) or using `.backup` command.
- **WSL export size**: Full WSL export with Ollama models can be 30GB+. Exclude model directories or use `--no-vhdx` flags.
- **GitHub rate limits**: Frequent pushes (>100/day) may hit rate limits. Soul layer changes are rare enough (<10/day) that this is not a concern in practice.

## Related Skills

- `agent-memory-management` — focuses on preventing memory bloat and pruning, complements backup strategy
- `system-health-audit` — broader system health checks that include backup verification
- `getnotes-sync-pipeline` — C013 sync pipeline (another data preservation mechanism)
