---
name: handwriting-ocr-pipeline
title: Handwriting OCR Pipeline
description: Canvas-based handwriting input → local/cloud OCR recognition → answer validation. For Chinese dictation/fill-in practice on tablets, with IME cheating prevention.
---

# Handwriting OCR Pipeline（手写识别管线）

将平板/触屏设备的 Canvas 手写输入转化为文字，用于语文/英语学科的填空题、听写题、默写题等**需要实际书写作答**的场景。

## 解决的问题

| 问题 | 键盘输入 | 手写输入 |
|------|---------|---------|
| 三年级孩子考试用手写 | ❌ 键盘不模拟真实场景 | ✅ 手指写字 = 模拟考试 |
| 输入法作弊（拼音→选字） | ❌ 孩子不写也填对 | ✅ 必须亲手写出来 |
| 浏览器无法禁用IME | ❌ `ime-mode:disabled` 无效 | ✅ 绕开输入法 |

> **核心设计原则**: 对三年级学生，键盘拼音选字的体验和手写完全不同——「碧」字不记得怎么写也能打出拼音选出。这导致练习成绩虚高。**手写输入不是功能补丁，而是语文练习的核心场景匹配需求。**

## Architecture (v2 — 用户验证后修正)

```
练习页 (practice.html)
  └── N 个独立 Canvas 格子（每字一格，田字格背景）
       ├── 格子0: 写第1个字
       ├── 格子1: 写第2个字
       └── ...
            └── 各格子独立 toDataURL → base64
                 └── POST /api/handwriting/recognize (逐格识别)
                      ├── ❶ 云端 Qwen-VL-OCR (主通道, ~0.3s, 高精度)
                      │    ⚠ 需要 data: URI 格式: `data:image/png;base64,...`
                      └── ❷ 本地 qwen3-vl:8b (降级, ~1-6s)
                           ⚠ 需要裸 base64 (无 data: 前缀)
                           └── { text: "快", source: "cloud|local" }
                                └── 拼合逐字结果
                                     └── 对比完整答案 → 逐字反馈
```

## 实现组件

### 1. Canvas 手写板（前端）

放在 `templates/` 中作为独立模板或 include 片段。

关键要素：
- **触摸事件**: `touchstart` / `touchmove` / `touchend` + `{ passive: false }`
- **鼠标事件**: `mousedown` / `mousemove` / `mouseup` / `mouseleave`（方便桌面调试）
- **高DPI适配**: `window.devicePixelRatio` 缩放 canvas
- **笔画压感模拟**: 根据移动速度动态调整 `lineWidth`
- **`touch-action: none`**: CSS 防止浏览器手势干扰

```javascript
// 核心绘制逻辑
function getPos(e) {
  const rect = canvas.getBoundingClientRect();
  return {
    x: (e.clientX || (e.touches && e.touches[0].clientX)) - rect.left,
    y: (e.clientY || (e.touches && e.touches[0].clientY)) - rect.top
  };
}
function draw(e) {
  const dx = pos.x - lastX, dy = pos.y - lastY;
  const speed = Math.sqrt(dx*dx + dy*dy);
  ctx.lineWidth = Math.max(2, Math.min(6, 6 - speed * 0.3)); // 慢→粗, 快→细
}
```

### 2. Flask 识别端点

```python
@api_bp.route('/handwriting/recognize', methods=['POST'])
def handwriting_recognize():
    data = request.get_json() or {}
    image_b64 = data.get('image', '')   # 可能是 data: URI 或裸 base64

    # 处理 base64 格式：Ollama 要裸数据，云端要 data: URI
    raw_b64 = image_b64
    data_uri = image_b64
    if raw_b64.startswith('data:'):
        raw_b64 = raw_b64.split(',', 1)[-1]     # 裸 base64 → Ollama
    else:
        data_uri = f'data:image/png;base64,{raw_b64}'  # data URI → 云端

    # 1. 本地 qwen3-vl:8b (Ollama via localhost)
    try:
        resp = http_req.post('http://localhost:11434/api/chat', json={
            'model': 'qwen3-vl:8b',
            'messages': [{
                'role': 'user',
                'content': '只输出识别的汉字，不要任何解释或标点',
                'images': [raw_b64],           # ← 裸 base64，无 data: 前缀
            }],
            'stream': False, 'options': {'temperature': 0.1}
        }, timeout=15)
        recognized = resp.json()['message']['content'].strip()
        source = 'local'
    except Exception as e:
        print(f'[handwriting] 本地失败: {e}')

    # 2. 云端 Qwen-VL-OCR 降级 (data: URI)
    if not recognized:
        resp = http_req.post(
            'https://dashscope.aliyuncs.com/compatible-mode/v1/chat/completions',
            headers={'Authorization': f'Bearer {api_key}', ...},
            json={
                'model': 'qwen-vl-ocr',
                'messages': [{
                    'role': 'user',
                    'content': [
                        {'type': 'image_url', 'image_url': {'url': data_uri}},  # ← data: URI
                        {'type': 'text', 'text': '识别手写汉字，只返回汉字本身'},
                    ],
                }],
            }, timeout=10
        )
        recognized = resp.json()['choices'][0]['message']['content'].strip()
        source = 'cloud'
    return jsonify({'text': recognized, 'source': source})
```

### 3. 提示词要求

极简输出，只返回汉字：

```
用户: 只输出识别的汉字，不要任何解释或标点
图片: [Canvas base64]
模型: (只输出) 月
```

## 题目适配

| 题型 | 手写板配置 |
|------|-----------|
| 单字填空 ("床前明___光") | 1格手写板，返回1个字 |
| 看拼音写词语 (zhōng guó) | 2-4格 → 模式A单行 |
| 反义词/近义词 | 1格手写板，写答案字 |
| 古诗默写 (多字) | 5字+ → 模式B/C/D 自适应 |
| 句子填空 | ≥15字 → 模式D全画布 |

## 4级自适应布局（v2.1 — 2026-05-16 新增）

**核心逻辑**: 根据 `answer_char_count` 自动选择布局模式和识别策略。一句话：字数决定画法，画法决定识别方式。

```javascript
var mode = count <= 4 ? 'single'    // A
         : count <= 8 ? 'dual'      // B
         : count <= 15 ? 'multi'    // C
         : 'full';                  // D
```

### 布局对照

| 模式 | 名称 | 字数 | 手机(cols) | Pad(cols) | 布局 | 识别 |
|------|------|------|-----------|----------|------|------|
| **A** | 单行 | ≤4 | 1行×N格 | 1行×N格 | 水平居中，无滚动 | 逐字 |
| **B** | 双行 | 5-8 | 2-3行×3列 | 2行×4列 | 自动换行，无滚动 | 逐字 |
| **C** | 多行滚动 | 9-15 | 4-5行×3列(最大高260px) | 3-4行×4列(最大高350px) | **固定高度 + 内部滚动** + 页指示器 | 逐字 |
| **D** | 全画布 | ≥15 | 320×120px | 500×160px | 单一大画布 + **引导线** | **整图一次识别** |

### 模式 D：全画布特殊处理

```javascript
function initFullCanvas(view, container, count) {
  // 画引导线（辅助书写比例控制）
  var lines = count <= 20 ? 1 : Math.ceil(count / 20);
  for (var l = 0; l < lines; l++) {
    var ly = h / (lines + 1) * (l + 1);
    // 绘制浅灰色水平引导线
  }
}

// 识别策略不同：整张图一次发送
async function recognizeFullCanvas(view) {
  var dataUrl = fs.canvas.toDataURL('image/png');
  var resp = await fetch('/api/handwriting/recognize', {
    body: JSON.stringify({ image: dataUrl }),
  });
  fs.fullText = result.text;  // 返回完整句子
}
```

### 模式 C：滚动指示器

```javascript
// 页面底部显示"第X屏（共N行）· 拖动查看"
function getPageText(view) {
  var pct = scrollTop / scrollH * 100;
  return '第 ' + (pct < 33 ? '1' : pct < 66 ? '2' : '3')
       + ' 屏（共' + totalRows + '行）· 拖动查看';
}
// 滚动时实时更新
scrollWrap.addEventListener('scroll', function() {
  pageEl.textContent = getPageText(view);
});
```

### 状态管理统一

`hwState[view]` 结构自适应：
- **模式 A/B/C**: `{ cells: [...], mode: 'single'|'dual'|'multi', colsPerRow: N, totalRows: N }`
- **模式 D**: `{ fullState: { canvas, ctx, dpr, drawn, fullText, ... }, mode: 'full' }`

所有公共操作函数（`clearAllCells`, `recognizeAllCells`, `submitHwAnswer`, `updateSubmitBtn`）内部根据 `s.mode` 分支处理。

### 逐格清除模式（✕ 按钮 — 2026-05-17 新增）

每个手写格子下方新增独立清除按钮（✕），支持**逐格擦除**而非一次性清空全部：

```html
<!-- 每个 cell wrap 内 -->
<div class="hw-wrap" style="position:relative;display:inline-block;">
  <canvas class="hw-cell" data-idx="N"></canvas>
  <div class="hw-cell-label" id="hwLabel_N">N+1</div>
  <button class="hw-btn-clear-sm" id="hwClear_N"
          style="display:none;position:absolute;top:-8px;right:-8px;
                 width:22px;height:22px;border-radius:50%;border:none;
                 background:rgba(255,80,80,0.85);color:#fff;font-size:12px;
                 cursor:pointer;line-height:22px;text-align:center;z-index:5;">✕</button>
</div>
```

**关键交互逻辑：**

| 触发条件 | 行为 | 实现 |
|---------|------|------|
| 格子刚画出笔迹（end事件） | ✕ 按钮出现 | `showClearBtn(idx)` → `btn.style.display = 'inline-block'` |
| 用户点击 ✕ 按钮 | 清除该格：清空canvas → `c.recognized=''` → `c.drawn=false` → label回到序号 | `clearCell(idx)` |
| 清除全部按钮点击 | 清除所有格子 + 隐藏所有 ✕ 按钮 | `clearAllCells()` 中遍历 `hideClearBtn(i)` |
| 格子重新画上笔迹 | ✕ 按钮重新出现 | 再次触发 end 事件 |

**🔴 陷阱：`clearAllCells()` 必须同步清理逐格按钮**

```javascript
// ❌ 错误 — 只清了画布，按钮还露在外面
function clearAllCells() {
  cells.forEach(c => { c.ctx.clearRect(...); c.drawn = false; c.recognized = ''; });
}

// ✅ 正确 — 遍历隐藏所有逐格按钮
function clearAllCells() {
  cells.forEach((c, i) => {
    c.ctx.clearRect(...); c.drawn = false; c.recognized = '';
    hideClearBtn(i);  // ← 不能漏！
  });
}
```

**设计原理：** 三年级孩子在触摸屏上写字可能某笔没写好（占格不对/笔画歪了），此时视觉上看到格子下方的 ✕ 按钮可直觉地只擦那一格重写，而非全部清空重来。这符合儿童认知心理：**指哪打哪，不殃及他格**。

### CSS 层级

```css
/* 通用格子 */
.hw-cells { display: flex; gap: 10px; justify-content: center; flex-wrap: wrap; }

/* 滚动容器 */
.hw-scroll-wrap { max-height: 350px; overflow-y: auto; scrollbar-width: thin; }
@media (max-width: 420px) { .hw-scroll-wrap { max-height: 260px; } }

/* 全画布 */
.hw-full-canvas canvas { 
  width: 500px; height: 160px;  /* pad */
  border: 2px solid rgba(255,255,255,0.2); border-radius: 10px; 
}
@media (max-width: 420px) { 
  .hw-full-canvas canvas { width: 320px !important; height: 180px !important; } 
}

/* 页指示器 */
.hw-page-indicator { font-size: 0.8rem; color: #666; text-align: center; margin-top: 4px; }
```

### 实现注意事项

- 模式 D 不能复用逐格识别的 `recognizeCell`，必须走专用 `recognizeFullCanvas`
- 模式 C 的滚动容器是动态创建的 `.hw-scroll-wrap`，内部再套 `.hw-cells`（两层DOM）
- `initHwView()` 现在包含隐藏/显示逻辑：每次初始化先隐藏所有模式专属容器，再渲染对应模式
- 提交答案时通过 `s.mode` 决定拼接方式：模式A/B/C用 `cells.map(c => c.recognized).join('')`，模式D用 `fullState.fullText`

## 交互模式（重要设计决策）

用户要求在试用手写原型后进行了明确的设计修正，形成了当前的交互模型：

### ❌ v1（已废弃 — 单 Canvas + 整句书写）

```
题目: "开心"的近义词是___
┌───────────────────────────┐
│   用手写  "快乐" 两个字    │
│   写在同一个大 Canvas      │
│   一次识别，识别精度差      │
└───────────────────────────┘
```

### ✅ v2（当前采用 — 逐字格子 + 独立识别）

```
题目: "开心"的近义词是___
                        
  ┌──────────┐    ┌──────────┐
  │ 田字格    │    │ 田字格    │   ← 每个格子 100x110px
  │ 写"快"   │    │ 写"乐"   │     田字格辅助线 (CSS)
  │          │    │          │     独立 Canvas，独立 ctx
  └──────────┘    └──────────┘
      ↓识别           ↓识别     
     "快"          "乐"
         → 拼合: "快乐" → 提交
```

**用户反馈驱动的设计修正**：
1. "本地模型识别不够准确" → 切换为云端主通道 (Qwen-VL-OCR)
2. "不支持一个字一个字地手写答案" → 改为逐字格子，每字独立画布

**核心原则**: 永远不要假设用户想要的交互模式。先出可交互原型，让用户上手验证后再固化设计。

### 前端实现要点

```javascript
// 田字格背景 (CSS)
.hw-cell {
  background-image:
    linear-gradient(rgba(0,0,0,0.06) 1px, transparent 1px),
    linear-gradient(90deg, rgba(0,0,0,0.06) 1px, transparent 1px);
  background-size: 50% 50%;
  background-position: center center;
}

// 格子激活状态边框
.hw-cell.active {
  border-color: #4a90d9;
  box-shadow: 0 0 12px rgba(74,144,217,0.4);
}

// 逐格绘制 (每个格子独立 Canvas)
cells[idx] = { canvas, ctx, dpr, drawn: false, recognized: '', w, h };
// 每个 canvas 绑定独立的 touch/mouse 事件
canvas.addEventListener('touchstart', (e) => startDraw(e, idx), { passive: false });
// 点击格子切换激活
cell.addEventListener('click', () => setActiveCell(i));
```

### 后端对比逻辑

```javascript
// 逐格结果拼合后，对比完整答案
const userAnswer = cells.map(c => c.recognized).join('');
const isCorrect = userAnswer === q.answer;
// 逐字反馈
const charResults = q.chars.map((ch, i) => {
  const got = cells[i]?.recognized || '?';
  return got === ch ? `✅${got}` : `❌${got}`;
});
```

> **设计教训**: 用户说"不能一个字一个字写"时，正确的响应不是解释为什么单 Canvas 够用，而是立刻重新设计为格子式交互。**原型的意义在于让用户指出不满意的地方**。

## Pitfalls

### Canvas 兼容性
- 平板必须设 `touch-action: none`，否则滑动页面时画不了
- `devicePixelRatio` 处理不当字会模糊或偏移
- touch 事件必须加 `{ passive: false }`，否则 `preventDefault()` 不生效

### 模型优先级（⚠️ 2026-05-16 更正 — 用户验证后翻转）

| 层级 | 模型 | 耗时 | VRAM | 适用说明 |
|------|------|------|------|---------|
| **主通道** 🏆 | **Qwen-VL-OCR** (阿里云百炼) | **~0.3s** | — | **云端主通道**。精度最高，延迟最低，免费额度充裕。是手写识别的首选。 |
| 降级 | **qwen3-vl:8b** (本地 Ollama) | 1~6s | ~6GB | 云端不可用或超时时回退。本地识别精度不如云端，用户反馈「不够准确」。 |

> **关于 qwen3.5-256k**: 虽然支持 vision（36s 识别正确），但延迟太高不适合交互式手写识别。已在代码中移除。不要尝试此模型。

### Ollama URL（WSL → Windows）

从 WSL 访问 Windows 宿主机上运行的 Ollama：
- ✅ `http://localhost:11434` — 正确！WSL 和 Windows 共享 localhost
- ❌ `http://host.docker.internal:11434` — 这是 Docker 容器的 DNS，在 WSL 中不可用

### base64 格式（重要）

Ollama 和云端 API 对 base64 的要求**不同**：

| API | 格式 | 示例 |
|-----|------|------|
| **Ollama** (本地) | **裸 base64**（无前缀） | `'images': ["iVBOR..."]` |
| **DashScope** (云端) | **data: URI**（带MIME前缀） | `image_url: { "url": "data:image/png;base64,iVBOR..." }` |

> 前端 Canvas 输出的 `toDataURL('image/png')` 是 data: URI 格式。**不能直接塞给 Ollama**，必须先 `split(',', 1)[-1]` 去掉前缀。

### 集成到出题引擎（重要 Bug 修复）

手写答案通过 `answer` 字段提交，与选择题共用同一个答题路由。需要确保：

1. 出题模板的 `answer` 和 `choices` 字段传递到前端（`generate()` 函数中显式保留）
2. `practice_answers` 表必须有 `answer_choices TEXT` 列（存储选项JSON，填空题可为NULL）
3. 答案提交路由必须同时支持 `request.get_json()`（AJAX）和 `request.form`（表单提交）

### 先出原型，后做固化（重要 — 2026-05-16 新增）

**不要替用户决定交互方式**。手写板的第一版（单Canvas整句书写）被用户直接否定：
- "不支持一个字一个字地手写答案" → 需要改为逐字格子
- "本地模型不够准确" → 需要切换为云端主通道

**正确流程**：
1. 出**可交互原型**（demo 页面，非最终集成版本）
2. 让用户在真实设备（Pad/平板）上测试
3. 收集用户的修改意见 → 再做最终版本
4. 最终版本固化前，**用户必须亲手试过**

### Vision 模型 JSON 输出的健壮性陷阱（⚠️ 2026-05-17 关键修复）

当要求 vision 模型返回 JSON 格式时（如 `{"char":"鬼","position_ok":true}`），模型可能输出**不合法的 JSON**：

| 模型输出 | 问题 | `json.loads` 结果 |
|----------|------|------------------|
| `{"char":"鬼","position_ok":ture}` | `ture` 拼写错误 | ❌ 抛出异常 |
| `{"char":"快","position_ok":fasle}` | `fasle` 拼写错误 | ❌ 抛出异常 |
| `只输出：{"char":"中","position_ok":true}` | 带前缀文本 | ❌ 抛出异常 |
| `{"char":"乐"` | 不完整 JSON | ❌ 抛出异常 |

**错误的 fallback（原始代码）：**
```python
except (json_mod.JSONDecodeError, TypeError):
    recognized = content  # ← 整段 JSON 字符串当成了汉字！
    position_ok = True
```

这导致 `recognized` 变成 `{"char":"鬼","position_ok":ture}`，提交时与正确答案"鬼"不匹配 → 判错。

**正确的 fallback（正则提取法）：**
```python
import re
try:
    parsed = json.loads(content)
    recognized = parsed.get('char', '')
    position_ok = parsed.get('position_ok', True)
except (json_mod.JSONDecodeError, TypeError):
    # 正则提取 char 字段值（容错 ture/fasle 等笔误）
    m = re.search(r'["\']char["\']\s*:\s*["\']([^"\']+)["\']', content)
    recognized = m.group(1) if m else content
    # 同时容错提取 position_ok
    m2 = re.search(r'["\']position_ok["\']\s*:\s*(true|false|ture|fasle|True|False)', content, re.IGNORECASE)
    position_ok = m2.group(1).lower() in ('true', 'ture') if m2 else True
```

**原则：** 任何要求 LLM/vision 模型输出 JSON 的场景，都不能只依赖 `json.loads()`。必须加一层正则兜底。常见的拼写错误包括：

| 正确 | 可能错误 |
|------|---------|
| `true` | `ture`, `treu`, `True` |
| `false` | `fasle`, `flase`, `False` |
| `null` | `nil`, `none` |

### 集成到练习系统
- 填空题用手写板，选择题保持选项点击
- 提交时 `answer` 字段存**识别结果文字**，不是图片
- 出题引擎模板必须显式传递 `answer` 和 `choices`，否则 `correct_answer` 和 `answer_choices` 会为空

## 田字格 + 位置质量评估（v3 — 2026-05-17 新增）

### 田字格实现

每个 Canvas 格子绘制 **米字格**（田字格 + 对角线虚线），使用 `ctx` 直接画在 Canvas 上（而非 CSS background）：

```javascript
function drawTianZiGe(ctx, w, h) {
  ctx.save();
  ctx.strokeStyle = 'rgba(180, 180, 200, 0.5)';
  ctx.lineWidth = 1;
  ctx.setLineDash([3, 3]);
  ctx.beginPath(); ctx.moveTo(w/2, 0); ctx.lineTo(w/2, h); ctx.stroke();
  ctx.beginPath(); ctx.moveTo(0, h/2); ctx.lineTo(w, h/2); ctx.stroke();
  ctx.beginPath(); ctx.moveTo(0, 0); ctx.lineTo(w, h); ctx.stroke();
  ctx.beginPath(); ctx.moveTo(w, 0); ctx.lineTo(0, h); ctx.stroke();
  ctx.setLineDash([]);
  ctx.restore();
}
```

**关键集成点：**
- `initHw()` 中每个 Canvas 创建后调用 `drawTianZiGe(ctx, cellW, cellH)`
- `clearCell()` / `clearAllCells()` 的 `clearRect` 后重绘
- Canvas 尺寸改为正方形 260×260（手机端 150×150）

### 书写位置质量评估

将 田字格 背景与手写笔画一同发送到 OCR API，修改提示词让模型同时评估位置质量：

```
图中田字格内手写的是什么汉字？判断该字是否书写位置正确
（笔画大致在田字格中央，不过于偏左偏右偏上偏下，大小不过分偏小）。
只返回JSON格式：{"char":"识别的汉字","position_ok":true/false}
```

**API 响应新增字段：** `position_ok: boolean`

### 前端处理

| 场景 | 表现 |
|------|------|
| 文字正确 + 位置正确 | 绿色标号 `字` |
| 文字正确 + 位置偏 | 橙色标号 `字 ⚠️位置偏` |
| 提交时位置偏 | `alert` 阻止提交，提示重写 |
| 清除后 | `position_ok` 重置为 `true` |

### 流程示意

```
用户写"好"字在田字格偏右上角
→ Canvas 截图（含田字格虚线）
→ Qwen-VL-OCR 识别：位置偏右 → position_ok: false
→ 前端显示 "好 ⚠️位置偏"（橙色）
→ 提交时拦截，alert 提示用✕清除重写
```

### 实现注意事项

- 田字格用 `ctx.draw` 而非 CSS background：CSS gradient 无法绘制对角线
- `clearCell` / `clearAllCells` 必须重置 `c.position_ok = true` 和 `label.style.color = ''`
- 本地降级模型（qwen3-vl:8b）JSON 输出不稳定，设 fallback `position_ok = True`

## 参考

- 演示实现: `templates/student/handwriting_demo.html`
- API 端点: `routes/api.py` → `handwriting_recognize()`
- 出题引擎: `services/practice_engine.py`
- 参考文件: `references/handwriting-demo-session.md` — 首次实现记录
- 参考文件: `references/session-20260516-cloud-grid-updates.md` — 云端/逐字格子版本更新
