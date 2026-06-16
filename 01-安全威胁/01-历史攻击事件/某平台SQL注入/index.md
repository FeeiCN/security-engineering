---
slug: /platform-sqli
title: 某平台SQL注入影响数十万企业
---

**关键基础服务的安全漏洞具有连锁放大效应。** 短信提供商掌控着大量企业的通信通道，一个 SQL 注入漏洞就能暴露所有客户的发送权限和用户数据。这类基础服务商承载的不只是自身的业务风险，而是其整个客户生态的安全风险。

<!-- truncate -->

## 1 漏洞入口

**弱口令与注入漏洞的叠加，使初始入口极易突破。** 通过之前发现的弱口令登录系统后，在统计查询接口的时间参数处发现注入点。该参数未经充分过滤，存在布尔盲注、报错注入和时间盲注三种可利用的注入类型，攻击者可借此枚举数据库结构并提取数据。

## 2 漏洞证明

**注入点一旦打通，整个数据库体系随之暴露。** 经过枚举，目标服务器存在 16 个数据库，涵盖短信业务的各个核心模块。

**DATABASES(16)**

```
[*] 400_gsms
[*] 400_list
[*] commission
[*] gsms2_init
[*] gsms_init
[*] information_schema
[*] list2_init
[*] list_init
[*] mos2_gsms
[*] mos_gsms
[*] mos_gsms2_1
[*] mos_gsms_2
[*] mos_list2_1
[*] mysql
[*] performance_schema
[*] test
```

**数据规模揭示了这个平台的实际体量。** 仅单库中的消息记录和联系人数据就已达到千万量级：

```
Database: mos_gsms2_1
+-------------------------------+---------+
| Table | Entries |
+-------------------------------+---------+
| gsms_msg_ticket | 4302497 |
| gsms_contact | 4191441 |
| gsms_statereport | 4031478 |
| gsms_deduct_record | 599589 |
| gsms_user_role | 360443 |
| gsms_user_role20130408 | 360110 |
| gsms_user_role20130409 | 360052 |
| gsms_msg_frame | 359263 |
| gsms_msg_pack | 241054 |
| gsms_non_white_list | 158632 |
| gsms_red_list | 124257 |
| gsms_account_carrier_price | 100360 |
| gsms_user | 58597 |
| gsms_user20130403 | 58523 |
| gsms_user20130401 | 58518 |
| gsms_user20130312 | 58275 |
| gsms_biztype_specnum | 53074 |
| gsms_enterprise_apply_detail | 49971 |
| gsms_user_business_type | 47656 |
| gsms_enterprise_specnum_bind | 39635 |
| gsms_user_account_bind | 33256 |
| gsms_business_type | 13511 |
| gsms_capital_account | 12545 |
| gsms_enterprise_apply | 12542 |
```

**Users(56)**

```
[*] 'cactiuser'@'192.168.10.89'
[*] 'censerver'@'192.168.10.89'
[*] 'censerver'@'localhost'
[*] 'chengxuan'@'192.168.%'
[*] 'chengxuan'@'localhost'
[*] 'innotop'@'192.168.10.83'
[*] 'monitor'@'192.168.10.89'
[*] 'monyog'@'192.168.%'
[*] 'monyog'@'localhost'
[*] 'moshengkuo'@'192.168.%'
[*] 'mscheck'@'127.0.0.1'
[*] 'mscheck'@'192.168.%'
[*] 'mscheck'@'localhost'
[*] 'private'@'192.168.10.86'
[*] 'private'@'localhost'
[*] 'program'@'172.16.200.10'
[*] 'program'@'172.16.202.101'
[*] 'program'@'172.16.202.102'
[*] 'program'@'172.16.202.203'
[*] 'program'@'183.232.65.44'
[*] 'program'@'192.168.%'
[*] 'program'@'192.168.10.100'
[*] 'program'@'192.168.10.101'
[*] 'program'@'192.168.10.102'
[*] 'program'@'192.168.10.98'
...
[*] 'program'@'localhost'
[*] 'repl'@'192.168.10.%'
[*] 'repl'@'192.168.10.130'
[*] 'root'@'%'
[*] 'root'@'192.168.%
[*] 'root'@'localhost'
[*] 'super'@'192.168.%'
[*] 'super'@'192.168.10.86'
[*] 'super'@'localhost'
[*] 'wanglianguang'@'%'
[*] 'wanglianguang'@'192.168.%'
[*] 'wanglianguang'@'localhost'
[*] 'xiaozhuan'@'192.168.%'
[*] 'xiaozhuan'@'localhost'
```

## 3 漏洞影响

![](/media/某平台SQL注入/v_xuanwu_01-1024x555.webp)
![](/media/某平台SQL注入/v_xuanwu_02-1024x598.webp)
![](/media/某平台SQL注入/v_xuanwu_03-1024x598.webp)
![](/media/某平台SQL注入/v_xuanwu_04-1024x570.webp)
![](/media/某平台SQL注入/v_xuanwu_05-1024x592.webp)
![](/media/某平台SQL注入/v_xuanwu_06-1024x592.webp)
![](/media/某平台SQL注入/v_xuanwu_07-1024x592.webp)
![](/media/某平台SQL注入/v_xuanwu_08-576x1024.webp)

**该平台所有产品数据库均已暴露，短信库中的企业用户数量可达数万家。** 这意味着攻击者不只获取了用户数据，还取得了每个企业客户的短信发送权限——可以以这些企业的名义向其终端用户发送任意内容。

**广泛的客户依赖关系本身就是一个安全风险集中点。** 当一个平台同时为金融、零售、物流、医疗等行业提供关键通信服务时，该平台的安全级别下限决定了所有客户的安全上限。涉及的客户类型包括：

- 各大银行
- 东风悦达起亚
- 安利中国
- BMW中国
- 人人快递网
- 中通物流
- 我要旅行网
- 娃哈哈集团
- GXG
- 唯品会
- 酷讯
- …

**掌握关键基础服务的厂商，其安全投入应当对齐其所服务的最高风险行业标准。** 银行、医疗、证券等行业有严格的安全合规要求，但当这些企业的短信通道外包给一个未达到同等安全标准的第三方时，合规边界就出现了缺口。这种依赖关系的安全脆弱性，是整个行业需要系统性解决的问题，而不只是单个厂商的修复任务。

漏洞已报告给CNCERT/乌云/补天或厂商且已修复完成，感谢厂商的奖励。
