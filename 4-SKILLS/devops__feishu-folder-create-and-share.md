---
name: feishu-folder-create-and-share
description: "在飞书云盘中创建专属文件夹并共享给用户（夜雨）的标准流程——包含API调用、Token注册、权限设置和常见陷阱"
version: 1.2.0
triggers:
  - user asks to create a Feishu Drive folder / 创建飞书文件夹
  - user asks to share a folder / 分享飞书文件夹
  - user needs a dedicated workspace / 需要专属工作区
  - I need to create a Feishu folder for organizing documents
  - user mentions "在飞书建个文件夹" or similar
---

# 飞书文件夹创建与共享技能

## 核心信息

| 项目 | 值 |
|------|-----|
| **用户 open_id** | `ou_50b21c92548fbb2173b049e57dfbdbec` |
| **共享权限** | `full_access`（完全权限） |
| **Token 注册表模块** | `~/.hermes/scripts/feishu_tokens.py` |
| **Token 注册表文件** | `~/.hermes_tools/knowledge_base/feishu_folder_tokens.json` |
| **认证方式** | 飞书 tenant_access_token（app_id/app_secret 从 `~/.hermes/.env` 读取） |
| **认证模块** | `feishu_tokens.get_fei_token()` |

### 常用父文件夹 Token

| 别名 | 名称 | Token | 状态 |
|------|------|-------|:----:|
| `system_docs` | 系统文档夹 | `LQx1fl0v8lB1XFdAI29czWeznVh` | ✅ |
| `hermes_generated` | Hermes生成文件夹 | `Otppfr9EelPIawdezL2csUXCnoh` | ✅ |
| `content_factory` | 内容工厂出品夹 | ~~`OaHdfQM9flT1vudSxmFcHVRMnEg`~~ | ❌ **404** |
| `projects_root` | 项目 | `F8apfVtErlmfEvdPWXGcF6mZn2g` | ✅ |

> ⚠️ **`content_factory` token 已确认漂移（404）** — 2026-05-16 上传内容系列时发现该 token 返回 HTTP 404 错误。飞书云盘中 Hermes生成文件夹（Otppfr9EelPIawdezL2csUXCnoh）下没有同名的「内容工厂出品」文件夹。**不要直接使用此 token**。替代方案：将内容上传到 Hermes生成文件夹下的「日常运营」或新建子文件夹。

**Hermes生成文件夹当前子结构（2026-05-16 确认）：**
```
Hermes生成 (Otppfr9EelPIawdezL2csUXCnoh)
├── 项目 (F8apfVtErlmfEvdPWXGcF6mZn2g)
├── 顾问团报告 (IR6wfpatklstu1d2LlHcyurondb)
├── 个人 (XMCHfepAYlIN8pddOmGcxqmEnZb)
├── 部门群产出 (CdquftRpTljAIJd3tIAcSEDEnxc)
├── 工作汇报 (KzSifJdMvlQYa8dW9RRcw3PsnQ5)
└── 日常运营 (LUbmfa67lljfPCdW4uPca0Ghnqh)
```

### 系统项目文件夹规范

所有**系统开发项目**（AI家庭教师、教育系统、工具应用等）的产出物（设计文档、测试案例、原型文件等）统一存放在此结构中：

```
Hermes生成文件夹 (Otppfr9EelPIawdezL2csUXCnoh)
  └── 项目 (F8apfVtErlmfEvdPWXGcF6mZn2g)  ← 所有开发项目根目录
       └── [系统名]                         ← 每个系统一个子文件夹
            ├── 设计文档 v1.0
            ├── 测试案例报告 v1.0
            └── ...
```

**创建项目子文件夹的标准流程（与创建普通文件夹相同，只是父文件夹固定为 `projects_root`）：**
```python
# 1. 创建 [系统名] 文件夹
parent_token = get_token("项目")  # 或使用 F8apfVtErlmfEvdPWXGcF6mZn2g
project_folder_token = create_and_share("系统名称", parent_token)

# 2. 在项目文件夹中创建文档
body = json.dumps({"title": "文档名称 v1.0", "folder_token": project_folder_token}).encode()
# ... 通过 API 创建文档，写入内容，再 share

# 3. 注册到 feishu_folder_tokens.json
with open(TOKENS_PATH) as f: registry = json.load(f)
registry["folders"].append({
    "name": "系统名称",
    "token": project_folder_token,
    "alias": "system_alias",
    "parent": parent_token
})
```

## 操作流程

### Step 1: 获取飞书认证 Token

```python
from feishu_tokens import get_fei_token
token = get_fei_token()  # 自动从 ~/.hermes/.env 读取 FEISHU_APP_ID / FEISHU_APP_SECRET
headers = {"Authorization": f"Bearer {token}"}
```

### Step 2: 创建文件夹

**API**: `POST https://open.feishu.cn/open-apis/drive/v1/files/create_folder`

```python
import json, urllib.request

body = json.dumps({
    "name": "文件夹名称",
    "folder_token": parent_token  # 父文件夹 token
}).encode()

req = urllib.request.Request(
    "https://open.feishu.cn/open-apis/drive/v1/files/create_folder",
    data=body,
    headers={
        "Authorization": f"Bearer {token}",
        "Content-Type": "application/json"
    })
resp = json.loads(urllib.request.urlopen(req).read())
folder_token = resp["data"]["token"]
```

**响应示例**:
```json
{
  "code": 0,
  "data": {
    "token": "AbCdEfGhIjKlMnOpQrStUvWxYz",
    "url": "https://xxx.feishu.cn/drive/folder/AbCdEfGhIjKlMnOpQrStUvWxYz"
  }
}
```

### Step 2b: 上传文件到飞书 Drive（非 docx 文档）

当需要上传 HTML / PDF / 图片等**二进制文件**到飞书云盘（而非创建 docx 文档）时，使用 `upload_all` API：

**API**: `POST https://open.feishu.cn/open-apis/drive/v1/files/upload_all`

**请求格式**: multipart/form-data

| 字段 | 值 | 说明 |
|:-----|:---|:------|
| `file_name` | `xxx-v1.9.html` | 文件名（含版本号） |
| `parent_type` | `explorer` | 上传到文件夹 |
| `parent_node` | `Otppfr9EelPIawdezL2csUXCnoh` | 目标文件夹 token |
| `size` | `57357` | 文件字节数 |
| `file` | (binary) | 文件内容 |

**Python 实现要点**：
- 使用 `email.mime.multipart.MIMEMultipart` 构建 multipart 请求体
- 注意 `Content-Disposition` 中 `name` 字段需匹配 API 参数名
- `Content-Type` 根据文件类型设置（HTML → `text/html`, 图片 → `image/png` 等）
- 上传后立即分享给用户（见 Step 3）

**完整代码模板**：

⚠️ **陷阱：不要使用 Python MIMEMultipart.as_bytes().split() 构建 multipart**
`email.mime` 库在 multipart 的 body 部分会使用 `\n` 换行，而飞书 API 要求 `\r\n`。手动 split + reassemble 会破坏 body 完整性，导致 `1062009: actual size is inconsistent with the parameter declaration size`。

✅ **正确做法：手动构建 multipart body：**
```python
import uuid, json, urllib.request, os

token = get_fei_token()
folder_token = 'Otppfr9EelPIawdezL2csUXCnoh'
fpath = '/path/to/file.html'
fname = 'file.html'
fsize = os.path.getsize(fpath)
boundary = '----' + uuid.uuid4().hex

# 构建 multipart 部件
parts = []

# 文本字段（每对 name=val 一个 part）
for name, val in [('file_name', fname), ('parent_type', 'explorer'),
                   ('parent_node', folder_token), ('size', str(fsize))]:
    part = f'--{boundary}\r\nContent-Disposition: form-data; name="{name}"\r\n\r\n{val}\r\n'
    parts.append(part.encode())

# 文件字段
with open(fpath, 'rb') as f:
    fb = f.read()

file_header = f'--{boundary}\r\n'
file_header += f'Content-Disposition: form-data; name="file"; filename="{fname}"\r\n'
file_header += 'Content-Type: image/png\r\n\r\n'  # 根据实际类型调整

# 组装完整 body
body_parts = b''.join(parts)
body_parts += file_header.encode()
body_parts += fb
body_parts += f'\r\n--{boundary}--\r\n'.encode()

req = urllib.request.Request(
    'https://open.feishu.cn/open-apis/drive/v1/files/upload_all',
    data=body_parts,
    headers={'Authorization': f'Bearer {token}',
             'Content-Type': f'multipart/form-data; boundary={boundary}'},
    method='POST')
resp = json.loads(urllib.request.urlopen(req, timeout=30).read())
file_token = resp['data']['file_token']
```

**Content-Type 对照表：**
| 文件类型 | Content-Type |
|---------|-------------|
| HTML | `text/html` |
| PNG | `image/png` |
| JPG/JPEG | `image/jpeg` |
| PDF | `application/pdf` |
| JavaScript | `text/javascript` |
| CSS | `text/css` |

### Step 3: 共享给用户

共享的 API 参数因对象类型不同而异：

| 对象类型 | `type=` 参数值 | 示例 |
|---------|---------------|------|
| **文件夹** | `folder` | `?type=folder` |
| **文档 (docx)** | `docx` | `?type=docx` |
| **其他文件** | `file` | `?type=file` |

**⚠️ 关键陷阱**：分享 docx 文档时必须用 `type=docx`，不要用 `type=file`（会报 1063001 Invalid parameter）。

**共享文件夹**：
```python
body = json.dumps({
    "member_type": "openid",
    "member_id": "ou_50b21c92548fbb2173b049e57dfbdbec",
    "perm": "full_access"
}).encode()

req = urllib.request.Request(
    f"https://open.feishu.cn/open-apis/drive/v1/permissions/{folder_token}/members?type=folder",
    data=body,
    headers={
        "Authorization": f"Bearer {token}",
        "Content-Type": "application/json"
    })
urllib.request.urlopen(req)
```

**共享文档**：
```python
body = json.dumps({
    "member_type": "openid",
    "member_id": "ou_50b21c92548fbb2173b049e57dfbdbec",
    "perm": "full_access"
}).encode()

req = urllib.request.Request(
    f"https://open.feishu.cn/open-apis/drive/v1/permissions/{doc_token}/members?type=docx",
    data=body,
    headers={
        "Authorization": f"Bearer {token}",
        "Content-Type": "application/json"
    })
urllib.request.urlopen(req)
```

### Step 4: 注册到 Token 注册表

将新文件夹信息写入 `feishu_folder_tokens.json`：

```json
{
  "folders": [
    ...,
    {
      "name": "文件夹名称",
      "token": "AbCdEfGhIjKlMnOpQrStUvWxYz",
      "alias": "english_alias",
      "parent": "父文件夹Token"
    }
  ]
}
```

**流程**:
1. 读取现有 JSON 文件
2. 在 `folders` 数组末尾追加新条目
3. 写回文件（覆盖）
4. 运行 `python3 -c "from feishu_tokens import validate_all; validate_all()"` 验证

### Step 5: 返回给用户

```
✅ 已创建并共享文件夹「名称」
📂 飞书链接：https://bytedance.feishu.cn/drive/folder/{folder_token}
```

## 常见陷阱

### 1. Token 漂移（Error 1770039）
飞书文件夹的 URL 中后 12 个字符可能会变化。创建/查找到的 token 务必立即记录。
**症状**：`error 1770039` — token 的末 12 位与实际值不符。
**解决**：始终从 API 返回的 `data.token` 中获取 token，不通过拼接 URL 猜 token。

### 2. 文件夹名称唯一性 → Token 漂移陷阱（🐛 2026-05-16）

在同一父文件夹中创建同名文件夹**不会**覆盖或报错，而是会创建多个同名文件夹。返回不同的 token，列表 API 会列出全部同名文件夹。

**后果**：\n- 注册表中记录的是旧 token，实际飞书浏览时用的是新 token\n- 上传文件到旧 token 返回 400（无效文件夹）\n- 用户在飞书界面看到多个同名文件夹，分不清哪个是当前在用\n- **最隐蔽的后果**：两次调用父文件夹列表 API 可能返回不同的 token（各指向不同的同名文件夹），第一次 listing 得到的 token 与第二次 listing 得到的 token 不同，导致上传失败后误判为\"API 不稳定\"\n\n**检测方法**：\n```python\n# 在 feishu_folder_tokens.json 中，检查同一父文件夹下是否有同名文件夹的多个条目\nfrom collections import Counter\nparents = [(entry.get('parent'), entry['name']) for entry in registry['folders']]\ndups = [item for item, cnt in Counter(parents).items() if cnt > 1]\nif dups:\n    print(f\"⚠️ 检测到重复文件夹: {dups}\")\n    # 需要清理：用父文件夹列表 API 确认哪个是当前有效的，删除多余的\n```\n\n**修复**：
- 上传前不要信任注册表中的 token 缓存，调用父文件夹列表 API 获取实际 token
- 在 `feishu_folder_tokens.json` 中，用 `parent` 字段+定期 `validate_all()` 检测漂移

```python
def resolve_folder_token(folder_name, parent_token, headers):
    \"\"\"获取父文件夹下指定子文件夹的最新 token\"\"\"
    url = f\"https://open.feishu.cn/open-apis/drive/v1/files?folder_token={parent_token}&page_size=100\"
    resp = json.loads(urllib.request.urlopen(urllib.request.Request(url, headers=headers)).read())
    for f in resp[\"data\"][\"files\"]:
        if f[\"type\"] == \"folder\" and f[\"name\"] == folder_name:
            return f[\"token\"]
    return \"\"
```

### 3. 权限传播
文件夹权限默认不会自动传播给子文件夹中的文档。每个文档的分享需要单独设置。

### 4. 路径引用
`feishu_tokens.py` 模块需要从 `~/.hermes/scripts/` 目录下导入。
```
from feishu_tokens import get_fei_token, get_token
```

### 5. feishu-md-writer.py 子进程空写入陷阱（⚠️ 已踩坑）

**症状**：从 Python `subprocess.run()` 或 `execute_code` 中调用 `feishu-md-writer.py` 时，返回 exit code 0，stdout/stderr 为空，但飞书文档包含 0 个 child blocks（用户看到空白页面）。

**根因**：`feishu-md-writer.py` 依赖 `feishu_tokens.py` 的导入，而 `feishu_tokens.py` 通过 CWD（当前工作目录）定位 `~/.hermes/scripts/`。当从 subprocess 调用时，CWD 可能是 `/tmp/` 或其它目录，导致模块导入失败但未抛出异常。

**修复**：始终从终端直接运行，且先 cd 到正确的目录：
```bash
cd ~/.hermes/scripts && python3 feishu-md-writer.py <doc_token> <md_path>
```

**验证**：写入后通过飞书 API 检查文档的 child blocks 数量 > 0：
```python
req = urllib.request.Request(
    f"https://open.feishu.cn/open-apis/docx/v1/documents/{doc_token}/blocks/{doc_token}",
    headers=headers)
resp = json.loads(urllib.request.urlopen(req).read())
children = resp.get("data", {}).get("block", {}).get("children", [])
print(f"Document has {len(children)} child blocks")
# 应 > 0，等于 0 说明空写入
```

### 6. 分享 docx 文档时 type 参数错误

**症状**：分享创建的飞书文档时报 `error 1063001: Invalid parameter`。

**根因**：分享 API 的 `type` 参数需要匹配对象类型：
- 文档 → `type=docx`
- 文件夹 → `type=folder`
- 其他文件 → `type=file`

**修复**：确认对象类型后选择正确的 `type` 值。创建了 docx 文档后用 `type=docx`。
⚠️ `type=file` 不适用于 docx 文档（会返回 `1063001`）。

### 7. 飞书文档 API 调用失败时的本地缓存兜底

**症状**：读飞书文档时 API 返回 404（可能是 token 漂移或 app 权限不足）。

**兜底策略**：检查 `~/.hermes/cache/` 目录下是否有本地缓存副本。常见文件名模式：
- `<项目名>-design-v<版本号>.html`（设计文档 HTML）
- `advisory-v<版本号>-<角色>.md`（顾问团评估文档）
- `<场景名>-v<版本号>.md`（其他文档缓存）

### 9. 父文件夹 token 有效性检查

**症状**：创建子文件夹或上传文件时返回 `HTTP 404`。

**根因**：注册表中的父文件夹 token 可能已漂移或文件夹被删除。例如 2026-05-16 发现内容工厂 token `OaHdfQM9flT1vudSxmFcHVRMnEg` 返回 404。

**修复**：上传前先通过 `list files` API 验证父文件夹是否存在：
```python
# 验证父文件夹 token
url = f"https://open.feishu.cn/open-apis/drive/v1/files?folder_token={parent_token}&page_size=1"
req = urllib.request.Request(url, headers=headers)
resp = json.loads(urllib.request.urlopen(req, timeout=10).read())
if resp.get("code") != 0:
    # 父文件夹 token 不可用——需要找到可用的父文件夹
    # fallback: 使用 Hermes生成文件夹 (Otppfr9EelPIawdezL2csUXCnoh)
    print(f"⚠️ 父文件夹 token {parent_token} 不可用 (code={resp.get('code')})，改用 Hermes生成文件夹")
    parent_token = "Otppfr9EelPIawdezL2csUXCnoh"
```

**预防**：
- 不要在 memory 和 SKILL.md 中缓存文件夹 token 作为"可用状态"——每次会话前通过 list API 验证
- 如果在注册表中发现 404 token，在 feishu_folder_tokens.json 中添加 `"drifted": true` 标记

使用 `POST /open-apis/docx/v1/documents` 创建的飞书文档是**空文档**（仅含标题）。如果只创建不填充，用户在飞书中打开看到的是一份空白页。

**症状**：用户反馈「文档内容都是空的」「只看到标题没有内容」。

**解决**：创建文档后必须使用 `feishu-md-writer.py` 写入内容：

```bash
cd ~/.hermes/scripts && python3 feishu-md-writer.py <doc_token> <markdown_file>
```

`feishu-md-writer.py` 会自动：
1. 读取 markdown 文件，解析为飞书blocks
2. 清空文档现有内容
3. 分批写入新内容（每批40个blocks，含0.3秒限速）

**工作流**：
1. 用 API 创建文档 → 获取 doc_token
2. 将内容写成 markdown 文件
3. 调用 feishu-md-writer.py 写入
4. 再分享给用户

**注意**：`feishu-md-writer.py` 依赖 `feishu_tokens.py`，必须从 `~/.hermes/scripts/` 目录运行。  
创建文件夹的 API 返回 `resp["data"]["token"]`（不是 `resp["data"]["node"]["token"]`，`_at_parent` 后缀已废弃）。  
对于创建文档（非文件夹），API 不同：`POST /open-apis/docx/v1/documents` 参数需要 `folder_token`。

## 完整 Python 示例脚本

```python
"""Create a Feishu folder and share with user"""
import json, urllib.request, os, sys

HOME = os.path.expanduser("~")
sys.path.insert(0, os.path.join(HOME, ".hermes", "scripts"))
from feishu_tokens import get_fei_token

def create_and_share(folder_name: str, parent_token: str) -> str:
    """Create folder under parent and share with user. Returns folder_token."""
    token = get_fei_token()
    headers = {"Authorization": f"Bearer {token}", "Content-Type": "application/json"}

    # Create folder (NOTE: use create_folder not create_folder_at_parent)
    body = json.dumps({"name": folder_name, "folder_token": parent_token}).encode()
    req = urllib.request.Request(
        "https://open.feishu.cn/open-apis/drive/v1/files/create_folder",
        data=body, headers=headers)
    resp = json.loads(urllib.request.urlopen(req).read())
    assert resp.get("code") == 0, f"Create failed: {resp}"
    folder_token = resp["data"]["token"]

    # Share with user (NOTE: use type=folder for folders)
    body = json.dumps({
        "member_type": "openid",
        "member_id": "ou_50b21c92548fbb2173b049e57dfbdbec",
        "perm": "full_access"
    }).encode()
    req = urllib.request.Request(
        f"https://open.feishu.cn/open-apis/drive/v1/permissions/{folder_token}/members?type=folder",
        data=body, headers=headers)
    urllib.request.urlopen(req)

    return folder_token


def share_document(doc_token: str, group_chat_id: str = None) -> str:
    """Share a docx document with the user and optionally with a group chat.
    Returns the doc URL for sending in conversation."""
    token = get_fei_token()
    headers = {"Authorization": f"Bearer {token}", "Content-Type": "application/json"}

    # Share with user (NOTE: use type=docx for docx documents, NOT type=file!)
    body = json.dumps({
        "member_type": "openid",
        "member_id": "ou_50b21c92548fbb2173b049e57dfbdbec",
        "perm": "full_access"
    }).encode()
    req = urllib.request.Request(
        f"https://open.feishu.cn/open-apis/drive/v1/permissions/{doc_token}/members?type=docx",
        data=body, headers=headers)
    urllib.request.urlopen(req)

    # Also share with group chat if provided (view permission)
    if group_chat_id:
        body = json.dumps({
            "member_type": "openchat",
            "member_id": group_chat_id,
            "perm": "view"
        }).encode()
        req = urllib.request.Request(
            f"https://open.feishu.cn/open-apis/drive/v1/permissions/{doc_token}/members?type=docx",
            data=body, headers=headers)
        urllib.request.urlopen(req)

    return f"https://rcncjr4hag8s.feishu.cn/docx/{doc_token}"

## 交付后必须发送链接到对话

分享完成后，必须将文档链接发到当前群聊/DM中，不可只上传不发消息：
```text
文档链接: https://rcncjr4hag8s.feishu.cn/docx/{doc_token}
```

两步缺一不可：上传到飞书 + 发送链接到对话。

## 验证

操作完成后运行以下命令确认注册表一致性：
```bash
cd ~/.hermes/scripts && python3 -c "from feishu_tokens import validate_all; validate_all()"
```

### 交付完成后验证（必做）

```python
# 1. 验证文档有内容（非空白页）
req = urllib.request.Request(f"https://open.feishu.cn/open-apis/docx/v1/documents/{doc_token}/blocks/{doc_token}", headers=headers)
children = json.loads(urllib.request.urlopen(req).read())["data"]["block"]["children"]
assert len(children) > 0, "❌ 文档内容为空！"

# 2. 验证分享完成
req = urllib.request.Request(f"https://open.feishu.cn/open-apis/drive/v1/permissions/{doc_token}/members?type=docx&page_size=50", headers=headers)
members = json.loads(urllib.request.urlopen(req).read())["data"]["items"]
assert any(m["member_id"] == "ou_50b21c92548fbb2173b049e57dfbdbec" for m in members), "❌ 未分享给用户"
if group_chat_id:
    assert any(m["member_id"] == group_chat_id for m in members), "❌ 未分享给群"
```
