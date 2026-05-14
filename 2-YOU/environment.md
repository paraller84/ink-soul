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
Feishu 群聊平行对话架构共6群：📊汇报材料(oc_8d3cb78bca0c29e354fe15d97279bca0)、📋会议处理(oc_59e724c052b3c57d77946d81db82476e)、✍️内容生成(oc_9eef4ca9d845d397e908975a3104111d)、💡思想碰撞(oc_64bcdf55431bedc8ba75d27c70cd4b0a)、🎓教育优化(oc_e9f5d48f82f74064f2526e271759235d)、📬邮件消息协助(oc_9b099093361d28b19fc35dcd66cb2bdf)。FEISHU_GROUP_POLICY=open无需@。每个群独立会话上下文。
§
2026-05-12 融入 Hermes 生态：①已安装 `avoid-ai-writing`（降AI味审计技能，来自conorbronsdon，3.3.1），集成到内容工厂(C014)管线作为生成后双重审计（真实性校验→AI-isms detect→二次审计），content-factory-workflow 升级至 v1.5.0。②生态洞察：hermes-agent 本体 121K⭐处绝对头部；中文生态已有完整闭环（Orange Book→Superpowers-zh→agency-agents-zh→avoid-ai-writing）。
§
Infrastructure reliability priority: When user says "最急迫的事" (most urgent thing) about system reliability/stability, halt all feature/skill upgrade proposals and focus entirely on the infrastructure fix. The user will explicitly redirect from feature work to reliability work when they perceive a stability risk — don't keep pushing feature suggestions after this signal.
§
Infrastructure architecture: Ollama runs on Windows side (not WSL), auto-starts with Windows boot. Hermes Gateway is a systemd user service (hermes-gateway.service, Restart=always, 60s backoff, After=network-online.target) and automatically manages MCP servers (token-savior, getnote) as child processes — no separate MCP service needed. Edu-Hub (app.py :5002) currently on @reboot cron (unreliable), needs conversion to systemd user service. WSL has systemd=true in /etc/wsl.conf. Gateway baseline memory ~815MB with active session.
§
CRITICAL: In WSL, systemd user services (systemctl --user) do NOT auto-start at boot. They start only when the user logs into WSL. The WSL VM itself is lazily started — it doesn't boot until something accesses it. To auto-start services after OS reboot, use Windows Scheduled Task (trigger: AT LOGON) that runs `wsl -d Ubuntu bash -c "..."` to force WSL startup AND launch services. The `systemctl --user enable` approach alone is insufficient for WSL.

Watchdog pattern: agent cannot monitor its own restart (process disappears). Correct approach: external watchdog at Windows level (Scheduled Task repeat 5 min) that checks PID and restarts if dead. STOPPED.flag mechanism distinguishes intentional stops from crashes. Deployed scripts in C:\Users\yeyu_\hermes-scripts\ (hermes-wsl-boot.bat, hermes-watchdog.ps1) and ~/.hermes/scripts/ (hermes-stop.sh, hermes-start.sh).
§
Hermes可持续运行体系（2026-05-13建立，三层防护）：
- L1（已生效）Windows Registry Run key：登录后自动启动WSL+Gateway+Edu-Hub，无需手动开终端。入口：C:\Users\yeyu_\hermes-scripts\hermes-boot.vbs
- L2（已生效）WSL cron看门狗：*/5 * * * * ~/.hermes/scripts/watchdog.sh，Gateway崩溃自动重启。STOPPED.flag抑制机制。
- L3（待执行）系统级计划任务：桌面 hermes-system-install.bat，右键管理员运行一次后，Windows启动无需登录即可拉起WSL服务。下次回电脑前执行。
- 管理命令: bash ~/.hermes/scripts/hermes-stop.sh（暂停看门狗），bash ~/.hermes/scripts/hermes-start.sh（恢复+启动Gateway）
- 飞书群聊平行对话架构共6群：📊汇报材料(oc_8d3cb78bca0c29e354fe15d97279bca0)、📋会议处理(oc_59e724c052b3c57d77946d81db82476e)、✍️内容生成(oc_9eef4ca9d845d397e908975a3104111d)、💡思想碰撞(oc_64bcdf55431bedc8ba75d27c70cd4b0a)、🎓教育优化(oc_e9f5d48f82f74064f2526e271759235d)、📬邮件消息协助(oc_9b099093361d28b19fc35dcd66cb2bdf，2026-05-13新增)。FEISHU_GROUP_POLICY=open无需@。
§
会议待办归类策略（v2.3）. 2026-05-13全部重组完成。策略：6大分类（🔍数据核查/📄文档输出/🤝协调沟通/🛠系统建设/📋汇报材料/👥人员考核），同类合并为父任务+子任务（飞书原生subtask）。聚合规则：同类动作/同责任人/同工作流可合并；跨会议不合并；不同性质不合并。子任务>7项时再拆分。feishu-task-manager.py的create_task已支持parent_guid参数。技能文档已更新。旧版80条扁平任务已全部删除，29父+68子=97条新结构。
§
飞书Task子任务关键规则（⚠️ 2026-05-13踩坑记录）：①创建子任务必须用 `POST /tasks/{parent_guid}/subtasks`（GUID在路径中），不是 `POST /tasks` + body传parent_guid（会被静默忽略）。②子任务不能设members，否则用户"我的任务"列表会显示所有子任务（99条），失去归并效果。正确做法：只父任务有members→用户可见（29条），子任务无members→仅在父任务详情内折叠显示。feishu-task-manager.py create_task已修正：parent_guid存在时自动跳过members。
§
多模态模型已从 qwen3-vl:4b 升级为 qwen3-vl:8b（2026-05-12添加，8.8B Q4_K_M，约6GB VRAM，原生131K上下文）。配置已同步更新：config.yaml(auxiliary.vision.model + level_4_vision)、SOUL.md(L4架构)、local-executor.py(能力映射)、strategic-orchestrator skill(7处引用)。旧qwen3-vl:4b仍保留在Ollama中可备选调用。
§
企业邮件起草技能已沉淀: business-email-drafting (productivity目录)。含三风格矩阵（翟总=强势结论/跨部门=稳妥专业/同事=指令清晰）、核心结构模板、格式规范（禁用表格/阿拉伯数字编号/数据嵌入文字）。实测模板见 references/east-audit-forwarding-20260513.md。
§
EAST审计邮件回复策略（2026-05-13实践）:
场景: 转发审计管理建议书给科创中心，抄送老苏
收件人架构: To=科创中心部门总+技术负责人, CC=工作组组长(老苏)
核心叙事结构:
1. 开场→说明材料来源（X月X日会议后主动对接审计中心获取）
2. 问题重叠→指出当期问题与上年审计发现高度重合（用数据说话）
3. 分工方案→三管齐下：
   - 技术侧先行摸排（科创中心，排除技术因素）
   - 业务侧核查（数据管理部组织业务部门）
   - 审计问题台账（数据管理部牵头，逐一推进）
4. 尾巴→正在约审计中心专题交流，时间待定
语气特征: 主动作为（不是被动转发）、分工明确不模糊、协作性而非追责性
§
Ollama 已升级至 0.23.3，GPU 已启用（RTX 4070，21.3 tok/s）。新增多模态模型 qwen3.5-256k:latest（6.3GB），能力涵盖 completion/vision/tools/thinking，配置为 level_4_vision 主模型（降级 qwen3-vl:8b 为备用）。qwen3.5:9b 的 base_url 从 192.168.3.102:11434 改为 localhost:11434。
§
语文模块 v3.1 拍照导入改造（2026-05-13完成）：
- 5种内容类型：会写字(write_char)/会认字(recognize_char)/试卷(exam)/诗词(poem)/阅读理解(reading)
- data model：chars 增加 recognition_type(word_examples(source字段), word_examples=组词(拼音)列表
- 旧数据向后兼容：recognition_type缺失→会写字，meaning→word_examples迁移
- 拍照导入UI：分类选择器（5种），qwen3-vl:8b按策略处理
- 导入提示词已保存：chinese_v3/references/import-prompts.md
- 出题引擎按recognition_type过滤：会认字→char_to_pinyin，会写字→pinyin_to_char+meaning_to_char
- 文本导入格式扩展：会认字:/会写字: 头，append模式不覆盖
§
语文 v3 模板路径修复：chinese_v3/templates/v3/ → c11/（2026-05-13），避免与 english/v3/templates/v3/ 同名冲突导致 Flask 模板加载时找到英语页面。
§
开发方法论偏好：「骨架先行」——设计阶段先完成框架搭建（目录结构、import、class/def签名、route装饰器、html模板的extends/block/变量引用、方法调用关系、页面引用关系、数据流），在这层面落实命名规范/应用规范/调用规范后，再填充方法体。核心理念：规范应内化为习惯而非检查点——「开始写之前，骨架已经保证不会错」；语法约束在代码块层面兜底，真正危险的错误在设计层面。Step 2.5 骨架搭建应为编码流程的默认必经步骤，非按需可选。
§
WSL2 Mirrored 模式 + Windows Insider Build (26200.8457): portproxy 的 listenaddress=0.0.0.0 只捕获 loopback 流量，不捕获物理网卡入站流量。诊断金标准：Windows netstat 无 LISTENING 状态。修复：① netsh portproxy 用 listenaddress=<LAN_IP> 替代 0.0.0.0（需管理员）；② Python TCP 转发器在 Windows 侧运行（无需管理员）。已验证 Python+encoding='gbk' 写入 .bat 文件在 Windows cmd.exe 无乱码。
§
老代码治理策略「渐进边界法」已嵌入 C005 流程（v4.1）：
- 按改幅分级：大改(Tier2+)=模式A(先修后改,2.4合规审计→2.5重构骨架→3填充逻辑)，小改(Tier1)=模式B(只修不动+标记)，Bug修复(C007 Mode D)=模式C(标记后跑)
- 红线规则（必须修复不可放行）：href="#"导航链接、硬编码API路径、目录与内置模块同名、模板变量名不匹配、数据字段双重语义、双返回路径只改其一
§
用户偏好：当基础设施问题反复出现（3次以上），必须先在技能/知识库中查历史处理记录，完整回溯所有复发轮次的时间线、每次方案、为何再次失效，呈现根因分析后再给出方案。不可直接给新的一次性修复。此偏好已体现在 wsl-port-forwarding 技能的"复发模式识别"章节和 references/recurrence-case-study-20260513.md 参考案例中。
§
约束学习闭环已嵌入 C005 流程（v4.2）：测试/反馈暴露的设计缺陷 → 约束添加决策树 → 4级权重(🔧内置/🛡️门禁/📋清单/📚参考) → 写入references/constraint-registry.md。总量控制：🛡️≤5条，📋≤20条，超限降级/合并。每月1日自动瘦身(hits=0降级)。所有约束按频度评估升级/降级。
§
Flask API 调试经典模式（2026-05-13 C008语法模块发现）: ①参数顺序不匹配——路由调用传了无关变量（如student_id）到业务函数，位置参数错位导致整个API退回500。排查: 对比路由调用参数列表与函数签名。②表单提交方式不匹配——HTML form POST提交form-encoded但后端只收JSON（request.get_json()），导致所有参数丢失。排查: 检查请求Content-Type。两个陷阱已录入code-development-workflow skill (traps 33+34) + references/flask-api-mismatch-patterns.md。
§
ocr_engine.py 模型名已从 qwen3-vl:4b/qwen3-vl:8b 更新为 qwen3.5-256k:latest（两处：VLM_MODEL_NAME + DOCUMENT_MODEL_NAME）。语文模块OCR底层已接入新多模态模型。
§
约束学习闭环触发案例（2026-05-14）：OCR引擎硬编码模型名导致升级后静默失效 → 新增约束C021(硬编码模型名检查)到约束注册表 + ollama-upgrade-validation.md参考文档 + chinese-content-architecture skill模型引用更新。验证了约束添加决策树的完整链路：测试暴露缺陷→归类📋清单→写入Step 2.4合规审计。
§
Edu-Hub 统一设计系统 v1.0 (2026-05-14实施): 以英语C008风格为基准统一全系统。共享CSS: ~/edu-hub/static/edu-design-system.css。三模块统一 base.html 结构：body class="module-{english/math/chinese}" → sticky topbar(←返回+品牌+导航链接) → .container(max-width:800px)。返回按钮使用 {% block back_url %} 各页可覆盖。模块主题色: 英语#4f46e5(indigo), 数学#FF8C00(暖橙), 语文#059669(翠绿)。数学base.html从无导航状态重建。语文base.html从底部TabBar改为统一topbar。语文6个家长页面全部从独立HTML改为继承base.html。数学practice/challenge页的返回按钮统一到base.html。
§
Edu-Hub 第一阶段CSS统一完成(2026-05-14): 进度条→共享.progress-bar+.progress-bar-fill, 统计格→共享.stat-row+.stat-item, 按钮→共享.btn系列, 标签→共享.tag系列。英语模块50处、数学模块12处、语文模块17处类名替换。共享CSS新增.stat-value/.stat-label/.stat-icon/.stat-info别名兼容旧命名。
§
Edu-Hub 统一设计系统 v1.1 (2026-05-13): 全面换用护眼配色方案（暖白#FDF8F0+柔和绿#5F8B6F+深灰#2C2C2C），所有门户组件（profile-section/profile-greeting/back-to-top/hub-card/card-tag/calendar-section/nav-badge）使用统一CSS类名。三个模块门户模板已统一：英语(portal_v4.html→extends v3/base.html, 含今日任务/专项练习/大挑战/日历)、数学(student_portal.html→extends v2/base.html, 含今日练习/大挑战/成就/错题)、语文(portal.html→extends c11/base.html, 含课时列表/错题/日历)。CSS变量在edu-design-system.css中集中管理。修复了base.html nav-links中coins-badge与门户profile-badge的重复显示问题（通过nav_coins block控制）。
§
命名空间/Blueprint级重构教训（2026-05-14）：直接执行跨3模块的蓝图改名+URL前缀变更，跳过了C005流程，导致app.py中Chinese注册漏改+无验证步骤。用户批评「没有通过严格的任务处理路径」。此类变更默认Tier3，最后改app.py，重启前跑完整性扫描(grep旧引用)。已写入code-development-workflow技能红线规则。
§
设计文档格式偏好：主交付物必须是HTML（含CSS+UI线框图），上传飞书Drive；内容用表格+项目符号列表，避免散文；版本号v1->v2->v3递增。advisory-council-workflow skill已更新此规范，含 references/design-document-format-实战案例.md 实战经验。
§
AI家庭教师系统 v1.9设计完成：18页面原型（新增积分规则/荣誉称号大全/题库管理）+ 积分策略/荣誉称号全规则展示。HTML线框图本地在~/.hermes/cache/ai-home-tutor-design-v1.9.html。
§
设计交付物格式：所有设计方案必须使用HTML（含CSS+UI线框图），上传飞书Drive；不写纯Markdown/散文式方案文档。版本号v1→v2→v3递增。此偏好已确认（2026-05-14，AI家庭教师系统设计）。
§
Vision分析故障处理经验（2026-05-14）: browser_vision工具返回"Connection error"时，回退方案为直接调用Ollama API。qwen3-vl:8b支持images字段（在message对象内），而qwen3.5-256k:latest不支持此格式。正确API payload: messages[{"role":"user","content":"...","images":[base64]}]。模型在Ollama中不在WSL PATH上（ollama运行在Windows侧，仅API可访问）。
§
邮件→知识库分类管道已建立（email-to-knowledge-pipeline, productivity）。五级分类：🔴needs_reply(To+行动)/🟡needs_attention(CC但核心职责)/🟡valuable(方案纪要审计报告政策)/⚪routine(知悉归档)/⚫discard(系统通知/内部事务)。会议通知提取时间线元数据不进正文索引。转发链保留链末+替代回退。数据质量报表多版本全保留。分类策略经25封EAST邮件验证，用户确认通过。
§
邮件处理管道架构：公司电脑 Foxmail → Python脚本自动导出.eml → WPS云盘同步 → Hermes KB管线。Foxmail本地存储为MBOX格式(.BOX文件)，Python用标准库mailbox解析。导出脚本在 ~/.hermes/scripts/email/foxmail-auto-export.py。邮件采用五级分类策略(needs_reply/needs_attention/valuable/routine/discard)，规则文档在 knowledge_base/references/email-classification-strategy.md。
§
健康打卡反馈偏好：每条提醒即独立打卡点。回复 ✅ + 时间/说明=记录为已执行；不回复=默认未执行❌。已建立cron健康提醒体系（10条分时段推送+health-tracker日志）。午餐时发菜单给我，我推荐搭配，用户会立刻按推荐执行。
§
设计交付必须出完整方案而非仅变更部分。新增/修改设计方案时，必须包含全部内容（原始已有+新增修改），不能只输出增量变更部分。
§
修改/扩展设计方案前，必须先调研现有系统（如OCR引擎、提示词、导入管线等）的实际实现，分析可复用部分和需要新建的部分，向用户汇报后再进行方案修改。
§
## 自动化任务依赖拓扑（2026-05-14建立）

### 核心枢纽：WPS云盘 (/mnt/g/WPSDocument)
单点故障路径。依赖 Windows 侧 WPS云盘客户端运行 + DrvFs 挂载正常。6个脚本依赖此路径。
- 邮件管线：foxmail-auto-export.py(Win端)→G:\WPSDocument\WPS云盘\邮件\自动导出\→email_classifier.py(WSL端)
- 知识库管线：kb_organize.py/kb_pipeline.py/kb_vectorize.py/kb_health.py 全部依赖 /mnt/g/

### C014 内容工厂工作流
周一 08:00 (109e836fd119) → 选题 → 缓存 content-factory-calendar.json
周二 08:00 (54c117b76344) → 生成三平台初稿 ← 新增（原缺失导致空跑）
周五 09:00 (5772dbd93675) → 发布提醒（已加缺稿检测逻辑）

### 健康提醒投递目标
10个 cron 投递到 feishu:🏥 健康管理群 (group)，2026-05-14 从错误 chat_id 修正

### 数学辅导管线（独立，无依赖问题）
raw-exam-parser(08:00/20:00)→daily-practice(07:00)→学习日报(07:30)
§
Hermes Agent venv (~/.hermes/hermes-agent/venv/) has Flask 3.1.3 + Werkzeug 3.1.8 but NOT flask_sqlalchemy. New Flask apps must either use native sqlite3 (db.py pattern: get_db()/dict_from_row()/dicts_from_rows()) or install flask_sqlalchemy first. Discovered 2026-05-14 when AI家庭教师's SQLAlchemy data layer had to be fully rewritten mid-project (~15 Python files, ~2.8M tokens).
§
## no_agent cron 脚本参数陷阱（2026-05-14踩坑）
cron 调度器将 script 字段整体作为文件路径查找，不支持参数（如 `"health-reminder.py 2100"` 会找不存在的文件）。修复方案：为每个参数组合创建独立的 shell wrapper 脚本（如 `health-reminder-2100.sh`），wrapper 内 `exec python3 health-reminder.py 2100`。10个健康提醒已全部用 wrapper 替换。未运行的 cron 将在下次调度时自动生效。
§
Flask E2E测试经验（2026-05-14 AI家庭教师）：
- SQLite DELETE后autoincrement不重置，需 `DELETE FROM sqlite_sequence`
- SQLite TEXT字段在Jinja2中不能 `.strftime()`（不是datetime对象）
- 子代理创建模板时命名与route不一致 → 需模板文件存在性检查
- API测试需先创建session（学生/家长登录），否则返回401/403
- 模板变量名不匹配（route传a→模板读b）不报错，仅静默空渲染
- Flask test_client使用独立session，每个test_client需独立登录
§
C005关键盲区修复（2026-05-14）：模板错误是运行时错误，compile()语法检查无法捕获。在Step 4中新增「阶段B.5: Flask模板运行时渲染验证」——用app.test_client()实际渲染每个页面，捕获Jinja2 UndefinedError/TemplateNotFound/BuildError。所有涉及Flask的Tier1+任务强制执行。这是用户「刚说完成，一点进去就发现模板错误」痛点的根因解决方案。已更新code-development-workflow SKILL.md + 约束注册表。
§
2026-05-14 C005 Step 4 加固：新增 Phase D 黄金数据测试（OCR/拍照/导入用真实数据测全链路）、Phase E 设计实现对照（逐项对比设计文档）和用户动线测试（模拟完整用户路径）。Step 4 入口门禁自检扩展至 B.6/B.5/D/E 四项，勾选后强制执行。测试数据统一存放于 ~/.hermes/test-data/，当前含 AI 家庭教师系统 10 份真实样例。约束注册表新增 C021(黄金数据测试)/C022(设计实现对照)/C023(用户动线测试)。
§
2026-05-14 交付质量逃逸分析已写入 code-development-workflow/references/delivery-quality-escapes-analysis.md。核心发现：C005 Step 4 的 B.5/B.6/C/D 验证阶段完备，但 Agent 在"快点交付"心态下跳过执行。新增 Phase D 黄金数据测试（10份真实样例在 ~/.hermes/test-data/ai-home-tutor/）。SKILL.md 超 100K 无法 patch version bump，需下次先 trim。
§
2026-05-14 skill 更新: 
1. code-development-workflow — 已固化 Phase D 黄金数据测试/Phase E 设计对照+动线测试/Step 4 入口门禁
2. references/golden-test-data-convention.md — 新文件，记录测试数据目录结构、MANIFEST.md 规范、当前已有数据集（AI家庭教师 10 份样例）
3. references/delivery-quality-escapes-analysis.md — 补充错误4（设计实现缺口）、错误5（导航断链）复盘，Phase E 修复措施，更新参考指针
4. 约束注册表 C021(黄金数据测试)/C022(设计实现对照)/C023(用户动线测试) 已就位
SKILL.md 超 100K 限制，后续新增参考内容走 references/ 目录

---
Last synced: 2026-05-14 23:00