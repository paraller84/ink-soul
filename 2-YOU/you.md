# 夜雨 (synced from user profile)

对知识库RAG系统非常有经验，熟悉7级去重规则。使用飞书→Wiki同步管道。有体系化能力注册表管理系统。
§
WSL: Ubuntu 24.04 LTS, 16GB RAM, 6核CPU, 8GB swap, NVIDIA RTX 4070 Laptop GPU(8GB VRAM, Compute 8.9), CUDA 12.6 Toolkit, nvidia-utils 590。用户 yeyu_。
§
个人信息: 42岁/上海/太平洋保险总公司数据管理部总经理助理/上海海事大学计算机本科。妻:平安健康保险科技产品负责人。子:10岁/三年级。工作黄金时间9:30-15:00。摩羯座。2006年起IT从业，2022转数据管理。个人品牌:AI应用专家(自媒体方向)。
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