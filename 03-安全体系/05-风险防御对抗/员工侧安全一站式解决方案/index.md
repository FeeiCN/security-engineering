---
slug: /gugu
title: 一种干掉所有密码的安全实践
icon: shield-check
sidebar_badge:
  text: 演讲
  color: secondary
---

<iframe
  src="/media/files/员工侧安全一站式解决方案最佳实践.pdf"
  title="一种干掉所有密码的安全实践 PDF"
  width="100%"
  height="640"
  style={{border: '1px solid rgba(15, 23, 42, 0.12)', borderRadius: '12px', background: '#fff'}}
>
</iframe>

<div style={{margin: '12px 0 24px'}}>
  <a className="button button--primary" href="/media/files/员工侧安全一站式解决方案最佳实践.pdf" target="_blank" rel="noopener noreferrer">
    打开 / 下载 PDF
  </a>
</div>

*2018年09月19日 · 西安 · SSC*

员工侧是企业最容易被忽视、又最容易被突破的攻击面。服务器和业务系统通常有专门的团队持续加固，员工手中的笔记本、手机和办公账号却分散在各处，一旦被攻破，攻击者就能以合法身份直接进入内网。当整体入侵生命周期中的安全产品布局趋于完善后，人反而成为最大的问题。GuGu 就是把员工侧的入网、鉴权、终端和密码风险收敛到一个客户端统一管控的实践。

## 背景与思路

### 办公网络的隐患

先看早期的网络架构：

- 通过 802.1x 认证账号密码通过后就进入内部办公网；
- 办公网为了访问效率拉了一条专线到 IDC 机房；
- 默认办公网和机房的网络是相通的，仅仅通过账号认证来限制访问。

这套结构让物理渗透变得非常容易：攻击者无需突破生产环境，只要潜伏进公司楼下的咖啡馆爆破办公室 Wi-Fi，或者混入公司用一根网线接入办公网络，就能拿到一部分内网访问权限——京东就出现过类似案例。风险的核心在于，进入办公网就等于拿到了入场券，而办公网的访问控制只有账号密码加 802.1x。

回过头看，办公网对日常工作的意义其实有限。员工日常工作可以分为两部分：访问各种内网系统（内网门户、工单审批、数据平台、业务营销管理后台、行政办公系统等），以及通过 SSH、FTP 等方式访问服务器。办公网的价值，主要是通过专线解决访问机房资源和局域网内文件互传的速度问题。

那为什么不把授权范围从"办公网"细化到"每个人的电脑"？让你的电脑而非你接入的网络来决定是否有权限进入内网。如果不考虑历史包袱，理想状态是去掉办公网这一层，每个人独立连入内网，在任何地方都能办公。

### VPN 软件的隐患

- SRC 曾收到白帽子报告的一个漏洞：通过员工泄露的邮箱账号密码拿到邮箱权限，翻看邮件发现附件里有 VPN 配置文件和 Google 二次验证配置文件，从而直接用账号密码加 Google 验证码连入内网。
- 原先的 VPN 基于 Tunnelblick，每次连接需要输入账号密码加 Google 验证码，且经常断线重连，严重影响效率；线上线下环境分离，研发调试时还要频繁切换 VPN 配置文件，非常繁琐。

针对第一个案例，我们强制所有员工邮箱绑定微信、每次登录扫码，缓解邮箱被盗的风险；但邮件只是泄露渠道之一，配置文件下次可能被传到 GitHub 或网盘，根源仍是 VPN 配置文件和鉴权配置文件本身会泄露。第二个案例则暴露了安全与体验的冲突：每次输入验证码足够安全，但体验糟糕。如果不考虑历史包袱，我们希望自研一套鉴权协议，并在其外面包一层 VPN，让它更方便也更安全。

### 账号密码的隐患

- 各类网站使用同一密码、密码长期不变都非常普遍。对外系统虽然有两步验证，但只要存在类似 SSRF 的漏洞能打进内网，就能畅通无阻。
- 在局域网下获取某个员工密码的方法还有很多，而知道账号密码就能登录其所有有权限的内部系统。

即便在密码策略上做增强——强制定期更换、强度校验、不允许相同相似密码——也无法彻底解决密码带来的问题，而给所有内部系统加二次验证又会严重拖累效率。如果不考虑历史包袱，我们希望去掉密码这种古老的验证方式，设计一套更方便、更有效的鉴权方案。

### 员工客户端的隐患

- 当拿到组件类漏洞（如 Struts2、Fastjson 的反序列化）或常规代码漏洞（如 SSRF、新型 SQL 注入）的情报时，安全团队可以用 Cobra 扫描集团项目判断影响范围；但当情报指向客户端软件时，团队无法及时主动排查受影响的范围。
- 一些同学离开座位不锁屏、电脑不设密码，甚至关闭部分安全设置。

我们全员使用 MacBook，终端上少了很多 Windows 的风险，但仍出过几次影响较大的事件，比如 macOS 端的几次高危漏洞：SourceTree 的远程命令执行、嵌入恶意后门的 XcodeGhost、JetBrains 全家桶（PyCharm、IDEA、PHPStorm、WebStorm）的远程命令执行。对于不安全的配置，仅靠意识无法兜底，必要的技术检测必须并行。理想状态下，我们希望有一个终端软件能把已安装软件、版本号和安全配置信息收集上来，出现威胁事件时快速判断影响范围。

<!-- truncate -->

## 从网络边界转向零信任

上面四类隐患指向同一个根因：传统模型把信任建立在网络位置上。只要接入了内网就被默认可信，VPN 成了单一的信任边界，一旦被绕过，内部便一马平川。零信任重新定义了信任的基础：

- **没有内网。** 不再以"是否在内网"作为信任依据，所有访问一视同仁。
- **没有密码。** 用更强的身份凭证替代静态口令，消除口令泄漏这一最大风险源。
- **一切访问都需要认证。** 任何资源的访问都要经过身份校验，不存在默认放行的区域。
- **认证同时绑定用户与设备。** 单有用户身份不够，必须同时确认设备可信，被盗的凭证在陌生设备上无法直接使用。
- **全流量可审计。** 所有访问行为留痕，为风险识别和事后溯源提供依据。

信任的根基从"网络位置"转移到"用户加设备"，这正是 GuGu 的设计出发点——用一个客户端，解决上面遇到的所有问题。

## 入网管控

通过 GuGu 控制密钥，让内置的 VPN 更安全也更方便，随时随地一键连入内网，杜绝从办公网入口渗透进内网的可能，也不会再出现 VPN 配置文件泄露导致的内网漫游。同时为提升开发效率，在 VPN 基础上增加了科学上网能力，方便开发过程中查阅资料和学习。

### 全平台 VPN 选型

在 OpenVPN、IKEv1（IPSec）、IKEv2 等方案中权衡跨平台、可扩展、稳定性和安全性，最终选型：

- Windows / macOS / Android 使用 OpenVPN；
- iOS 使用 IKEv2。

落地中主要的坑集中在下发路由表和下发 DNS 上。

### VPN 认证方案

![VPN 认证方案](/media/员工侧安全一站式解决方案/gugu_vpn_auth.webp)

将计算出的最终结果当作密码去连接 VPN。VPN 验证服务发现密码中有四处 `gu`/`ug` 分隔符、且长度为 89 位时，就走 GuGu SID 验证逻辑。

```markdown
# 【拼接原始数据】拼接 sessionID（会话ID）、serialNumber（序列号）、timestamp（时间戳）、分隔符（gu/ug）
original = gu + sessionID(32) + ug + serialNumber(34) + gu + timestamp(10)

# 【统一平台数据交互】转换为小写
original = tolower(original)

# 【防止请求重放攻击】取 original sha1 的前五位，并和 original 拼接
result = original + ug + sha1(original)[:5]
# len = 89 = 2*4 + 32 + 34 + 10 + 5
```

### 入网架构

![入网架构](/media/员工侧安全一站式解决方案/gugu_network_architecture.webp)

## 一键免密

GuGu 登录过程中集成了多重安全策略，保证账号安全。其他软件可以信任 GuGu 的登录状态，从而形成一条信任链。登录和使用大部分日常系统时都会唤起 GuGu 的授权框，实现一键免密鉴权。当所有系统的免密鉴权全部覆盖后，就不再有需要密码的系统，GuGu 即可接管全部系统，把密码替换为随机 Token，员工不必再记住任何密码。

### Web 端免密

![Web 端免密方案](/media/员工侧安全一站式解决方案/gugu_password-free01.webp)

GuGu 启动时拉起本地 HTTP Server：

- 默认监听 `127.0.0.1` 的 `51273` 端口；
- 若 `51273` 被占用则监听 `51274`；
- 若 `51274` 也被占用则放弃启动，并上报信息给 GuGu 服务端。

处理获取 Token 请求时：

- 不存在的接口一律返回 403；
- 不存在的 HTTP Method 一律返回 405；
- 参数格式错误返回 `1002`；
- 接口形式为 JSONP；
- 只允许固定 User-Agent 和固定域名（Referer）调用；
- 一切正常返回 `1001`。

接入方客户端的调用流程：

- **判断是否安装 GuGu**：先请求 `127.0.0.1:51273` 的版本接口，超时设为 1s；超时则用 `51274` 重试；仍超时则说明未安装，提示用户使用备选方式登录。
- **获取 Token**：若已安装，则请求本地 Token 接口，超时设为 33s（弹窗询问时间为 30s）。

### App 端免密

![App 端免密方案一](/media/员工侧安全一站式解决方案/gugu_free-password-for-mobile01.webp)
![App 端免密方案二](/media/员工侧安全一站式解决方案/gugu_free-password-for-mobile02.webp)

### 三方系统免密

![三方系统免密方案](/media/员工侧安全一站式解决方案/gugu_third-party_authentication.webp)

## 终端安全

GuGu 的登录体系和信任设备机制保障了基础安全；通过预先收集电脑已安装的软件、版本号和安全配置信息，在出现软件安全情报时可以快速确认影响范围；通过开启自动锁屏功能，缓解离开电脑时未锁屏的风险。

## 自身加固

### 登录限制

预警信息包含规则名、用户名、错误次数、IP、序列号及电脑名称：

- 预警单电脑爆破账号密码；
- 预警多电脑单 IP 爆破（序列号可伪造时，使用 IP 预警）；
- 预警账号被爆破（IP 和序列号都被破解时，预警账号爆破）。

以下信息可在 Metabase 动态配置：

- 各规则的错误次数及封禁时间阈值（调优最佳参数）；
- 公司 IP 白名单（防止公司 IP 变动）；
- 序列号白名单（防止紧急情况下用户无法连接内网）。

### Token 签名算法

将所有待提交参数的 key 取出并按字典序排序，拼接对应的 value 计算 sha1，作为 `sign` 一并提交。例如提交 `k1=v1&k2=v2`，则 `sign = sha1(v1+v2)`，最终提交 `k1=v1&k2=v2&sign=sha1(v1+v2)`。

```python
import hashlib

def generate_sign(params):
    """
    Generate sign by params
    :param params:
    :return:
    """
    if not isinstance(params, dict):
        raise TypeError
    values = '#%$'.join(str(params[x]) for x in sorted(params))
    sha = hashlib.sha1()
    sha.update(values.encode())
    return sha.hexdigest()

params = {
    'k2': 233,
    'k1': '.cn',
    'a': 'feei',
}
sign = generate_sign(params)
assert sign == '7344da1feae2faa432c2fa8aa70839d6bedde142'
```

```java
import java.util.List;
import java.util.Map;
import java.util.Objects;
import java.util.Set;
import java.util.stream.Collectors;

public static final String SEPERATE_CHAR = "#%$";
public static String generateSign(Map<String, Object> param) {
    Set<String> keys = param.keySet();
    //过滤掉sign
    List<String> sortedKeys = keys.stream().filter(key -> !Objects.equals(key, "sign")).sorted().collect(Collectors.toList());
    String calcSign = sortedKeys.stream().map(param::get).map(String::valueOf).collect(Collectors.joining(SEPERATE_CHAR));
    calcSign = DigestUtils.sha1Hex(calcSign);
    return calcSign;
}
```

### Token 交互流程

![Token 交互流程](/media/员工侧安全一站式解决方案/gugu_token.webp)

## 项目成型

项目起始时，我们也调研了几家大企业对这些问题的解决方案，大多是常规的单项治理。而这些问题大都由密码引起，于是提出了一个大胆的想法——去密码：把所有密码都去掉，交由可信设备处理。GuGu 的雏形由此而来，用一个客户端解决上面遇到的所有问题。整个项目涉及安全、运维、内网、IM 四大团队，覆盖 macOS、Windows、Android、iOS 四大平台。

典型场景：

- 在家 / 公司办公：macOS / Windows 入网管控；
- 外出旅游：Android / iOS 入网管控；
- 查阅资料：科学上网；
- 不用密码登录各类系统：一键免密鉴权；
- 离开座位锁屏：自动锁屏；
- 新型高危软件漏洞应急响应：软件版本情报信息收集。

![GuGu 项目组](/media/员工侧安全一站式解决方案/IMG_3455.webp)

*GuGu 项目组*
