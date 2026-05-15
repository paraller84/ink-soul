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
Edu-Hub: ~/edu-hub/app.py :5002 (统一Flask进程). 三大模块同属Edu-Hub工程: C003数学/cron驱动, C008英语/最成熟, C011语文/双分支. 术语: 称「模块」不称「系统」. 统一设计系统v1.1: 护眼配色, 共享CSS edu-design-system.css.
AI家庭教师: ~/edu-hub/ai-family-tutor/ (独立项目, 非Edu-Hub). 
飞书8部门群体系: 📊战略材料部 / 📋会议情报部 / ✍️内容市场部 / 💡战略研究部 / 🎓教育产品部 / 📬对外联络部 / 🗓️办公室 / 🔧系统运维部. FEISHU_GROUP_POLICY=open无需@.
§
=== FLYWHEEL (C类系统) ===
C012飞书待办: feishu-task-manager.py. 两组分组(👤你的/🤖我的). 子任务必须POST /tasks/{parent_guid}/subtasks (GUID在路径中), 不能用POST /tasks传parent_guid. 子任务不设members. 三层模型: 🌲全景仪表盘+🌳C012行动清单+🌟日工作建议.
C013 Get笔记: 每15分钟轮询, 4类分流(会议→待办, 阅读→wiki/raw/get-notes/, 灵感→wiki/queries/, 一般→wiki/raw/get-notes/). 说话人指认规则: 提供2-4段最具识别性段落让用户指认, 不可自行推断.
C014内容工厂: 周一选题→周二生成→周四/五发布. 话题池4分类10话题. 集成 avoid-ai-writing 审计. 话题发现机制: 周五集中批处理.
§
=== FEISHU API & RULES ===
用户open_id: ou_50b21c92548fbb2173b049e57dfbdbec, perm=full_access. 飞书Token注册表: feishu_folder_tokens.json. 三大父夹: 系统文档(LQx1fl0v8lB1XFdAI29czWeznVh), Hermes生成(Otppfr9EelPIawdezL2csUXCnoh), 内容工厂(OaHdfQM9flT1vudSxmFcHVRMnEg).
文档版本规则(不可违反): 修改已有文档时保留旧版本不动, 新建带版本号的新文件(v1→v2→v3). feishu-md-writer.py阻塞: 必须在 ~/.hermes/scripts/ 目录下运行(依赖CWD导入feishu_tokens).
文件夹创建: POST /drive/v1/files/create_folder. 上传文件: POST /drive/v1/files/upload_all (multipart). 分享: 文件夹用type=folder, 文档用type=docx, 文件用type=file. 创建文档: POST /docx/v1/documents + feishu-md-writer.py写内容.
§
=== TOKEN TRACKING ===
回溯总计: 718会话, $30.84 (V4 Flash $21.26 + Reasoner $9.58). 日常对话占58.8%($18.14). 日报告(23:55) + 周报告(周日20:00) 投递🔧系统运维部群(oc_3b372c56ef4e1c176f74bafde5484149). 异常规则: 环比>30%/单会话>500K/凌晨>100K/成本翻倍. 仪表盘: UVFgdIdB3oBkyKx39qwcxRpsnqf (Token管理体系文件夹).
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
AI家庭教师OCR管线已升级为三级提示词体系（shengzibiao/cn_recog/cishu）+ Schema校验。词库数据模型以(student_id, semester, char)为唯一键，支持upsert合并（补拼音/补组词/升格类型）。vocab_service.py新增batch_import_shengzibiao/cn_recog/cishu，routes新增semester API和vocab/by-semester查询。photo_import.html模板已创建（学生→类型→学期→拍照→预览→确认全流程）。
§
工作方式要求：必须先观察再动手——用户发来图片数据时，必须先审视排版/内容/特性，确认理解正确后再设计提示词和代码。跳过「先看」直接改代码会被纠正。流程纪律：会认字那次走的流程（审视→描述→设计→测试→反馈→调整→固化）是用户认可的，不能跳步骤。
§
memory-tencentdb Hermes 插件已安装（v0.3.5，2026-05-15）。架构: Python MemoryProvider → Node.js Gateway sidecar (port 8420) → SQLite + sqlite-vec。插件路径: ~/.hermes/hermes-agent/plugins/memory/memory_tencentdb/（软链至 ~/.memory-tencentdb/）。npm包路径: ~/.memory-tencentdb/node_modules/@tencentdb-agent-memory/memory-tencentdb/。配置: memory.provider: memory_tencentdb。LLM: DeepSeek V4 Flash（TDAI_LLM_MODEL=deepseek-chat）。L0对话搜索已测试通过；L1提取需LLM异步处理；短期Offload尚未实现（仅OpenClaw专有）。参考: agent-memory-management skill → references/memory-tencentdb-setup.md
§
用户期望的OCR图片处理流程：先批量看完全部图片→识别排版类型→总结分类给用户确认→确认后再设计提示词→最后才改代码。不要跳到代码修改前。直接用大模型视觉能力看图（Qwen-VL-OCR），不要问用户"这是什么排版"——由我自己分析后给结论让用户纠错。
§
TencentDB Agent Memory 已集成完成（2026-05-15）。memory_tencentdb Hermes 插件已安装。架构: Python MemoryProvider (auto-start) → Node.js Gateway sidecar (port 8420) → SQLite + sqlite-vec。LLM=DeepSeek V4 Flash。config.yaml memory.provider=memory_tencentdb。重启 Hermes Gateway 后生效。

---
Last synced: 2026-05-15 23:00