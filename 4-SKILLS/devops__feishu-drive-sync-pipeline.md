---
name: feishu-drive-sync-pipeline
title: Feishu Drive → Wiki Sync Pipeline
description: Build an incremental sync pipeline from Feishu Drive folders to a local Karpathy-style Wiki using lark_oapi, cron scripting, and agent-based content summarization.
category: devops
trigger: User wants to sync Feishu cloud docs to a local wiki or knowledge store
---

# Feishu Drive → Wiki Sync Pipeline

## Overview

Build an incremental sync pipeline from Feishu Drive folders to a local Wiki (Karpathy-style). Used for syncing LLM discussion records, meeting notes, or any cloud docs to a local knowledge base.

## Files Created

- `~/.hermes/scripts/feishu-wiki-sync.py` — sync script
- `~/.hermes/scripts/feishu-wiki-sync-state.json` — tracks already-synced docs

## Setup Steps

### 1. Create a Drive Folder

Use lark_oapi's `drive/v1/files/create_folder` endpoint. **folder_token goes in the BODY**, not as a query param. Response format varies: check `data.folder.token` first, fallback to `data.token`.

**PITFALL**: `feishu_doc_create(type="folder")` creates a **doc-space** folder (token prefixed `S31...`). Drive API (`drive/v1/files`) cannot list files in doc-space folders. Use raw API to create proper **drive-space** folders.

### 2. Write the Sync Script

Location: `~/.hermes/scripts/feishu-wiki-sync.py`

Key APIs:
- List files in folder: `GET /open-apis/drive/v1/files?folder_token={token}&page_size=100`
- Read doc plain text: `GET /open-apis/docx/v1/documents/{doc_token}/raw_content`

### 3. Register as Cron Job

Script path must be **bare filename** relative to `~/.hermes/scripts/`. No absolute paths.

## ⚠️ KNOWN BUG (FIXED 2026-05-01): Pipe Table Blocks Displaced Backwards

### History

**Previously misdiagnosed** as a Feishu rendering engine issue. The actual root cause is in `feishu-md-writer.py`'s `parse_markdown_to_blocks` — **condition ordering** in the parser.

### Symptom

Pipe-style markdown tables appeared AFTER the next heading/divider in the Feishu document, not at their source position.

```
Source (intended order):            Feishu rendered (ACTUAL):
  ### 顾问角色                        ### 顾问角色
  | # | 角色 | ...                   ### 4阶段询问流程 ← heading came first!
  | 1 | ...                          | # | 角色 | ...  ← table pushed here
  ### 4阶段询问流程                   | 1 | ...       
  | 阶段 | ...                       | 阶段 | ...
```

### Actual Root Cause (Fixed)

In `feishu-md-writer.py`'s `parse_markdown_to_blocks()`, the **heading check** (`if heading_match`) was placed **before** the **table-end check** (`if in_table and not ("|" in line)`). When a heading was encountered while `in_table=True`:

1. Heading check matched first → created heading block + `continue`
2. Table buffer **never flushed** before the next section
3. All tables accumulated until the final `---` divider or EOF
4. `blocks.extend(table_blocks)` **appended** at end instead of inserting at source position

The fix (2026-05-01):
1. Moved table-end check to **before** heading/divider/quote/list checks (right after empty-line check)
2. Changed `blocks.extend(table_blocks)` → `blocks[table_insert_at:table_insert_at] = table_blocks` to insert at the correct position (right after the heading that precedes the table)

### Diagnostic Technique

To verify block ordering in a Feishu document:

```python
# 1. Get all child block IDs
children = get_child_blocks(doc_token, page_block_id)

# 2. Read each block's type and text
for child_id in children:
    result = _api("GET", f"/open-apis/docx/v1/documents/{doc_token}/blocks/{child_id}")
    block = result["data"]["block"]
    bt = block["block_type"]
    # 3=heading1(文档标题), 4=heading2, 5=heading3, 2=text
    # Check block_type + text_run content to verify order
    text = block.get("text", {}).get("elements", [{}])[0].get("text_run", {}).get("content", "")
    print(f"type={bt} | {text[:60]}")
```

If a `###` heading (type 5) appears before the table text blocks (type 2) that should precede it, you have a condition-ordering bug in the parser.

### If Pipe Tables Still Have Issues

After the fix, pipe tables render correctly as text blocks with `|` separators. If displacement still occurs, check:

1. Is `feishu-md-writer.py` the latest version? (should have `table_insert_at` logic and table-end check before heading check)
2. Are there `---` divider lines between the table and the next heading? (dividers also need the table-end check to come first)
3. Test by parsing locally: `blocks = parse_markdown_to_blocks(content)` and dump block indices to verify order

### Fallback: Numbered Lists

If rendering issues persist for complex tables, fall back to numbered lists:

```markdown
❌ Pipe table (may still have issues in edge cases):
| 题目 | 试商 | 对/错 |
| 156÷13 | 10 | ( ) |

✅ Numbered list (always safe):
(1) 156÷13，试商10。判断：(    )
(2) 274÷28，试商9。判断：(    )
```

**Exception**: Code blocks (triple backtick) with `|` are always safe (block_type 14).

## Writing Docx Block Content

### CORRECT Block Type Numbering (1-based)

| Type | block_type | JSON field |
|------|-----------|------------|
| Text | **2** | `text` (NOT `paragraph`) |
| h1/h2/h3 | 3/4/5 | `heading1`/`heading2`/`heading3` |
| Bullet | **12** | `bullet` |
| Ordered | **13** | `ordered` |
| Code | **14** | `code` |
| Quote | **15** | `quote` |
| Divider | 16 (broken!) | Use text separator instead |

### `text_run` Wrapper is MANDATORY

**❌ WRONG** (error 1770024):
```python
{"content": "hello", "text_element_style": {}}
```
**✅ CORRECT**:
```python
{"text_run": {"content": "hello", "text_element_style": {}}}
```

### Markdown → Blocks Pipeline

Script at `~/.hermes/scripts/feishu-md-writer.py`:
```bash
python3 feishu-md-writer.py <doc_token> <markdown_file>
```

Key constraints:
- Max 50 blocks per request (error 99992402)
- 3 req/s rate limit
- Token expires ~2 hours, refresh for long operations

## Token Drift: The 12-Char Shift

Hardcoded folder tokens silently expire when folders are recreated. The **last 12 characters** of the token string change while the prefix stays the same:

```
Old (stale): NGDcfURb2l13bLd6Am5c6QEmnUd
New (valid): NGDcfURb2l13tPd2d8FcrVORnPh  ← last 12 chars differ!
```

Error `1770039 "folder not found"` almost always means token drift, not an actual missing folder. Always verify by listing the parent folder's contents via API and comparing tokens.

## Avoiding Circular Sync

### Pattern: Three-Way Isolation

- **User uploads** → dedicated folder (scanned by sync pipeline)
- **System output** → non-scanned subfolder (daily-practice, exam-analysis)
- **Reference docs** → non-scanned subfolder (review-book)

### Steps

1. Create dedicated upload folder (NOT at root)
2. Point sync pipeline ONLY at upload folder
3. Force ALL system generators to write to NON-scanned folders
4. Track system-generated docs in global `generated_docs` list
5. Check `is_generated_doc()` before analyzing any document

### Naming Convention

Suffix `_试题` / `_答案` in doc names triggers auto-tagging. Tools like `exam-answer-splitter.py` split mixed docs.

## Feishu Drive Folder & File Management API

### Create a Folder

```python
# POST /open-apis/drive/v1/files/create_folder
body = json.dumps({"name": "文件夹名", "folder_token": "parent_folder_token"}).encode()
req = urllib.request.Request(
    "https://open.feishu.cn/open-apis/drive/v1/files/create_folder",
    data=body, headers={**headers, "Content-Type": "application/json"}, method="POST"
)
result = json.loads(urllib.request.urlopen(req).read())
# Token is in data.token (NOT data.data.folder.folder_token)
token = result.get("data", {}).get("token", "")
```

**PITFALL**: The response is `{\"code\":0,\"data\":{\"token\":\"...\",\"url\":\"...\"}}`. The token is in `data.token` (NOT nested under `data.data.folder.folder_token` — that path returns empty string). Always extract with `result.get(\"data\", {}).get(\"token\", \"\")`.

### List a Folder's Contents

```python
# GET /open-apis/drive/v1/files?folder_token={token}&page_size=100
# Paginated — check has_more for next page
def list_folder(ftoken):
    all_files = []
    page_token = None
    while True:
        url = f"https://open.feishu.cn/open-apis/drive/v1/files?folder_token={ftoken}&page_size=100"
        if page_token:
            url += f"&page_token={page_token}"
        req = urllib.request.Request(url, headers=headers)
        data = json.loads(urllib.request.urlopen(req).read())
        if data.get("code") != 0: break
        all_files.extend(data.get("data", {}).get("files", []))
        if not data.get("data", {}).get("has_more"): break
        page_token = data.get("data", {}).get("page_token")
    return all_files
```

### Move a File to Another Folder

```python
# POST /open-apis/drive/v1/files/{file_token}/move
body = json.dumps({"type": "docx", "folder_token": "target_folder_token"}).encode()
req = urllib.request.Request(
    f"https://open.feishu.cn/open-apis/drive/v1/files/{file_token}/move",
    data=body, headers={**headers, "Content-Type": "application/json"}, method="POST"
)
result = json.loads(urllib.request.urlopen(req).read())
# code=0 means success
```

**TRIAL-AND-ERROR DISCOVERY**:
- ❌ `PATCH .../files/{token}` with `{"parent_token": "target"}` → `code=99992402 field validation failed`
- ❌ `POST .../move` with `{"type": "file", "folder_token": "target"}` → `code=1061003 not found`
- ✅ `POST .../move` with `{"type": "docx", "folder_token": "target"}` → `code=0 success`
- The `type` must match the actual file type (`docx` for Feishu docs, `file` for uploaded files like images)

### Delete a File or Folder

```python
# Delete a file (image, PDF, etc.)
url = f"https://open.feishu.cn/open-apis/drive/v1/files/{token}?type=file"

# Delete a Feishu doc
url = f"https://open.feishu.cn/open-apis/drive/v1/files/{token}?type=docx"

# Delete an empty folder
url = f"https://open.feishu.cn/open-apis/drive/v1/files/{token}?type=folder"

req = urllib.request.Request(url, headers=headers, method="DELETE")
resp = urllib.request.urlopen(req)
result = json.loads(resp.read())
# code=0 success
```

**PITFALL**: Folders must be EMPTY before deletion. Delete all files inside first, then delete the folder.

### Rename a File (PATCH)

```python
body = json.dumps({"name": "新名称"}).encode()
req = urllib.request.Request(
    f"https://open.feishu.cn/open-apis/drive/v1/files/{token}",
    data=body, headers=headers, method="PATCH"
)
```

**PITFALL**: `PATCH` only supports `name` (rename). Does NOT support `parent_token` for moving — use the `move` endpoint instead.

### Empty Subfolder Listing 400 Bug

Some subfolder tokens (especially under 练习题/) return HTTP 400 when listed directly:
```python
# ❌ This may fail with 400 for some subfolders
url = f"https://open.feishu.cn/open-apis/drive/v1/files?folder_token={subfolder_token}"
# ✅ Always works: list parent and verify by name
url = f"https://open.feishu.cn/open-apis/drive/v1/files?folder_token={parent_token}&page_size=100"
```

If a folder appears in the parent listing (type=folder) but direct listing fails with 400, it may be empty or an API ghost. Retry without `page_size` parameter.

## Config Consistency: Multi-File Token Updates

When changing the Feishu folder structure (create/delete folders), ALL config files must be updated in sync:

| Config File | Variables |
|:-----------|:----------|
| `~/.hermes/scripts/edu_system_common.py` | PARSED_FOLDER, WRONG_FOLDER, BACKUP_FOLDER, DAILY_FOLDER |
| `~/edu-hub/scripts/edu_common.py` | MATH_FOLDER, WRONG_FOLDER, DAILY_FOLDER |
| `~/edu-hub/feishu-folder-tokens.json` | JSON map of all tokens |
| `~/.hermes/scripts/raw-exam-parser.py` | RAW_FOLDER, PARSED_FOLDER, WRONG_FOLDER, BACKUP_FOLDER |
| `~/edu-hub/scripts/raw-exam-parser.py` | Same as above |
| Cron job prompt | Description of where files go |

Missing one → scripts silently write to wrong folders or get 404 errors.

## Centralized Token Registry Pattern

Instead of hardcoding Feishu folder tokens across multiple scripts, maintain a single source of truth:

### Registry File: `~/.hermes/feishu-tokens.json`

```json
{
  "_meta": { "version": "1.0", "last_updated": "ISO_TIMESTAMP" },
  "folders": [
    {
      "name": "解析试题",
      "alias": "parsed_exams",
      "token": "QWKhfNWCWlhU...",
      "parent": "PARENT_FOLDER_TOKEN",
      "purpose": "System parsed exam docs"
    }
  ]
}
```

Each entry should include:
- `name` — Chinese name (human-readable)
- `alias` — English alias (stable identifier, not affected by name changes)
- `token` — The Feishu folder token
- `parent` — Parent folder's token (for drift detection via backtracking)
- `purpose` — What the folder is used for

### Access Module: `~/.hermes/scripts/feishu_tokens.py`

```python
from feishu_tokens import get_token, get_token_by_alias, validate_all

token = get_token("解析试题")               # by name
token = get_token_by_alias("parsed_exams")  # by alias (stable)
report = validate_all()                     # check all tokens
```

### Integration Pattern

In each config module, load tokens from registry with hardcoded fallback:

```python
def _load_token(name: str, fallback: str) -> str:
    try:
        from feishu_tokens import get_token
        return get_token(name) or fallback
    except Exception:
        return fallback

PARSED_FOLDER = _load_token("解析试题", "HARDCODED_FALLBACK_TOKEN")
```

This ensures:
- Single point of truth for all tokens
- Graceful degradation if registry file is missing
- Easy batch updates when folder structure changes

### Token Validation Script

The `feishu_tokens.py` module includes a `validate_all()` function that:

1. **For each folder**: tries to list its contents directly
2. **If direct listing succeeds** (even 0 files) → token is valid
3. **If direct listing fails with 400** → falls back to parent backtracking: lists the parent folder, finds the child by name, and compares the actual token
4. **Detects token drift**: when a folder's token changes (typically last 12 characters shift)

### Daily Token Validation Cron

```bash
# Runs at 3:00 AM daily, non-peak hours
cron: "0 3 * * *"
script: feishu_tokens.py  # defaults to --validate when no args
deliver: local  # only log, no notification unless drift detected
```

The cron prompt should instruct the agent to check for ❌ invalid or ⚠️ drifted entries and update `feishu-tokens.json` if needed.

## Document Creation & Content Writing

To create a new Feishu document in a specific folder and write markdown content to it, see `references/doc-creation-workflow.md` for the complete pattern (create doc → write content → share → register in sync pipeline).

Key folder tokens for delivery:
- 顾问团报告: `ONPQf8IeSl649NdG4dDcdSN6nLf`
- 我的系统: `LQx1fl0v8lB1XFdAI29czWeznVh`
- Hermes生成文件: `Otppfr9EelPIawdezL2csUXCnoh`

## Dependencies: Installing `lark-oapi`

The `feishu-wiki-sync.py` script depends on `lark-oapi`. On Debian/Ubuntu WSL systems with externally managed Python interpreters:

### Installation (Debian/Ubuntu WSL)

```bash
# ❌ Will fail — system Python is externally managed:
# pip3 install lark-oapi              # error: externally-managed-environment

# ❌ uv pip install also fails for the same reason:
# uv pip install lark-oapi --system   # error: externally managed

# ✅ Break-system-packages flag:
pip3 install lark-oapi --break-system-packages --timeout 30
```

The `--break-system-packages` flag is required on Debian 12+ / Ubuntu 24.04+.

### Dependencies Installed

```
lark-oapi==1.6.4
pycryptodome==3.23.0
requests_toolbelt==1.0.0
websockets==15.0.1
```

### Network Issues

`pycryptodome` is a large binary wheel (~50MB). If pypi.org downloads are slow or timeout:
- Use `--timeout 30` to allow enough time
- Retry if first attempt times out (packages may have been partially cached)
- The `uv` cache at `~/.cache/uv/` is not shared with pip — mixed tooling doesn't help

## Pitfalls & Learnings

1. **Pipe-table displacement**: All `|` text blocks shift past next heading. Never use pipe tables in Feishu md content.
2. **Token drift**: Last 12 chars of folder tokens change on recreate. Error 1770039 = token drift. Verify by listing parent.
3. **Doc-space vs Drive-space**: `feishu_doc_create(type="folder")` creates Doc-space tokens (S31...) invisible to Drive API.
4. **Bot App Space isolation**: Bot-created folders live in private space. Share explicitly via permission API.
5. **Self-built bot not searchable**: Cannot be found by user. Share docs/folders via API + give direct link.
6. **Script path**: Cron script paths must be bare filenames relative to `~/.hermes/scripts/`.
7. **lark_oapi POST failures**: Error 9499 → switch to raw urllib.request.
8. **Cron stdout**: Format as structured JSON with `--- SYNC_RESULT ---` delimiter.
9. **Move ≠ Delete/PATCH**: Use `POST .../move` with proper `type` field. `PATCH parent_token` does NOT work.
10. **Empty before delete**: Folders must be emptied (all files removed) before the folder itself can be deleted.
11. **File type matter**: `POST .../move` needs `type: "docx"` for Feishu docs, not `"file"`.
12. **lark-oapi not installed**: If `feishu-wiki-sync.py` fails with `ModuleNotFoundError: No module named 'lark_oapi'`, install via `pip3 install lark-oapi --break-system-packages --timeout 30` (Debian/Ubuntu externally-managed Python workaround). On first runs after creation this is a common gap.

## Common Error Codes

| Code | Meaning | Fix |
|------|---------|-----|
| 1770024 | invalid operation | Missing `text_run` wrapper or wrong block field |
| 1770039 | folder not found | **Token drift** — last 12 chars differ. Re-list parent. |
| 99992402 | too many blocks / field validation | Split into batches of 50. Or wrong API params. |
| 9499 | Bad Request (SDK) | Switch to raw HTTP |
| 1061003 | not found | File token is wrong or type mismatch in move/delete |
| 1061002 | params error | Wrong URL format or missing required query params |
