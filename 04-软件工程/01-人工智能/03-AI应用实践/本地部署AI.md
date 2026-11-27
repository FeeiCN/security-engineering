---
slug: /local-ai-deployment
title: 本地部署 AI
icon: cpu-icon
description: 本地 AI 选型需要同时匹配硬件、权重格式、上下文和并发；能启动只是起点，还要测试质量、速度、内存、API 与安全边界。
last_reviewed: '2026-07-10'
---

# 本地部署 AI

本地部署的价值是控制模型版本、运行环境和数据路径。它也会将硬件、驱动、模型格式、容量、监控和安全更新交给自己。

本页的目标是帮助读者选择一条当前可用的推理路线，并用同一组指标判断它能否承担真实任务。工具安装、模型标签和硬件支持变化很快，执行前仍应查看链接的官方文档。

## 先确认本地部署是否值得

适合优先评估本地部署的场景包括：

- 原始数据不允许发送到外部模型服务。
- 必须在断网、内网或固定版本环境中运行。
- 需要反复调用开源模型，并愿意承担硬件和维护成本。
- 需要控制量化、推理参数、模型路由或服务更新节奏。

如果只是低频使用、需要前沿大模型，或不希望维护推理基础设施，先试用[商业 AI](/commercial-ai) 通常更容易得到结论。本地运行也不会自动满足隐私和合规要求，日志、界面、插件、工具和网络出口仍需要单独审查。

## 用四个约束选模型与框架

**权重内存。** 量化是用更低精度保存模型权重，以减少内存和计算开销；`4-bit` 表示每个权重按 4 位估算，实际质量和框架兼容性仍要测试。只计权重的粗略下界可以写成：

$$
M_{\text{weights}}\ (\mathrm{GB}) \approx \frac{N_{\text{B}}\times b}{8}
$$

其中 $N_{\text{B}}$ 是以十亿为单位的参数量，$b$ 是每个权重的位数。

:::note[权重下界不等于设备需求]

例如，8B 模型的 4-bit 权重原始下界约为 4GB。实际运行还需要程序、中间激活、KV cache 和系统预留，所以不能把权重下界当成设备需求。

:::

**模型格式。** 格式规定权重和相关元数据怎样存储，推理框架必须明确支持。`GGUF` 常用于 llama.cpp 生态；Hugging Face 权重和 MLX 转换版采用其他加载路径，不能随意混用。

**上下文与并发。** KV cache 是生成过程中保存注意力中间状态的内存；上下文越长、同时请求越多，它通常占用越高，调度压力也越大。评估必须使用真实上下文长度和并发量。

**平台与驱动。** CUDA 和 ROCm 分别是 NVIDIA、AMD GPU 常用的计算软件栈。Apple Silicon、NVIDIA CUDA、AMD ROCm 和纯 CPU 路线的支持范围不同。先确认操作系统、架构、驱动和 Python 版本，再执行安装命令。

## 按使用场景选择路线

**个人电脑快速试用：Ollama 或 LM Studio。** Ollama 提供简单 CLI 和本地 API；LM Studio 提供图形界面、模型管理和 OpenAI 兼容服务。前者适合终端与脚本，后者适合希望在界面中完成下载、加载和参数调整的用户。

**Apple Silicon 专用实验：MLX-LM。** MLX-LM 面向 Apple Silicon 的统一内存架构，适合命令行生成、量化实验和本地 API 试用。其内置 server 官方只定位为基础开发用途，不应直接承担生产服务。

**Linux 服务器与并发 API：vLLM。** vLLM 的稳定快速开始以 Linux 和特定加速器路线为前提，适合将已支持模型暴露为 OpenAI 兼容服务。Mac 用户不应直接套用 CUDA 安装命令。

**底层控制与 GGUF：llama.cpp。** llama.cpp 支持多种 CPU 和 GPU 后端，可以直接控制模型文件、GPU offload（把部分模型层放到 GPU 计算）、上下文和服务参数。各平台安装方式不同，应跟随官方 README 选择对应后端。

## 个人电脑的最小链路

### Ollama

按 [Ollama Quickstart](https://docs.ollama.com/quickstart) 为 macOS、Windows 或 Linux 安装对应版本，然后运行官方示例模型：

```bash
ollama run gemma4
```

安装后的本地 API 默认在 `http://localhost:11434/api`，接口说明见 [Ollama API](https://docs.ollama.com/api/introduction)。将它接入其他工具前，先确认模型能在本地对话、卸载后可重新加载，并记录真实内存和速度。

### LM Studio

安装并至少启动一次 LM Studio 后，可以使用随应用安装的 `lms` CLI。下面的 `<model-name>` 和 `<model-key>` 需要替换为实际模型：

```bash title="LM Studio 下载、加载与 API 验证" {1,3}
lms get <model-name>
lms ls
lms load <model-key> --identifier local-model
lms server start --port 1234
curl http://localhost:1234/v1/models
```

显式设置端口可以避免 CLI 沿用上次服务端口。系统前提、CLI 和 OpenAI 兼容接口分别见 [LM Studio 系统要求](https://lmstudio.ai/docs/app/system-requirements)、[CLI](https://lmstudio.ai/docs/cli) 和 [Local Server](https://lmstudio.ai/docs/developer/core/server)。

## 需要更多控制时的路线

### llama.cpp

根据 [llama.cpp README](https://github.com/ggml-org/llama.cpp) 安装适合当前平台的版本后，可以直接读取 GGUF 文件：

```bash
llama-cli -m model.gguf
llama-server -m model.gguf --port 8080
```

服务启动后，可先检查 `http://localhost:8080`。不同版本的 server API 会变化，集成前使用当前 README 和 changelog 核对路径。

### MLX-LM

当前官方稳定说明面向 Apple Silicon、macOS 14+ 和原生 Python 3.10+。安装并使用同一模型进行命令行与 API 验证：

```bash
pip install mlx-lm
mlx_lm.generate \
  --model mlx-community/Mistral-7B-Instruct-v0.3-4bit \
  --prompt "请用一句话解释本地模型。"
mlx_lm.server --model mlx-community/Mistral-7B-Instruct-v0.3-4bit
curl http://localhost:8080/v1/models
```

详细参数与 server 安全边界见 [MLX-LM](https://github.com/ml-explore/mlx-lm) 和 [Server 文档](https://github.com/ml-explore/mlx-lm/blob/main/mlx_lm/SERVER.md)。

## Linux 服务器的并发路线

以 [vLLM 稳定版 Quickstart](https://docs.vllm.ai/en/stable/getting_started/quickstart/) 当前核对的 NVIDIA CUDA 路线为例，先在独立 Python 环境安装，再启动一个小模型验证 API：

```bash
uv venv --python 3.12 --seed
source .venv/bin/activate
uv pip install vllm --torch-backend=auto
vllm serve Qwen/Qwen2.5-1.5B-Instruct
curl http://localhost:8000/v1/models
```

这组命令不适用于 Apple Silicon、AMD ROCm、TPU 或其他加速器。目标平台不同时，使用 vLLM 官方的对应安装路线。

## 用同一套指标验收

“能启动”只能证明权重和框架在最小输入下工作。正式选型前，在每条路线上使用相同的真实任务集，记录：

- 经外部验收的任务成功情况，以及多次运行的波动。
- 首字延迟、输出速度、总时间和长任务稳定性。
- 模型加载、真实上下文和目标并发下的内存峰值。
- 重启后能否恢复，请求失败时是否有明确错误和健康检查。
- OpenAI 兼容客户实际需要的 endpoint、流式输出、工具调用和返回字段是否支持。
- 模型权重、代码、量化和商业使用是否符合各自许可证。

需要图形界面时，可将推理后端连接到 [Open WebUI](https://docs.openwebui.com/getting-started/quick-start/)。标准 Open WebUI 是界面和应用层，不自动包含已配置的推理引擎。快速开始的 `:main` 是浮动标签，需要可复现时应固定发布版本。

![Ollama 与 Open WebUI 界面](/media/本地部署AI/Ollma-with-Open-WebUI.jpg "Ollama 与 Open WebUI 组合的本地界面。")

## 常见故障与安全边界

**无法加载。** 检查权重格式、架构支持、可用内存和量化；减小模型、上下文或并发后重新测试。

**速度过慢。** 确认是否真正使用目标 GPU 后端，分别测量模型加载、首字和持续输出。调整量化、GPU offload、上下文和并发时，每次只改一个变量。

**输出异常。** 核对分词器、聊天模板、系统提示和量化版本。模型能生成文字并不证明客户端协议和模板已经正确匹配。

**API 无法连接。** 核对监听地址、端口、base URL、路径和容器网络。开发阶段默认绑定 `127.0.0.1`，不用一个公网端口来解决本地连接问题。

:::warning[共享服务的安全边界]

团队共享服务需要身份认证、最小权限、TLS、请求限制、日志脱敏、密钥轮换和网络隔离。不将 Ollama、MLX-LM、llama.cpp 或 Open WebUI 的开发端口直接暴露到公网。更完整的威胁模型见 [AI 安全](/ai-security)。

:::
