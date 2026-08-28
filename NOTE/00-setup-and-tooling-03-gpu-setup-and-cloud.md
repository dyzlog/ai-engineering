# 00-setup-and-tooling · 03-gpu-setup-and-cloud 学习笔记

> 科目：AI 工程课程（ai-engineering-from-scratch）
> 阶段：Phase 0 Setup & Tooling · 第三小节 GPU Setup & Cloud
> 类型：Build · 语言：Python
> 日期：2026-08-28
> 状态：✅ 完成（本机无独显，走 Colab 云端 T4 路线）

---

## 0. 核心问题：训练需要 GPU

- Phase 1-3（数学/ML 基础）**CPU 完全够跑**
- Phase 4+（CNN/Transformer/LLM 训练）需要 GPU：CPU 8 小时 → GPU 10 分钟
- 三大选项：本地 GPU（$0）/ Google Colab 免费 T4（$0）/ 云 GPU（$0.2-2/时）

## 1. 本机结论：无独显

```bash
nvidia-smi   # → Command not found = 无 NVIDIA 独显（华为笔记本）
```

**不要** `sudo apt install nvidia-utils-*`（没卡装了没用）。CPU 跑学习，重训练用 Colab。

## 2. Google Colab 免费 T4 配置（本机方案）

1. 打开 colab.research.google.com（Google 账号登录）
2. **Runtime → Change runtime type**（笔记本设置）
3. **硬件加速器** 下拉 → 选 **T4 GPU**（免费；H100/A100/L4 等要付费，别选）
4. 其他保持默认（Python 3 / 最新运行时）
5. 点**保存**，等右上角显示 T4 已连接
6. 验证：`!nvidia-smi`（注意**感叹号前缀**，Colab 里跑 shell 命令要用 !）

### 本机实测 nvidia-smi 输出解读

```text
|   0  Tesla T4                       Off |   00000000:00:04.0 Off |                    0 |
| N/A   48C    P8             12W /   70W |       0MiB /  15360MiB |      0%      Default |
+-----------------------------------------------------------------------------------------+
|  No running processes found                                                             |
```

| 字段 | 值 | 含义 |
|---|---|---|
| GPU Name | Tesla T4 | 免费额度给的显卡型号 |
| Driver | 580.82.07 | 驱动版本 |
| CUDA Version | 13.0 | 驱动支持的最高 CUDA |
| Memory-Usage | 0MiB / 15360MiB | 已用 0 / 总 15GB |
| GPU-Util | 0% | 空闲 |
| Processes | No running | 无进程占用 |

## 3. PyTorch 云端验证（本机实测）

```python
import torch
print(f"PyTorch: {torch.__version__}")            # 2.11.0+cu128
print(f"CUDA available: {torch.cuda.is_available()}")  # True
print(f"GPU: {torch.cuda.get_device_name(0)}")    # Tesla T4
props = torch.cuda.get_device_properties(0)
print(f"Memory: {props.total_memory / 1e9:.1f} GB")    # 15.6 GB
print(f"fp16 最大模型: ~{props.total_memory / 1e9 / 2:.0f}B 参数")  # ~8B
```

（Colab 默认没 torch 时先 `!pip install torch`）

## 4. 本机 GPU 检查脚本（课程提供）

```bash
cd ~/projects/ai-engineering
python phases/00-setup-and-tooling/03-gpu-setup-and-cloud/code/gpu_check.py
```

预期（本机无 GPU）：PyTorch version + CUDA available: False + "No GPU detected. That's fine for most lessons." + CPU 基准数据。

## 5. CPU vs GPU 基准测试

```python
import torch, time
size = 5000
a, b = torch.randn(size, size), torch.randn(size, size)

start = time.time(); c = a @ b; cpu_time = time.time() - start
print(f"CPU: {cpu_time:.3f}s")

if torch.cuda.is_available():
    a_g, b_g = a.to("cuda"), b.to("cuda")
    torch.cuda.synchronize()          # ★ 必须：等 GPU 运算真正完成再计时
    start = time.time(); c = a_g @ b_g
    torch.cuda.synchronize()
    gpu_time = time.time() - start
    print(f"GPU: {gpu_time:.3f}s  Speedup: {cpu_time / gpu_time:.0f}x")
```

**为什么 GPU 计时前要 `torch.cuda.synchronize()`**：GPU 运算是**异步**的，不等待就会把"还没算完"的时间计进去，测不准。

## 6. 关键概念速记

| 术语 | 含义 |
|---|---|
| CUDA | NVIDIA 并行计算平台（让代码在 GPU 上跑） |
| VRAM | GPU 显存，**与系统内存分开**，决定模型上限 |
| fp16 | 半精度，内存是 fp32 一半，精度损失极小 |
| Tensor Core | GPU 矩阵乘法专用核心，比普通核快 4-8 倍 |

## 7. fp16 显存估算规则（quiz 必考）

> **2 字节/参数**：参数量(B) = VRAM(GB) ÷ 2

| VRAM | 可装模型（fp16） |
|---|---|
| 15.6 GB（T4） | ~8B 参数 |
| 24 GB | ~12B 参数（quiz 答案） |
| 80 GB（A100） | ~40B 参数 |

## 8. 本机"GPU 双轨"策略

```text
本地 WSL：无 GPU → CPU 跑（Phase 1-3 学习实验）
云端 Colab：T4 免费 → 重训练（Phase 4+ 用）
```

- 平时 CPU 学习零成本，需要 GPU 打开 Colab 切 T4；
- **现阶段不买云 GPU**（Lambda/RunPod 等 $0.2-2/时，等真做训练再说）；
- 免费 T4 每日有额度，用完会提示等待或购买。

## 9. 记忆口诀

1. **nvidia-smi 没有 = 没独显**，正常，别装驱动包；
2. **Colab 三步**：Runtime → Change runtime type → T4 GPU → 保存；
3. **Colab 跑 shell 命令要加 !**（如 !nvidia-smi）；
4. **GPU 计时前先 synchronize**，否则测的是"排队时间"；
5. **fp16 规则**：显存 GB ÷ 2 = 能装多少 B 参数。
