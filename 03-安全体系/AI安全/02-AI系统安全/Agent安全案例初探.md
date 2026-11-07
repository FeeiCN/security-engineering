---
slug: /agent-security-case-notes
title: Agent 安全案例初探
description: 从 Semantic Kernel 与 AutoJack 的公开研究看模型可控参数如何穿越工具、沙箱和本地服务边界。
published_at: '2026-10-25'
---

# Agent 安全案例初探

首批案例来自公开的一手研究材料，记录攻击成立条件与信任边界。本文仅完成公开资料分析，**没有进行本地 PoC 复现**；披露演示不等于生产环境实际入侵。系统性控制原则见[Agent 与工具调用安全](/ai-agent-tool-security)，证据要求见[AI Agent 安全研究方法](/ai-agent-security-research)。

## Semantic Kernel：模型参数进入动态执行

**CVE-2026-26030** 涉及 Semantic Kernel Python SDK 的 In-Memory Vector Store 过滤功能。微软披露的必要条件是：攻击者能够影响 Agent 输入；目标 Agent 使用相关 Search Plugin 和默认过滤配置；运行版本低于 1.39.4。模型控制的参数被拼接到 Python lambda 表达式并交给 `eval()`，原有 AST 阻断逻辑仍可被绕过。修复建议是升级至 1.39.4 或更高版本。

关键边界是 **不可信自然语言 → 模型生成的工具参数 → 可执行表达式**。模型是否识别恶意意图并不是最终执行边界；即使工具调用结构合法，参数仍须按照不可信输入处理。

**CVE-2026-25592** 涉及 Semantic Kernel .NET SDK 的 SessionsPythonPlugin。微软披露，原本用于从隔离会话下载文件的 `DownloadFileAsync` 意外暴露为模型可调用工具，且宿主机目标路径缺少必要约束。攻击链可使 Agent 将沙箱内文件写到宿主机指定位置；在研究演示中，写入 Windows 启动目录可能导致后续登录时执行。微软说明受影响版本为低于 1.71.0 的 .NET SDK，修复包括撤销该工具的模型调用暴露、规范化路径并限制允许目录。

这里的边界是 **沙箱内文件 → 宿主文件系统**。沙箱隔离并未失效于容器本身，而是被具有宿主副作用的工具接口绕开。

来源：[Microsoft Security Research，2026-05-07](https://www.microsoft.com/en-us/security/blog/2026/05/07/prompts-become-shells-rce-vulnerabilities-ai-agent-frameworks/)。

## AutoJack：浏览器内容跨越 localhost 控制面

微软于 2026-06-18 披露 AutoGen Studio 开发分支中的 AutoJack 攻击链：不可信网页内容经浏览 Agent 触达本地 MCP WebSocket，进而可能在宿主执行进程。研究团队称上游通过提交 `b047730` 加固，且**受影响的 WebSocket 接口未进入 PyPI 发布版本**，不能据此推断普通 PyPI 安装者受该特定攻击链影响。

关键边界是 **网页内容 → Agent 行为 → 本地高权限服务**。localhost 不是授权机制；本地控制面仍需身份认证、动作授权和网络隔离。

来源：[Microsoft Security Research，2026-06-18](https://www.microsoft.com/en-us/security/blog/2026/06/18/autojack-single-page-rce-host-running-ai-agent/)。

## 如何验证防御是否有效

上述案例提示三类需要独立测试的控制：

1. **工具参数约束**：不允许模型生成的字符串直接进入动态代码执行、Shell 或不受限路径。
2. **副作用边界**：沙箱文件传输、宿主写入、网络出站和本地控制面分别进行资源侧授权。
3. **模型外强制**：Prompt 约束、工具描述和模型拒答不能替代参数校验、沙箱、凭证隔离与审计。

评估时同时记录攻击成功率和正常任务完成率。可参考 [AgentDojo（NeurIPS 2024）](https://proceedings.nips.cc/paper_files/paper/2024/hash/97091a5177d8dc64b1da8bf3e1f6fb54-Abstract-Datasets_and_Benchmarks_Track.html) 与 [AgentDyn（2026）](https://arxiv.org/abs/2602.03117)。两者是研究评测基准，**不是本页所述生产漏洞的实际入侵证据**。

## 待完成

- 对照 CVE/GHSA 和修复提交核验精确受影响版本及修复差异。
- 在授权隔离环境中复现，并保留环境、配置、结果和失败条件。
- 扩展案例库并按信任边界去重；每条案例单独维护证据等级与验证状态。
