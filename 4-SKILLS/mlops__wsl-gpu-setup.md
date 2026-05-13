---
layout: default
name: wsl-gpu-setup
title: WSL GPU Passthrough — Diagnostics & Setup
description: Diagnose and configure NVIDIA GPU passthrough for WSL2. Covers environment diagnostics (cross Windows/WSL boundary), CUDA Toolkit installation, nvidia-utils version matching, WSL restart, and verification. Designed for Ubuntu 24.04 + NVIDIA GPUs.
tags: [wsl, cuda, nvidia, gpu, ubuntu]
---

# WSL GPU 透传诊断与配置

诊断并配置 WSL2 的 NVIDIA GPU 透传支持。适用于：用户已有 WSL 环境，但有 GPU（RTX 系列等）却无法在 WSL 内使用。

## 触发条件

用户问「WSL 怎么用 GPU」「WSL 里 CUDA 装不上」「WSL 里 nvidia-smi 找不到」或类似问题。

## 步骤

### 1. 诊断当前 WSL 环境

```bash
# 基础状态
echo "=== 发行版 ===" && lsb_release -a 2>/dev/null || cat /etc/os-release
echo "=== 内存 ===" && free -h
echo "=== CPU ===" && nproc
echo "=== WSL 内核 ===" && uname -r
echo "=== GPU 设备节点 ===" && ls /dev/nvidia* 2>&1 || echo "❌ 无 /dev/nvidia*"
echo "=== nvidia-smi ===" && nvidia-smi 2>&1 || echo "❌ nvidia-smi 不可用"
echo "=== CUDA ===" && nvcc --version 2>&1 || echo "❌ CUDA Toolkit 未安装"
```

### 2. 从 WSL 内部探测 Windows 端 GPU

```bash
# 使用 cmd.exe（WSL Interop）在 Windows 端查 GPU
/mnt/c/Windows/System32/cmd.exe /c "wmic path win32_VideoController get name /format:list"
/mnt/c/Windows/System32/cmd.exe /c "nvidia-smi" | head -10
# 查找已存在的 .wslconfig
find /mnt/c/Users/ -maxdepth 2 -name ".wslconfig" 2>/dev/null
```

### 3. 诊断标准

| 指标 | 正常 | 异常 |
|------|------|------|
| `/dev/nvidia*` | `nvidia0 nvidiactl nvidia-modeset` | 不存在 → 需要重启 WSL |
| Windows `nvidia-smi` | 显示 GPU 型号 + 驱动版本 | 无 → Windows 端未装驱动 |
| Windows 驱动版本 | ≥ 525.60（WSL 支持门槛） | 过低 → 更新驱动 |
| `.wslconfig` | 存在且包含 `memory=16GB` 等 | 不存在或配置不合适 |
| `nvcc --version` | 显示版本号 | 无 → 需安装 CUDA Toolkit |
| WSL 版本 | 2.x | 1.x → `wsl --set-default-version 2` |

### 4. 安装 CUDA Toolkit（WSL 专用）

```bash
# 添加 NVIDIA WSL 源
cd /tmp
wget https://developer.download.nvidia.com/compute/cuda/repos/wsl-ubuntu/x86_64/cuda-keyring_1.1-1_all.deb
sudo dpkg -i cuda-keyring_1.1-1_all.deb
sudo apt update

# 安装 CUDA Toolkit
sudo apt install -y cuda-toolkit-12-6

# 添加 PATH
echo 'export PATH=/usr/local/cuda-12/bin:$PATH' >> ~/.bashrc
echo 'export LD_LIBRARY_PATH=/usr/local/cuda-12/lib64:$LD_LIBRARY_PATH' >> ~/.bashrc
source ~/.bashrc
```

### 5. 安装 nvidia-utils（WSL 内可运行 nvidia-smi）

```bash
# 查找匹配 Windows 端驱动版本的包
apt-cache search nvidia-utils | sort

# 安装最接近的版本（版本号≤ Windows 驱动版本即可）
sudo apt install -y nvidia-utils-590
```

### 6. 重启 WSL

**关键步骤**。在 Windows PowerShell / CMD 中执行：

```powershell
wsl --shutdown
wsl
```

重启后验证：
```bash
nvidia-smi          # 应显示 RTX 4070 / 4090 等 GPU 信息
ls /dev/nvidia*     # 可选（见下方注意事项）
nvcc --version      # CUDA 版本
free -h             # 16GB 内存确认
```

> **⚠️ WSL 内核 6.6.87.2+ 的已知行为**：`/dev/nvidia*` 设备节点可能**不会**自动出现（即使 `nvidia-smi` 正常工作）。这是因为新版本 WSL 内核中 NVIDIA 内核模块采用**惰性设备节点创建**策略。**以 `nvidia-smi` 为准**——如果它有输出且显示 GPU 信息，GPU 透传即正常工作。

### 7. 验证 GPU 加速

**方式 A：PyTorch（需先安装）**
```bash
# 安装 PyTorch with CUDA
pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cu126

# 验证
python3 -c "import torch; print(torch.cuda.is_available()); print(torch.cuda.get_device_name(0))"
```

**方式 B：CUDA C 基础验证（不依赖 PyTorch，推荐优先使用）**
```bash
# 编译并运行一个小 CUDA 程序，验证 CUDA 运行时 + 驱动完整可用
cat > /tmp/cuda_test.cu << 'EOF'
#include <cuda_runtime.h>
#include <stdio.h>
int main() {
    int deviceCount = 0;
    cudaError_t err = cudaGetDeviceCount(&deviceCount);
    printf("CUDA device count: %d (err: %s)\n", deviceCount, cudaGetErrorString(err));
    if (deviceCount > 0) {
        cudaDeviceProp prop;
        cudaGetDeviceProperties(&prop, 0);
        printf("Device: %s\n", prop.name);
        printf("Memory: %.1f GB\n", prop.totalGlobalMem / 1024.0 / 1024.0 / 1024.0);
        // 测试内存分配
        float *d_data;
        cudaMalloc(&d_data, 1024 * sizeof(float));
        cudaMemset(d_data, 0, 1024 * sizeof(float));
        cudaFree(d_data);
        printf("CUDA compute: OK (malloc + memset + free passed)\n");
    }
    return 0;
}
EOF
nvcc -o /tmp/cuda_test /tmp/cuda_test.cu && /tmp/cuda_test
```

## 注意事项

- `.wslconfig` 在 Windows 端 `%UserProfile%`（即 `C:\Users\<用户名>`），**不是** WSL 内
- 修改 `.wslconfig` 后必须 `wsl --shutdown` 再启动才生效
- WSL 内 **不需要** 安装 NVIDIA 显卡驱动，Windows 端驱动包含 WSL GPU 内核模块
- `nvidia-utils` 版本可以低于 Windows 驱动版本，但不能高于
- `nvidia-smi` 在 WSL 重启前不可用 — 不要误判为安装失败
- 如果 `/dev/nvidia*` 在重启后仍不出现：可能是 Windows 驱动版本过旧（< 525.60），或是 WSL 内核版本过旧
- 重启 WSL 会终止所有 WSL 进程（包括当前会话）— 先通知用户保存工作

## 诊断 Ollama GPU 加速（Windows 端）

当 Ollama 推理速度异常慢（< 15 tok/s 对 8B 模型），或视觉模型无法处理图片时，问题通常是 **Ollama 未检测到 GPU**，全部推理在 CPU 上运行。

> **详细诊断参考**：`references/ollama-gpu-diagnostics.md`
>
> 包含完整日志片段、CUDA 13.2 兼容性分析、网络受限环境的升级策略。

### 诊断流程

```bash
# 1. 检查 nvidia-smi 是否有 Ollama 进程占用显存
nvidia-smi
# 正常：Ollama runner 进程应占用 4-7GB VRAM
# 异常：仅显示 Xwayland 等非 Ollama 进程

# 2. 测试模型推理速度
curl -s http://localhost:11434/api/generate \
  -d '{"model": "qwen3.5:9b", "prompt": "test", "stream": false}' \
  | python3 -c "import sys,json; d=json.load(sys.stdin); print(f'{d.get(\"eval_count\",0)/d.get(\"eval_duration\",1)*1e9:.1f} tok/s')"
# 正常（GPU）：> 40 tok/s
# 警告（CPU）：< 15 tok/s

# 3. 查看 Ollama 服务日志确认 GPU 检测
grep -E "inference compute|offloaded.*layer" /mnt/c/Users/%USERNAME%/AppData/Local/ollama/server.log
# 正常输出：
#   inference compute: id=cuda ... 
#   offloaded 36/37 layers to GPU
# CPU 模式（问题信号）：
#   inference compute: id=cpu  ← 只检测到 CPU！
#   offloaded 0/37 layers to GPU ← 零层卸载！
```

### 根因分析

| 日志信号 | 含义 | 解法 |
|----------|------|------|
| `id=cpu library=cpu` | GPU 未检测到 | 升级 Ollama 或检查驱动兼容性 |
| `offloaded 0/37 layers to GPU` | GPU 检测到但未使用 | 检查 `CUDA_VISIBLE_DEVICES` 环境变量 |
| `user overrode visible devices` | 用户手动限制了 GPU | 检查 `CUDA_VISIBLE_DEVICES` 设置 |
| `OLLAMA_NEW_ENGINE:false` | 使用旧版引擎 | 尝试设置 `OLLAMA_NEW_ENGINE=true` |

### 已知兼容性问题

**Ollama 0.22.x + CUDA 13.2 驱动（NVIDIA 595.97）不兼容**：
- 症状：所有模型在 CPU 上运行，VRAM 仅 1.5GB，8B Q4_K_M 推理仅 9.9 tok/s
- 根因：Ollama 内置 CUDA 运行时较旧，无法调用新版 CUDA 13.2 API
- 修复：升级 Ollama 到最新版（详见 `references/ollama-gpu-diagnostics.md`）

### 升级 Ollama

详见 `references/ollama-gpu-diagnostics.md` → 三种方法（浏览器/PowerShell/批处理）+ 网络受限策略

**已验证成功的升级路径**（RTX 4070, CUDA 13.2, NVIDIA 595.97）：
```
Ollama 0.22.1 → 0.23.3 (升级后验证)
├── 推理速度: 9.9 tok/s (CPU) → 21.3 tok/s (GPU) ↑2.15x
├── GPU检测: "offloaded 0/37 layers to GPU" → GPU正常参与
├── 视觉模型: qwen3-vl:8b CPU超时35s → qwen3.5-256k可用(21.3tok/s)
└── nvidia-smi: 显存使用从1.5GB升至正常范围
```

### 升级后验证

```bash
# 1. 确认推理速度恢复正常
curl -s http://localhost:11434/api/generate \
  -d '{"model": "qwen3:8b", "prompt": "test", "stream": false}' \
  | python3 -c "import sys,json; d=json.load(sys.stdin); print(f'{d.get(\"eval_count\",0)/d.get(\"eval_duration\",1)*1e9:.1f} tok/s')"

# 2. 确认日志显示 GPU 加速
grep "inference compute" /mnt/c/Users/%USERNAME%/AppData/Local/ollama/server.log | tail -3

# 3. 测试视觉模型处理图片
python3 -c "
import base64, json, urllib.request
with open('test_image.jpg', 'rb') as f:
    b64 = base64.b64encode(f.read()).decode()
payload = json.dumps({'model': 'qwen3-vl:8b', 'prompt': '描述图片', 'images': [b64], 'stream': False}).encode()
req = urllib.request.Request('http://localhost:11434/api/generate', data=payload, headers={'Content-Type': 'application/json'})
resp = urllib.request.urlopen(req, timeout=120)
d = json.loads(resp.read())
print(d.get('response', 'N/A'))
"
```

## 常见故障

| 现象 | 原因 | 解法 |
|------|------|------|
| `nvidia-smi` 在 WSL 内找不到 | WSL 未重启 | `wsl --shutdown` 后重开 |
| `.wslconfig` 不生效 | 路径不对 | 确认在 `C:\Users\<用户名>\.wslconfig` |
| 16GB 内存出不来 | `.wslconfig` 语法错误 | 用 `wsl --shutdown` 重启检查 |
| CUDA 编译报错 | CUDA Toolkit 版本与 nvidia-utils 不匹配 | 统一用 12.x 系列 |
| `/dev/nvidia*` 重启后仍缺失 | 驱动版本过旧 | 更新 Windows 端驱动到 525.60+ |
| Ollama 推理 < 15 tok/s | Ollama 未检测到 GPU（CPU 模式） | 见上文「诊断 Ollama GPU 加速」 |
| 视觉模型处理图片无限超时 | GPU 未加速 + 图片编码耗 CPU | 先修复 Ollama GPU 检测 |
