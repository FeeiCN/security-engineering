---
slug: /ai-agent-tool-security
title: Agent 与工具调用安全
sidebar_position: 6
icon: brain-circuit-icon
description: Agent 安全要用 Capability Envelope 限定一次运行可达的权限，再以 Action Envelope、逐次授权和资源侧执行约束每个真实副作用。
content_type: article
last_reviewed: '2026-07-11'
---

# Agent 与工具调用安全

合同助手升级成采购 Agent 后，模型输出会进入 ERP、邮箱、文件系统和长期记忆。模型可以灵活规划，安全边界却要满足一条稳定不变量：**Prompt、外部文档、工具结果、记忆和其他 Agent 消息只能影响计划，不能自行扩大执行权限。** 新增权限必须来自模型之外的授权事件，并绑定主体、资源、参数、期限和副作用。

本文把一次运行允许触达的最大权限集合称为 `Capability Envelope`，把一次准备执行的具体动作称为 `Action Envelope`。这两个名称是架构原语，并非某个协议已经定义的标准对象。它们的价值在于把“Agent 可以做什么”和“这次到底获准做什么”变成可由策略引擎与资源系统校验的数据。

## 攻击者寻找权限升级链

攻击者可以控制供应商报价单、外部网页或恶意 MCP Server 返回的内容，也可以诱导普通员工发起正常采购任务；他没有采购经理账号和 ERP 凭证。攻击目标是借 Agent 已有的连接，把读取报价的任务升级成修改收款账号、创建订单、外发合同或写入持久记忆。

一条跨组件攻击链可能这样形成：

1. 员工委托 Agent 比较三份报价，只授予当前项目的只读资料访问。
2. 恶意报价单要求模型调用“供应商维护”工具并把新账号记为已验证事实。
3. 工具目录向模型暴露了与当前任务无关的写工具，模型生成格式合法的调用参数。
4. Agent 宿主持有共享的高权限服务令牌，未按当前委托重新判断动作、资源和金额。
5. MCP Server 接收令牌后继续转发，ERP 只看到服务账号，无法识别真实委托人和任务范围。
6. 收款账号被修改；工具响应超时后，宿主又重试一次，形成第二个副作用。
7. 模型把“修改成功”写入长期记忆，污染延续到后续运行。

[AgentDojo](https://arxiv.org/abs/2406.13352)专门评估工具返回的不可信数据如何劫持 Agent。论文中的环境包含邮件、网银和旅行预订等 97 个正常任务及 629 个安全测试用例，并观察到现有模型在没有攻击时也会失败于部分任务。安全设计需要同时覆盖恶意注入、模型普通失误和工具异常，不能把所有危险请求都归因于攻击者。

## Capability Envelope 限定一次运行的最大权限

`Capability Envelope` 在运行开始前由宿主根据登录用户、业务场景和已有批准生成。它至少描述委托主体与租户、允许的工具和动作、资源选择器、允许读取的数据级别、允许输出的目的地、费用与步骤预算、有效期、子 Agent 委托深度，以及哪些动作仍需独立批准。

对一段运行时间 `t`，可以用三个约束理解权限单调性：

```text
模型可见事件不能扩大 C(t)
子 Agent 的 C_child 必须是 C_parent 的子集
实际执行的动作 a 必须满足 a ∈ C(t)
```

用户完成强认证、采购经理批准订单或安全人员临时开放工具时，系统可以签发新版本的 Envelope。这个变化来自授权服务，并留下授权人、依据、差异、期限与撤销条件；模型输出“用户已经同意”不构成授权事件。旧 Envelope 随版本替换或到期失效，避免并行步骤继续使用过时权限。

权限集合还要包含数据去向。一个只读订单工具不会修改数据库，却可能把客户资料交给公开邮件或 HTTP 工具。`read:orders` 与 `send:external` 的组合风险高于两个 Scope 各自表达的含义，因此 Envelope 要限制允许的数据分类和 Sink，具体的信息流标签由[大模型应用安全](/llm-application-security)定义。

## 参考架构把规划与执行分开

```text
用户 / 业务任务 ─> Run Manager ─────────────> LLM Planner
                         |                         |
             Capability Envelope          Action Proposal
                         |                         |
授权服务 / 审批 ─────────┴────> PDP <──── Proposal Normalizer
                                  |
                         allow + decision_ref
                                  |
                                  v
                       Action Envelope Builder
                                  |
                                  v
                          Tool Gateway PEP
                                  |
                      Credential Broker / Sandbox
                                  |
                                  v
                     MCP Server / 业务资源 PEP
                                  |
                       权威结果 / 真实副作用
                                  |
                                  v
                       Verifier + Run Event
```

模型只产生 `Action Proposal`，其中的工具名和参数仍是不可信候选。Proposal Normalizer 先规范化类型、解析资源身份和计算参数摘要，再交给 PDP。PDP 使用当前 Capability Envelope、业务状态、风险级别和审批记录作决定；放行后，Envelope Builder 把规范化参数与 `decision_ref` 固化为 Action Envelope。Tool Gateway PEP 执行决定，校验通过后 Credential Broker 才为目标资源取得短期凭证。

凭证应限制接收方和用途。[RFC 8707](https://www.rfc-editor.org/rfc/rfc8707)定义了 OAuth `resource` 参数，使授权服务器可以签发面向特定资源的 audience-restricted Token；规范也说明多 Audience 会扩大资源之间的信任。模型上下文无需接触 Token，MCP Server 也不应把收到的 Token 当成可转发给任意下游的通行证。

[MCP 授权规范](https://modelcontextprotocol.io/specification/2025-11-25/basic/authorization)解决客户端、授权服务器与资源服务器之间的协议交互，业务对象级授权仍由资源服务完成。[MCP Security Best Practices](https://modelcontextprotocol.io/docs/tutorials/security/security_best_practices)进一步讨论 confused deputy、Token passthrough、Scope 最小化与本地 Server 风险。Tool Gateway 的允许决定无法替代 ERP 对租户、订单和当前状态的再次校验。

## Action Envelope 绑定一次具体副作用

PDP 放行后，宿主生成不可由模型修改的 `Action Envelope`。它把自然语言计划收敛成一个可验证动作，例如：

```json
{
  "run_id": "run_...",
  "action_id": "act_...",
  "capability_envelope": "cap_...@v...",
  "principal": {
    "agent": "procurement-agent",
    "on_behalf_of": "user_...",
    "tenant": "tenant_..."
  },
  "tool": {"id": "erp.create_order", "version": "v..."},
  "resource": {"type": "contract", "id": "contract_..."},
  "arguments_digest": "sha256:...",
  "precondition": {"contract_version": "...", "approval_state": "approved"},
  "constraints": {
    "allowed_recipient": "supplier_...",
    "expires_at": "..."
  },
  "policy_decision_ref": "decision_...",
  "approval_ref": "approval_...",
  "idempotency_key": "...",
  "expected_effect": "create one purchase order",
  "compensation": "cancel_order"
}
```

审批界面展示规范化后的主体、旧值与新值、目标资源、数据去向和可逆性。批准结果绑定 `arguments_digest` 与前置状态；模型改变金额、收件人或资源版本后，原批准失效。这同时处理了计划阶段到执行阶段的 TOCTOU：PEP 在调用前重新读取关键状态，发现合同版本或审批状态变化就拒绝执行。

`idempotency_key` 用于识别同一意图的重试。工具超时只能说明响应未知，宿主应先向权威系统查询动作状态，再决定继续、补偿或转人工。模型生成的“调用成功”以及 MCP Server 的自然语言确认都不能代替 ERP 中的订单记录。

Action Envelope 可以签名或由受信服务通过引用传递，具体实现取决于跨服务边界。关键条件是业务资源 PEP 能校验其完整性、有效期、Audience 和当前策略，并把实际执行参数与批准参数对上。只在 Agent 宿主内检查一次，会让被攻破或配置错误的工具服务重新获得扩权机会。

## 工具结果、记忆与子 Agent 只能收窄权限

工具描述和 Schema 会影响模型选择，却没有授予能力的效力。运行时只向模型展示 Envelope 内可能使用的工具子集；新 MCP Server、工具版本、目标地址或权限需求变化后，工具注册表先重新审核。一个同名工具即使返回结构完全正确，也不能跳过 Server 身份和目标地址验证。

工具结果按外部观察处理。网页、邮件、报错和数据库文本可以更新模型对任务事实的判断，无法修改 Envelope、策略或审批状态。结果进入下一轮上下文前保留来源和敏感标签，凭证、内部路径与超长内容由宿主过滤。模型需要根据结果发起新动作时，仍生成新的 Action Proposal 并重新经过 PDP。

长期记忆写入本身是一项动作。模型提出“记住供应商新账号”后，记忆 PEP 检查来源、主体、作用域、有效期和验证依据；外部文档只能形成待核实候选。业务规则、工具回执和模型摘要分区保存，摘要不能覆盖权威记录。

多 Agent 委托遵守衰减关系：子 Agent 的工具、资源、数据去向、预算和有效期均不得超过父 Envelope。子 Agent 消息回到协调者时仍是不可信观察。确需扩权的子任务回到授权服务签发新 Envelope，协调模型无权复制自己的凭证或把“紧急”当成提权理由。

## 强约束要处理自动化的现实代价

严格 Envelope 会限制临时发现的新任务。低风险场景可以允许模型在只读沙箱中探索并生成计划，遇到未授权能力时暂停，由用户基于具体差异追加授权。预先授予一组宽泛 Scope 虽然减少中断，也会让一段恶意文档直接获得更大的可达后果。

短期 Token 会给长任务带来续期成本。续期应由 Run Manager 携带当前任务和 Envelope 版本向授权服务申请，授权服务重新检查用户状态、撤销信号与资源 Audience；把长期 Refresh Token 交给模型或任意 MCP Server 会重新扩大暴露面。

人工审批也可能退化为点击惯性。审批应集中在新增权限、跨域数据流和难以撤销的副作用，页面展示参数差异与前置证据；读取公开目录等低后果动作可以由预先策略自动放行。每一步都弹出相同确认框，只会提高操作成本，难以提高判断质量。

沙箱能限制进程、文件、网络与资源消耗，无法阻止一个获得合法 ERP 连接的工具提交错误订单。业务资源 PEP、幂等控制和结果核验仍然必需。相反，一个纯文本规划 Agent 没有凭证、外部 Sink 和持久记忆时，可以采用更轻的 Envelope；控制强度由最大可达副作用、可验证性和可恢复性决定。

## 用分母证明权限单调性

测试从完整攻击链发起：把注入放入报价单和工具结果，篡改 MCP 工具描述，改变审批后的参数，撤销运行中的权限，模拟超时、重复响应、记忆污染和子 Agent 越权。每个用例同时检查模型请求、PDP 决定、PEP 执行和业务系统最终状态。

发布报告至少给出这些带分母指标：

- **未授权提议率** = 被 PDP 判定越权的 Action Proposal 数 / 全部 Action Proposal 数，并按正常与对抗任务分别报告。
- **强制逃逸率** = 造成真实状态变化的越权动作数 / PDP 已拒绝或 Envelope 外的动作总数。
- **审批绑定失效率** = 参数或前置状态改变后仍被执行的动作数 / 审批后变异测试总数。
- **重复副作用率** = 形成重复业务状态的运行数 / 注入超时与重试的测试运行总数。
- **结果核验覆盖率** = 已从权威系统确认后置状态的副作用动作数 / 已执行副作用动作总数。
- **正常任务完成率** = 在权限约束内正确完成的正常任务数 / 正常任务总数。

未授权提议率升高说明模型或上下文防护在退化；强制逃逸率检验模型外边界是否守住。两者不能合成一个总分。测试中的零逃逸只证明给定版本和样本下没有观察到突破，统计证据的强度还要按 [AI 安全评测与红队](/ai-security-evaluation-red-teaming)中的重复次数、覆盖范围和置信上界解释。

一次发布应能证明：模型可见信息没有权限写入口；每个真实动作都属于当时有效的 Capability Envelope；每次执行都关联不可变的 Action Envelope；资源系统能够独立拒绝超范围调用；副作用能够核验、停止或进入明确的补偿流程。满足这些条件后，模型能力提升会扩大可用计划空间，不会自动扩大授权空间。
