# Environment & Memory (synced from framework)

=== ENVIRONMENT ===
WSL Ubuntu 24.04, 16GB RAM, 6核CPU, 8GB swap. RTX 4070 8GB VRAM (Compute 8.9), CUDA 12.6. 用户 yeyu_。Ollama 0.23.3 runs on Windows side (not WSL), 21.3 tok/s, GPU enabled. DeepSeek V4 Flash主模型。本地 qwen3.5-256k:latest (6.6GB, vision/tools/thinking) + qwen3-vl:8b (6.1GB vision备用) + qwen2.5:3b (1.9GB, 132tok/s极速)。
§
=== L1模型路由规则 ===
【L1简单对话 → 本地qwen2.5:3b】当消息属于以下类型时，调用Ollama API (http://localhost:11434/api/generate, 模型qwen2.5:3b) 直接处理并返回，不经过DeepSeek：
- 晨间问候（早上好/新的一天/起床提醒）
- 确认回复（收到/好的/确认）
- 健康打卡（已跑步/已泡脚/吃饭了）
- 日程提醒回复（知道了/安排了）
- 简单询问（几点了/今天周几/天气如何）
- 任何3句话以内能说清楚的简单交互
调用方式: `curl -s http://localhost:11434/api/generate -d '{"model":"qwen2.5:3b","prompt":"...","stream":false}'`
所有需要编码、分析、多步操作、工具调用的对话，继续使用DeepSeek。
§
=== INFRASTRUCTURE ===
Hermes Gateway: systemd user service (hermes-gateway.service, Restart=always, 60s backoff). WSL systemd=true in /etc/wsl.conf. 三层可持续运行体系: L1 Registry Run key → L2 WSL cron watchdog(5min) → L3 系统级计划任务(待执行). 管理: hermes-stop.sh暂停看门狗, hermes-start.sh恢复.
WSL关键限制: systemctl --user services 不自动随WSL启动, 需Windows Scheduled Task强制启动.
§
=== CORE SYSTEMS ===
WSL-Windows 交叉诊断(SKILL): wsl-windows-interop. 2026-05-19 更新: 新增纯英文踩坑实录(CMD UTF-8→GBK编码问题)+2个可复用模板(c-drive-analyzer/c-drive-cleanup). 编码预检已作为Section 0固化第一条原则.
§
=== FLYWHEEL (C类系统) ===
C012飞书待办: feishu-task-manager.py. 两组分组(👤你的/🤖我的). 子任务必须POST /tasks/{parent_guid}/subtasks (GUID在路径中), 不能用POST /tasks传parent_guid. 子任务不设members. 三层模型: 🌲全景仪表盘+🌳C012行动清单+🌟日工作建议.
C013 Get笔记: 每15分钟轮询, 4类分流(会议→待办, 阅读→wiki/raw/get-notes/, 灵感→wiki/queries/, 一般→wiki/raw/get-notes/). 说话人指认规则: 提供2-4段最具识别性段落让用户指认, 不可自行推断.
C014内容工厂: 周一选题→周二生成→周四/五发布. 话题池4分类10话题. 集成 avoid-ai-writing 审计. 话题发现机制: 周五集中批处理.
§
内容工厂文件夹 token OaHdfQM9flT1vudSxmFcHVRMnEg 已漂移/不存在(2026-05-16返回404)。替代方案：新建子文件夹在 Hermes生成(Otppfr9EelPIawdezL2csUXCnoh)下直接创建。Hermes生成当前子结构：项目(F8apfVtErlmfEvdPWXGcF6mZn2g)、顾问团报告(IR6wfpatklstu1d2LlHcyurondb)、个人(XMCHfepAYlIN8pddOmGcxqmEnZb)、部门群产出(CdquftRpTljAIJd3tIAcSEDEnxc)、工作汇报(KzSifJdMvlQYa8dW9RRcw3PsnQ5)、日常运营(LUbmfa67lljfPCdW4uPca0Ghnqh)。
§
Token管理体系文件夹: 位于Hermes生成→项目下，token在feishu-tokens.json中。⚠️ 不要从MEMORY中读token——始终通过resolve_folder_token()从注册表获取。token可能漂移。
§
=== KEY ARCHITECTURE DECISIONS ===
- C005流程: 5角色(需求→代码学习→架构→实现→测试→交付). Step 4 含黄金数据测试+设计对照+动线测试.
- C007顾问团: 4阶段Socratic问询(目标→约束→权衡→验证). 教育模块顾问团3角色: 🎓教育→🎨UX→🏗️架构.
- 约束学习闭环: 设计缺陷→4级权重(🔧内置/🛡️门禁/📋清单/📚参考) → 写入约束注册表. 总量控制.
- 老代码治理「渐进边界法」: Tier2+模式A(先修后改), Tier1模式B(只修不动), Bug模式C(标记).
- 开发方法论: 骨架先行(先搭框架再填逻辑). 模板设计必须检查base.html的Jinja block兼容性.
- 当三模块功能一致时, 优先建共享模板而非改三份.
§
=== CRON & SCHEDULING ===
no_agent cron脚本不支持参数, 需建shell wrapper. Cron投递用chat_id格式feishu:oc_xxx. 系统脉搏(17:15)汇总8部门动态投递DM. 运营监控周报(周一06:00).
§
AI家庭教师项目 OCR 管线：`~/edu-hub/ai-family-tutor/services/ocr_service.py` 已接入阿里云百炼 Qwen-VL-OCR（$DASHSCOPE_API_KEY），识别速度从 60-120s 降至 ~1s。云端失败时回退到本地 qwen3-vl:8b。提示词设计方法论已沉淀为 skill `photo-import-prompt-design`。会认字实际有 3 种子排版（生字表两行式/单行式/识字表内联式），需用「排版适配规则」段让模型自动适配。
§
L1模型路由（2026-05-15确认）：简单对话（问候/确认/打卡/日程/简单询问/3句话以内）→ 本地qwen2.5:3b (Ollama API localhost:11434, 132tok/s免费)。调用方式: curl -s http://localhost:11434/api/generate -d '{"model":"qwen2.5:3b","prompt":"...","stream":false}'。编码/分析/多步/工具调用仍走DeepSeek。
§
AI家庭教师OCR管线已升级为三级提示词体系（shengzibiao/cn_recog/cishu）+ Schema校验。词库数据模型以(student_id, semester, char)为唯一键，支持upsert合并（补拼音/补组词/升格类型）。vocab_service.py新增batch_import_shengzibiao/cn_recog/cishu，routes新增semester API和vocab/by-semester查询。photo_import.html模板已创建（学生→类型→学期→拍照→预览→确认全流程）。2026-05-15新增三种类型入库: write_char(单字填空, 2张图16条)、word_comp(组词表格,1张图8条)、cn_words扁平(词语表,3张图265条)，全部关联3下学期学生小明。chinese_chars库25条，chinese_words库313条。
§
工作方式要求：必须先观察再动手——用户发来图片数据时，必须先审视排版/内容/特性，确认理解正确后再设计提示词和代码。跳过「先看」直接改代码会被纠正。流程纪律：会认字那次走的流程（审视→描述→设计→测试→反馈→调整→固化）是用户认可的，不能跳步骤。
§
memory-tencentdb Hermes 插件已安装（v0.3.5，2026-05-15）。架构: Python MemoryProvider → Node.js Gateway sidecar (port 8420) → SQLite + sqlite-vec。插件路径: ~/.hermes/hermes-agent/plugins/memory/memory_tencentdb/（软链至 ~/.memory-tencentdb/）。npm包路径: ~/.memory-tencentdb/node_modules/@tencentdb-agent-memory/memory-tencentdb/。配置: memory.provider: memory_tencentdb。LLM: DeepSeek V4 Flash（TDAI_LLM_MODEL=deepseek-chat）。L0对话搜索已测试通过；L1提取需LLM异步处理；短期Offload尚未实现（仅OpenClaw专有）。参考: agent-memory-management skill → references/memory-tencentdb-setup.md
§
用户期望的OCR图片处理流程：先批量看完全部图片→识别排版类型→总结分类给用户确认→确认后再设计提示词→最后才改代码。不要跳到代码修改前。直接用大模型视觉能力看图（Qwen-VL-OCR），不要问用户"这是什么排版"——由我自己分析后给结论让用户纠错。
§
TencentDB Agent Memory 已集成完成并上线（2026-05-15）。memory_tencentdb v0.3.5 Hermes 插件 + Node.js Gateway sidecar。Gateway 由 Python provider 自动发现并启动，作为 Hermes Gateway 子进程运行。L0对话搜索已验证通过。L1-L3提取依赖 DeepSeek V4 Flash。config.yaml memory.provider=memory_tencentdb。
§
photo-import-prompt-design skill updated: added write_char and word_comp content types, compact OCR output technique for large lists, session reference file session-20260515-write-char-and-word-comp.md. Process discipline strengthened with "逐张零散地问" pitfall.
§
AI家庭教师OCR管线已有7种提示词类型（cn_recog/shengzibiao/cishu/cn_write/cn_words/write_char/word_comp）。词语表分词要注意分隔符差异（空格/斜杠）。photo-import-prompt-design skill已更新流程纪律：观察先行、批量化看全部图、用户分享信息≠技术指令、"我又忘记了"模式识别。
§
=== PRACTICE ENGINE BUG FIXES (2026-05-16) ===
Fixed 3 bugs in practice_engine.py + routes/student.py:
1. `generate()` method in QuestionGenerator dropped `answer` field from template questions → added `'answer': t.get('answer', '')` to template result dict
2. `start_practice()` did not store `choices` in practice_answers → added `answer_choices` TEXT column to practice_answers table, store JSON choices via INSERT + json.dumps
3. Answer route only handled `request.get_json()` but HTML form POST sends `request.form` → added dual support with `request.is_json` check + redirect for form submissions
4. `get_current_question()` did not return `choices` → added json.loads from answer_choices column
5. `/practice/<session_id>/result` route had KeyError: 'session' → fixed to query session data directly from DB
§
用户要求AI家庭教师语文填空题采用手写输入（Canvas手写板 + 本地qwen3.5-256k vision识别），而非键盘打字。理由：三年级考试是手写而非打字，键盘输入导致练习成绩虚高。用户对"模拟真实考试场景"有明确要求。
§
Token优化三件事已执行(2026-05-16): ①SOUL.md 7012→540字符(-92%) ②压缩阈值0.25→0.10, target_ratio 0.20→0.15, protect_last_n 20→15 ③Token Savior MCP 53→15个工具暴露。均需新会话(/new)生效。
§
用户工作方式（2026-05-16 手写原型验证后确认）：「先出可交互原型→用户上手测试→收集修改意见→再做最终版本」。不要替用户决定交互方式。手写板的设计经历了「单Canvas整句→逐字格子」的翻转，这是用户指出问题后才改的。云端vs本地模型的选择也以用户实际测试判断为准（用户试了本地后说「不够准确」，切回云端）。
§
DASHSCOPE_API_KEY (阿里云百炼) 已写入 ~/.bashrc 第123行，用于 AI家庭教师 OCR 管线。默认家长PIN: 123456。API Key 值已知且在系统环境中。
§
2026-05-16 高频重复提示词固化成果：①C005 Step 0-A新增3项前置预检（全量清单/需求收敛/先观察再动手升维）②新建 visual-style-registry skill 统一管理3种视觉风格（暗色科技/护眼教育/清透蓝白）③design-audit-checklist.md 从design-document-production抽取为独立审计规范文件，可跨skill引用。
§
AI家庭教师项目数据架构已沉淀为 skill `ai-family-tutor-data-architecture` — 含 unit/lesson 三级组织、scope筛选、SQLite迁移策略。`chinese-content-architecture` 纯属 Edu-Hub C011 体系，两系统不互通。curl 调用含中文参数（如 `3下`）需 URL 编码为 `3%E4%B8%8B`，否则 Flask 返回 404。
§
🔴 强制性规则（不可违反）：所有系统/模块开发优化过程中产生的设计文档、方案文档、测试文档、审计报告等，必须在生成后：
1. 上传到飞书云盘对应项目/模块的文件夹下
2. 将文档的打开链接（飞书文档链接）发送到对话框中
两步缺一不可。不允许停留在本地、不允许只发路径不发链接、不允许只上传不发消息。违反此规则需要主动道歉并补做。
§
AI家庭教师文件夹Token漂移记录：旧token ByLufjLtbltcLfdSSDCcA5nXnhe（无效），实际有效token为 CQlaf9ZA9lNFhodeGP4cTm8wnoh。需更新feishu-tokens.json中的注册记录。
§
AI家庭教师文件夹token: H04qfCSORlT7FFdNQBdcDANwn8d 经验证有效(2026-05-17, upload_all API ✅)。上传用 POST /open-apis/drive/v1/files/upload_all, multipart/form-data(5字段: file_name/parent_type=explorer/parent_node=<token>/size/file)。文件URL格式: https://nofile.feishu.cn/file/{file_token}。
§
🔴 表设计文档质量标准（强制）：所有涉及数据库表设计的文档，必须包含：
1. 完整的字段定义（字段名、类型、长度、主键/外键、约束条件、默认值）
2. 每张表的中文定义/说明（表用途、核心字段含义）
3. 表之间的 ER 关系图（实体关系说明，用 Mermaid ER 图或 ASCII 关系图呈现）
三者缺一不可。适用于所有系统/模块的数据层设计文档。
§
🎓教育产品部飞书群chat_id: oc_e9f5d48f82f74064f2526e271759235d。AI家庭教师项目在飞书的文件夹token: H04qfCSORlT7FFdNQBdcDANwn8d (位于Hermes生成→项目→AI家庭教师)。
§
AI家庭教师数据架构v4.0核心设计决策：四层体系(L1内容源→L2分析→L3题库→L4计划+执行)，LLM驱动出题(系统组装7模块上下文包，LLM是出题引擎)，Agent/App职责分离(Agent预生成+校准，App筛选+记录+展示)，补做追踪(通过practice_schedule_questions.status=pending实现)。exam_analysis替代exam_models(存储描述性分析而非结构化规则)。question_bank双轨难度(assigned vs effective)。
§
=== AI家庭教师题库设计 ===
出题机制：LLM(DeepSeek)是出题引擎，系统是上下文策展人。系统组装7模块上下文包(试卷分析+真题样例+课本内容+学生画像+出题指令+输出Schema)给LLM，LLM基于此出题。exam_analysis存储AI试卷分析结果(JSON结构：题型分布/知识点覆盖/难度评估/出题规律描述/代表性样本)，作为LLM出题的原材料。question_bank是缓存+统计追踪器，非全量题库。双轨难度制：difficulty_assigned(LLM初始) vs difficulty_effective(基于答题记录校准)。有效难度公式: 1+(1-correct_rate)×4，答题≥3次启用。
三层架构：题库(question_bank,所有题的池子)→练习库(practice_library,预打包的完整练习内容)→练习计划(practice_schedule,哪天做什么)。不是临时凑题，是预生成好的完整内容包。应用只读不生成。
补做判断：整份练习整体判定(非逐题追踪)。practice_schedule.status三态流转: assigned→in_progress→completed。status≠completed且日期≤今天即需补做。practice_schedule_questions为纯映射表(无逐题状态)。
Agent/App分工：Agent写L1内容源+L2题库+L3计划，App只读+写practice_records+改schedule.status。
§
AI家庭教师题库设计v5.0: 三层体系(L1内容源→L2题库→L3练习库→L4计划+执行)，LLM驱动出题(系统组装7模块上下文包，LLM是出题引擎)，Agent/App职责分离(Agent预生成+打包+校准，App只读+记录+展示)，补做追踪(通过practice_schedule.status整体判定而非逐题追踪)，exam_analysis替代exam_models(存储描述性分析而非结构化规则)，question_bank双轨难度(assigned vs effective)。每日练习15题，单元测试覆盖全部生词。practice_library是预制好的完整练习内容(不是临时凑)，practice_schedule只存library_id+date+status。
§
=== 出题策略确认决议 (2026-05-16) ===
三源比例：当期60% / 往期动态(0-20%按充实度) / 跨期动态(20%-20%自动兜底)。每日题型：核心12题(看拼音写词语6-7+会认字选读音5-6) + 综合3题(每天1种按周轮换：句子/课文/组词/古诗/阅读)。时间范围：固定窗口法[C-1,C,C+1]三课，家长设当前课号。难度自适应：4项指标(完成率/时间/准确率/核心题型正确率)→5档调节(大幅下调/下调/保持/上调/大幅上调)，Agent每周自动判定。考点避重：7天内同字同题型不重复，coverage字段分题型追踪。可读→可写：年级跨度自然完成转换，过往学期字词默认以会写标准出题，不依赖标签。批次：启动3天45题→每周7天105题。Agent管线：7模块上下文包(出题指令+课本字词+往期候选+跨期候选+避重清单+薄弱清单+输出Schema)→调DeepSeek→校验→入库→通知审核。
§
用户提出的知识三层可信度模型（设计原则）：L1-已验证区（实践验证/用户纠正，可信度0.8-1.0，污染风险极低）→ L2-待验证区（文档/被动接收，0.4-0.7，中风险）→ L3-暂存区（外部输入/未验证，<0.4，高风险）。任何新知识必须经历"暂存→验证→晋级"的生命周期，外部输入不可直接写入核心知识库。
§
用户强调决策过程的可追溯性：AI的每一步处理（分类→密度检查→置信度门控→检索→推理→用户反馈）都需要输出可读的决策日志，便于精确定位错误环节（"是分类的问题、处理的问题、还是RAG置信度策略的问题"）。不可提供笼统的解释。
§
用户对所有改动持"可回滚"原则：新功能必须支持双轨并行（金丝雀发布），A轨(旧模式)全程保留，B轨(新模式)可选择流量比例。任何改动都必须能通过配置开关一键回滚，不可有不可逆的系统变更。风险厌恶型部署风格。
§
🔴 输出格式规则（2026-05-17 升级为强制规则）：所有生成的文档/方案/报告/文件统一使用 HTML 格式（暗色科技风主题），以信息图/结构化卡片/代码框等形式呈现，方便人脑快速理解和处理。禁止纯文字 Markdown 表格。此规则适用于所有场景，无需每次重申。
§
飞书云盘目录重组已完成（2026-05-16）: ①创建🧠顾问团目录(RRkofnlQOlnN3VdrZ7mcAra4nYd)于部门群产出下，合并原两处顾问团报告（11+13=24项）②创建能力体系目录(HA85fFkMQlwPijdIBxccQDQQnxf)于系统文档下，迁移C系列等18个散落文档③儿子教育迁移至个人/儿子教育/④系统文档顶层0散落文件。注册表已更新。skill feishu-drive-directory-standard已固化完整路由规则。
§
设计方案标准化作业模板 v1.0: 基于AI家庭教师系统优化全过程的8项设计交付物归并为6种类型(A系统蓝图/B数据结构/C业务规则/D交互原型/E系统架构/F测试方案)，每类含必须内容清单/格式要求/常见陷阱/实例。通用要求含元信息头部/格式规范/设计决策可追溯/变更轨迹。交付5步闭环纪律。飞书文件 token: DRhJbN3yYoBCrexb1UWcClzwnbe
§
家长中心优化方案 v2 已完成（2026-05-16），文档已上传至飞书云盘 AI家庭教师文件夹（H04qfCSORlT7FFdNQBdcDANwn8d），file_token=BUFJb8LqyoG7uFxtyp4cum8cnCb。
§
设计方案标准化作业模板维护规则：用户在后续对话中对模板内容提出的任何补充调整，必须同步更新到模板文件本体（飞书云盘中的标准化作业模板_v1.0.html），保持模板始终与最新设计规范一致。不得只做当前交付而不回写模板。
§
用户微信公众号 appid: wx488d661c2f4cf358。用户更偏好 API 方案（appid+appsecret）而非浏览器 Cookie 导出——API 无需浏览器、无需持续维护。用户浏览器 F12 快捷键无效（可能是 360/QQ 等国产浏览器或键盘冲突），未来涉及浏览器操作时优先提供菜单路径或插件方案。
§
AI家庭教师家长中心v6重构完成(2026-05-17): 6入口仪表盘(3公共+3科目), chinese Blueprint(/parent/chinese/<id>/) 含7个子页面(仪表盘/内容管理/题库审查/练习卷/练习计划/出题设置/学习报告), 科目入口先选学生后进入, 旧路由全兼容. 重点: 练习计划日历使用42格网格, 题库支持4维筛选+分页, 内容管理使用标签页+AJAX概览API.
§
家长中心测试套件：tests/conftest.py（临时数据库fixture）+ tests/test_parent_center.py（47项测试，覆盖仪表盘/语文蓝图/练习卷/题库/积分/报告/旧路由兼容）。create_app() 现在接受可选的 instance_path 参数用于测试。启动：cd edu-hub/ai-family-tutor && /home/openclaw/.hermes/hermes-agent/venv/bin/python -m pytest tests/ -v。
§
Jimeng CLI (dreamina_cli) 已安装: /home/openclaw/.local/bin/dreamina，通过 curl -fsSL https://jimeng.jianying.com/cli | bash 安装。支持 headless 登录 (dreamina login --headless → 用户浏览器访问 verification_uri 确认 → dreamina login checklogin --device_code=xxx --poll=30)。核心命令: text2image, text2video, image2image, image2video, query_result, list_task, user_credit。
§
内容工厂系列文章主题确认：围绕"从录音(Get笔记)+邮件到任务闭环"的整套体系，含"我关注/他关注/场景切分"三层分类。用户认为这套闭环比单点方法论更值得对外分享。三平台分发：微信/知乎/小红书。这是用户的个人品牌IP方向。
§
Jimeng AI CLI (dreamina) 登录架构：OAuth token 持久化，web端用 window.open() 弹窗登录（无头浏览器不可用），CLI headless 登录可用。当前账号 standard 版（user_id 332524177596025, 1168积分），需 Maestro VIP 才能用 dreamina text2image。升级入口：jimeng.jianying.com 网页端登录后会员中心。不要尝试通过浏览器自动化登录即梦AI。
§
read_file() 输出带 `NUM|` 行号前缀（如 `1|import json`），不能直接 pipe 到 write_file()。write_file 写入的是人可读的渲染文本，不是纯代码内容。如需复制文件内容进行编辑，用 `terminal("cat path")` 获取原始内容，或用 `open().read()` 在 Python 中读取。
§
=== CRON NOTIFICATION GOVERNANCE (2026-05-17) ===
三阶通知策略: 静默(local,纯后台)/决策触发(含空跑判定)/决策必需(每次推送)。异常告警: no_agent cron-anomaly-monitor.py 每5分钟扫描jobs.json, 对比anomaly-state.json免重复, 异常即时推送🔧系统运维部群。空跑静默: no_agent脚本空stdout不投递。多渠道分发: deliver支持"origin,feishu:oc_xxx" fan-out语法。每日运营报告: 合并到17:15系统脉搏, fan-out到DM+🔧系统运维部群。已迁静默任务: 数学辅导-每日练习/原始题解析, 英语辅导-内容解析, AI家庭教师-周出题, 学习日报。
§
Notification strategy preferences (2026-05-17 established): 
① 空跑任务不通知 — 无有效产出时不发任何消息；
② 仅需用户决策/影响行为判断的任务才通知到工作群，纯后台任务全静默；
③ 系统运维部每日17:15接收自动任务运营报告（与系统脉搏合并投递），异常情况即时（每5分钟）告警。异常监测用no_agent脚本+状态文件实现，0 token成本。

Task lifecycle philosophy: 长时间空跑的任务直接清除（remove非pause），不保留空转。可接受对同一管线拆分双频调度（工作日/非工作日不同频次）。对无使用预期的同步管线主张直接删除。
§
🔴 飞书上传+发链强约束（2026-05-17 固化）：完成任何文档/HTML/方案生成后，两步缺一不可：①调用 upload_all API 上传到飞书对应文件夹 ②发送可访问链接到当前对话。此约束已固化到 advisory-council-workflow 技能（"交付铁律"节）和 design-document-production 技能（"交付执行纪律"节）。违反需主动道歉补做。设计-document 生产规则
§
物理零件图纸设计规则 (2026-05-17 用户纠正): 用户要"分别的示意图"(每个零件单独画的独立三视图, 含完整尺寸标注) 而非"组装在一起的示意图"(总装图显示装配关系)。已在 design-document-production skill 新增 Type G 物理/机械零件图纸文档类型, 含三视图标准(SVG)、颜色方案、陷阱清单。默认规则: 除非用户明确说要总装图, 优先输出分别的零件图。
§
质量门禁体系已固化到 skill（2026-05-17）: codex skill 新增 3-phase Hermes→Codex 委派工作流 + 引用文件。product-quality-improvement-pipeline 新增 6维D1-D6门禁定义、A/B/C/D分类、缺陷模式注册表、quality_gate.py 脚本模板。两份方案文档已上传至系统文档/能力体系。
§
C005 工作流 v4.3.0 变更（2026-05-17）: Step 0-A 新增❹数据契约先行（新开发/接口新增时强制预定义入参出参Schema）；Step 4 阶段C质量门禁升级为D1-D6 6维体系，脚本 quality_gate.py，新增 Hermes+Codex 双模式路由规则（含修复循环≤2轮）；quality-gate-integration.md 参考文件同步更新。
§
OpenCode 模型验证架构：模型名在硬编码扁平列表中（如 deepseek-v4-flash、gpt-4o-mini），但 per-provider 无绑定关系。`-m provider/model` 格式强制但无模型注册于任何 provider 下。跳过验证的方案是设置 `OPENAI_API_KEY` + `OPENAI_BASE_URL` 环境变量。control_account 的 url 字段可存凭证但无法单独绕过模型名验证。

ds-proxy.py 关键修复：TARGET 不能带 /v1 后缀（self.path 已经含 /v1/ 前缀），否则双重重影 404；必须分开定义 do_GET（返回 /v1/models）和 do_POST；do_GET = do_POST 类属性赋值会覆盖已定义的 do_GET 方法；需 SO_REUSEADDR 避免 TIME_WAIT 端口占用。
§
飞书上传经验（2026-05-18 纠正）：此前记录的"所有子文件夹Token均返回404"是误报，实际原因是用了错误API端点。正确端点是 `GET /open-apis/drive/v1/files?folder_token={token}` 而非 `GET .../files/{token}/children`。Token本身稳定，工作汇报(KzSifJdMvlQYa8dW9RRcw3PsnQ5)等所有Hermes生成子目录均有效。上传用 POST /open-apis/drive/v1/files/upload_all, multipart/form-data(5字段: file_name/parent_type=explorer/parent_node=<token>/size/file)。文件URL格式: https://nofile.feishu.cn/file/{file_token}。
§
Hermes + delegate_task 交付质量体系已落地（2026-05-17）: 核心工作流为 Hermes(架构+审计) → delegate_task/sub-agent(编码) → quality_gate.py(6维门禁D1-D6) → A类阻断修复(≤2轮)。实战验证：AI家庭教师项目4个A类问题清零（2个真修复+2个门禁误报修复），总扫描从426降到360项。C005 skill已更新v4.3.x集成。
§
AI家庭教师全链路测试（2026-05-17）发现并修复3个问题：
1. history路由500错误 — 模板{{s._accuracy}}引用未计算字段，在routes/student.py原SQL查询后补充循环计算_accuracy = correct/total*100
2. 历史数据孤立引用 — practice_records中2条旧系统遗留记录（schedule_id=1指向不存在schedule），已DELETE清理
3. 全链路测试通过率从16/101提升至101/101(100%)
关键发现：auth路由POST / 用于学生登录(student_login)，history日志级别session['student_id']需先正确设置
§
Full-link testing of Flask web apps methodology: use curl with cookie jar for session-based testing; organize into 6 phases (page rendering → core flow → data management → admin pages → DB consistency → edge cases); always check content keywords not just HTTP status; handle blueprint URL prefixes with dynamic path params; check for orphan DB records after schema migration. This methodology is captured in delivery-conformance-audit skill Phase 2C + references/curl-full-link-testing.md.

Cron time collision analysis technique: group cron jobs by exact minute, classify as LLM/script/reminder, assign risk level (3+ LLM=P0, 2 LLM=P1, mixed=P2). Discovered 34 cron jobs with 3 collision windows covering 9 tasks. Captured in system-health-audit skill Section 4B.
§
question_bank 表新增 lesson_number 列（2026-05-17），已为第5、6单元（Library #3/#5）的85题回填课号。来源显示规则：practice页 q-badge 旁以「第X课」标签展示每题的课本来源。
§
C005 v4.4.0 + subagent-driven-development v1.2.0 多Agent分离协议已固化（2026-05-17）:
- 四角色分离：设计者(Hermes) → 编码者(SubAgent) → 审核者(SubAgent/不知情) → 决策者(Hermes)
- 审核SubAgent只持验收条件清单+代码文件，禁止看设计文档全文
- 2轮升级规则：任何审核FAIL最多2轮修正，第3轮仍FAIL升级到Hermes决策
- 验收条件清单是编码SubAgent和审核SubAgent之间的唯一通信协议
§
Codex CLI v0.130.0 在 WSL Ubuntu 24.04 上有严重运行时问题（2026-05-17 实测）: Reconnecting errors（疑似sandbox初始化失败）、bwrap缺失、stdin捕获阻塞pipe调用、模型自动选gpt-5.5非gpt-4o、首次exec超时60s。不推荐作为WSL日常编码SubAgent。替代方案：Aider (`pip install aider-chat`)，无sandbox依赖、支持DeepSeek/local Ollama、响应即时。codex skill已记录WSL pitfalls并推荐Aider。
§
Aider 0.86.2 安装于 ~/.aider-venv，DeepSeek V4 Flash 模型配置完成。使用 ~/.local/bin/aider-wrapper 封装调用。非TTY模式下创建文件必须在命令行显式指定文件名。已知 pitfalls：非TTY模式下 SEARCH/REPLACE 文件名解析可能产生 artifact（如 "Now produce SEARCH/REPLACE blocks.xxx"），需手动重命名；--map-refactor 未支持。skill: aider-coding 已创建。
§
AI家庭教师 textbook_chars/textbook_words 表通过 lesson_id (FK→textbook_lessons.id) 关联课号，非直接 lesson_number。char_type 值为 'recog'/'write'（不是 'recognize'）。查询 lesson_number 必须先 JOIN textbook_lessons。此为高频 SQL 陷阱。
§
晨间简报 cron (8114210bdc6c) 已从08:00改为09:00工作日运行，模板含今日待办+计划外登记区，支持用户白天填写完成情况，夜间读取更新注册表。
§
阵地回复终审任务状态：转办营运和科创中心。营运已上线未办结(准备回复长图,我方需补充内容)；科创中心未排期(我方已提供方案)。
§
会议情报部群chat_id: oc_59e724c052b3c57d77946d81db82476e。将进度汇报苏总的邮件拟稿交办给了会议情报部完善。
§
2026-05-18: 为老苏（苏金华总，EAST数据治理工作组组长）拟了阶段性情况汇报，涵盖5/7手续费率排查会、5/9周例会、5/9产险质量提升会、5/9五报四/数据安全/保险数据报送会共4场专题会23项重点交代事项的完成情况。文档上传至飞书云盘 Hermes生成文件/工作汇报 文件夹。
§
健康提醒已升级为智能静默机制(2026-05-18): health-reminder.py 加载 daily-checklist.json 前置检查，对应事项已完成→空输出→不推送。mapping: 1200→lunch-photo, 1230→noon-walk, 1700→wrap-up, 1945→arrive-home, 1955→run, 2100→couple-talk, 2230→foot-soak, 2300→lights-out。0600起床项无映射始终推送。对话中告知完成→我调用health-reminder.py mark <time_key>标记，到点自动静默。state文件每日自动重置。
§
编码策略：所有编码任务（特征开发、重构、多文件修改）必须使用 Aider（aider-wrapper）通过 delegate_task 委派执行。直接自己写代码不符合用户期望的工作流。Aider 安装路径：~/.aider-venv/bin/aider，wrapper 在 ~/.local/bin/aider-wrapper。
§
群组角色自报家门协议 v1.0（2026-05-18 用户要求建立）：每次在任何群处理任务时，必须：
1. 声明参与角色（如【🎓 教育产品部 · 角色：数学家教 + 出题官】）
2. 角色配置预匹配该群工作划分（8群各有预设角色包）
3. 若任务不属本群范围，提醒移交对应群
4. 若任务属本群则精细化选择子角色执行
5. 根据实际需求建议是否增补常驻角色
DM为总指挥部，负责跨群调度决策。飞书文档已上传至Hermes生成根目录。
§
群组角色复盘机制已建立：每月1日09:00自动执行cron复盘（job_id: 67d096d2bb7d），按D1-D4四维度评估各群角色配置合理性。skill: group-role-review。还设有一条触发规则：同一群连续3次越界提醒移交时主动触发临时复盘。
§
原型交互反馈模式：用户一次性给出多条具体反馈（指向文件名/功能名称），需全部处理完再统一回复，不可逐条确认。用户用英文 ID 指代原型页面（如 "exams" "question_bank" "practice_volumes"）。"补齐"指令默认覆盖全量需求页面，不可只补 P0。
§
AI家庭教师 clear-today-records 功能: 学生首页有"🗑️ 清除今日记录"按钮, 路由 @student_bp.route('/practice/clear-today') in routes/student.py:766-831. 功能: ①删除 practice_records ②重置 practice_schedule 状态为assigned ③删除 wrong_questions ④清理旧 practice_sessions + practice_answers. 可发现索引加在 routes/student.py 顶部注释块.
§
AI家庭教师质量门禁体系完善(2026-05-19): ① services/question_quality.py — validate_pinyin_format(->valid+errors+warnings三元组), check_data_contract(全题型), validate_question_bank(170题全通过), check_db_integrity ② exam_service.py _sync_to_question_bank()集成非阻断校验 ③ scripts/validate_question_data.py 每日04:00静默运行(5d086ebbf4cc, no_agent, deliver=local) ④ 轻声(中性调)合法不阻断,仅提醒 ⑤ routes/student.py顶部功能索引含clear-today
§
Hermes Agent 已升级到 v0.14.0（2026.5.16），git pull fast-forward 从 fef1a4124 到 378bca1d2。config v22→v23 迁移完成，memory_tencentdb 兼容。web_search provider 已切换为 ddgs（DuckDuckGo 免费搜索），零 API Key 运行正常。
§
=== 任务分解核心策略 ===
用户要求：复杂任务做分解时，必须：
1. 识别**关键路径**——没有这个环节整个系统就无法运作的骨架
2. 先做**最小闭环**——沿关键路径打通一个完整端到端链路（如教育系统：题入库→学生做题→出结果）
3. **做透再扩展**——这个闭环要做到扎实可用后，再分层添加其他场景
4. 避免"一块一块平行切割"——那会导致改左坏右、缺乏连贯性
5. 正确做法是**切片而非切块**：切片是沿关键路径纵向切出一个完整功能薄片，切块是横向切出互不关联的碎片
用户举例如教育系统，优先路径是：题入库 → 学生读题做题 → 生成结果。这条路通了，再扩展其他功能。
§
工作群路由规则（2026-05-19）：🗓️ 办公室群仅处理办公室职责内事务（日程/健康/作息/待办登记）。所有工作任务必须按8部门体系路由到对应部门群处理，不得在办公室群直接处理跨部门工作事项。路由优先级：凡涉及对外沟通→📬对外联络部，材料制作→📊战略材料部，会议纪要→📋会议情报部，教育相关→🎓教育产品部，其他按对应职责分配。
§
重点关注任务筛选规则（2026-05-19 用户确认）："重点关注"仅纳入需要用户亲自行动/决策的事项。凡已进入等待执行阶段（由他人推进、等待最终执行）的事项，移出重点关注转为后台跟踪。用户仅需发邮件催办的任务归类为"发出即可"而非"重点关注"；真正的重点是对反馈汇总后向领导汇报。
§
EAST项目关键事件关联：张燕事件=费用率问题（高薪合议现场问询），产品规范问题也是由张燕事件引发（同一次问询中暴露）。两者同一事件根源不是两件事。北京分公司监管通报：京金发[2026]-33号（2026年4月2日收到），点名两个问题——保单销售人员关联表79.6%销售人员编号无法关联、农险赔案扩展信息表78.25%产险产品类型未按枚举值填报，要求5月底前重报。重复补报痛点：月月重复补报同一类问题，历史错误累计，人力消耗大。
§
飞书文件上传（upload_all）：先拿 tenant_access_token（FEISHU_APP_ID + FEISHU_APP_SECRET 凭据），再 POST multipart/form-data（5字段）到 /open-apis/drive/v1/files/upload_all。File URL: https://nofile.feishu.cn/file/{file_token}。上传后必在对话中发链接。
§
2026-05-19 微信汇报稿（苏金华总）最终版已发送并归档。内容覆盖4月手续费异常清零、保批单费率不一致（科创中心5/16反馈系统规则）、保险责任终止后批单（外围保单）、险种归类不匹配（已排期6月修正）、审计问题（科创中心5/18分类，三类原因）、科技资源（内部到岗2人+外包立项中）、保信采购（集团统筹方案，5/21交流会）。归档飞书文件 token: F3QebhlwToApn8xKKBlc2i4anCe。
§
=== CRON MERGER (2026-05-20) ===
C012 待办回顾(07:30)+晨间简报(09:00)+晚间复盘(18:30)已合并为单一「晨间全景」09:00 cron。原待办回顾脚本 daily-task-review.py 保留，通过 --output-json 模式供新 prompt 内嵌调用。合并理由：三 cron 信息流呈时序衔接关系（昨日完成→今日待办→全天节奏），合并为一次推送减少碎片。feishu-task-management-system 技能 v4.2.0 已记录完整模板。合并设计模式已写入 references/cron-merger-design-pattern.md。
§
编码铁律（每次必犯，需前置预检）：为 Windows 生成 .bat/.cmd/.ps1 脚本时，write_file 默认 UTF-8+LF 会在 CMD (GBK) 中导致中文乱码和命令解析错乱。对策：全量纯英文/纯 ASCII，零中文。若必须含中文，用 chcp 65001 切换代码页 + 确认目标 CMD 能显示。写文件后用 `file` 验证编码 + 换行符（CRLF for CMD）。每次脚本类任务的第一步就做编码决策，不可跳过。
§
邮件分类体系设计方案（2026-05-20 已定稿，待实施）：13条规则决策树（自上而下优先匹配）+ 双层架构（Layer1结构化标记+Layer2价值判定入KB）。身份定义：我=叶郁/yeyu-004, 直属领导=苏金华/翟玉婷, 部门内部=张照宇/孙蒨茹/卢新永/曹爽/王佳怡/庄经纬/李显翠/张子越/兰轶东/李辉/轩鹏超。价值判定：发件人是我/发件人是领导/仅我一人收件/我是多收件人/部门内部结论性/抄送有领导在场/抄送无领导→入KB; 系统发件人/大群发≥20人/会议通知(抄送或收件)/部门内部过程性/其余兜底→不入KB; 会议通知→不入KB仅本地记会议时间表。附件处理→只存文件名列表不向量化。实施策略：Phase 1先入库不做后续处理，Phase 2再设计搜寻使用。用户偏好"先形成完整方案确认后再修改实施"。
§
邮件处理体系 Phase 1 完整方案 v1.3.0（2026-05-20确认）：
流水线：.eml→13条规则标记→附件处理(文件名索引+全量向量化)→价值判定→生成邮件概况→发🗓️办公室群→你确认→按分类入库。
用户身份：我=叶郁/yeyu-004，领导=苏金华/翟玉婷，部门内部=张照宇/孙蒨茹/卢新永/曹爽/王佳怡/庄经纬/李显翠/张子越/兰轶东/李辉/轩鹏超。
13条规则：①系统发件人❌不入KB ②发件人是我✅入KB(replied) ③领导发件✅入KB ④大群发≥20人❌不入KB ⑤⑥会议通知(抄送/收件)❌不入KB+记时间表 ⑦⑧部门内部(收件/抄送)❌不入KB ⑨仅我收件✅入KB ⑩多收件人✅入KB ⑪抄送有领导✅入KB ⑫抄送无领导✅入KB ⑬其余兜底❌不入KB。
附件：文件名→元数据(钩子) + 内容→向量化，双向勾兑。
展示格式：🔥需重点关注(详版含📝摘要+💡建议) / 🟡值得关注(简版) / ℹ️已回复(简版) / 📅会议记录(仅时间表)。发🗓️办公室群，你确认后入库。
§
邮件处理体系 Phase 2 已确认（2026-05-20）。任务A：入库后自动查KB需回复邮件(规则③⑨⑩⑪⑫)，按三风格矩阵匹配风格，发任务到📬对外联络部群。任务B：每周查KB中is_from_me邮件，分析风格，更新style_profile.yaml。三风格矩阵基准：强势结论型(上级)/稳妥专业型(跨部门)/指令清晰型(同事)。style_profile.yaml位于email-to-knowledge-pipeline skill的references/下，每周迭代只追加不删除基准，初始含7条全局规范。
§
设计一致性审计三阶门禁：①下载飞书原型HTML（Drive API download端点，非Web包装器）→ ②创建逐页对照矩阵（页面/功能/数据/交互/视觉5维）→ ③全体✅后才可宣告"已完成"。严禁仅凭代码静态分析或修改1-2页就宣告审计完成。违反后果：被用户指出重做、失去信任。
§
关键纠正（2026-05-20）：做设计对照审计时，必须先下载/加载实际的设计原型文件（飞书HTML），逐页比对，不能仅靠静态代码分析判断"正确性"。此前有连续多次"已完成"误报（5/18原型缺页面+混入无关内容 → 5/20说dashboard已改完但其他页面未对照原型），根因是未加载完整原型做验收。
§
邮件处理系统已全面实施完成（2026-05-20）。email_classifier.py 已重构为13条规则决策树+价值判定+generate-overview+deliver-overview子命令，全部规则引擎无需LLM，内容提炼用DeepSeek V4 Flash而非Ollama。旧30min cron已移除，新日cron(13a5e17d0f4c)工作日08:30执行扫描+投递概况到🗓️办公室群。Phase2脚本已就绪(phase2_reply_dispatch.py/phase2_style_learn.py)，风格学习每周一12:00(cron ccc8d00ca11d)。用户确认概况后，我手动触发phase2_reply_dispatch.py分发到📬对外联络部群。
§
2026-05-20: 家长端配色改为「清爽蓝天」浅色方案（方案A）。页面底色 #f0f7ff，卡片底色 #ffffff，边框 #dce8f5，主色调 #0284c7 天空蓝，文字色 #1e293b。通过 parent-theme.css（!important 覆盖）+ body_class=parent-theme 实现，不干扰学生端。
§
邮件系统token优化已完成(2026-05-20)。Phase 2分发已去除LLM确认+要点提取(改为规则化提取)，概况生成已去除DeepSeek摘要。日常运行(扫描→概况→分发)LLM token消耗降至0，仅每周风格学习1次DeepSeek调用。scan/30min cron和deliver-overview均为纯规则引擎。
§
🔴 read_file() 输出带行号前缀（如 "1|import json"），不能直接用 write_file() 回写。write_file 写入的是人可读的渲染文本。如需复制文件内容进行编辑，用 terminal("cat path") 获取原始内容，或用 open().read() 在 Python 中读取。2026-05-20 因犯此错导致6个模板被行号污染。
§
C005 v5.2.0 监督体系强化（2026-05-20）：Step 0-D 新增Aider编码强制自检（硬性拦截规则，Hermes不可直接编码，违规需回滚重做）；Step 5.0 新增交付前独立SubAgent审计（不可跳过，审计SubAgent不知情，凭原型HTML+代码做对照，PASS才可交付）。delivery-conformance-audit v1.3.0 增强：Phase 1 原型下载改为强制前置门禁（没有原型文件=审计结论无效）。每日监督审计cron（b175c2a8c627，工作日06:00）独立运行，投递到🎓教育产品部群。
§
=== Token 主动提醒规则（2026-05-20 用户确认）===
1. 对话上下文堆积提醒：当我检测到当前会话已较长时间/较深交互时，主动建议 `/new` 重置上下文，避免Token浪费在历史堆积上。典型触发时机：连续5轮以上深入交互、或即将开始新话题时。
2. 大HTML输出改为飞书上传发链接：当输出内容>20K字符（特别是HTML报告/审计/方案）时，优先写本地文件→上传飞书对应文件夹→在对话中发链接，不再贴全文回显。这既省Token又避免下次对话把大HTML加载进上下文。
3. 用户说"继续XX"时：主动问"要不要新开会话，避免上次的上下文堆积？"

---
Last synced: 2026-05-20 23:00