---
name: feishu-doc-versioning
description: "飞书文档版本管理规则 — 每次修改已有文档时，保留旧版本不动，新建带版本号的新文件（v1→v2→v3），便于回溯比对。"
version: 1.0.0
triggers:
  - user asks to modify/update/rewrite/change an existing Feishu document
  - user asks to revise published article content in Feishu
  - I'm about to write new content to an existing Feishu doc_token
---

# 飞书文档版本管理规则

## 核心规则

**每次调整文件时，保留旧版本文件不动，新建带版本号的新文件，不直接在原文件上覆盖。**

v1 → v2 → v3 ... 所有版本在飞书文件夹中并列存在，便于用户回溯比对历史版本。

## 操作流程

### 当需要修改一个已有飞书文档时：

1. **不要**调用 `feishu-md-writer.py` 写入旧的 doc_token
2. **而是**：
   a. 在飞书同目录下创建新文档（API: POST /open-apis/docx/v1/documents，传 folder_token）
   b. 标题追加版本号：原标题 → `原标题 v2`
   c. 用 `feishu-md-writer.py` 写入新 doc_token
   d. 记录新旧 doc_token 对应关系

### 例外规则

- 仅当用户明确说「直接覆盖原文件」时才走覆盖模式
- 仅当文档第一次写入（尚无版本历史）时，不走版本管理

## 版本号规则

- v1：原始版本
- v2：第一次修改
- v3：第二次修改
- 以此类推

## 本地文件同样适用

本地生成的文章 JSON 文件也遵循相同规则：
- 不覆盖旧文件
- 新建带版本标记的文件名
- 例如：`wechat_主题_v1.json` → `wechat_主题_v2.json`
