---
name: feishu-docx-content-creation
description: 通过 Feishu Open API 创建结构化文档、添加内容块、管理文档内容的完整工作流
version: 1.0.0
author: Hermes Agent
metadata:
  hermes:
    tags: [feishu, docx, api, document, blocks, content-creation]
    use_cases:
      - 需要在飞书云盘创建包含结构化内容的文档并归档
      - 需要将工作信息、报告、反馈等以格式化文档存入飞书
      - 任何需要在飞书中生成包含标题/段落/列表等富文本内容的场景
---

# 飞书文档内容创建工作流

## 一、整体流程

```
1. 获取 tenant_access_token
2. （如需归档到已有文件夹）导航到目标文件夹获取 folder_token
3. 创建空白文档 → 获取 document_id
4. 通过 blocks API 批量添加内容
5. 验证文档内容完整
6. 构造文档 URL 交付给用户
```

## 二、关键 API 端点

| 操作 | HTTP | 端点 |
|------|------|------|
| 获取 Token | POST | `/open-apis/auth/v3/tenant_access_token/internal` |
| 创建文档 | POST | `/open-apis/docx/v1/documents` |
| 获取文档内容 | GET | `/open-apis/docx/v1/documents/{doc_id}/raw_content` |
| 批量添加块 | POST | `/open-apis/docx/v1/documents/{doc_id}/blocks/{block_id}/children` |
| 批量删除块 | DELETE | `/open-apis/docx/v1/documents/{doc_id}/blocks/{block_id}/children/batch_delete` |
| 查询文件夹子项 | GET | `/open-apis/drive/explorer/v2/folder/{token}/children` |
| 获取文件列表 | GET | `/open-apis/drive/v1/files` |

> ⚠️ **关键陷阱**：添加子的正确端点是 `/children`（POST），**不是** `/children/batch_create`（后者返回 404）。

## 三、支持的块类型

### ✅ 支持（可通过 children POST 批量创建）

| block_type | key | 说明 |
|:----------:|:----|:-----|
| 2 | `text` | 普通段落 |
| 3 | `heading1` | 一级标题 |
| 4 | `heading2` | 二级标题 |
| 5 | `heading3` | 三级标题 |
| 1 | `page` | 页面根块（文档自动创建） |

### ❌ 不支持（children POST 返回 invalid param）

| block_type | key | 替代方案 |
|:----------:|:----|:---------|
| 27 | `bullet` | 使用 text block（type=2），内容前加空格缩进 + Unicode 项目符号（如 `•`） |
| 17 | `divider` | 用 text block 加空行或 `─` 分隔线 |
| 其他列表类型 | 同上 | 同上 |

> 经验：有测试发现 H1/H2/H3 均成功，但 bullet 和 divider 返回 1770001 (invalid param)。

## 四、创建文档的步骤

### Step 1: 获取 Token

```python
import requests
r = requests.post("https://open.feishu.cn/open-apis/auth/v3/tenant_access_token/internal",
    json={"app_id": APP_ID, "app_secret": APP_SECRET})
TOKEN = r.json()["tenant_access_token"]
```

> ⚠️ 注意使用 requests/curl 时，app_secret 中可能包含特殊字符（如 `'`），
> 在 shell 命令中需要转义或用 `@file` 方式传参。

### Step 2: 创建空白文档

```python
r = requests.post("https://open.feishu.cn/open-apis/docx/v1/documents",
    headers={"Authorization": f"Bearer {TOKEN}"},
    json={"title": "文档标题", "folder_token": "目标文件夹token"})
doc_id = r.json()["data"]["document"]["document_id"]
```

- `folder_token` 可选，不传则创建在根目录
- 返回的 `document_id` 同时也是根 page block 的 `block_id`

### Step 3: 批量添加内容块

```python
payload = {
    "children": [
        {"block_type": 4, "heading2": {
            "elements": [{"text_run": {"content": "标题文本", "text_element_style": {"bold": False}}}],
            "style": {}
        }},
        {"block_type": 2, "text": {
            "elements": [
                {"text_run": {"content": "加粗部分", "text_element_style": {"bold": True}}},
                {"text_run": {"content": "普通部分", "text_element_style": {"bold": False}}}
            ],
            "style": {}
        }},
    ],
    "index": 0  # 0=追加到末尾
}
r = requests.post(
    f"https://open.feishu.cn/open-apis/docx/v1/documents/{doc_id}/blocks/{doc_id}/children",
    headers={"Authorization": f"Bearer {TOKEN}", "Content-Type": "application/json"},
    json=payload
)
```

### Step 4: 构造文档 URL

文档 URL 格式：`https://{tenant_prefix}.feishu.cn/docx/{doc_id}`

tenant_prefix 可通过文件列表 API 返回的 `url` 字段推断，或直接从飞书 Web 端地址获取。

## 五、目录导航（查找目标文件夹）

### 导航到子文件夹

```python
# 获取顶层文件夹列表
r = requests.get(f"https://open.feishu.cn/open-apis/drive/v1/files?page_size=50",
    headers={"Authorization": f"Bearer {TOKEN}"})

# 使用 v2 explorer 查看文件夹子项
r = requests.get(
    f"https://open.feishu.cn/open-apis/drive/explorer/v2/folder/{folder_token}/children?page_size=50",
    headers={"Authorization": f"Bearer {TOKEN}"})
```

> ⚠️ `drive/v1/folders/{token}/children` 返回 404，需使用
> `drive/explorer/v2/folder/{token}/children`。

## 六、清理测试块

如需删除已添加的测试块（按 range 批量删除）：

```python
import requests
r = requests.delete(
    f"https://open.feishu.cn/open-apis/docx/v1/documents/{doc_id}/blocks/{doc_id}/children/batch_delete",
    headers={"Authorization": f"Bearer {TOKEN}", "Content-Type": "application/json"},
    json={"start_index": 21, "end_index": 27}
)
```

> `start_index` 和 `end_index` 基于 children 列表的 0-based 索引位置。

## 七、常见陷阱

1. **quote 转义**：app_secret 含特殊字符时，shell 命令中 curl 的 `-d '...'` 可能出错。
   改为 `-d @file.json` 或直接用 Python requests 库。
2. **block payload 格式**：每个块必须包含 `block_type` 和对应的类型 key
   （type=2 → `text`，type=3 → `heading1`，type=4 → `heading2` 等）。
3. **多个块一次性提交**：children 数组支持一次性创建多个块，减少 API 调用次数。
   已验证：24 个块在单次 API 调用中一次性添加成功。
4. **Python True/False**：在 Python 代码中写 JSON 字面量时，必须用 `True`/`False`
   （大写），不能用 `true`/`false`（JS 风格）。
5. **`index` 参数**：`index: 0` 表示追加到末尾。也可以指定位置插入。
6. **文档 token 漂移**：飞书文件夹 token 会随操作漂移，使用前应通过列表 API 重新获取。
7. **`upload_all` 对 HTML 文件不可靠**：`drive/v1/files/upload_all` 上传 .html 文件返回
   成功（code=0+file_token），但文件不会出现在 drive 文件列表中（metas API 返回 404）。
   ❌ 不要用 upload_all 上传 HTML 报告。✅ 应创建 docx 文档并通过 blocks API 填充内容。
8. **中文内联字符串语法陷阱**：在 Python 的 `execute_code` 或 inline 代码块中，
   包含中文引号、特殊字符（如 `·`、`「」`）的字符串字面量可能导致 SyntaxError
   （"closing parenthesis ']' does not match opening parenthesis '{'"）。
   **缓解方案**：将 JSON/payload 写入独立文件（`write_file`→磁盘），再从文件读取，
   而非在 inline 代码中直接构造字面量。参考 `references/chinese-inline-string-workaround.md`。

## 八、参考文件

- `references/docx-api-quirks-session-20260516.md` — docx API 调试记录
- `references/chinese-inline-string-workaround.md` — 中文内联字符串的语法规避方案
