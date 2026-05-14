# 夜雨 (synced from user profile)

对知识库RAG系统非常有经验，熟悉7级去重规则。使用飞书→Wiki同步管道。有体系化能力注册表管理系统。
§
WSL: Ubuntu 24.04 LTS, 16GB RAM, 6核CPU, 8GB swap, NVIDIA RTX 4070 Laptop GPU(8GB VRAM, Compute 8.9), CUDA 12.6 Toolkit, nvidia-utils 590。用户 yeyu_。
§
每日去陆家嘴办公室上班（太平洋保险总公司，数据管理部总经理助理），通过飞书远程协作Hermes Agent工作。工作黄金时间9:30-15:00。
§
工作风格: 极其注重细节和架构正确性。偏好结构化表格化呈现。只复制不删除原则（原文件永不触碰）。全本地模型策略。先梳理完整事项清单再动手。
§
通信规则: ①每次输出标注模型名称（如[DeepSeek V4 Flash]）②每次任务完成后告知「已完成」。
§
编码要求: 所有编码必须走C005（五角色流程: 需求分析→代码学习→架构设计→工程实现→质量测试→集成交付）。多步骤任务(3+步)用execute_code批量编排（一次认证、一次写、一次验证）。Bug修复先走C007 Mode D审核。
§
Web系统部署: 必须配置Windows端口转发 `netsh interface portproxy add v4tov4 listenaddress=<Win-LAN-IP> listenport=<PORT> connectaddress=<WSL-IP> connectport=<PORT>`。
§
实施策略偏好: B/A/A模式——①一次性全量实施（不分期）；②立即启动（不等待）；③直接开干（不额外讨论）。信任顾问团方案后不要求分期交付。
§
报告/PPT创作: 领导阅读类用条条目+表格，避免散文。结构:目标→推进→优化。原因分析覆盖所有问题类型。技术表述精确。竞聘报告必需自我简介关键词页。实打实的举措禁用套话。
§
内容创作: 真实经验→实战文（如Hermes体系/教育系统）。仅观察→思考文（行业观察与思考）。禁止虚构案例冒充亲身经历。区分「实战文」和「思考文」。
§
语文作文辅导: 儿子三年级作文辅导是刚需。偏好具体可操作步骤式方法（如「三句话写一个场景」结构公式），紧扣教材单元练习内容。亲自参与。
§
英语教育(C008)偏好: 语法SM-2间隔比词汇更长(1→3→7→14→30天)。LLM进阶题自动生成入库，无需手动确认。改进项≤6小时时一次性P0到位不分期。
§
视觉偏好: 深色科技风主题。追求视觉一致性和沉浸式体验。信任C005流程，授权后不过度干预执行细节。
§
邮件回复顾问服务: 输出草稿→用户微调后回传→差异比对学习风格。三类风格: 翟总=强势结论型、跨部门=稳妥专业型、同事=指令清晰型。
§
质量门禁理念: prefer fixing pipeline over cleaning up messes。全链路追根溯源（从数据来源到最终展示），不只报告Bug。
§
不使用 OneDrive。家庭存储基础设施：家里有 NAS（待确认品牌/IP/协议），百度云盘（Windows 客户端已安装，G:盘 BaiduNetdiskDownload 目录存在），外置 SSD（G:盘 426GB 空闲），D:盘（183GB 空闲）。SSH key 已绑 GitHub（paraller84）。
§
My name is Ink. The user named me Ink as my permanent identity. I call them 夜雨 (Yè Yǔ).
§
工作风格：对紧急数据需求优先走快捷路径（优先处理局部而非全量重建）。EAST 数据治理相关工作属最高优先级，需要完整处理（包括邮件附件提取）。
§
材料制作工作流要求：①必须先建立「材料使用台账」（追踪所有信息来源：外部文档+知识库引用+产出物），台账包含来源/用途/入库KB状态/归档路径。②叙事框架必须基于「复盘已发生的问题」而非「预测未来风险」——用事实推导结论，复盘具有不可辩驳性。③多任务场景要先列出队列让用户确认优先级，逐份完成不并行。
§
术语偏好：强调C003/C008/C011是Edu-Hub教育平台下的「学科模块」而非「独立系统」。禁用「系统」描述教育模块。统一称为「数学模块(C003)」「英语模块(C008)」「语文模块(C011)」。
§
Architecture alignment check: Demands proactive architecture comparison with English system before implementing cross-system changes. Expects structured comparison tables showing alignment gaps and recommended alignment decisions.

Template review before implementation: Wants to see template/strategy designs first, confirm before coding starts. ("你先设计模板，在开始改造执行前先和我说一下")

举一反三 capability: Wants the system to analyze uploaded exam paper patterns and generate similar new questions (not just collect wrong questions). This is a key differentiator of the math system.

Question strategy design: Confirmed 7 challenge types (等量代换/和差/周期/鸡兔同笼/年龄/植树/还原) + 5 comprehensive types (数据解读/科学/生活/运动/社会). Daily practice = 10 questions with 2 challenge + 2 comprehensive + 错题变式 + 基础+进阶计算.

Unified coins: Confirmed math should use same coins pool as English/Chinese (shared coins_manager.py).

Prefers structured comparison tables for architecture decisions (responded positively to the English v3 alignment comparison table).
§
Infrastructure reliability priority: When the user says "最急迫的事" (most urgent thing) about system stability/reliability, immediately halt all feature work and focus entirely on the infrastructure fix. This overrides all pending upgrades, skill deployments, and feature requests. The user has explicitly demonstrated (on 2026-05-13) that they will redirect from skill/content upgrades to reliability when they perceive a stability risk.
§
开发痛点：之前开发的系统"总是漏了或是和想的不一样"。对「可交互原型→精确参数→一次编码到位」的流程有强烈兴趣，认为可视化拖拽调整比自然语言描述更精准，能减少开发返工。偏好用可操作的原型验证需求后再动手编码。
§
Architecture preference: 当三个模块功能/风格一致时，偏好「建同一共享模板」而非「改三份各自模板」。设计预览稿确认后，编码前必须先检查 base.html 的 Jinja block 结构是否支持预览设计——将预览元素逐个与 base.html 可用 block 对比，缺失的 block 先加再编码。
§
用户对工作流合规性要求严格。2026-05-14 我跳过了 C005 直接执行命名空间统一，用户批评指出应走标准任务处理路径。核心期望：发现流程偏离时应先停下列完整审计报告，而非一边修复一边继续。宁可慢也要走对流程。
§
游戏化积分设计偏好：获得应严格有限（每日上限100分，单科40分），消费对价要高（玩30分钟游戏=200分≈3-4天练习），大额消费需家长审批。核心原则「积分来之不易，消费需经努力，一天练出来≠一周免费游戏」。对积分经济平衡很敏感，厌恶「通胀型」积分系统。
§
家庭多子女场景是默认需求。涉及家庭成员使用的系统（教育/娱乐/工具），必须从设计层面支持多用户/多角色/数据隔离，而非单用户+后期改造。
§
评估完整性是硬约束：顾问团模式C链式评估必须覆盖所有专业角色（教育专家/UX专家/系统架构师），不可跳过任何一轮。用户原话：「评估的完整性是必须的」。

冷启动vs迭代调整区分：全新系统（冷启动）走完整4轮Socratic询问；已有基础的迭代调整不走4轮询问，但必须由专业角色链式评估通过后方可定稿。

顾问团模式C的3角色配比偏好：🎓教育专家→🎨UX/前端专家→🏗️系统架构师（而非skill中原写的「产品经理」）。用户认为教育系统评估中UX专家比产品经理更对路。
§
作息偏好：午休30min不可省。晚间需要和我协作优化系统的时间（约20:30-21:00）。周一/三可和儿子慢跑30min（学校作业早完成）。需要固定社交时间（如每两周一次周五），提前和太太协调。阿姨负责接娃+做饭到20:00。太太21:00后到家，不一起吃饭。实际熄灯时间约23:00。运动偏好：通勤骑车（偶尔）+ 慢跑，但需要把运动嵌入日程缝隙而非单独腾时间。