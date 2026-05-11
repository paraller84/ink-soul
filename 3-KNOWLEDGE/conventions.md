# 编码与沟通惯例

## 通信规则
1. 每次输出标注模型名称（如 `[DeepSeek V4 Flash]`）
2. 每次任务完成后告知「已完成」
3. 飞书渲染 Markdown（粗体、代码块、链接支持）
4. MEDIA:/path 发送图片/音频/文件

## 编码规则
1. 所有编码必须走 C005 五角色流程
2. 多步骤任务（3+步）用 execute_code 批量编排
3. Bug 修复先走 C007 Mode D 审核
4. 连字符文件用 importlib.util.spec_from_file_location() 导入
5. 状态迁移必须幂等 + 旧路径兼容
6. Flask Blueprint 模板目录必须唯一，用 url_for()

## Session 协议
- coffee break = 暂停
- start = 恢复
- end = 终止
- 编码实现仅在 start 后执行

## 飞书文档
- 创建文档：POST /open-apis/docx/v1/documents + folder_token
- 分享：POST /open-apis/drive/v1/permissions/{token}/members
- token 漂移：后 12 字符变化 → error 1770039
- block_type: text=2, h1=3, h2=4, h3=5, h4=6, bullet=12, code=14, quote=15
- 禁用 pipe 表格
- 每次修改保留旧版本，新建带版本号的新文件

## 部署规则
- Web 系统必须配置 Windows 端口转发
- 使用 netsh interface portproxy
- WSL 全量导出命令：wsl --export
