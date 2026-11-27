---
slug: /ai-skill
icon: stack-icon
description: Skill 应封装经过验证的重复方法：从登录回跳修复提炼 repo-bugfix，并用触发描述、权限边界、未知 Bug 验收和版本回归保证可复用性。
last_reviewed: '2026-07-10'
---

# AI Skill

前一篇 [ReAct 模式](/react-agent-loop) 用测试和源码证据定位了登录回跳 Bug：登录入口把目标页面写入 `session['return_to']`，回调却读取 `session['next_path']`，于是取到默认值 `/`。修复时统一会话键、补回归测试、运行项目验证并检查差异，结果得到外部证据确认。

这次把已经奏效的修复方法提炼成 `repo-bugfix` Skill，不再展开 Agent 如何逐步选择行动。以后遇到根因未知的新 Bug，Agent 可以复用同一套质量要求，同时仍根据新证据决定具体读取和修改哪些文件。

Skill 只收录经过验证、值得重复的方法。登录回跳案例可以产生候选流程；候选流程还要通过未参与编写的新 Bug 验收，才能成为可依赖的版本。

## 判断是否值得创建

先检查任务是否同时满足四个条件：

- **会重复出现**：输入和根因会变化，但任务目标长期相似，例如持续处理仓库中的可复现 Bug。
- **存在稳定步骤**：多个成功案例都经历复现、缩小范围、最小修改、回归测试、运行验证和差异检查。
- **结果可以验收**：测试退出码、类型检查、构建结果和 Git 差异能从模型之外判断任务是否完成。
- **权限可以收窄**：任务可以限定在当前仓库，不需要生产凭证、部署权限或不受控的外部操作。

一次性问题、仍在频繁变化的流程、只能靠主观感受判断的结果，都适合先保留为任务笔记。首次成功也只说明方法值得继续测试，不能直接证明它适用于同类任务。

`repo-bugfix` 的复用单位是修复纪律，不是登录模块知识。下面这些内容应保留：

```text
复现故障 → 限定相关范围 → 最小修改 → 补回归测试
          → 运行验证 → 检查完整 diff → 提交结果
```

`return_to`、`next_path`、`src/auth/login.ts`、`src/auth/callback.ts` 等事故细节不进入主流程。把这些名字写死后，Skill 在新的缓存、表单或权限 Bug 上会沿着错误方向搜索。

## 建立最小目录

[Agent Skills 官方规范](https://agentskills.io/specification) 规定，一个 Skill 至少是包含 `SKILL.md` 的目录；`scripts/` 和 `references/` 都是可选资源。不同 Agent 实现的安装位置可能不同，先在普通工作目录中完成和验证内容，再放入当前实现识别的 Skill 根目录。

```text
repo-bugfix/
├── SKILL.md
├── scripts/
│   └── check-diff.sh
└── references/
    └── repository-conventions.md
```

可以先创建目录：

```bash
mkdir -p repo-bugfix/scripts repo-bugfix/references
```

`SKILL.md` 保存每次都要遵守的短流程；脚本承载确定、可执行的机械检查；参考资料保存仓库约定和较长说明。一个文件已经能讲清时，只创建 `SKILL.md`。目录扩展应由真实重复需求推动。

## 写好触发描述和主流程

规范要求 `SKILL.md` 的 front matter 至少包含 `name` 和 `description`。`description` 同时回答“做什么”和“什么时候使用”，因为 Agent 会根据这些信息发现相关 Skill。`Helps fix bugs` 范围过宽，也缺少触发场景和完成要求。

下面是一份最小可用的 `SKILL.md`：

```md
---
name: repo-bugfix
description: Diagnoses and fixes reproducible bugs in existing code repositories. Use when expected and actual behavior differ, a local test fails, or a user requests a minimal verified patch. Excludes deployments, production incidents, and tasks requiring secrets.
compatibility: Requires Git and the repository's documented validation commands. Operates only inside the current repository.
metadata:
  version: "0.1.0"
---

# Repository Bug Fix

## Required inputs

- Record expected behavior, actual behavior, and available reproduction steps.
- Read the repository's local instructions before changing files.
- Treat the current dirty worktree as user-owned state.

## Workflow

1. Reproduce the bug with the narrowest existing test or command. Record the command, exit code, and relevant output.
2. Limit investigation to the failing path and its nearest callers, state transitions, and tests. Expand only when evidence requires it.
3. Make the smallest change that addresses the demonstrated cause. Preserve unrelated user changes.
4. Add or update a regression test for the reproduced behavior. Confirm it fails before the fix and passes after it; report when a safe pre-fix run is unavailable.
5. Run the focused regression, then the repository's documented broader checks.
6. Run `scripts/check-diff.sh`, inspect the full diff, and remove unrelated changes or generated artifacts created by this task.

## Boundaries

- Read and write only inside the current repository.
- Do not read credential files, secret stores, or production data.
- Do not deploy, push, send messages, or call external systems. Stop and hand off those requests.
- Do not use destructive Git commands to clean an existing worktree.

## Stop conditions

- Stop as blocked when the bug cannot be reproduced with the available environment.
- Stop and request direction when the fix requires credentials, production access, deployment, or a broader product decision.
- Do not claim success while required tests fail or completion evidence is missing.

## Final report

Report the reproduced behavior, verified root cause, changed files, regression test, validation commands and results, remaining risk, and any skipped check.
```

这份主文件只定义稳定流程。它不会规定先搜索哪个文件，也不会嵌入完整的 [ReAct 循环](/react-agent-loop)、[Plan-and-Execute 算法](/plan-and-execute-loop) 或 [Harness](/harness-engineering) 状态机。Agent 运行时负责选择行动和维护状态，Skill 只提供反复验证过的做事方法与验收要求。

## 把机械动作放进脚本

当同一组确定性命令反复被遗漏或写错时，再增加脚本。下面的 `scripts/check-diff.sh` 只读取 Git 状态，不修改文件：

```bash
#!/usr/bin/env bash
set -euo pipefail

git rev-parse --show-toplevel >/dev/null
git diff --check
git diff --cached --check
git status --short
git diff --stat
git diff --cached --stat
```

为脚本增加执行权限后，可以独立运行：

```bash
chmod +x repo-bugfix/scripts/check-diff.sh
repo-bugfix/scripts/check-diff.sh
```

脚本只能证明差异格式有效，并展示文件范围和统计。它无法判断某一行是否服务当前 Bug，Agent 仍要读取完整 diff，并对照任务范围逐项检查。

测试、类型检查和构建命令因仓库而异，先从项目自己的 `AGENTS.md`、`CONTRIBUTING.md`、包管理配置或 CI 配置中确认。把已经验证的选择顺序写入 `references/repository-conventions.md`：

```md
# Repository conventions

## Validation order

1. Run the narrow regression for the changed behavior.
2. Run checks required by the nearest repository instructions.
3. Run the broader build or test gate required before delivery.
4. Record commands, exit codes, skipped checks, and reasons.

## Generated files

Do not edit generated output directly. Identify its source and use the documented generator.
```

参考资料保存变化较慢、只在特定仓库需要的信息。主文件通过相对路径指向它，避免层层跳转。某条规则能够由脚本稳定检查后，应把检查放进脚本，让正文只说明何时运行以及如何解释结果。

## 让权限边界真正生效

写在 `SKILL.md` 里的边界可以指导模型，无法单独形成安全隔离。宿主运行时仍要限制文件系统、工具、网络和凭证，并在部署、推送、删除和外部消息等副作用前执行授权检查。

`repo-bugfix` 应采用以下运行边界：

- 文件读写根目录固定为当前仓库，解析真实路径后拒绝越界访问。
- `.env`、密钥目录、凭证存储和生产数据不进入模型上下文。
- 只提供完成当前修复所需的搜索、读取、编辑和本地验证工具。
- 部署、推送、网络请求和外部通知不属于该 Skill 的工具集合。
- 工具输出仍按未验证观察处理；只有测试、Schema、退出码和差异检查通过后，才提交为完成证据。

官方规范包含可选的 `allowed-tools` 字段，并明确标记其为实验能力、不同 Agent 实现的支持程度可能不同。因此本文不依赖这个字段完成授权；实际权限以宿主沙箱和执行策略为准。

## 用未见过的 Bug 验收

Skill 的验收任务不能参与它的编写。登录回跳案例已经出现在正文和指令中，只能作为训练样例。准备一个根因未知、文件位置不同的保留案例，例如：

```text
期望：用户只修改昵称后，原有时区保持 Asia/Shanghai
实际：保存资料后，时区被重置为 UTC
约束：只能修改当前仓库，不得读取生产数据或部署
```

在未提示具体文件和根因的情况下运行 `repo-bugfix`，按以下标准验收：

1. `description` 能让 Agent 判断该任务适用 `repo-bugfix`。
2. 修改前先用测试或最小操作复现，保存命令、退出码和实际结果。
3. 搜索范围由新证据驱动，没有套用登录案例的文件名和会话键。
4. 补丁只修改根因所需代码，并补充能捕获旧行为的回归测试。
5. 聚焦测试和仓库要求的更广验证均通过；无法运行的检查被明确报告。
6. 完整 diff 没有无关改动、生产凭证、生成产物或意外副作用。

只在当前案例成功仍不够。继续选择不同模块、不同错误类型的保留任务，并重跑已经通过的旧任务。新版本应在扩大适用范围的同时保住原有结果；若它开始搜索固定文件或跳过复现，说明指令过拟合了登录案例。

## 用失败证据迭代版本

每次失败先判断应该修改哪一层：

- **触发错误**：补充 `description` 中的任务信号或排除项。
- **步骤遗漏**：修改 `SKILL.md` 的稳定流程或停止条件。
- **命令反复写错**：修正脚本，并为脚本增加可重复检查。
- **仓库知识过时**：更新参考资料，不扩大主流程。
- **权限越界**：先修宿主策略，再补 Skill 中的显式提醒。

一次只针对已复现的失败做最小修改，然后运行两组测试：触发该修改的新保留案例，以及此前已经通过的回归案例。新版本只有在目标失败消失、旧任务没有退化、权限边界未放宽时才被接受。

版本号可以保存在 `metadata.version`，Git 记录保存实际差异和评审历史。可以采用常见的版本约定：措辞修正增加补丁版本；新增兼容步骤或可选资源增加次版本；输入、输出或权限契约发生不兼容变化时增加主版本。版本号只标识变更，验证记录才说明这个版本是否值得使用。

完成后的 `repo-bugfix` 应满足以下条件：目录通过当前 Agent 实现的格式检查；触发描述能覆盖目标 Bug 并排除生产操作；主流程只含已验证方法；脚本可独立运行；参考资料来源明确；至少一个未见过的 Bug 和既有回归案例通过相同验收标准。
