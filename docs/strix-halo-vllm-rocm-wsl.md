# 在 Strix Halo（Radeon 8060S，gfx1151）上尝试 WSL2 + vLLM ROCm 的经验与踩坑

> 设备：AMD Ryzen AI MAX+ 395 w/ Radeon 8060S（gfx1151）  
> 系统：Windows 11 Pro build 26200 + WSL2 Ubuntu 24.04  
> 目标：在 WSL2 内用 Docker 跑 vLLM ROCm 版  
> 结论：**未成功稳定运行，WSL2 不是当前可行路径**

---

## 1. 为什么选这条路？

Strix Halo 的统一内存架构对本地大模型推理很有吸引力：128 GB 共享内存可以让大模型直接跑在核显上。vLLM 是 serving 场景的事实标准，而 AMD 从 ROCm 7.2 开始通过 **ROCDXG** 方案支持 WSL2，看起来是一条能走通的路线。

## 2. ROCDXG 是什么？

**ROCDXG（librocdxg）** 是 AMD 为了让 ROCm 程序能在 WSL2 里跑而做的用户态兼容层。

- 原生 Linux：ROCm → `/dev/kfd` → Linux 内核显卡驱动 → GPU
- WSL2：ROCm → `librocdxg` → `/dev/dxg`（DXCore）→ Windows 主机 AMD 驱动 → GPU

WSL2 里没有 `/dev/kfd`，所以必须靠 librocdxg 把调用翻译成 Windows 驱动能理解的 DXCore 调用。

## 3. 我们走到了哪一步？

### 3.1 基础环境打通 ✅

- 修复了损坏的 WSL 安装（从 GitHub release 重装 2.9.4）
- 装了 Ubuntu 24.04、Docker 29.1.3
- 在 Ubuntu 里装了 ROCm 7.2.4 用户态 + `librocdxg` 1.2.0
- 手动补了 `dids.conf`
- 验证：容器内 `rocminfo` 能看到 `gfx1151`，PyTorch 能跑 GPU `matmul`

### 3.2 拉镜像 ✅

| 镜像 | 来源 | 大小 | 速度 |
|---|---|---|---|
| `vllm/vllm-openai-rocm:v0.21.0` | 华为 ddn-k8s 镜像站 | 51.8 GB | ~9.6 MB/s |
| `vllm/vllm-openai-rocm:v0.25.1` | `docker.1ms.run` | 44.7 GB | ~31 MB/s |

> 踩坑：华为站没有 v0.25.1 tag；daocloud 在 WSL 里只有 ~0.6 MB/s；`docker.1ms.run` 是目前最快的国内源。

### 3.3 代码层补丁 ✅

vLLM 默认高度依赖 `amdsmi`，而 WSL 下没有 `/dev/kfd`，`amdsmi_init()` 必然失败。我们打了三处补丁：

1. `vllm/platforms/__init__.py`：平台探测直接通过 `/dev/dxg` + `HSA_ENABLE_DXG_DETECTION=1` 认定 ROCm
2. `vllm/platforms/rocm.py`：`_get_gcn_arch`、`get_device_name`、`get_device_uuid` 改用 `torch.cuda`
3. `vllm/logger.py`（v0.25.1 需要）：绕开日志引起的循环导入

### 3.4 服务启动 ✅ 但推理崩溃 ❌

打完补丁、降低显存预算（`--gpu-memory-utilization 0.3`）、关闭 CUDA graph（`--enforce-eager`）后，vLLM 0.21.0 能启动到 `Application startup complete.`。

但一进入 EngineCore/KV cache/模型执行阶段，就出现：

```
Exited (255)
```

或者更糟——整个 WSL 虚拟机重启。`dmesg` 里持续出现：

```
misc dxg: dxgk: dxgkio_query_adapter_info: Ioctl failed: -22
```

### 3.5 尝试 Lemonade SDK 预构建包 ❌

社区有专门为 gfx1151 验证过的 Lemonade SDK 预构建包（`vllm0.21.0-rocm7.13.0-gfx1151`）。我们下完、解压、补依赖、打补丁后，同样看到 vLLM 启动 banner，但随后 WSL VM 还是崩溃重启。

## 4. 崩溃的根本原因（一句话）

> vLLM 的 ROCm 后端在 WSL2 里一上负载，就把 AMD-Windows 的 DXCore/dxg 透传路径跑崩了，导致整个 WSL 虚拟机重启。

这不是镜像、参数或补丁能彻底解决的，是 **WSL2 + ROCDXG + Strix Halo(gfx1151)** 这条技术路径目前的底层稳定性问题。

## 5. AMD 官方怎么说？

查过 [ROCm 7.14.0 兼容矩阵](https://rocm.docs.amd.com/en/latest/compatibility/compatibility-matrix.html)：

- Ryzen AI Max 300 系列 / Radeon 8060S（gfx1151）官方支持：
  - **Ubuntu 24.04.4 / 26.04**
  - **Windows 11 25H2**
- **WSL2 不在支持列表里**

所以用 WSL2 跑 vLLM ROCm 属于官方未验证环境，出问题不意外。

## 6. 可行的替代方案

| 方案 | 是否 vLLM | 平台 | 备注 |
|---|---|---|---|
| **原生 Ubuntu 双系统** | ✅ 是 | Linux | 官方支持路径，最稳 |
| **ONNX Runtime + DirectML** | ❌ 不是 | Windows | 原生 Windows，用 iGPU 推理 |
| **llama.cpp Windows 版（ROCm/Vulkan）** | ❌ 不是 | Windows | 社区构建，跑 GGUF |
| **AMD Ryzen AI SDK / AMUSE** | ❌ 不是 | Windows | AMD 官方 GUI 工具，针对 Strix Halo 优化 |
| **Ollama Windows 版** | ❌ 不是 | Windows | 底层 llama.cpp，安装简单 |

## 7. 给后来者的建议

1. 如果你**必须用 vLLM**：直接上原生 Linux，别在 WSL2 上浪费时间。
2. 如果你只要**Windows 上能跑本地大模型**：优先试 Ollama / AMUSE / llama.cpp。
3. 如果你**想等 WSL2 成熟**：关注 `ROCm/ROCm#4952`、`microsoft/WSL#40321` 这两个 issue。

## 8. 关键文件/命令备忘

```bash
# 验证 GPU 是否通过 ROCDXG 识别
export HSA_ENABLE_DXG_DETECTION=1
/opt/rocm/bin/rocminfo | grep gfx

# 最快的国内 vLLM ROCm 镜像源
docker pull docker.1ms.run/vllm/vllm-openai-rocm:v0.25.1

# 容器启动参数（最终失败前用的配置）
docker run -d --name vllm-serve \
  --device=/dev/dxg \
  --cap-add=SYS_PTRACE \
  --security-opt seccomp=unconfined \
  -v /usr/lib/wsl/lib/libdxcore.so:/usr/lib/libdxcore.so \
  -v /opt/rocm/lib/librocdxg.so.1.2.0:/usr/lib/librocdxg.so \
  -v /opt/rocm/share/rocdxg/dids.conf:/usr/share/rocdxg/dids.conf \
  -v <patched_platforms_init.py>:/usr/local/lib/python3.12/dist-packages/vllm/platforms/__init__.py:ro \
  -v <patched_rocm.py>:/usr/local/lib/python3.12/dist-packages/vllm/platforms/rocm.py:ro \
  -e HSA_ENABLE_DXG_DETECTION=1 \
  -e HF_ENDPOINT=https://hf-mirror.com \
  -e PYTHONUNBUFFERED=1 \
  -p 8000:8000 --ipc=host --shm-size 8G \
  vllm-rocm:local \
  --model Qwen/Qwen3-0.6B \
  --gpu-memory-utilization 0.3 \
  --enforce-eager \
  --max-model-len 512
```

---

*最后更新：2026-07-24*
