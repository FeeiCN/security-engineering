---
slug: /ai-crown-jewels-infrastructure-security
title: AI 核心资产与基础设施安全
sidebar_position: 2
icon: lock-icon
description: 从核心资产价值和攻击者能力出发，保护模型权重、训练方法、研究知识以及训练与推理基础设施，控制访问、外传与单点失陷的爆炸半径。
published_at: '2026-09-23'
---

# AI 核心资产与基础设施安全

AI 系统的核心资产不只有模型文件。对能力领先或业务关键的系统，真正需要保护的是一组能够复制、恢复或显著推进能力的资产：

- **Weights**：基础权重、Checkpoint、Adapter、量化版本和可恢复副本。
- **Recipe**：训练与后训练方法、超参数、数据配比、RL / Synthetic Data 方法和未公开架构。
- **Research Knowledge**：实验结果、失败经验、Notebook、内部论文、路线判断和下一步研究方向。
- **Data / Eval**：高价值训练数据、私有评测集、能力与安全边界测试结果。
- **Infrastructure Knowledge**：训练拓扑、集群管理面、部署架构、内部工具和访问路径。

模型供应链安全主要回答“生产运行的是不是经过验证的那组制品”；这里重点回答另一个问题：**攻击者能否取得足够多的核心资产，从而复制能力、窃取研究成果或控制承载这些资产的基础设施？**

## 从资产价值和攻击者能力决定 Assurance

安全基线不应只由“我们是一家互联网公司还是 AI 公司”决定。更可操作的模型是：

> **Asset Criticality × Adversary Capability → Required Assurance Level**

威胁分层可以从普通机会型攻击者、组织化网络犯罪，逐渐提升到资源充足的专业攻击团队和国家级对手。具体组织处于哪一层，需要结合资产价值、暴露面、历史情报和业务环境判断，而不是因为使用了大模型就默认采用最高等级。

当 Threat Tier 上升，控制往往不是线性增加几个安全产品，而是发生架构变化：更严格的 Need-to-know、更强的环境隔离、多方授权、更少的外部依赖、更高的物理和人员安全要求，以及能够针对目标攻击者反复验证的红队体系。

Leopold Aschenbrenner 在《[Lock Down the Labs](https://situational-awareness.ai/lock-down-the-labs/)》中以高价值前沿模型为背景讨论了这种威胁升级。文章包含作者对未来 AI 与地缘政治的判断；这些判断不作为本文事实前提。这里保留的是通用安全方法：**威胁模型要覆盖真正有能力、也有动机获取核心资产的攻击者。**

## 权重和算法秘密需要不同保护

Weights 是集中而巨大的制品，通常经过训练存储、Checkpoint、Registry、推理加载和备份链路；Recipe 与 Research Knowledge 更分散，可能存在于 Git、Notebook、实验平台、文档、聊天、邮件和人的工作环境中。

因此不能用“保护模型仓库”代替核心资产保护。即使权重完全无法下载，完整的训练方法、关键数据构造和实验结论泄露，也可能显著降低复制能力所需的时间和成本。

反过来，研究文档保护得很好，也不能忽略推理集群：只要节点实际加载完整权重，它就进入 Weight Security Boundary。

> **Where the weights execute, the weight-security boundary follows.**

训练与推理环境应分别枚举权重以何种形式存在于对象存储、内存、GPU/加速器、Host、Snapshot、Crash Dump、Cache 和 Backup 中，以及哪些 Principal 和控制面能够读取或导出这些状态。

## 保护到达资产的路径，而不只是资产本身

攻击者不一定直接下载 `model.bin`。真正的保密边界应覆盖能够把资产转换、复制或恢复成等价知识的全部路径：

```text
Asset
  ↓
Principal
  ↓
Access Path
  ↓
Transformation
  ↓
Egress
  ↓
Destination
```

源码不能直接上传，不代表不能被复制到另一种文档再导出；权重下载被禁止，也不代表 Checkpoint、Snapshot、Debug、Backup、推理节点、对象存储复制和内部数据传输都被同样约束。

因此 DLP 和 Egress Control 不能只匹配原始文件。应同时检查身份、数据来源、转换链、目的地、体量、时间模式和业务理由。对高价值资产，读取和导出最好是不同权限：大量读取不自动意味着允许复制到任意外部位置。

## 用 Compartmentalization 限制单点失陷

Need-to-know 的目标不是制造流程障碍，而是避免一个账号、一台终端或一个团队被攻陷后直接获得完整 Capability Recipe。

```text
Research A  → 某类训练算法
Research B  → 数据与评测
Engineer C  → 训练基础设施
Operator D  → 推理与生产运行

             ↓

极少数受控 Principal
才能跨 Compartment 组合完整资产
```

Compartment 不只按组织架构划分，也可以按模型代际、项目、数据密级、研究方向和生产职责划分。跨区访问需要明确理由、短期授权和可审计审批；高影响操作可以要求 Multi-party Authorization。

同时要防止“逻辑上分区，现实中全部同步到同一终端”。Endpoint、Notebook、CI Runner、聊天与文档系统的访问边界必须与 Compartment 一致，否则 IAM 上的隔离会在数据落地后失效。

## 人员和终端属于核心资产边界

算法秘密大量存在于人的工作流中，因此 Personnel Security 与 Endpoint Security 不是外围控制。

高价值环境至少要回答：谁能进入核心项目，岗位变化后权限多久收回；管理员能否同时访问多个 Compartment；个人终端是否能缓存核心资料；离职、转岗、外包和供应商人员如何撤权；异常批量访问、压缩、转换和外传如何被检测。

这不意味着对所有 AI 团队采用高强度人员审查。控制强度应与 Threat Tier 和资产价值匹配，并遵守适用法律、劳动规则与隐私边界。

## 基础设施的控制面比算力本身更重要

训练集群的 GPU 很贵，但安全上更值得关注的是能够改变集群行为的 Control Plane：Scheduler、IAM、Secrets、Artifact Registry、Object Storage、Cluster Admin、Image Registry、CI/CD、Network Fabric 和管理跳板。

攻击者如果取得这些控制点，可能不需要突破训练框架本身，就能读取 Checkpoint、替换镜像、复制存储、创建调试任务或改变网络出口。

因此高价值集群应把管理面与计算面分开建模，限制长期管理员，使用短期工作身份，收紧调试与 Break-glass 权限，并让大规模权重读取、Snapshot、跨区复制和新 Egress Path 成为独立的高风险事件。

## 用攻击路径验证，而不是只做控制清单

最终验收不应是“权重加密了、员工有 MFA、集群有 EDR”。应该选择核心资产，构造完整 Exfiltration Path：

```text
初始身份失陷
  ↓
能否发现 Crown Jewel
  ↓
能否扩大访问
  ↓
能否读取 / 恢复等价资产
  ↓
能否转换或分片
  ↓
能否通过允许的出口带走
  ↓
检测能否及时关联并阻断
```

每个关键路径都要明确独立阻断点。只在最后依赖 DLP，或者只在入口依赖 MFA，都意味着单个控制失效后剩余纵深不足。

可以长期观察几类指标：核心资产完整归属率、跨 Compartment 授权数量和存续时间、未授权大规模读取/导出事件、权重读取路径覆盖率、训练与推理环境的异常 Egress、Break-glass 使用与复核，以及针对目标 Threat Tier 的红队成功路径。

真正的目标不是“把 AI Lab 做成最高安全等级”，而是让安全强度与资产价值和现实对手匹配，并能回答：

> **如果最重要的一份 AI 能力今天被人盯上，攻击者需要连续突破哪些彼此独立的边界，才能真正把它带走？**
