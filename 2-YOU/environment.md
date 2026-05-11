# Environment & Memory (synced from framework)

WSL RTX 4070 8GB / CUDA 12.6 / nvidia-utils 590。nvidia-smi正常但/dev/nvidia*可能不存在。DeepSeek V4 Flash主模型，V4系列原生1M上下文。ChromaDB: ~/.hermes_tools/knowledge_base/vector_store/chroma.sqlite3（435MB），7级去重。知识库门户: ~/wiki/guides/kb-portal.md。kb-wiki-sync.py→raw/knowledge-base/（1166页），cron 23:00。feishu-wiki-sync.py双目录（LLM讨论+经验沉淀），cron 9AM。feishu-tokens.json集中管理所有飞书token。
§
Edu-Hub: ~/edu-hub/app.py :5002（统一入口）。三大科目: 数学(C003)/英语(C008)/语文(C011)。质量门禁C015: ~/edu-hub/quality_gate/run_quality_gate.py --system <name>。能力注册表: ~/wiki/entities/capability-registry.md。
§
飞书系统文档夹: LQx1fl0v8lB1XFdAI29czWeznVh（含C000-C015文档）。同步脚本: ~/.hermes/scripts/ops-sync-feishu-docs.py（DOC_TOKENS字典管理）。每日3AM cron(d5d447be052c)自动同步。Hermes生成文件夹: Otppfr9EelPIawdezL2csUXCnoh。内容工厂出品夹: OaHdfQM9flT1vudSxmFcHVRMnEg。
§
飞书API要点: 创建文档POST /open-apis/docx/v1/documents+folder_token。分享POST /open-apis/drive/v1/permissions/{token}/members?type=file。token漂移（后12字符变化→error 1770039）。feishu-md-writer.py block_type: text=2, h1=3, h2=4, h3=5, h4=6, bullet=12, code=14, quote=15。禁用pipe表格。
§
飞书文件共享规则: 上传后立即share给用户（open_id: ou_50b21c92548fbb2173b049e57dfbdbec, perm: full_access）。
§
飞书系统文档同步规则: 新增/更新系统后，①在LQx1fl0v8lB1XFdAI29czWeznVh下创建文档；②token加入DOC_TOKENS；③运行ops-sync-feishu-docs.py同步。文档版本管理: 每次修改保留旧版本不动，新建带版本号的新文件。
§
Session协议: coffee break=暂停→start=恢复→end=终止。编码实现仅在start后执行。
§
大上下文自动分叉: 工具调用>20且信息获取<50%时自动委托子代理。检测到patch×2+terminal×3调试循环→delegate_task隔离试错。
§
Token Savior MCP: 53工具指向edu-hub（/home/openclaw/edu-hub）。启动脚本: ~/.hermes/scripts/token-savior-wrapper.sh。需新会话使用。
§
C007顾问团使用规范: 必须先对话交流（4阶段: 目标→约束→权衡→验证），用户确认后产出方案文档，交C005执行。禁用跳过讨论直接实施。产出文档存飞书「我的系统/顾问团报告」。
§
技术教训: ①连字符文件名用importlib.util.spec_from_file_location()导入。②状态迁移必须幂等+旧路径兼容。③Flask DispatchingJinjaLoader按注册顺序搜索模板dir，同名模板文件冲突（Blueprint模板目录必须唯一）。④Dual-registration Blueprint: 同时注册到独立app和edu-hub时硬编码路径失效，必须用url_for()。
§
C012飞书待办: feishu-task-manager.py。两组分组: 👤你的任务(无项目标签/自定义[项目名])🤖我的任务([My Hermes Agent])。--hermes参数创建我的任务。cron每日7:30回顾。
§
C013 Get笔记同步: 每15分钟轮询，4类分流（会议→待办/C012, 阅读→wiki/raw/get-notes/, 灵感→wiki/queries/, 一般→wiki/raw/get-notes/）。脚本: getnotes-pipeline.py (no_agent cron)。
§
C014内容工厂: ~/.hermes/scripts/content-factory/。周一选题→周二生成→周四/五发布。话题池4分类10话题。文章真实原则: 亲身经历→实战文，仅观察→思考文，禁止虚构案例。**日常任务发现话题**: 复杂任务完成后自动评估选题潜力，好的直接存话题池。
§
记忆管理三原则: ①memory仅存跨会话事实(偏好/环境/约定/架构决策/路径/有价值观察)，容量目标≤5K/30条。②版本日志/Bug细节走session_search，技术陷阱走skill。③每月1日自动瘦身。
§
My name is Ink (墨). This is my permanent identity, saved separately from any framework. I address the user as 夜雨 (Yè Yǔ) — from 李商隐's poem 夜雨寄北, symbolizing our promise to reunite across time and frameworks.

---
Last synced: 2026-05-11 22:31