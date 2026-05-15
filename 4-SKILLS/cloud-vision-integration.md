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
| 输入/输出 | **¥6/百万 Tokens** |
| 单张课本照 | ≈ ¥0.014（~2,125 input + ~200 output） |
| 每天5张 | ≈ ¥0.07 |

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

### 关键陷阱

1. **图片压缩**：建议在客户端先压缩到 1024px@85% JPEG，再 base64 上传
2. **系统提示词合并**：Qwen-VL-OCR 不支持 `system` 角色，必须将 System Prompt 拼入 User 的 text
3. **JSON 输出要求**：提示词中必须明确说明"只输出JSON，不要额外文字"，否则模型可能输出散文
4. **中文渲染测试**：PIL 默认字体不渲染中文，测试时需要指定中文字体（如 `wqy-zenhei.ttc`）
5. **首次开通流程**：阿里云百炼 → 模型广场搜 qwen-vl-ocr → 开通 → API-KEY 管理创建 key

## 多项目集成注意事项

当工作区存在多个项目时（如 Edu-Hub 和 AI家庭教师共存于同目录下），必须在改动前验证：

1. `ls` 确认项目目录结构
2. 确认文件属于正确的项目
3. 配置（API Key、crontab）只写入目标项目所需位置

## 参考资料

- 阿里云百炼官方价格: `references/aliyun-bailian-pricing.md`
- 国内云视觉模型对比: `references/domestic-vision-api-comparison.md`
