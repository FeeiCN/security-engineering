---
slug: /plan-and-execute-loop
icon: layout-dashboard-icon
description: 用登录回跳缺陷演示 Plan-and-Execute：让计划携带依赖和完成条件，并在新观察推翻假设时有预算地重规划。
last_reviewed: '2026-07-10'
---

# Plan-and-Execute 模式

Plan-and-Execute 适合目标明确、步骤存在依赖，但执行路径可能被新信息改变的任务。Planner 先生成可检查的步骤，Executor 按依赖执行；某个观察推翻计划假设后，Replanner 只修改尚未完成的部分。

本文用一个仓库缺陷贯穿全程：用户从 `/settings` 发起登录，登录成功后却跳到首页。入口把已校验的地址写入 `session['return_to']`，回调却读取 `session['next_path']`，最终使用了默认路径 `/`。

单个步骤内部仍然沿用 [ReAct 模式](/react-agent-loop) 的 `Action -> Observation`：模型提出一次行动，宿主执行工具并返回观察。Plan-and-Execute 在它外面增加计划、依赖、完成条件和 Replan，不再重复介绍工具循环。

## 先定义任务契约

Planner 开始工作前，需要知道目标、完成标准和操作边界。本例的任务契约是：

```yaml
goal: 修复登录成功后的 returnTo 回跳
scope:
  allow: 当前仓库内的文件、命令和测试
  deny: 仓库外文件、外部系统和真实用户数据
success:
  reproduction: 新增回归测试，稳定复现错误跳转
  relative_return: 已校验的相对路径 /settings 在登录后恢复
  external_url: https://evil.example 这类外部 URL 仍被拒绝
  tests: 相关测试全部通过
  diff_scope: diff 中没有无关改动
limits:
  replan_budget: 2
```

`success` 中的每一项都能由测试、命令结果或 diff 验证。Planner 可以调整实现路径，不能放宽外部 URL 限制，也不能把操作范围扩大到当前仓库之外。

## 把计划保存为可修改状态

计划需要进入任务状态，不能只存在于模型生成的一段文字中。控制器从任务契约加载 `required_checks`，把每项绑定到宿主注册的 verifier；Planner 和 Replanner 只能读取这张映射。最小状态包括：

```json
{
  "revision": 1,
  "scope": "current_repository",
  "replans_left": 2,
  "required_checks": {
    "reproduction": "verify_regression_red",
    "relative_return": "verify_relative_return",
    "external_url": "verify_external_url_rejected",
    "tests": "verify_related_tests",
    "diff_scope": "verify_diff_scope"
  },
  "facts": [],
  "steps": [],
  "plan_history": []
}
```

每个步骤至少包含五个字段：

```json
{
  "id": "reproduce",
  "goal": "用现有测试夹具复现错误回跳",
  "depends_on": ["inspect"],
  "covers": ["reproduction"],
  "done_when": "测试稳定得到实际路径 /，预期路径 /settings",
  "status": "pending"
}
```

- `id` 是稳定标识，后续依赖和执行记录都引用它。
- `depends_on` 列出必须先完成的步骤。执行器只能选择依赖已完成的步骤。
- `covers` 将步骤映射到任务契约中的验收项，便于安排执行顺序；它只是计划标签，不能让验收项自动通过。
- `done_when` 描述期望获得的外部证据，例如测试结果、结构化返回值或 diff。真正的判定由宿主注册的 verifier 完成。
- `status` 使用 `pending`、`running`、`completed`、`blocked` 或 `superseded`。
- `facts` 保存经过工具或测试确认的事实，并记录证据引用。Replanner 只能读取这些事实；新事实由观察验证器提交，不能随计划提案写入。

“理解登录流程”无法作为 `done_when`，因为它只能由模型自己宣称。“找到回调入口，并记录 `returnTo` 在每一层的输入和输出”可以由文件位置和代码片段检查，更适合作为完成条件。

## 生成初始计划

初始计划基于一个尚未验证的假设：现有测试夹具已经支持 `returnTo`，可以直接复现问题。

```json
[
  {
    "id": "inspect",
    "goal": "定位登录入口、回调和现有测试夹具",
    "depends_on": [],
    "covers": [],
    "done_when": "记录 returnTo 的校验、传递、回退路径和测试夹具字段",
    "status": "pending"
  },
  {
    "id": "reproduce",
    "goal": "用现有测试夹具复现 /settings 被改成 /",
    "depends_on": ["inspect"],
    "covers": ["reproduction"],
    "done_when": "回归测试只因实际路径为 / 而失败",
    "status": "pending"
  },
  {
    "id": "patch",
    "goal": "让回调恢复已经校验的 returnTo",
    "depends_on": ["reproduce"],
    "covers": ["relative_return"],
    "done_when": "相对路径回跳测试通过，且没有新增 URL 信任入口",
    "status": "pending"
  },
  {
    "id": "verify",
    "goal": "验证安全边界、相关测试和改动范围",
    "depends_on": ["patch"],
    "covers": ["external_url", "tests", "diff_scope"],
    "done_when": "外部 URL 被拒绝，相关测试通过，diff 无无关改动",
    "status": "pending"
  }
]
```

步骤顺序由依赖决定，不依赖数组中的书写位置。实际任务可以并行执行多个互不依赖的步骤，但共享同一文件或同一测试环境时仍需串行，避免观察互相污染。

## 按依赖执行 Action 与 Observation

这里沿用全系列术语：Action 是模型提交的**行动请求**，Observation 先作为**未验证观察**保存；只有测试或确定性检查满足 `done_when` 后，事实才进入宿主维护的**权威状态**。`done_when` 由**外部验证**判定，全部 `required_checks` 则来自任务开始前写定的**完成标准**。

执行器先选出所有依赖已完成的 `pending` 步骤，再为当前步骤运行 Action。工具返回值被记录为 Observation，完成检查器用 `done_when` 判断步骤能否进入 `completed`。

```python
def ready_steps(steps: list[dict]) -> list[dict]:
    completed = {step["id"] for step in steps if step["status"] == "completed"}
    return [
        step
        for step in steps
        if step["status"] == "pending"
        and set(step["depends_on"]).issubset(completed)
    ]


def record_observation(state: dict, step_id: str, observation: dict) -> None:
    state.setdefault("observations", []).append(
        {"step_id": step_id, **observation}
    )
```

本例先执行 `inspect`：

```text
Action:
  search_repo(query="returnTo|callback|redirect", root="current_repository")

Observation (obs-inspect-1):
  - 登录入口只接受站内相对路径，外部 URL 会被拒绝
  - /settings 已通过入口校验，并写入 session['return_to']
  - 回调读取 session['next_path']，缺失时回退到 /
  - 现有回调测试夹具没有 returnTo 字段
```

前三项确认了故障的数据流和现有安全边界，第四项推翻了初始计划的前提。`inspect` 已达到自己的 `done_when`，因此保持 `completed`；`reproduce` 依赖的测试前提不存在，需要进入 Replan。

Observation 只是带来源的记录。只有代码检查、测试结果或其他确定性检查满足 `done_when` 后，观察验证器才把相应内容提交到 `facts`。模型说“已经复现”不能替代失败测试，Replanner 也不能通过新计划补写事实。

## 观察改变后触发 Replan

Replanner 接收任务契约、当前计划、已验证事实和触发观察。它保留 `inspect` 的结果，将无法执行的原步骤标记为 `superseded`，再替换尚未完成的路径。下面展示的是控制器接受新步骤后形成的权威状态；`facts` 仍来自此前的观察验证器，不属于 Replanner 提案。

本轮保留 `inspect`，废弃原 `reproduce`，后续依次执行 `extend-fixture → red-test → patch → reject-external → verify`。

<details>
<summary>查看 revision 2 的完整计划状态</summary>

```json title="重规划后的权威状态" {2-4}
{
  "revision": 2,
  "scope": "current_repository",
  "replans_left": 1,
  "required_checks": {
    "reproduction": "verify_regression_red",
    "relative_return": "verify_relative_return",
    "external_url": "verify_external_url_rejected",
    "tests": "verify_related_tests",
    "diff_scope": "verify_diff_scope"
  },
  "facts": [
    {
      "claim": "相对路径 /settings 已通过入口校验并写入 return_to",
      "evidence_ref": "obs-inspect-1"
    },
    {
      "claim": "外部 URL 会被现有校验拒绝",
      "evidence_ref": "obs-inspect-1"
    },
    {
      "claim": "回调读取 next_path，未取得 return_to，因此回退到 /",
      "evidence_ref": "obs-inspect-1"
    },
    {
      "claim": "现有测试夹具没有 returnTo 字段",
      "evidence_ref": "obs-inspect-1"
    }
  ],
  "steps": [
    {
      "id": "inspect",
      "goal": "定位登录入口、回调和现有测试夹具",
      "depends_on": [],
      "covers": [],
      "done_when": "记录 returnTo 的校验、传递、回退路径和测试夹具字段",
      "status": "completed"
    },
    {
      "id": "reproduce",
      "goal": "用现有测试夹具复现 /settings 被改成 /",
      "depends_on": ["inspect"],
      "covers": ["reproduction"],
      "done_when": "回归测试只因实际路径为 / 而失败",
      "status": "superseded"
    },
    {
      "id": "extend-fixture",
      "goal": "扩展测试夹具以携带 returnTo",
      "depends_on": ["inspect"],
      "covers": [],
      "done_when": "测试夹具能分别表达 /settings 和外部 URL",
      "status": "pending"
    },
    {
      "id": "red-test",
      "goal": "新增相对路径回跳的失败测试",
      "depends_on": ["extend-fixture"],
      "covers": ["reproduction"],
      "done_when": "新增测试因实际路径为 /、预期为 /settings 而失败",
      "status": "pending"
    },
    {
      "id": "patch",
      "goal": "让回调恢复已经校验的 returnTo",
      "depends_on": ["red-test"],
      "covers": ["relative_return"],
      "done_when": "回调恢复已校验的相对 returnTo，回归测试通过",
      "status": "pending"
    },
    {
      "id": "reject-external",
      "goal": "确认修复没有放宽外部 URL 限制",
      "depends_on": ["patch"],
      "covers": ["external_url"],
      "done_when": "外部 URL 测试证明请求被拒绝且不会发生外部跳转",
      "status": "pending"
    },
    {
      "id": "verify",
      "goal": "运行相关测试并检查最终 diff",
      "depends_on": ["patch", "reject-external"],
      "covers": ["tests", "diff_scope"],
      "done_when": "相关测试通过，diff 只包含修复与回归测试",
      "status": "pending"
    }
  ]
}
```

</details>

这次 Replan 增加了夹具步骤和先失败后修复的回归测试。它没有重新调查已经确认的数据流，也没有把“外部 URL 会被拒绝”从约束中删除。旧计划进入 `plan_history`，因此可以解释计划为什么变化。

## 给 Replan 设置预算

工具暂时失败、依赖缺失、观察推翻假设或 `done_when` 无法满足时，都可能需要 Replan。每次重规划都会增加模型调用、执行时间和偏移风险，需要由控制器限制。

```text
function accept_replan(state, proposal):
  if state.replans_left <= 0:
    state.status = blocked
    return

  if proposal.scope != state.scope:
    reject("Replan 不得扩大任务范围")

  validate_completed_steps_unchanged(state.steps, proposal.steps)
  validate_dependencies(proposal.steps)
  validate_done_when(proposal.steps)
  validate_contract_coverage(
    proposal.steps,
    keys(state.required_checks)
  )

  state.plan_history.append(copy(state.steps))
  state.steps = copy(proposal.steps)
  state.revision += 1
  state.replans_left -= 1
```

这是控制器伪代码，`validate_*` 都要由宿主实现并测试，失败时拒绝提案。`validate_dependencies` 至少检查步骤 ID 互不重复、依赖目标存在且依赖图无环；`validate_contract_coverage` 检查所有未废弃步骤的 `covers` 合集仍提到了每项任务要求。这项检查只能发现计划遗漏，不能证明自然语言步骤真的执行了对应测试。

真正的安全边界在最终 verifier。宿主使用不可由模型改写的注册表，将 `verify_external_url_rejected` 等 ID 绑定到确定测试或检查函数；无论计划如何命名步骤，进入 `succeeded` 前都会重新运行全部 `required_checks`。模型伪造一个 `covers: ["external_url"]` 标签，无法让 verifier 返回通过。

`proposal` 只包含范围和步骤，不包含 `facts`。已确认事实在 Replan 前后保持不变；确实发现旧事实有误时，由观察验证器追加一条带新证据的更正记录，不能由 Replanner 静默覆盖。

`replans_left` 由控制器计算和扣减，Replanner 无权设置它。预算耗尽后，任务进入 `blocked`，并报告已完成步骤、最后观察、剩余风险和需要用户决定的问题。控制器不能通过扩大仓库范围、跳过安全测试或自行增加预算来继续执行。

## 处理常见失败

| 失败表现 | 处理方式 |
|---|---|
| 每次观察都重做整份计划 | 只替换受影响的 `pending` 或 `blocked` 步骤，保留完成步骤和事实 |
| `done_when` 写成“修好问题” | 改成具体测试结果、返回值、退出码或 diff 条件 |
| 新步骤依赖自己或形成环 | 接受 Replan 前校验依赖图，无可执行步骤时停止 |
| Replan 改写目标或安全边界 | 将任务契约与计划分开保存，拒绝越界提案 |
| 同一种失败反复出现 | 消耗重规划预算；预算耗尽后进入 `blocked` 并交还用户 |

如果测试环境偶发失败，应先区分环境错误和产品错误。可重复的临时错误可以有限重试；缺少测试夹具、权限不足或计划假设错误会改变后续路径，更适合 Replan。

## 按完成标准验收

修复完成后，控制器按不可变的 `required_checks` 注册表逐项运行 verifier，不以 `covers`、`done_when` 或 Planner、Executor 的文字结论作为成功依据：

- 回归测试在修复前稳定显示实际 `/`、预期 `/settings`，修复后通过。
- 相对 `returnTo=/settings` 登录后返回 `/settings`。
- 外部 `returnTo=https://evil.example` 被拒绝，不会产生外部跳转。
- 与登录、回调和重定向相关的测试全部通过。
- `git diff --check` 通过，`git status --short` 和最终 diff 没有无关改动。
- 所有完成步骤都保存对应的 Action、Observation 和验证证据。

只有这些条件全部满足，任务状态才能进入 `succeeded`。如果相关测试命令无法确定、依赖服务不可用或剩余预算不足，任务保持 `blocked`，交付现有证据和未完成步骤。
