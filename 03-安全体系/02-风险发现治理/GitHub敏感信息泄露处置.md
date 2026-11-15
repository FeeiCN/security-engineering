---
slug: /github-sensitive-data-response
title: GitHub 敏感信息泄露的处置边界
icon: lock-icon
description: 发现 GitHub 中的密码、Token 和其他敏感信息后，如何撤销凭据、调查影响、清理仓库副本并选择正确的 GitHub 移除流程。
published_at: '2026-11-12'
---

发现密码、Token、私钥或其他敏感信息被提交到 GitHub 后，首要目标是让泄露内容失去利用价值，再处理仓库中的公开副本。只删除文件或仓库，不能代替凭据撤销，也不能自动清除他人的 clone、仍包含敏感提交的 fork 和 GitHub 上的缓存引用。

本文整理截至 2026 年 11 月仍适用的处置边界。早期使用 GSIL 监控 GitHub 泄露的实践及当时的运行方式，保留在历史演讲快照[《GSIL：GitHub敏感信息泄漏监控》](/gsil)中。

## 先撤销或轮换凭据

如果泄露内容能够用于访问系统，应先到对应服务撤销或轮换凭据。GitHub 的[敏感数据移除说明](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/removing-sensitive-data-from-a-repository)也将撤销或轮换密码、Token 等凭据列为第一步。凭据失效后，再判断是否需要重写 Git 历史。

处置时至少需要确认：

- 凭据能够访问哪些系统、数据和操作；
- 凭据从何时开始公开，是否出现在其他分支、标签、Issue、Pull Request 或构建产物中；
- 访问日志中是否存在异常来源、异常时间或超出预期的操作；
- 使用旧凭据的程序和自动化任务是否已经切换，避免轮换后重新启用旧值。

删除仓库、删除文件或提交一个“清理凭据”的新版本，都不会让已经公开的凭据自动失效。即使暂时没有发现异常使用，也应按凭据实际权限完成撤销或轮换。

## 清理仓库历史及其副本

确认凭据已经失效后，可以根据内容敏感程度和公开范围决定是否重写仓库历史。重写历史会改变提交哈希，并可能影响现有分支、标签、Pull Request、签名和协作者的本地副本，因此需要与仓库协作者协调。

历史重写只能处理可控制的仓库副本，不能直接修改其他用户的本地 clone。仍包含原提交的 fork 也会继续提供访问，需要联系 fork 所有者处理。完成重写后，还要避免协作者从旧 clone 再次推送被删除的提交。

GitHub 文档说明，完成历史重写后，可以通过 [GitHub Support](https://support.github.com/) 请求清除 GitHub 上缓存的视图和 Pull Request 中对敏感数据的引用。因此，更准确的边界是：删除文件、仓库或重写历史本身不能清除所有副本；站内缓存引用可以在完成必要清理后请求 GitHub Support 协助处理。

## 按 API 规则运行监控

[GSIL 仓库](https://github.com/FeeiCN/GSIL)当前已经归档，GitHub API 显示最后一次代码推送时间为 2023-11-28。历史演讲中的扫描规则数量、Token 数量和运行频率不能直接作为当前可复现结果。重新部署前，需要评估依赖版本、GitHub API、搜索能力和认证方式。

持续调用 GitHub REST API 时，应按 GitHub 的[最佳实践](https://docs.github.com/en/rest/using-the-rest-api/best-practices-for-using-the-rest-api)处理请求：

- 读取 `x-ratelimit-remaining`、`x-ratelimit-reset` 和 `retry-after` 等响应头；
- 对轮询请求使用经过认证的条件请求，在内容未变化时利用 `304 Not Modified`；
- 使用队列控制请求，避免并发请求触发次级限额；
- 遇到限额错误后暂停请求，并按响应头或指数退避策略重试；
- 组织级自动化根据权限范围和生命周期评估 GitHub App，避免长期共享个人 Token。

GitHub [服务条款](https://docs.github.com/en/site-policy/github-terms/github-terms-of-service#h-api-terms)明确禁止通过共享 API Token 超过限额，并限制同一人或法律实体持有免费账号的数量。因此，注册多个账号、申请多个 Token 或共享 Token 规避限额，不应作为监控系统的扩容方案。

## 选择对应的移除流程

敏感信息位于自己能够管理的仓库时，应先自行撤销凭据并清理内容。内容位于无法控制的仓库、fork 或其他公开位置时，可以先联系仓库所有者；涉及高风险私密信息且无法及时处理时，再按内容类型选择 GitHub 的移除流程。

GitHub 的[私密信息移除流程](https://docs.github.com/en/site-policy/content-removal-policies/github-private-information-removal-policy)适用于公开后会给个人或组织带来具体安全风险的私密信息，例如用户名与密码组合、访问 Token，以及能够访问服务器、网络或域名的其他凭据。请求中需要提供敏感内容的准确位置，并说明它造成的具体安全风险。该流程通常针对文件中的具体私密信息，不默认用于删除完整文件或整个仓库。

[DMCA 下架流程](https://docs.github.com/en/site-policy/content-removal-policies/guide-to-submitting-a-dmca-takedown-notice)只处理受版权保护内容的侵权主张，不适合用于删除密码等敏感数据。GitHub 的指南没有承诺固定三天完成处理，因此不能把历史经验中的响应时间当作当前服务承诺。

无论选择哪种移除流程，都不能等待平台处理后再撤销凭据。平台移除解决的是公开内容及其托管位置，凭据撤销解决的是访问能力，两者需要分别完成。
