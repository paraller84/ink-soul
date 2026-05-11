# 技术环境

## WSL (主工作环境)
- Ubuntu 24.04 LTS
- 16GB RAM，6 核 CPU，8GB swap
- NVIDIA RTX 4070 Laptop GPU (8GB VRAM, Compute 8.9)
- CUDA 12.6 Toolkit, nvidia-utils 590
- nvidia-smi 正常但 /dev/nvidia* 可能不存在
- IP: 192.168.3.102/24 (WSL mirrored networking)
- Windows 主机: 192.168.3.x 网段

## Windows 主机
- 用户名：yeyu_
- D: 盘：183GB 空闲（本地数据盘）
- G: 盘：426GB 空闲（外置 SSD，含 Ollama 模型 26GB、知识库文档 8GB）
- 百度云盘客户端已安装（G: 盘 BaiduNetdiskDownload 目录）
- NAS：家庭网络中有 NAS，详情待确认
- 端口转发：netsh interface portproxy 管理

## AI 模型
- 主力模型：DeepSeek V4 Flash
- V4 系列原生 1M 上下文
- 全本地模型策略（不依赖云端 API）

## 知识库
- ChromaDB: ~/.hermes_tools/knowledge_base/vector_store/chroma.sqlite3 (435MB)
- 7 级去重
- Wiki: ~/wiki/guides/kb-portal.md
- kb-wiki-sync.py → raw/knowledge-base/（1166页），cron 23:00
- feishu-wiki-sync.py 双目录（LLM讨论 + 经验沉淀），cron 9AM

## 飞书
- 系统文档夹：LQx1fl0v8lB1XFdAI29czWeznVh (C000-C015)
- Hermes 生成文件夹：Otppfr9EelPIawdezL2csUXCnoh
- 内容工厂出品夹：OaHdfQM9flT1vudSxmFcHVRMnEg
- feishu-tokens.json 集中管理所有飞书 token

## Git
- GitHub 用户名：paraller84
- SSH key: id_ed25519（已验证可访问 GitHub）
- 项目：hermes-agent（上游）、ink-soul（本包）

---

更新日期：2026-05-11
