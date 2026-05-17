---
name: cloud-vision-integration
version: 1.0.0
category: mlops/inference
description: >-
  集成云端视觉API（OCR/图像识别）的标准模式。覆盖国内主流云视觉模型的选型、
  API接入、定价对比、架构模式。当前聚焦阿里云百炼 Qwen-VL-OCR，
  可扩展至其他国内云视觉服务。
tags:
  - cloud-vision
  - OCR
  - qwen-vl-ocr
  - aliyun
  - baidu-vision
  - bailian
---

# 云端视觉API集成

## 核心架构模式：云端优先，本地回退

```
拍照 →
  客户端质检（canvas检测模糊/亮度/倾斜，不上传）→
  云端API主通道（~1s）→
  失败时回退本地Ollama（~60-120s）→
  结构化结果返回
```

## 阿里云百炼 Qwen-VL-OCR

### 接入要点

| 项目 | 值 |
|:-----|:---|
| 模型名 | `qwen-vl-ocr-latest` |
| Base URL | `https://dashscope.aliyuncs.com/compatible-mode/v1` |
| API格式 | OpenAI 兼容（可直接用 openai Python SDK） |
| 认证 | Header: `Authorization: Bearer {DASHSCOPE_API_KEY}` |
| 图片参数 | `min_pixels: 32*32, max_pixels: 32*32*2048` |
| 新用户免费 | 7,000万 Tokens（约3万张照片） |

### 价格

| 项 | 单价 |
|:---|:----|
| Qwen-VL-OCR 输入/输出 | **¥6/百万 Tokens** |
| Qwen-VL-OCR 单张课本照 | ≈ ¥0.014（~2,125 input + ~200 output） |
| Qwen-VL-OCR 每天5张 | ≈ ¥0.07 |
| Qwen3-VL-Plus 输入/输出 | **¥2/百万 Tokens**（2026-05实际：¥0.0017/张） |
| Qwen3-VL-Plus 27张批量 | ≈ ¥0.046 |

对比：
- Qwen-VL-Plus: ¥0.75/百万输入（通用视觉，非OCR专用）
- GLM-4.6V-Flash: 完全免费（智谱AI，能力偏通用）
- 豆包Seed-1.6-vision: ¥0.8/百万输入，有每日50万免费额度

### API 调用示例（原生 urllib，不依赖 openai 库）

```python
import json, urllib.request

body = {
    "model": "qwen-vl-ocr-latest",
    "messages": [{
        "role": "user",
        "content": [
            {"type": "text", "text": prompt},  # 提示词需含格式要求
            {"type": "image_url", "image_url": {
                "url": f"data:image/jpeg;base64,{compressed_b64}",
                "min_pixels": 32*32,
                "max_pixels": 32*32*2048,
            }},
        ]
    }],
    "max_tokens": 2048,
    "temperature": 0.1,
}

req = urllib.request.Request(base_url, data=json.dumps(body).encode("utf-8"),
    headers={"Authorization": f"Bearer {api_key}", "Content-Type": "application/json"},
    method="POST")
resp = urllib.request.urlopen(req, timeout=60)
result = json.loads(resp.read())
text = result["choices"][0]["message"]["content"]
```

### 多模型协作模式

### 场景：彩色背景词语检测

Qwen-VL-OCR 不识别彩色背景标记（课本中的高亮词语）。需要配合一个支持颜色识别的视觉模型：

| 职责 | 模型 | 特长 | 成本 |
|:-----|:-----|:-----|:----|
| 常规文字提取 | Qwen-VL-OCR | 快速文本识别（~1s/张） | ¥0.014/张 |
| 彩色标记检测 | Qwen3-VL-Plus | 可分辨粉色/黄色/蓝色背景的词语 | ¥0.0017/张 |

**架构模式**：双轨并行 — 先 Qwen-VL-OCR 提取全文，再用 Qwen3-VL-Plus 检测有背景色的词语。两条结果通过 `page_number` 字段关联。

### Qwen3-VL-Plus 接入要点

| 项目 | 值 |
|:-----|:---|
| 模型名 | `qwen3-vl-plus` |
| API格式 | DashScope OpenAI 兼容（同 Qwen-VL-OCR） |
| 图片分辨率关键 | 保持原图分辨率（不压缩到800px），否则颜色区域不可辨 |
| 价格 | **¥2/百万 Tokens**（约 ¥0.0017/张课本照） |

### API 调用示例

```python
import json, urllib.request, base64

# 原图直接 base64（不压缩，保留颜色细节）
with open(image_path, "rb") as f:
    b64 = base64.b64encode(f.read()).decode()

body = {
    "model": "qwen3-vl-plus",
    "messages": [{
        "role": "user",
        "content": [
            {"type": "text", "text": prompt},
            {"type": "image_url", "image_url": {
                "url": f"data:image/png;base64,{b64}",
                "min_pixels": 256*256,
                "max_pixels": 1280*1280*4,  # 高分辨率保留颜色细节
            }},
        ]
    }],
    "max_tokens": 4096,
}

req = urllib.request.Request(base_url, data=json.dumps(body).encode(),
    headers={"Authorization": f"Bearer {api_key}", "Content-Type": "application/json"})
resp = urllib.request.urlopen(req, timeout=120)
result = json.loads(resp.read())
```

### 颜色检测提示词模板

```python
PROMPT_COLORED_WORDS = """你是一个语文课本彩色标记词语提取器。

请分析这张课文图片：
1. 提取课文全文
2. 注意页面上**有颜色背景（粉色/黄色/蓝色等）**的词语，这些是重点高亮词
3. 只输出JSON格式，不要额外文字

输出格式：
{
    "type": "cn_highlight_words",
    "lesson_title": "课文标题",
    "lesson_number": 课号,
    "page_number": 页码数字,
    "source_text": "完整课文文字",
    "words": ["词语1", "词语2", "词语3"]
}"""
```

## 关键陷阱

1. **图片压缩**：Qwen-VL-OCR 建议压缩到 1024px@85% JPEG；但 Qwen3-VL-Plus 辨识颜色时需要用 PNG 原图（不压缩），否则颜色区域模糊
2. **系统提示词合并**：Qwen-VL-OCR 不支持 `system` 角色，必须将 System Prompt 拼入 User 的 text
3. **JSON 输出要求**：提示词中必须明确说明"只输出JSON，不要额外文字"，否则模型可能输出散文
4. **中文渲染测试**：PIL 默认字体不渲染中文，测试时需要指定中文字体（如 `wqy-zenhei.ttc`）
5. **首次开通流程**：阿里云百炼 → 模型广场搜 qwen-vl-ocr → 开通 → API-KEY 管理创建 key
6. **WSL 环境 API Key 加载**：DashScope API Key 在 `~/.bashrc` 中设置，但 terminal() 的 Python 子进程不会自动 source. 需要在 ocr_service.py 中增加 .bashrc 备选读取：

```python
DASHSCOPE_API_KEY = os.environ.get("DASHSCOPE_API_KEY", "")
if not DASHSCOPE_API_KEY:
    bashrc = os.path.expanduser("~/.bashrc")
    if os.path.exists(bashrc):
        import re
        with open(bashrc) as f:
            match = re.search(r'DASHSCOPE_API_KEY=[\"\\\']?([^\"\\\'\\\\n]+)', f.read())
            if match:
                DASHSCOPE_API_KEY = match.group(1)
```

## 多项目集成注意事项

当工作区存在多个项目时（如 Edu-Hub 和 AI家庭教师共存于同目录下），必须在改动前验证：

1. `ls` 确认项目目录结构
2. 确认文件属于正确的项目
3. 配置（API Key、crontab）只写入目标项目所需位置

## 参考资料

- 阿里云百炼官方价格: `references/aliyun-bailian-pricing.md`
- 国内云视觉模型对比: `references/domestic-vision-api-comparison.md`
