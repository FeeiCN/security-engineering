---
slug: /llms-from-scratch
title: 从零构建大语言模型
sidebar_position: 0
pagination_next: 网络安全/人工智能安全/通用人工智能/AI基础认知/从零构建大语言模型/从零认识大语言模型
icon: book-icon
description: 沿七章学习笔记，从文字编码、注意力与 GPT 结构读到预训练、短信分类微调和指令微调，理解数据、参数与生成的关系。
---

# 从零构建大语言模型

这组笔记按 Sebastian Raschka 的《Build a Large Language Model (From Scratch)》前七章梳理 GPT 的工作过程，面向能阅读简单程序、希望先理解概念和数据流的读者。它与[模型训练与推理](/model-training-inference)的概览互补，按下面的顺序逐步展开。

| 章节 | 解决的问题 |
| --- | --- |
| 第1章：[从零认识大语言模型](/llms-from-scratch/chapter-1) | 模型怎样从数据中学习并逐步生成文字？ |
| 第2章：[文字怎样变成模型的输入](/llms-from-scratch/chapter-2) | 字符串怎样变成 token、训练样本和向量？ |
| 第3章：[注意力如何利用上下文](/llms-from-scratch/chapter-3) | 当前位置怎样从允许参考的前文中取得信息？ |
| 第4章：[把注意力组装成完整的 GPT](/llms-from-scratch/chapter-4) | 多个部件怎样组成能输出词表分数的 GPT？ |
| 第5章：[让 GPT 从预测错误中学习](/llms-from-scratch/chapter-5) | 怎样计算损失、更新参数并检查过拟合？ |
| 第6章：[把 GPT 微调成短信分类器](/llms-from-scratch/chapter-6) | 怎样把续写模型改成固定类别的分类器？ |
| 第7章：[让 GPT 学会按照指令回答](/llms-from-scratch/chapter-7) | 怎样用指令与参考回答训练生成模型？ |
