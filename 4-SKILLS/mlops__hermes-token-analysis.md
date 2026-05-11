---
name: hermes-token-analysis
description: 分析Hermes Agent的token消耗情况，包括数据库查询、成本计算和报告生成
tags:
  - hermes
  - token
  - cost
  - analysis
  - billing
  - usage
---

# Hermes Token消耗分析

当用户询问Hermes Agent的token消耗、成本分析或使用统计时使用此技能。提供完整的分析工作流，包括数据库查询、日志检查和成本计算。

## 触发条件

- "请分析token消耗情况"
- "查看最近的使用统计"
- "计算一下成本"
- "token使用量怎么样"
- "分析大模型使用情况"
- "检查API使用量"

## 分析流程

### 1. 定位Hermes数据文件

首先确定Hermes数据目录位置，通常是`~/.hermes/`：

```bash
# 检查Hermes目录结构
ls -la ~/.hermes/
```

关键文件：
- 会话数据库（SQLite格式）
- 日志文件
- 配置文件

### 2. 数据库分析

使用SQLite查询会话数据：

```bash
# 查看数据库结构
sqlite3 ~/.hermes/state.db ".tables"
sqlite3 ~/.hermes/state.db "PRAGMA table_info(sessions);"

# 查询最近会话
sqlite3 ~/.hermes/state.db "SELECT id, datetime(started_at, 'unixepoch') as time, input_tokens, output_tokens, estimated_cost_usd, billing_provider, model FROM sessions ORDER BY started_at DESC LIMIT 10;"

# 按日期统计
sqlite3 ~/.hermes/state.db "SELECT date(datetime(started_at, 'unixepoch')) as date, COUNT(*) as sessions, SUM(input_tokens) as total_input, SUM(output_tokens) as total_output, SUM(estimated_cost_usd) as total_cost FROM sessions GROUP BY date ORDER BY date DESC;"
```

### 3. Python脚本分析

对于复杂分析，使用Python脚本：

```python
import sqlite3
import time
from datetime import datetime

def analyze_token_usage(hours=48):
    """分析指定时间范围内的token使用情况"""
    db_path = "/home/openclaw/.hermes/state.db"
    conn = sqlite3.connect(db_path)
    cursor = conn.cursor()
    
    # 计算时间范围
    now = time.time()
    time_threshold = now - (hours * 3600)
    
    # 查询会话数据
    cursor.execute("""
    SELECT id, started_at, input_tokens, output_tokens, 
           cache_read_tokens, cache_write_tokens, reasoning_tokens,
           estimated_cost_usd, billing_provider, model
    FROM sessions 
    WHERE started_at >= ?
    ORDER BY started_at DESC
    """, (time_threshold,))
    
    rows = cursor.fetchall()
    
    # 计算统计
    stats = {
        'session_count': len(rows),
        'total_input': 0,
        'total_output': 0,
        'total_cache_read': 0,
        'total_cache_write': 0,
        'total_reasoning': 0,
        'total_cost': 0.0,
        'providers': {}
    }
    
    for row in rows:
        session_id, started_at, inp, out, cache_r, cache_w, reasoning, cost, provider, model = row
        
        stats['total_input'] += inp if inp else 0
        stats['total_output'] += out if out else 0
        stats['total_cache_read'] += cache_r if cache_r else 0
        stats['total_cache_write'] += cache_w if cache_w else 0
        stats['total_reasoning'] += reasoning if reasoning else 0
        stats['total_cost'] += cost if cost else 0
        
        # 按提供商统计
        if provider:
            if provider not in stats['providers']:
                stats['providers'][provider] = {
                    'sessions': 0,
                    'input': 0,
                    'output': 0,
                    'cost': 0.0
                }
            stats['providers'][provider]['sessions'] += 1
            stats['providers'][provider]['input'] += inp if inp else 0
            stats['providers'][provider]['output'] += out if out else 0
            stats['providers'][provider]['cost'] += cost if cost else 0
    
    conn.close()
    return stats

# 使用示例
stats = analyze_token_usage(48)
print(f"会话数量: {stats['session_count']}")
print(f"总输入token: {stats['total_input']:,}")
print(f"总输出token: {stats['total_output']:,}")
print(f"总成本: ${stats['total_cost']:.6f}")
```

### 4. 日志文件检查

补充日志分析：

```bash
# 查看最近活动
tail -50 ~/.hermes/logs/agent.log

# 搜索关键事件
grep -i "response ready" ~/.hermes/logs/agent.log | tail -10
grep -i "inbound message" ~/.hermes/logs/agent.log | tail -10
```

### 5. 成本计算

根据实际使用推算定价：

```python
def calculate_pricing(input_tokens, output_tokens, total_cost):
    """根据实际使用计算每百万token成本"""
    if input_tokens + output_tokens == 0:
        return None
    
    input_ratio = input_tokens / (input_tokens + output_tokens)
    output_ratio = output_tokens / (input_tokens + output_tokens)
    
    input_cost_per_million = (total_cost * input_ratio) / (input_tokens / 1_000_000)
    output_cost_per_million = (total_cost * output_ratio) / (output_tokens / 1_000_000)
    
    return {
        'input_per_million': input_cost_per_million,
        'output_per_million': output_cost_per_million,
        'input_tokens': input_tokens,
        'output_tokens': output_tokens,
        'total_cost': total_cost
    }
```

## 报告生成

### 基本报告结构

```
## Token消耗分析报告

**分析时间**: [当前时间]
**时间范围**: [分析的时间范围]
**数据来源**: Hermes会话数据库

### 📊 总体统计
- 总会话数: [数量]
- 总输入Token: [数值] ([百分比]%)
- 总输出Token: [数值] ([百分比]%)
- 总Token: [数值]
- 估算成本: $[金额]

### 📅 详细统计
[按日期或提供商的详细数据]

### 💰 成本分析
- 推算定价: 输入~$[价格]/M, 输出~$[价格]/M
- 成本效率: [评价]

### ⚠️ 注意事项
1. [数据局限性]
2. [时间戳问题]
3. [缓存token说明]

### 🎯 建议
1. [优化建议1]
2. [优化建议2]
```

## 常见问题处理

### 1. 时间戳异常

数据库可能显示异常时间戳（如未来日期）。处理方法：
- 关注相对时间而非绝对时间
- 使用`ORDER BY started_at DESC`获取最新记录
- 在报告中说明时间戳问题

### 2. 数据不完整

某些时间段可能缺少数据：
- 检查日志确认实际活动
- 说明数据局限性
- 建议启用详细日志记录

### 3. 缓存token

缓存读取token通常不计入成本：
- 区分缓存token和计费token
- 在报告中单独说明缓存使用量
- 缓存命中率高表示系统优化良好

## 提供商特定信息

### DeepSeek
- 参考定价: 输入$0.14/M, 输出$0.28/M
- 缓存读取: 通常免费
- 注意实际定价可能变化

### 其他提供商
- 检查配置文件中的提供商设置
- 查询提供商最新定价
- 考虑本地模型的零成本优势

## 优化建议

基于分析结果提供建议：

1. **成本优化**:
   - 使用成本更低的模型处理简单任务
   - 提高缓存命中率
   - 设置成本预警阈值

2. **使用模式优化**:
   - 分析高峰使用时段
   - 识别高消耗任务类型
   - 优化提示词减少token使用

3. **监控改进**:
   - 启用详细token跟踪
   - 设置定期报告
   - 实现成本预警系统

## 验证与确认

完成分析后：
1. 交叉验证数据库和日志数据
2. 检查时间范围是否正确
3. 验证成本计算合理性
4. 确认数据完整性

## 扩展功能

可选的进阶分析：
- 趋势预测（日/周/月）
- 异常检测算法
- 成本效益分析
- 多用户使用统计