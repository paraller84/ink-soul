---
name: feishu-folder-create-and-share
description: "在飞书云盘中创建专属文件夹并共享给用户（夜雨）的标准流程——包含API调用、Token注册、权限设置和常见陷阱"
version: 1.0.0
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

| 别名 | 名称 | Token |
|------|------|-------|
| `system_docs` | 系统文档夹 | `LQx1fl0v8lB1XFdAI29czWeznVh` |
| `hermes_generated` | Hermes生成文件夹 | `Otppfr9EelPIawdezL2csUXCnoh` |
| `content_factory` | 内容工厂出品夹 | `OaHdfQM9flT1vudSxmFcHVRMnEg` |

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

### Step 3: 共享给用户

**API**: `POST https://open.feishu.cn/open-apis/drive/v1/permissions/{token}/members?type=folder`

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

### 2. 文件夹名称唯一性
在同一父文件夹中创建同名文件夹会创建多个同名文件夹。建议：
- 名称中加时间戳或用 UUID 后缀
- 创建前先检查目录下是否已有同名文件夹

### 3. 权限传播
文件夹权限默认不会自动传播给子文件夹中的文档。每个文档的分享需要单独设置。

### 4. 路径引用
`feishu_tokens.py` 模块需要从 `~/.hermes/scripts/` 目录下导入。
```
from feishu_tokens import get_fei_token, get_token
```

### 6. 创建文档后内容为空

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
```

## 验证

操作完成后运行以下命令确认注册表一致性：
```bash
cd ~/.hermes/scripts && python3 -c "from feishu_tokens import validate_all; validate_all()"
```
