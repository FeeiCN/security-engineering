---
slug: /ai-projects
title: AI 项目
icon: library-icon
description: 通过开源项目和真实实践，检验 AI 技术是否解决了工作流中的关键瓶颈，以及方法能否安装、复现和维护。
---

# AI 项目

## 研究方法

研究 AI 开源项目，先看它处在什么层，再看它有没有真的解决生产里的关键环节。Star 只能说明关注度，不能说明可用性。

- **模型层**：本地运行、推理加速、量化、服务化，例如 [Ollama](https://github.com/ollama/ollama)、[vLLM](https://github.com/vllm-project/vllm)
- **工作流层**：把模型接到任务循环、工具调用和编排系统，例如 [LangGraph](https://github.com/langchain-ai/langgraph)、[OpenHands](https://github.com/OpenHands/OpenHands)
- **知识层**：把文档、数据库和检索接进上下文，例如 [RAGFlow](https://github.com/infiniflow/ragflow)
- **编辑器与终端层**：让 AI 直接进入开发工作区，例如 [Continue](https://github.com/continuedev/continue)
- **协议层**：把模型和外部工具、数据源接起来，例如 [MCP](https://github.com/modelcontextprotocol/modelcontextprotocol)

判断一个项目值不值得追，重点看四件事：

- 是否卡住了关键链路，替代了最重的那一步人工操作
- 是否有清晰的安装方式、API、示例和文档
- 是否持续活跃，release、issue 和 PR 的处理是否稳定
- 是否沉淀了自己的工作流或数据，而不只是依附某个更强模型

## FEEI.CN

[FEEI.CN](https://github.com/FeeiCN/FEEI.CN) 是本站的源码，基于 Docusaurus + Markdown 构建的个人知识站点，内容涵盖健康幸福、事业有成、财务自由、人生丰富四个方向。

开发、内容更新、功能迭代全部通过 Claude Code 完成，AI Coding 100%。部署链路：提交代码 → GitHub Action 构建 → Nginx 提供静态文件。

## slide-writer

[slide-writer](https://github.com/FeeiCN/slide-writer) 是一个 Claude Code Skill，将想法、大纲、演讲稿或草稿转换为企业级 HTML 演示文稿。

支持 14 种品牌主题（阿里、腾讯、字节等），自动识别并应用对应的视觉风格。输出单一 HTML 文件，CSS、JS 和图片全部内嵌，无需任何依赖，浏览器直接打开。

## antding-cli

[antding-cli](https://github.com/FeeiCN/antding-cli) 逆向分析 Antding（钉钉 Mac 定制版）本地 SQLite 数据库的加密方式，离线解密并将聊天记录导出为 Markdown。

通过 Frida hook `CCCrypt` 在运行时抓取解密 key，绕过 Hardened Runtime 限制，支持全库解密与按会话导出。AI Coding 100%。

## auto-translate

[auto-translate](https://github.com/FeeiCN/auto-translate) 是一个 Chrome 插件，支持在 X（Twitter）时间线、YouTube 字幕和 abc.com 视频字幕上自动翻译内容，免费使用、无需任何翻译 API Key。

基于 OpenAI Codex 完成开发与迭代，AI Coding 100%。

**功能：**

- 自动识别并翻译 X 时间线中的推文正文和文章卡片标题
- YouTube 字幕稿翻译，右侧逐句展示英文与中文译文
- abc.com 视频字幕中英双语化
- 多翻译源自动降级（MyMemory → Google GTX → LibreTranslate）
- 翻译缓存与请求节流，降低接口压力
- 自动隐藏广告推文
- 右键菜单支持：选中文字或图片一键发送到 Grok 分析

## 如何继续看

开源 AI 项目不是同一条赛道上的竞争者。`Ollama`、`vLLM`、`RAGFlow`、`LangGraph`、`OpenHands`、`Continue`、`MCP` 分别落在模型、知识、工作流、开发环境和协议的不同层，重点不是谁更“热”，而是谁更接近真实工作流里的瓶颈位置。
