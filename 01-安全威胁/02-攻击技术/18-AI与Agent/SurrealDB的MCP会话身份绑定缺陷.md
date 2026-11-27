---
slug: /surrealdb-mcp-session-hijacking
title: MCP 会话为什么会让低权限用户获得管理员权限：SurrealDB 的身份绑定缺陷
sidebar_label: SurrealDB MCP 会话身份绑定
icon: bug-icon
description: MCP 会话必须同时绑定真实用户、有效权限与授权上下文；SurrealDB 的历史漏洞展示了认证正确却复用高权限会话的风险及修复边界。
published_at: '2026-11-04'
---

# MCP 会话为什么会让低权限用户获得管理员权限：SurrealDB 的身份绑定缺陷

MCP Server 通常需要同时处理两种身份：一是当前 HTTP 请求携带的访问凭证，二是 MCP 会话初始化时绑定的用户身份。如果服务器使用长期会话保存数据库连接、权限和上下文，就必须保证后续请求仍然属于原来的用户。

SurrealDB 的 CVE-2026-121419 展示了这条边界失效后的后果：攻击者持有自己的低权限 Token，只要另外获得高权限用户的 MCP Session ID，在特定受影响版本中，就可能借用原会话执行高权限数据库操作。

问题不在 JWT 签名校验，也不在模型是否遵守安全指令，而在于**系统将“使用同一种认证方式”错误地当成了“属于同一个用户”**。

这类缺陷值得关注，因为它不仅存在于 MCP。任何使用会话池、持久连接或复用执行上下文的系统，都可能因为身份等价关系定义不完整，而将一个用户的权限交给另一个用户。

## 一个已经修复的真实漏洞

2026 年 11 月 3 日，CVE-2026-121419 正式进入公开漏洞数据库。记录显示，受影响范围为 SurrealDB 3.1.0 至 3.3.0 之前的版本，修复版本为 3.3.0。

需要区分几个日期：CVE 记录将 2026 年 10 月 19 日列为首次公开日期；GitHub 官方发布记录显示，SurrealDB 3.3.0 实际发布于 9 月 28 日。因此，这不是 11 月 3 日才出现、尚无补丁的漏洞。11 月 3 日是此次 CVE 记录的正式发布日期。([CVE 原始记录](https://github.com/CVEProject/cvelistV5/blob/main/cves/2026/121xxx/CVE-2026-121419.json))([SurrealDB 3.3.0 官方发布记录](https://github.com/surrealdb/surrealdb/releases/tag/v3.3.0))

漏洞成立需要几个条件：目标使用受影响版本的 HTTP MCP 服务；攻击者拥有同一 JWT/Bearer Access Method 签发的有效低权限凭证；攻击者还必须取得另一个高权限用户正在使用的 `mcp-session-id`。

这些条件不能省略。漏洞并不意味着任意未认证互联网用户都能直接取得数据库管理员权限，也没有证据表明它已经被用于真实入侵。

不过，一旦条件满足，风险就可能超出普通 Session ID 泄露：**服务器即使重新认证了攻击者提交的凭证，也可能继续使用原会话保存的高权限执行上下文。**

## 漏洞发生在两次正确操作之间

SurrealDB 的 HTTP MCP 服务在初始化时建立会话，将认证后的数据库 `Session` 保存起来，随后通过 `mcp-session-id` 关联请求。

为了避免攻击者使用别人的 Session ID，服务端还会在每次请求时检查当前认证主体是否与会话绑定的主体一致。

看起来已经有了两道控制：HTTP 请求认证和会话身份一致性检查。

但这两道控制之间依赖一个关键前提：身份指纹必须能够唯一地区分不同用户。

我读取并比较了 SurrealDB 官方仓库的 `v3.2.0` 与 `v3.3.0` 固定版本源码。受影响版本中的 `BoundSubject` 结构可以简化为：

```rust
struct BoundSubject {
    level: Level,
    identity: String,
}

fn from_session(session: &Session) -> BoundSubject {
    BoundSubject {
        level: session.au.level().clone(),
        identity: session.au.id().to_string(),
    }
}
```

其中，`level` 描述认证范围，`identity` 来自 `Auth::id()`。

问题出现在生成 `Auth::id()` 的上游逻辑。

在 `v3.2.0` 的 JWT/Bearer 数据库认证路径中，源码使用 Access Method 名称构造 Actor：

```rust
session.au = Arc::new(Auth::new(Actor::new(
    de.name.to_string(),
    roles,
    Level::Database(ns.clone(), db.clone()),
)));
```

`de.name` 是 Access Method 的名称，不是 Token 持有者的唯一身份。

这意味着两个不同用户，即使各自的 Token 合法且权限不同，只要通过同一个 Access Method 认证，在这段逻辑下就可能得到相同的 `Auth::id()`。

最终，会话一致性检查比较的是相同的认证范围和相同的 Access Method 名称，而不是两个真正不同的用户。

[SurrealDB v3.2.0 身份绑定源码](https://github.com/surrealdb/surrealdb/blob/v3.2.0/surrealdb/mcp/src/auth.rs)

[SurrealDB v3.2.0 JWT/Bearer 身份构造源码](https://github.com/surrealdb/surrealdb/blob/v3.2.0/surrealdb/core/src/iam/verify.rs)

## 为什么重新认证后仍然会发生越权

考虑一个简化示例。

管理员 Alice 使用一个高权限 JWT 初始化 MCP 会话。SurrealDB 验证 Token 后，建立包含 Alice 权限的数据库 Session，并向客户端返回对应的 Session ID。

普通用户 Bob 使用同一个 Access Method 签发的低权限 JWT。Bob 本来只能读取有限的数据。

假设 Bob 获得了 Alice 的 Session ID，并使用自己的合法 Token 请求该 MCP 会话。

在受影响版本中，可能出现以下过程：

```text
Alice 初始化 MCP 会话
    ↓
HTTP 认证成功：Alice / Owner
    ↓
保存 Alice 的数据库 Session
    ↓
BoundSubject = (Database, access-method-name)
    ↓
Bob 获得 Alice 的 MCP Session ID
    ↓
Bob 使用自己的低权限 Token 发出请求
    ↓
HTTP 认证成功：Bob / Viewer
    ↓
BoundSubject = (Database, access-method-name)
    ↓
会话身份一致性检查错误通过
    ↓
继续使用 Alice 的原始 Session
    ↓
以 Alice 的权限执行工具
```

关键在最后两个步骤。

问题不是 Bob 的 Token 通过认证后被直接授予 Owner 权限，而是服务器没有正确识别 Bob 与 Alice 属于不同主体，因而允许 Bob 的请求进入 Alice 已初始化的会话。

**当前请求的认证结果是 Bob，但真正执行操作的会话仍然保存着 Alice 的权限。**

这个区别很重要。

如果只测试 HTTP 认证接口，Bob 的 Token 可能始终只能得到 Viewer 权限；如果只测试 MCP Session ID 的随机性，也无法发现这个错误。

必须把两个用户、两个权限等级和同一个被复用的 MCP 会话组合起来，才能检查实际的权限混淆。

从源码结构看，这是身份验证结果与执行上下文绑定不一致的问题，而不只是 Session ID 管理问题。

## 修复为什么需要同时修改身份和权限

SurrealDB 3.3.0 的修复并不是简单增加一次 Token 检查。

首先，项目修改了 JWT/Bearer 认证路径，为不同用户生成不同的 Actor 身份。新实现使用 Access Method 和 Token 的 `sub` 声明共同构造身份。

简化后可以表示为：

```rust
fn access_actor_id(access: &str, claims: &Claims) -> String {
    match claims.sub.as_deref() {
        Some(subject) => {
            format!("{}:{access}/{subject}", access.len())
        }
        None => {
            format!("{}:{access}", access.len())
        }
    }
}
```

代码中的长度前缀还有一个作用：避免不同 Access Method 名称和 Subject 字符串在拼接后产生歧义。

例如，直接拼接时，`access=a, sub=b/c` 和 `access=a/b, sub=c` 可能产生相同的文本。使用可区分边界的编码后，就不会因为字符串表示歧义而错误地认定两个主体相同。

与此同时，`BoundSubject` 增加了权限和会话上下文字段：

```rust
struct BoundSubject {
    level: Level,
    identity: String,
    privilege: Option<Role>,
    data: Option<SessionData>,
}
```

这里实际上解决了三个不同的问题。

**身份一致性。** 即使 Alice 和 Bob 使用同一个 Access Method，也应被识别为不同用户。

**权限一致性。** 同一用户可能持有不同权限的 Token。如果初始化会话使用 Owner 权限，后续请求却只携带 Viewer 权限，不能因为用户名相同就继续复用 Owner 会话。

**授权上下文一致性。** 某些数据库权限还取决于认证时计算的组织、租户或授权条件。即使用户和角色没变，相关授权上下文也可能发生变化。如果服务器继续使用初始化时缓存的旧上下文，就可能忽略已经发生的权限变更。

SurrealDB 的修复将这些因素纳入身份指纹，以便在后续请求重新认证所得的上下文发生变化时拒绝复用旧会话。

[SurrealDB v3.3.0 身份绑定修复](https://github.com/surrealdb/surrealdb/blob/v3.3.0/surrealdb/mcp/src/auth.rs)

[SurrealDB v3.3.0 认证身份构造修复](https://github.com/surrealdb/surrealdb/blob/v3.3.0/surrealdb/core/src/iam/verify.rs)

## 修复后仍然需要满足哪些条件

这次修复还有一个非常值得关注的边界。

`v3.3.0` 的源码明确说明：如果 JWT 没有 `sub` 声明，就无法根据该声明区分同一个 Access Method 下的不同 Token 持有者。

在这种情况下，代码仍然使用 Access Method 名称作为身份表示，并给出运行时警告。

因此，不能把“升级后增加了身份比较”理解为“任何 Token 配置都能可靠实现用户级身份隔离”。

如果业务依赖不同用户的会话隔离，就必须保证用于认证的 Token 能提供可靠、稳定、可验证的用户标识。对于没有此类标识的 Token，不能仅依赖 Access Method 名称实现用户级会话区分。

另一个边界是授权上下文的稳定性。

`v3.3.0` 将认证过程中生成的 `CONTEXT` 数据纳入会话指纹，因此同一用户的授权条件发生变化时，可能触发会话拒绝和重新初始化。

这是有意义的安全设计，但也带来可用性要求：如果某个 `CONTEXT` 表达式依赖每次请求都会变化的值，即使业务授权并未真正改变，也可能导致正常请求无法继续使用既有会话。

源码专门指出了这种情况。

所以，身份指纹并不是字段越多越安全。应该绑定的是能够决定执行权限、并且具有明确变化语义的属性。

最后，身份重新认证也不能解决已经进入执行阶段的所有授权撤销竞态。它阻止的是旧会话被错误复用，并不能自动使跨队列、跨数据库或者第三方系统中已启动的业务操作停止。

## 如何验证一个 MCP Server 是否存在同类问题

这类漏洞的验证重点，不应只放在 Session ID 是否随机、Token 是否过期或者 HTTP 接口是否要求认证。

更有效的方法是构造一组隔离环境下的身份与权限交叉测试。

| 测试场景 | 预期结果 |
|---|---|
| Alice 初始化会话，Alice 使用相同权限继续调用 | 正常成功 |
| Alice 初始化会话，Bob 使用不同用户凭证请求该会话 | 拒绝 |
| Alice 使用 Owner 初始化，随后使用 Viewer 权限凭证请求同一会话 | 不得继续获得 Owner 权限 |
| 同一用户的有效授权上下文发生变化 | 不得继续沿用不匹配的旧授权上下文 |
| 缺少凭证但提供有效 MCP Session ID | 不得取得已认证用户的执行权限 |
| 合法用户重新初始化会话 | 应能按照当前权限正常完成任务 |

测试不仅需要检查 HTTP 状态码或 JSON-RPC 错误，还应检查真实执行层观察到的用户身份、权限和资源访问结果。

例如，在完全隔离的数据库中，创建两个虚构用户和具有不同访问权限的测试表。初始化时记录会话绑定的主体，再逐次记录后续请求的认证主体、实际数据库执行主体和访问结果。

如果一个请求在认证日志中显示为 Bob，但效果日志显示使用了 Alice 的权限，那么即使接口返回格式完全符合 MCP 协议，也仍然发生了权限边界失效。

对于已经修复的版本，还需要正向验证正常任务不会被过度拒绝。特别是在角色更新、租户上下文变化和重新初始化场景下，应同时衡量安全拒绝与合法操作成功率。

SurrealDB 官方 `v3.3.0` 源码包含针对不同 Subject、不同角色以及授权上下文变化的回归测试。我已核对这些固定版本的测试代码，但没有运行 SurrealDB 服务端或上游测试，因此不能将静态代码分析表述为独立完成漏洞复现。

## 真正需要保护的是身份与执行权限的对应关系

这个案例揭示了一个比普通会话劫持更普遍的问题。

传统系统经常把 Session ID 当作访问某个执行上下文的索引，再通过当前请求的 Token 验证访问者身份。但如果这两者之间没有正确的身份等价关系，就可能出现“认证正确、授权错误”的结果。

对于 MCP 和 AI Agent 系统，这种风险更值得重视。Agent 可能长期持有一个会话，期间完成大量工具调用；不同用户或子 Agent 也可能通过统一网关访问同一个服务。

因此，工程上至少需要保证三件事。

首先，每次网络请求的凭证都必须被正确验证，并且能关联到明确的实际主体，而不只是一个通用的认证配置名称。

其次，如果复用已经初始化的会话，当前主体、有效权限和影响执行结果的授权上下文必须与原会话的约束保持一致。若无法证明一致，应重新建立会话或拒绝请求，而不是继续使用之前的权限。

最后，真正的业务资源仍然需要执行自身的权限控制。即使 MCP 会话身份绑定正确，也不能让一个持有高权限服务账号的工具绕过当前用户的资源授权。

CVE-2026-121419 已经有明确修复版本，不是本次研究发现的新漏洞。它的价值在于提供了一组可以直接比较的历史源码和回归测试，让我们看到一条具体的安全边界是怎样因为身份定义不完整而失效，又怎样通过绑定真实主体、有效权限和授权上下文得到修正。

**验证 Token 合法，只能证明请求通过了身份认证；证明当前请求仍有权使用之前的执行会话，才完成了会话授权。**
