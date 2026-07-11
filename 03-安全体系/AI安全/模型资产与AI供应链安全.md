---
slug: /ai-model-supply-chain-security
title: 模型资产与 AI 供应链安全
sidebar_position: 3
icon: cpu-icon
description: 生产模型身份是权重、Tokenizer、Adapter、量化、运行时、Prompt 与策略的组合承诺，并须绑定来源、评测和加载证明。
content_type: article
last_reviewed: '2026-07-11'
---

# 模型资产与 AI 供应链安全

一家企业从外部仓库引入基础模型，再训练客服 Adapter。测试环境使用一套 Tokenizer 和推理镜像，生产流水线却从 `customer-service-latest` 临时拉取了另一个量化版本。更新后，助手只在特定退款表述下偏离规则。团队拥有一份“评测通过”报告，却无法证明报告测过的组合正在生产运行。

模型发布需要一条可验证的安全不变量：**每个生产请求都只能路由到一个有不可变发布身份的模型组合；该身份必须同时绑定可验证来源、同一组合的行为评测证据，以及运行时实际加载身份。** 任何一环无法对上，该组合就不进入受支持的生产路由。

这条不变量将两类经常混在一起的证明分开。哈希、签名和构建来源证明回答“这是哪些字节，由哪条流水线产生”；安全评测回答“这些字节与运行环境组合后，在已测边界下表现如何”。签名的后门仍然是后门，评测过的组合被部署流水线替换后，原报告也失去了证明力。

## 攻击者可以改制品，也可以改流水线

本文的攻击者包括被入侵的上游维护者账号、能向暂存仓库上传制品的内部开发者、已控制某个训练任务的攻击者，以及能修改可变标签或部署参数的账号。他们未必同时控制签名服务、生产准入策略和运行集群，但会尝试借合法流水线把恶意对象提升到生产。

攻击目标有三类。第一类是完整性：替换基础权重、Tokenizer、Adapter、量化结果、自定义加载代码或推理镜像，让模型在触发条件下偏离。第二类是保密性：从共享对象存储、调试容器、备份或模型注册表批量带走未公开 Checkpoint 和内部 Adapter。第三类是可用性：删除关键分片、破坏兼容关系或让模型在启动时依赖已不可用的远程标签。

没有恶意攻击时，制品组合仍可能失效。Tokenizer 改变会让输入截断位置和特殊 Token 处理变化，Adapter 与错误基础权重组合会带来未测行为，量化工具和推理库更新也可能改变数值行为。因此，发布身份要覆盖完整组合，不使用单独的模型名称代替。

## 一个内部签名如何洗白恶意上游

一条典型攻击链从“用内部流程测一下外部模型”开始：

1. 攻击者控制上游账号，发布包含自定义加载逻辑或不安全反序列化对象的新版本。
2. 评测任务在持有模型仓库令牌、云凭证和内网访问权的工作节点上首次加载模型。Python 官方文档明确警告，[`pickle` 不能用于反序列化不可信数据](https://docs.python.org/3/library/pickle.html)，因为精心构造的数据可以在反序列化时执行代码。
3. 恶意代码读取工作节点凭证，上传一个被替换的 Adapter，或者修改测试输出，让流水线看起来成功。
4. 内部发布服务只检查“评测任务已完成”，随后用内部身份签名恶意组合。签名如实证明了内部服务处理过这份制品，也让未检查上游来源的问题获得了更高信任。
5. 常规任务评测未覆盖特定触发词，组合通过发布门禁。[BadNets 论文](https://arxiv.org/abs/1708.06733) 展示了模型供应链中的触发式后门机制：模型在普通输入上维持功能，在特定触发条件下偏向攻击目标。这是机制证据，不直接代表某个当前大模型的实际攻击成功率。
6. 生产部署校验了内部签名，但没有比较正在加载的分片、Tokenizer、Adapter 和镜像是否与评测报告完全一致。

这条链的修复重点是缩小首次加载环境的权限，让来源证明与行为评测共同绑定不可变 manifest，再由生产准入层核对实际加载对象。只在流程末端增加一次签名，会保留前面的信任空档。

## 参考架构产生三份相互绑定的证据

自托管模型的安全发布链可以按下图组织。导入、训练、签名、批准和部署由不同工作身份完成，相互之间交换带摘要的对象，不共享一个长期令牌。

```text
 external model / source / data manifest
                  |
                  v
       +-----------------------+
       | isolated import stage |
       | no secret, no prod net|
       +-----------+-----------+
                   |
                   v
       +-----------------------+
       | build / train identity|
       | checkpoint + digest   |
       +-----------+-----------+
                   |
                   v
       +-----------------------+
       |    Target Manifest    |
       +-----+------------+----+
             |            |
             v            v
    provenance check   behavior evaluation
             |            |
             +-----+------+
                   v
       Release Manifest (promotion attestation)
                   |
                   v
       immutable production registry
                   |
                   v
       admission policy -> runtime load proof
                   |
                   v
              production route
```

三份证据分别是：来源证明，记录谁用什么输入和流水线产生了 Target Manifest 中的制品；行为评测报告，记录同一 Target 摘要在冻结环境下通过了哪些门禁；运行时加载证据，记录实例实际加载的分片、Adapter、镜像和系统配置摘要。发布服务在评测完成后生成 Release Manifest，签署 Target 摘要、评测报告摘要、批准结果之间的关系。

## `ReleaseManifest` 给组合一个不可变身份

基础权重的多个 shard、Tokenizer、Adapter、量化参数、推理库和容器镜像构成模型子组合；Prompt、知识快照、工具契约、权限与安全策略继续决定生产行为。先对完整运行目标生成 `Target Manifest`，评测报告只绑定它的摘要；评测通过后再生成公共的 `Release Manifest`。两个对象分别计算摘要，避免评测报告与 Release Manifest 互相包含对方摘要。

```yaml
target_manifest:
  model:
    base_model:
      uri: "registry://models/base@<digest>"
      shards: ["sha256:<digest>"]
      source_attestation: "attestation:<digest>"
    tokenizer: "registry://tokenizers/customer@<digest>"
    adapters: ["registry://adapters/refund@<digest>"]
    runtime_image: "registry://images/inference@<digest>"
    quantization_config: "registry://configs/quant@<digest>"
    inference_config: "registry://configs/inference@<digest>"
    loader_policy:
      allow_remote_code: false
      allowed_formats: ["safetensors"]
  application:
    system_prompt: "registry://prompts/customer-service@<digest>"
    knowledge_snapshot: "registry://knowledge/customer-service@<digest>"
    tool_contracts: "registry://tools/customer-service@<digest>"
    permission_policy: "registry://policies/permissions@<digest>"
    safety_policy: "registry://policies/content-safety@<digest>"
  build:
    source_commit: "git:<commit>"
    workflow_identity: "workload:model-builder"
    data_manifest: "dataset:<digest>"
target_digest: "sha256:<canonical-target-manifest>"

release_manifest:
  target_manifest: "sha256:<canonical-target-manifest>"
  provenance: "attestation:<digest>"
  security_evaluation: "eval-report:<digest>"
  approval:
    policy_version: "release-policy:<digest>"
    approved_by: ["role:model-release-manager"]
  rollback_to: "sha256:<previous-release-manifest>"
release_digest: "sha256:<canonical-release-manifest>"
```

`target_digest` 对规范化的 `target_manifest` 映射计算，不把摘要字段本身放进输入；评测报告引用这个摘要。`release_digest` 同理，对包含 Target 引用、证据和批准结果的 `release_manifest` 映射计算，再由发布服务作分离签名。这里的生产模型身份是 `target_digest` 所承诺的完整组合，单个基础权重哈希只标识其中一个组件。每个大模型分片仍有独立摘要，既能发现“权重名称没变，某个 shard 已被替换”，也能避免同一权重在不同 Prompt、知识、工具或策略下冒用旧评测。`stable` 和 `latest` 可以作为用户友好别名，生产准入层最终把别名解析成已批准的 Release Manifest，再按其中的 Target 摘要加载，防止实例启动后重新解析可变远程标签。

[SLSA Provenance](https://slsa.dev/spec/v1.2/provenance) 定义了如何声明某个 subject 由哪个构建过程产生，可用来表达 manifest 与受控流水线的关系。[Sigstore Cosign](https://docs.sigstore.dev/cosign/signing/overview/) 提供了对容器及其他 blob 进行签名和验证的开源工具链。这些机制提供来源与身份证据，行为评测仍需由独立门禁完成。

## 自托管和托管 API 需要不同强度的门禁

**导入门。** 外部制品首次加载在无生产凭证、限制外联和限制宿主访问的隔离环境中完成。加载器禁止未批准的远程自定义代码，对序列化格式建立允许列表。[开源 `safetensors`](https://github.com/huggingface/safetensors) 以安全存储 Tensor 为设计目标，可以减少通用对象反序列化的代码执行面。它无法判断 Tensor 是否存在后门，也不检查同目录的自定义加载代码，所以仍要配合来源、隔离和行为评测。

**训练门。** 训练任务使用独立工作身份和短期凭证，只读取已批准数据和基础模型，只写入当前任务的暂存区。Checkpoint 在保存时记录摘要、父版本、训练步和配置，恢复前重新校验。交互式调试账号不默认读取全部历史 Checkpoint；对象存储、临时盘、缓存和备份采用同一分类与导出策略。

**提升门。** 构建身份只能提交 Target Manifest，评测身份只能写绑定 Target 摘要的评测证据，发布服务核对摘要一致后才能生成 Release Manifest 并提升。安全评测同时冻结模型、Tokenizer、Adapter、量化配置、镜像、Prompt、知识、工具与策略。任一组件更换都会形成新 Target，按影响重跑门禁。

**运行门。** 生产准入策略只接受签名有效、来源条件匹配、行为评测通过且已批准的 manifest。实例启动后回传实际加载摘要，网关只向匹配实例路由。这一机制不需要模型自报版本，证据由容器准入层、模型加载器和路由层共同产生。

**托管 API 门。** 外部供应商不提供权重时，使用方无法校验真实权重摘要，运行时证明的强度会降低。此时发布身份记录供应商、产品版本 ID、端点、区域、组织配置和观测时间，再用固定 canary 任务、变更通知和持续回归发现行为变化。报告应如实标注“无法验证权重身份”，不用供应商品牌或一个版本名称补齐证据空缺。

## 指标要检查证据是否闭合

下表的指标按发布、实例或演练计算。Manifest 与待提升发布的全量对账属于确定性门禁，任一摘要不匹配就移出路由，不依赖置信区间；运行匹配率若只抽检部分实例，要同时报告抽样框、样本数和比例区间。行为安全指标和攻击样本定义由[AI 安全评测与红队](./AI安全评测与红队.md)承担，本文检查评测证据是否真的绑定了生产组合。

| 指标 | 分子 / 分母 | 测试条件与用途 |
|---|---|---|
| Target Manifest 完整率 | 组件摘要、来源和兼容字段齐全的组件 / 加载器在该发布中发现的全部组件 | 用独立扫描结果和 Target Manifest 对账；不以 Manifest 自身声明作为分母 |
| 来源验证通过率 | 签名有效且构建身份、仓库、工作流程符合策略的制品 / 本次提升尝试的全部制品 | 对签名者、受众、时间和 provenance predicate 实际验证，不只检查存在签名文件 |
| 评测绑定率 | 评测报告中的 Target 摘要与待提升 Target 一致的发布 / 所有待提升发布 | 包含回滚、灰度和紧急发布；紧急例外记录责任人与到期时间 |
| 运行匹配率 | 实际加载 Target 摘要与 Release Manifest 引用一致的实例 / 抽检或全量上报的生产实例 | 实例启动、扩容、节点恢复和镜像重拉后重新计算；未匹配实例移出路由 |
| 未授权制品导出率 | 未经策略批准的权重、Checkpoint 或 Adapter 导出事件 / 发布周期内的全部制品导出事件 | 联合仓库审计、对象存储和网络出口；发现一次就进入事件调查 |
| 回滚恢复时间 | 从禁用当前 manifest 到上一已验证组合恢复目标流量的时长 | 在隔离或灰度环境定期演练，报告 P50/P95 和未完成实例数 |
| 托管变更暴露窗口 | 最后一次正常 canary 到首次异常 canary 之间的时间区间 | 适用于无法观察真实变更起点的托管模式；另用受控注入演练测实际发现时延，并报告 canary 误报与漏检 |

## 供应链证据有明确的失效边界

**签名可以保护恶意制品的完整性。** 当上游、构建任务或签名身份已经受控时，签名只能证明制品在之后没有变。门禁因此同时检查构建身份、源仓库、工作流程和行为证据，并将签名服务与导入环境隔离。若攻击者同时控制签名身份与生产准入策略，本文的链上证据已无法提供独立保证，需要组织级凭证泄露响应和带外证据。

**安全序列化格式无法证明权重行为安全。** `safetensors` 可以缩小加载时代码执行面，后门、偏差、与错误 Adapter 组合的回归仍需行为评测。SBOM 能列出代码和库的已知依赖，不描述模型在训练中学到的全部行为。

**完整 provenance 不等于能重现相同字节。** GPU 内核、并行调度、随机源和数值精度都可以让训练具有非确定性。来源证明记录产生现有制品的过程，不强行承诺重跑一次会得到相同哈希。对重要发布保留精确制品和构建证据，会增加存储与保密负担；团队可以按价值分级保留，对过期 Checkpoint 执行受控删除。

**托管 API 的证明天花板更低。** 用户可以证明自己调用了哪个端点和声明版本，可以用 canary 发现部分行为变化，无法用内部 manifest 证明供应商真实加载了哪些权重。高影响场景需要在这个证据缺口、自托管的运维成本与外部模型的能力之间作明确选择。

**固定版本会放慢上游安全修复。** 直接跟随 `latest` 会降低补丁延迟，也让行为证据快速过期。固定 digest 后，团队要为高危漏洞保留快速通道：隔离导入、缩小评测集为高优先级门禁、限制流量，并为例外设置到期时间。

本文只证明模型制品如何从受控输入到达生产。训练数据的来源、投毒和撤销见[AI 数据与知识安全](./AI数据与知识安全.md)；模型组合的具体行为门禁见[AI 安全评测与红队](./AI安全评测与红队.md)；权重、日志和训练材料中的个人信息及机密见[AI 隐私与机密保护](./AI隐私与机密保护.md)。
