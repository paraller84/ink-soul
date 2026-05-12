# Environment & Memory (synced from framework)

WSL RTX 4070 8GB / CUDA 12.6 / nvidia-utils 590。nvidia-smi正常但/dev/nvidia*可能不存在。DeepSeek V4 Flash主模型，V4系列原生1M上下文。ChromaDB: ~/.hermes_tools/knowledge_base/vector_store/chroma.sqlite3（435MB），7级去重。知识库门户: ~/wiki/guides/kb-portal.md。kb-wiki-sync.py→raw/knowledge-base/（1166页），cron 23:00。feishu-wiki-sync.py双目录（LLM讨论+经验沉淀），cron 9AM。feishu-tokens.json集中管理所有飞书token。
§
Edu-Hub: ~/edu-hub/app.py :5002（统一入口，单 Flask 进程）。三大模块（非独立系统）：数学模块(C003)/英语模块(C008)/语文模块(C011)，同属 Edu-Hub 代码工程，通过 Flask Blueprint 注册共享 app.py、auth、质量门禁 C015。数学模块(cron脚本驱动)/英语模块(最成熟,v1→v2→v3)/语文模块(两个分支并行chinese/ + chinese-v2/)。术语标准：称"模块"不称"系统"。
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
§
C012 feishu-task-manager.py bug fixed: list-user crashed on `str` object `due.get("timestamp")` at 4 locations (analyze_tasks, generate_review_report, cmd_list_user, cmd_get). Root cause: registry stores `due` as either dict `{"timestamp": "..."}` or string `""`/`"0"`. Fixed with isinstance() branching + try/except. Also added Feishu doc dashboard workaround for app-created task invisibility UX.
§
C012飞书待办系统关键发现：创建任务时需加 members=[{"type":"user","id":"ou_50b21c92548fbb2173b049e57dfbdbec","role":"assignee"}] 才能在用户飞书任务App中可见。meeting命令已支持 "标题::描述" 格式。feishu-task-manager.py已修复list-user的due字段类型bug（due可能是string也可能是dict）。
§
会议创建待办的正确流程：①用 meeting 命令批量创建，自动设置 project=会议名；②create_task 必须带 members 字段（指定用户为 assignee），任务才能在飞书任务App中可见；③建议用 "标题::描述" 格式附带会议关键讨论点。C012脚本已完整支持此流程。
§
material-production-system 是纯对话工作流（skill定义行为），不需编码实现。直接可实战，按两阶段走：材料顾问团4轮问询→制作命令书→用户确认→4角色流水线输出逐页内容。无需构建技术系统。
§
用户是EAST数据治理专项工作组核心成员（数据管理部），工作小组组长为苏金华（苏总），工作小组成员包括翟玉婷、高如海、毕姝晨、施月华、宋为翔。用户负责在季度工作会上做主要汇报。工作邮箱 yeyu-004@cpic.com.cn。收到3份关键背景文档《关于成立EAST数据治理专项工作组的通知》《EAST数据治理专项工作方案》《转发_Re_关于召开 EAST 数据治理专项工作组会议通知》，已存档于 ~/.hermes/cache/documents/。
§
kb-wiki-sync cron job_id=fe8aa1197d7b (每日23:00, no_agent)。feishu-wiki-sync cron job_id=1079e9c6f7fd (每日09:00, no_agent)。wpsdoc_cache.json位置: ~/.hermes_tools/knowledge_base/wpsdoc_cache.json（非cache/子目录）。--full/--organize-only跳过增量扫描，全量重建需先rm cache再跑无参数模式。OLLAMA_HOST已全部修复为localhost:11434（4脚本: pipeline/health/retrieve/vectorize）。
§
材料制作飞书Drive目录结构: Hermes生成文件夹→材料制作(PzFWfbMo7lgew1dyk4hcMk91n6b)→各项目子文件夹。已建: EAST二季度工作会(BTvefUaJ5lifJFd5MlJculUUnSc), 保险数据报送项目进展汇报-苏总(KbiwfKwfplj50Sdk03TcSBjsnRe)。所有产出物(内容纲要/文稿/PPT素材/台账)以飞书文档形式存入对应项目文件夹。feishu-tokens.json注册表中已登记别名 material_production_root / east_q2_report / baosong_project_report。
§
飞书文件夹创建共享技能已沉淀: feishu-folder-create-and-share (devops目录)。完整流程：get_fei_token()→POST create_folder_at_parent→POST permissions/{token}/members→写入feishu_folder_tokens.json→validate_all()。用户open_id=ou_50b21c92548fbb2173b049e57dfbdbec, perm=full_access。三大父夹: 系统文档(LQx1fl0v8lB1XFdAI29czWeznVh), Hermes生成(Otppfr9EelPIawdezL2csUXCnoh), 内容工厂(OaHdfQM9flT1vudSxmFcHVRMnEg)。
§
⚠️ **飞书文档版本规则（不可违反）**：修改已有飞书文档时，禁止覆盖修改原文件。必须：①保留旧版本不动（永不调用feishu-md-writer.py写入旧doc_token）；②在同目录创建新文档，标题追加版本号（v1→v2→v3）；③用feishu-md-writer.py写入新doc_token。唯一例外：用户明确说"直接覆盖"。每次违反将加重记忆权重。
§
材料制作系统规范: ①每份材料建独立Feishu Drive项目子文件夹（飞书云盘→Hermes生成文件夹→材料制作/）; ②必须维护材料使用台账（外部来源+KB引用+产出物，一笔不漏）; ③源材料归档到项目目录后入库KB; ④所有修改必须建v1/v2/v3新版本文件，严禁覆盖已有文档。
§
汇报材料叙事偏好: ①用监管质检数据说话而非自评（如质检异常数量趋势）; ②问题→协作框架的逻辑链路，语气是"协作"（如何一起做好）而非"追责"（谁有问题）; ③用实际案例推演方案，让听众当场看到新规则的效果。
§
Edu-Hub 架构修正：C003/C008/C011 是 Edu-Hub 统一教育平台下的学科模块（非独立系统），共享同一个 Flask 应用（app.py）、认证、session、质量门禁。术语标准：「系统」=独立部署应用，「模块」=Edu-Hub 内学科单元。C003=数学模块、C008=英语模块、C011=语文模块。
§
会议纪要处理—说话人指认的正确流程：从录音转写原文中提取每个"说话人X"最具识别性的段落（2-4段，含人名/项目/数据），呈现给用户指认。不可自行推断映射，不可跳过此步骤直接进入待办提取。用户原话："提供每个需要指认人说的最具备识别性的一段话,方便我指认"。此约束已写入meeting-minutes-workflow (Step 0) 和 getnotes-sync-pipeline (Speaker Mapping v1.1)。
§
C003 数学模块改造完成（2026-05-12）。架构从 bundle模式改为session模式（逐题API）。新增：大挑战引擎（20题/30分钟限时）、挑战题（7类轻奥数）、综合题（5类跨学科）、积分共享英语池、成就页。新文件：session_manager.py, challenge_engine.py, student_portal.html, challenge/session.html, achievements.html。API前缀 /api/v2/。每日练习10题分布：变式2+基础2+进阶1+应用1+挑战1+综合2+口算1。
§
Tool quirk: `patch()` tool (from hermes_cli) can silently fail — returned "ok" for 6 files but modified 0. Root cause unknown (possibly the old_string didn't exactly match due to whitespace/encoding, but tool returned ok anyway). When critical file modifications fail silently, always verify with a secondary check (read back or grep). For guaranteed writes, use direct Python file operations or `patch` with careful string matching and post-write verification.
§
Feishu 群聊平行对话架构: 5个群已创建（oc_8d3cb78bca0c29e354fe15d97279bca0=汇报材料, oc_59e724c052b3c57d77946d81db82476e=会议处理, oc_9eef4ca9d845d397e908975a3104111d=内容生成, oc_64bcdf55431bedc8ba75d27c70cd4b0a=每日思想碰撞, oc_e9f5d48f82f74064f2526e271759235d=教育系统优化）。FEISHU_GROUP_POLICY=open无需@。每个群独立会话上下文，可同时并行处理不同任务。已注册到feishu-tokens.json。

---
Last synced: 2026-05-12 23:01