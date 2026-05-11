---
name: vector-retrieval-deployment
description: 部署完整的向量化检索系统，支持多格式文档索引、语义搜索和RAG集成。针对资源受限环境优化。
tags:
  - vector-search
  - chromadb
  - sentence-transformers
  - fastapi
  - rag
  - resource-constrained
trigger: |
  当用户需要：
  1. 部署向量检索系统
  2. 搭建文档/知识库检索系统
  3. 在无GPU环境中运行嵌入模型
  4. 创建语义搜索服务
  5. 为RAG应用构建后端
category: mlops
---

# 向量检索系统部署指南

## 概述

本技能提供完整的向量检索系统部署方案，支持多格式文档索引、语义搜索和RAG集成。特别针对资源受限环境（无GPU、16GB以下内存）优化。

## 核心概念

### 向量检索系统组件
1. **文档处理器**: 提取和分块多格式文档
2. **嵌入模型**: 将文本转换为向量表示
3. **向量数据库**: 存储和检索向量
4. **检索API**: 提供搜索接口
5. **管理工具**: 系统监控和维护

### 部署架构选择
- **L1 (轻量级本地)**: ChromaDB + Sentence Transformers (CPU)
- **L2 (中等规模)**: Qdrant + Ollama (本地Docker)
- **L3 (云原生)**: Weaviate/Pinecone + 云API

## 部署流程

### 1. 环境准备
- 创建项目目录结构
- 设置Python虚拟环境
- 安装系统依赖

### 2. 核心文件创建
需要创建以下文件结构：
```
vector-retrieval-deployment/
├── src/
│   ├── utils/document_processor.py    # 文档处理
│   ├── models/vector_store.py         # 向量存储
│   ├── models/embedding_models.py     # 嵌入模型
│   ├── api/main.py                    # API服务
│   └── cli/__init__.py                # 命令行工具
├── config/pipeline_config.yaml        # 流水线配置
├── .env                              # 环境变量
├── requirements.txt                  # Python依赖
└── deploy.sh                         # 部署脚本
```

### 3. 依赖安装
核心Python包：
- chromadb (向量数据库)
- sentence-transformers (嵌入模型)
- fastapi + uvicorn (API服务)
- pydantic (数据验证)
- 文档处理库 (pypdf2, python-docx等)

### 4. 配置文件
关键配置项：
- 向量存储路径 (`./data/chromadb`)
- 嵌入模型类型 (`sentence_transformer`)
- 模型名称 (`all-MiniLM-L6-v2` - CPU友好)
- 分块大小 (`800` - 内存优化)
- 重叠大小 (`150`)

### 5. 系统验证
验证步骤：
1. 检查Python包导入
2. 测试嵌入模型加载
3. 验证向量存储连接
4. 测试API服务启动

**自动化验证脚本**: 创建`scripts/verify.sh`进行系统健康检查
```bash
#!/bin/bash
# 验证向量检索系统安装
# 检查Python环境、依赖包、目录结构、配置文件、模型加载、API服务等
```

**快速验证命令**:
```bash
cd ~/vector-retrieval-deployment
./scripts/verify.sh  # 全面验证
./deploy.sh --status # 检查服务状态
python -m src.cli stats  # 查看系统统计
curl http://localhost:8000/health  # API健康检查
```

**部署脚本关键功能**:
- `--setup-env`: 创建Python虚拟环境
- `--install-deps`: 安装依赖包（带超时和镜像源）
- `--init-dirs`: 初始化目录结构和示例文档
- `--create-config`: 生成配置文件
- `--start-api`: 启动API服务（处理初始化错误）
- `--index-docs`: 索引文档
- `--quick-start`: 一键完成所有步骤

## 最佳实践

### 针对资源受限环境
1. **内存优化**:
   - 使用小型嵌入模型 (all-MiniLM-L6-v2, 22MB)
   - 减小分块大小 (600-800字符)
   - 限制批量处理大小
   - WSL环境: 监控WSL2内存使用，考虑配置`.wslconfig`限制内存

2. **CPU优化**:
   - 安装CPU版本的PyTorch
   - 使用本地文件缓存模型
   - 避免频繁模型重载
   - WSL环境: 启用WSL2 GPU支持（如果可用）或优化CPU线程数

3. **存储优化**:
   - 定期清理临时文件
   - 压缩向量存储
   - 使用增量索引
   - WSL环境: 使用WSL2的文件系统性能优于WSL1

4. **WSL特定优化**:
   - **网络连接**: 使用国内镜像源加速下载
   - **文件I/O**: 避免在`/mnt/c/`路径下存储向量数据库，使用WSL原生文件系统
   - **内存限制**: 默认WSL2使用80%主机内存，可在`.wslconfig`中设置限制
   - **启动性能**: 禁用自动挂载Windows驱动器减少启动时间
   - **Docker替代**: 无Docker环境下使用纯Python部署方案

### 性能调优
- **响应时间**: 目标 < 100ms 搜索延迟
- **内存使用**: 监控峰值内存，保持在系统限制的80%以下
- **并发能力**: 根据内存限制调整最大并发数

## 故障排除

### 常见问题

#### 内存不足
- 减小配置中的分块大小
- 使用更小的嵌入模型
- 增加系统交换空间

#### 模型下载失败
- 使用国内镜像源
- 手动下载模型到本地缓存
- 使用离线模式

#### 向量存储错误
- 检查文件权限
- 清理损坏的存储文件
- 验证存储路径有效性

#### API端口冲突
- 修改默认端口
- 检查占用进程
- 使用环境变量配置

### 实际部署中发现的问题及解决方案

#### 1. ChromaDB集合初始化失败
**问题**: `chromadb.errors.NotFoundError: Collection [documents] does not exist`
**原因**: 新部署时集合不存在，但代码只捕获`ValueError`
**解决方案**: 同时捕获`ValueError`和`chromadb.errors.NotFoundError`
```python
try:
    self.collection = self.client.get_collection(self.collection_name)
except (ValueError, chromadb.errors.NotFoundError):
    self.collection = self.client.create_collection(name=self.collection_name)
```

#### 2. Python导入循环依赖
**问题**: FastAPI路由模块中导入`get_vector_store`导致循环导入
**原因**: 路由模块导入主模块，主模块又导入路由模块
**解决方案**: 创建独立的`dependencies.py`文件
```python
# src/api/dependencies.py
vector_store = None
embedding_model = None

def get_vector_store():
    if vector_store is None:
        raise HTTPException(status_code=500, detail="向量存储未初始化")
    return vector_store

# src/api/main.py中设置全局变量
from .dependencies import set_global_store
set_global_store(vector_store, embedding_model)
```

#### 3. Pydantic环境变量解析失败
**问题**: `JSONDecodeError`解析`CORS_ORIGINS`
**原因**: `.env`文件中的列表格式不正确
**解决方案**: 使用正确的JSON数组格式
```bash
# 错误格式
CORS_ORIGINS=http://localhost:3000,http://localhost:8000

# 正确格式  
CORS_ORIGINS=["http://localhost:3000","http://localhost:8000"]
```

#### 4. 依赖安装超时
**问题**: `pip install`在WSL环境中下载缓慢超时
**解决方案**: 使用国内镜像源和超时设置
```bash
pip install --default-timeout=180 -i https://pypi.tuna.tsinghua.edu.cn/simple sentence-transformers
```

#### 5. 模块路径问题
**问题**: `ModuleNotFoundError: No module named 'src.api.models'`
**原因**: Python相对导入在多层包结构中混乱
**解决方案**: 在API主模块中添加项目根目录到路径
```python
import sys
from pathlib import Path
sys.path.insert(0, str(Path(__file__).parent.parent.parent))
```

#### 7. Python包导入名称不匹配
**问题**: `ModuleNotFoundError: No module named 'sentence-transformers'`
**原因**: pip安装的包名为`sentence-transformers`（带连字符），但导入时Python模块名为`sentence_transformers`（带下划线）
**解决方案**: 在验证脚本和代码中使用正确的导入名称
```bash
# pip安装
pip install sentence-transformers

# Python导入
import sentence_transformers  # 注意是下划线，不是连字符

# 验证脚本中检查
python3 -c "import sentence_transformers"  # 正确
python3 -c "import sentence-transformers"  # 错误（语法错误）
```

## 扩展建议

### 功能增强
1. **多语言支持**: 使用多语言嵌入模型
2. **图像检索**: 集成CLIP模型
3. **代码搜索**: 添加代码解析器
4. **增量更新**: 实时文档监控

### 生产化改进
1. **认证授权**: API访问控制
2. **监控告警**: 系统健康监控
3. **备份恢复**: 自动化备份策略
4. **负载均衡**: 多实例部署

### 性能优化
1. **缓存策略**: 查询结果缓存
2. **索引优化**: 向量索引类型选择
3. **批量处理**: 异步索引任务
4. **压缩存储**: 向量量化压缩

## 工具集成

### 可选的向量数据库
- ChromaDB: 简单易用，本地存储
- Qdrant: 高性能，支持过滤
- Weaviate: 云原生，内置模块
- Pinecone: 完全托管服务

### 嵌入模型选项
- Sentence Transformers: 本地CPU模型
- Ollama: 本地LLM集成
- OpenAI Embeddings: API服务
- Cohere Embed: 多语言优化

### 前端集成
- Streamlit: 快速Web界面
- Gradio: 交互式演示
- 自定义前端: React/Vue应用

## 维护指南

### 日常维护
1. 日志监控和轮转
2. 磁盘空间检查
3. 性能指标收集
4. 备份验证

### 定期更新
1. Python包安全更新
2. 模型版本升级
3. 配置文件优化
4. 文档和脚本更新

### 灾难恢复
1. 定期完整备份
2. 恢复流程测试
3. 监控告警设置
4. 故障切换演练

---
## 附录 A: WSL 快速部署（从 vector-retrieval-deployment-wsl 合并）

> 针对 16GB 内存、无 GPU、无 Docker 的 WSL 环境，提供可直接复制的目录结构、配置文件和脚本。

### A1. 目录结构

```bash
mkdir -p ~/vector-retrieval-deployment
cd ~/vector-retrieval-deployment
mkdir -p {src/{utils,models,api,cli,pipeline},config,data/{raw,processed,index},logs,scripts,tests}
```

### A2. WSL 优化配置

#### `.env`
```env
VECTOR_STORE_TYPE=chromadb
VECTOR_STORE_PATH=./data/vector_store
EMBEDDING_MODEL=all-MiniLM-L6-v2
API_HOST=0.0.0.0
API_PORT=8000
API_WORKERS=2
MAX_CHUNK_SIZE=800
CHUNK_OVERLAP=150
BATCH_SIZE=32
```

#### `config/pipeline_config.yaml`
```yaml
pipeline:
  document_processors:
    pdf: {enabled: true, extract_images: false}
    docx: {enabled: true}
    pptx: {enabled: true}
    txt: {enabled: true}
    markdown: {enabled: true}
  chunking:
    chunk_size: 800
    chunk_overlap: 150
    separator: "\n\n"
  embedding:
    model_name: "all-MiniLM-L6-v2"
    device: "cpu"
    normalize_embeddings: true
  vector_store:
    type: "chromadb"
    persist_directory: "./data/vector_store"
    collection_name: "documents"
  memory:
    max_documents_per_batch: 10
    cleanup_interval: 100
```

### A3. 一键部署脚本

参见 `scripts/deploy-wsl.sh`（在 `vector-retrieval-deployment-wsl` 技能中有完整实现）。

### A4. 内存使用预估（16GB WSL）

| 任务 | 内存使用 | 说明 |
|------|----------|------|
| API服务 | 500MB | FastAPI + ChromaDB |
| 嵌入模型 | 800MB | all-MiniLM-L6-v2 |
| 文档处理 | 1-2GB | 取决于文档大小 |
| **峰值** | **3-4GB** | 安全边界内 |

### A5. WSL 特有陷阱

1. **文件 I/O**: 向量数据库不要放在 `/mnt/c/` 下，使用 WSL 原生文件系统
2. **内存限制**: 默认 WSL2 使用 80% 主机内存，可在 `.wslconfig` 中设置限制
3. **网络加速**: 使用国内镜像源（如 `pip install -i https://pypi.tuna.tsinghua.edu.cn/simple`）
4. **ChromaDB 重置**: `rm -rf ./data/vector_store` 可清理损坏的向量存储

## 成功指标

### 技术指标
- 搜索响应时间 < 100ms
- 系统可用性 > 99.5%
- 索引延迟 < 10分钟/GB
- 内存使用稳定

### 业务指标
- 搜索准确率 > 90%
- 用户满意度评分
- 平均搜索深度
- 系统使用频率

## 注意事项

### 安全考虑
- 限制文件上传类型
- API访问速率限制
- 敏感数据过滤
- 定期安全扫描

### 法律合规
- 数据隐私保护
- 版权合规检查
- 用户协议明确
- 数据使用透明

### 成本控制
- 资源使用监控
- 自动伸缩策略
- 成本优化建议
- 预算预警设置

---

本技能提供从零开始部署向量检索系统的完整指导，特别关注资源受限环境的优化和实际问题解决方案。