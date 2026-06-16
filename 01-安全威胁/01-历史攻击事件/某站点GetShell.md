---
slug: /site-getshell
title: 某站点GET SHELL
---

**Git 泄露和在线更新脚本的组合，会将正常的部署工具变成攻击入口。** 当 auth-token 暴露在版本控制系统中，一次提交就能完成对服务器的完整接管。这个案例的根本问题不在于某个具体的代码漏洞，而在于开发便利性工具在没有访问控制保护的情况下直接暴露在生产环境中。

## 1 信息泄露

**公开的 Git 泄露可以直接暴露私有代码仓库的访问凭据。** 通过浏览器插件检测发现目标站点存在 Git 信息泄露。

![](/media/01-历史攻击事件/v_baixing_01-1024x618.webp)

Edge 插件检测（左下角提示）到百姓金融存在 Git 泄漏。

![](/media/01-历史攻击事件/v_baixing_02.webp)

**泄露的 git 信息指向了一个部署在 GitHub Private 的项目，且携带了 auth-token。**

![](/media/01-历史攻击事件/v_baixing_03.webp)

携带 auth-token 的仓库地址意味着可以直接以该 token 为授权码克隆代码，无需任何额外的身份验证。

<!-- truncate -->

## 2 GET SHELL

**在线更新脚本是开发便利性的产物，也是未受保护的远程执行接口。** `pull.php` 这类脚本在开发团队中很常见——它的作用是让服务器通过 HTTP 请求直接从 Git 拉取最新代码，省去了 SSH 登录部署的步骤。这个设计在内部开发流程中提升了效率，但一旦脚本暴露在外网且没有任何访问控制，它就变成了任何持有仓库写权限的人都可以直接触发的远程代码执行入口。

代码审计后未发现硬编码密码等直接可利用的内容，但发现了 `pull.php`。通过 auth-token 将 webshell 直接 commit 到代码仓库，然后访问 `pull.php` 触发服务器拉取最新代码，即可 GetShell。

![](/media/01-历史攻击事件/v_baixing_03.webp)
![](/media/01-历史攻击事件/v_baixing_05-1024x561.webp)

DB 账号信息

![](/media/01-历史攻击事件/v_baixing_06-1024x561.webp)

DB 数据

## 3 供应链风险链

**多个项目使用相同的弱防御机制，说明安全设计缺陷是系统性的。** 在同一组织下的另一个项目（聚车商管理后台）中发现了完全相同的问题：代码仓库 URL 携带 auth-token，生产环境同样存在 `pull.php`。这种重复性说明安全风险的来源是统一的开发规范或工具模板，而不是某个开发者的个人失误。

`https://29ab43d962**********c0480b3ed157687:x-oauth-basic@github.com/****/jucheshang_admin`

通过该地址直接上传 webshell 并 commit，再通过 `http://cartier.****.cn/pull.php` 触发部署，即可在另一个域名下同样完成 GetShell。

![](/media/01-历史攻击事件/v_baixing_07-1024x619.webp)

SHELL 地址：`http://cartier.****.cn/js/faq.php`

![](/media/01-历史攻击事件/v_baixing_08.webp)

**Git 泄露的防治需要从两个层面入手。** 第一层是防止 `.git` 目录被 Web 服务器直接访问，可通过 Nginx 或 Apache 配置屏蔽该路径。第二层是防止 auth-token 出现在任何公开或可能泄露的位置，包括 HTTP 响应头、页面源码、配置文件和 Git 历史记录。对于 `pull.php` 这类部署脚本，至少应当添加 IP 白名单或 token 验证，避免任何能访问 URL 的人都能触发代码更新。

漏洞已报告给CNCERT/乌云/补天或厂商且已修复完成，感谢厂商的重视及现金奖励。
