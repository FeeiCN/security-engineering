---
slug: /esd
title: 枚举子域名
icon: magnifier-icon
sidebar_position: 27
---

## 事情背景

开始渗透一个网站前，需要知道网站的网络资产，比如域名、IP、端口服务等，而IP和域名有着直接的解析关系，所以如何找到网站所有子域名是关键。

<!-- truncate -->

## 实现思路

> 在知道网站主域名的情况下，可以通过以下几种方式进行域名收集。

### 搜索引擎

使用百度、Google等搜索引擎，可通过`site`关键字查询所有收录该域名的记录，而子域名权重较高会排在前面。

以`feei.cn`为例，可以通过搜索`site:feei.cn`来获取`feei.cn`被收录的子域名。

- 百度:`https://www.baidu.com/s?wd=site:feei.cn`
- Google:`https://www.google.com.hk/search?q=site:feei.cn`
- Bing:`https://www.bing.com/search?q=site:feei.cn`
- Yahoo:`https://search.yahoo.com/search?p=site:feei.cn`

![搜索引擎查询子域名](./esd_01.webp)

缺点：接口性质的子域名不会被搜索引擎收录，存在遗漏

### HTTPS证书

#### HTTPS证书透明度

Google透明度报告中的[证书透明度项目](https://transparencyreport.google.com/https/certificates)是用来解决HTTPS证书系统的结构性缺陷，它能够让所有人查询各个网站的HTTPS证书信息，从而能发现签发了证书的子域名。[`feei.cn`的证书透明度结果](https://transparencyreport.google.com/https/certificates?cert_search_auth=&cert_search_cert=&cert_search=include_expired:true;include_subdomains:true;domain:feei.cn&lu=cert_search)

![Google证书透明度查询子域名](./esd_02.webp)

#### HTTPS证书内置支持的域名和子域名

- 获取域名的A记录IP
- 通过向该IP443端口发送请求获取其证书支持的域名和子域名

```python
# -*- coding: utf-8 -*-
"""
    CAInfo
    ~~~~~~
    根据域名A记录通过HTTPS证书查询其支持的域名和子域名
    :author:    Feei <feei@feei.cn>
    :homepage:  https://github.com/FeeiCN/ESD
    :license:   GPL, see LICENSE for more details.
    :copyright: Copyright (c) 2018 Feei. All rights reserved
"""
import os
import asyncio
import aiodns
import socket
import ssl
import traceback
class CAInfo(object):
    def __init__(self, domain):
        self.domain = domain
    @staticmethod
    def query_a(domain):
        """
        查询根域名A记录
        :param domain: 被查询的域名(某些情况下根域名可能没有A记录，可尝试www的A记录)
        :return:
        """
        loop = asyncio.new_event_loop()
        asyncio.set_event_loop(loop)
        resolver = aiodns.DNSResolver(loop=loop)
        f = resolver.query(domain, 'A')
        result = loop.run_until_complete(f)
        # 可以多个IP都尝试下，但正常情况下没有区别
        return result[0].host
    @staticmethod
    def get_cert_domains(ip):
        """
        向IP的443端口查询支持的域名情况
        :param ip:
        :return:
        """
        s = socket.socket()
        s.settimeout(2)
        cert_path = os.path.join(os.path.dirname(os.path.abspath(__file__)), 'cacert.pem')
        connect = ssl.wrap_socket(s, cert_reqs=ssl.CERT_REQUIRED, ca_certs=cert_path)
        connect.settimeout(2)
        connect.connect((ip, 443))
        cert_data = connect.getpeercert().get('subjectAltName')
        return cert_data
    def get_domains(self, only_subdomains=False):
        """
        根据HTTPS证书获取支持的域名
        :param only_subdomains: 是否只需要子域名，比如搜索baidu.com只需要*.baidu.com，不需要baidu.cn
        :return:
        """
        domains = []
        try:
            ip = self.query_a(self.domain)
            cert_domains = self.get_cert_domains(ip)
            for cert_domain in cert_domains:
                domain = cert_domain[1]
                if not domain.startswith('*'):
                    if only_subdomains:
                        if domain.endswith(self.domain):
                            domains.append(domain)
                    else:
                        domains.append(domain)
            return domains
        except Exception as e:
            return domains
if __name__ == '__main__':
    try:
        ret_domains = CAInfo('baidu.com').get_domains(only_subdomains=False)
        print(ret_domains)
    except Exception as e:
        traceback.print_exc()
```

```bash
['baidu.cn', 'baidu.com', 'baidu.com.cn', 'w.baidu.com', 'ww.baidu.com', 'www.baidu.com.cn', 'www.baidu.com.hk', 'www.baidu.hk', 'www.baidu.net.au', 'www.baidu.net.ph', 'www.baidu.net.tw', 'www.baidu.net.vn', 'wwww.baidu.com', 'wwww.baidu.com.cn', 'www.baidu.cn']
```

缺点：对于只签根域名证书的情况，存在遗漏

### 自身泄露

- 流量代理：通过[WebProxy](https://github.com/FeeiCN/WebProxy)代理电脑所有流量，再分析流量中中出现的子域名
  - 域名跳转记录中的子域名
  - Response中存在的子域名
  - 网络请求资源中的子域名
- GitHub搜索：`https://github.com/search?o=desc&q=%22feei.cn%22&s=indexed&type=Code&utf8=✓`
- DNS域传递：`dig @8.8.8.8 axfr www.feei.cn`
- crossdomain.xml：`http://www.feei.cn/crossdomain.xml`
- DNSPod：`http://www.dnspod.cn/proxy_diagnose/recordscan/feei.cn?callback=feei`

![GitHub查询子域名](./esd_03.webp)

缺点：不全面，存在遗漏

### 三方工具

- Shodan/Zoomeye/Fofa/Censys等网络资产工具
- Sublist3r/subbrute/subDomainBrute等集成爆破工具

### DNS查询

域名的存在是为了避免让大家记住IP而出现的，因此域名都是对应着IP的。所以可以通过收集常用域名字典，去DNS服务商查询是否有解析记录来枚举子域名。 比如`feei.cn`，通过`dig`命令可以看到二级域名`papers.feei.cn`的DNS解析记录（ANSWER SECTION）。

```bash
$ dig papers.feei.cn
; <<>> DiG 9.9.7-P3 <<>> papers.feei.cn
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 45877
;; flags: qr rd ra; QUERY: 1, ANSWER: 3, AUTHORITY: 0, ADDITIONAL: 1
;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 512
;; QUESTION SECTION:
;papers.feei.cn.			IN	A
;; ANSWER SECTION:
papers.feei.cn.		599	IN	CNAME	feeicn.github.io.
feeicn.github.io.	3599	IN	CNAME	sni.github.map.fastly.net.
sni.github.map.fastly.net. 29	IN	A	151.101.77.147
;; Query time: 146 msec
;; SERVER: 8.8.8.8#53(8.8.8.8)
;; WHEN: Fri Mar 02 17:28:15 CST 2018
;; MSG SIZE  rcvd: 128
```

缺点：速度，存在少量漏报

## 最佳实践

每种思路都存在漏报的可能，结合起来查询的结果才能最全面。着重说下第四种方式，通过DNS查询来枚举子域名。 通过DNS来枚举需要解决两个问题，字典和速度。

### 更全的字典

#### DNS服务商：从子域名中来，到子域名中去

DNS服务商的字典是最准确有效的，先找到一份DNSPod公布的使用最多的子域名：[dnspod-top2000-sub-domains.txt](https://github.com/DNSPod/oh-my-free-data/blob/master/src/dnspod-top2000-sub-domains.txt)

#### 通用字典

> 一些基础的组合的字典，在大小和命中做取舍。

- 单字母：`f.feei.cn`（大都喜欢短一点的域名，单字符的最为常见）
- 单字母+单数字：`s1.feei.cn`
- 双字母：`sd.feei.cn`（大都喜欢业务的首字母缩写）
- 双字母+单数字：`ss1.feei.cn`
- 双字母+双数字：`ss01.feei.cn`
- 三字母：`www.feei.cn`（部分业务名称为三个字）
- 四字母：`webp.feei.cn`（部分业务名称为四个字）
- 单数字：`1.feei.cn`
- 双数字：`11.feei.cn`
- 三数字：`666.feei.cn`

#### 常用词组

> 一些最常见中英文的词组。

- `fanyi.feei.cn`（中）`tranlate.feei.cn`（英）
- `huiyuan.feei.cn`（中）`member.feei.cn`（英）
- `tupian.feei.cn`（中）`picture.feei.cn`（英）

#### 同类爆破工具的字典

> 同类工具各自都收集整理了独有的字典，全部结合起来。

- [subbrute](https://github.com/TheRook/subbrute):[names_small.txt](https://github.com/TheRook/subbrute/blob/master/names_small.txt)
- [subDomainsBrute](https://github.com/lijiejie/subDomainsBrute):[subnames_full.txt](https://github.com/lijiejie/subDomainsBrute/blob/master/dict/subnames_full.txt)
- [dnsbrute](https://github.com/Q2h1Cg/dnsbrute):[53683.txt](https://github.com/Q2h1Cg/dnsbrute/blob/v2.0/dict/53683.txt)

### 更快的速度

使用常见的多进程、多线程及gevent等都无法发挥出最大的作用。 使用Python中的[asyncio](https://github.com/python/asyncio)+[aioDNS](https://github.com/saghul/aiodns)来获取最大速度。 一个简单的例子:

```python
import asyncio
import aiodns
loop = asyncio.get_event_loop()
resolver = aiodns.DNSResolver(loop=loop)
f = resolver.query('google.com','A')
result = loop.run_until_complete(f)
print(result)
```

可能会有疑问，大并发下会不会被DNS Server封禁？

由于DNS查询特性，当我们通过114DNS（`114.114.114.114`）查询`feei.cn`的A记录时，114DNS若没有该记录则会去查询`CN`域名根节点服务器。 同理，114DNS自身也会被其他DNS Server当做根节点服务器被大量查询，同时使用114的DNS的正常用户太多，对于速度的封禁会很容易造成误杀，至少目前并不用太担心封禁问题。

#### 实现流程

- 判断是否是泛解析域名
  - 查询一个绝对不存在域名的A记录，比如`enumsubdomain-feei.feei.cn`
  - 根据是否返回IP来判断是否是泛解析（若返回IP则是泛解析域名）
- 加载字典
  - 读取固定文本字典
  - 根据固定文本字典中的可变字典来动态生成新的字典
  - 合并去重
- 协程批量查询
  - 建立17万个任务
  - 每次最多同时跑10000个任务来避免任务加载时间过长
  - 记录每次查询结果
- 结果
  - 记录耗时
  - 写入结果文件

#### 字典生成

对于通用类型的字典，比如常见的字母和数字组合可以动态生成。

```python
# https://github.com/FeeiCN/ESD/blob/master/ESD.py
def generate_general_dicts(self, line):
    letter_count = line.count('{letter}')
    number_count = line.count('{number}')
    letters = itertools.product(string.ascii_lowercase, repeat=letter_count)
    letters = [''.join(l) for l in letters]
    numbers = itertools.product(string.digits, repeat=number_count)
    numbers = [''.join(n) for n in numbers]
    for l in letters:
        iter_line = line.replace('{letter}' * letter_count, l)
        self.general_dicts.append(iter_line)
    number_dicts = []
    for gd in self.general_dicts:
        for n in numbers:
            iter_line = gd.replace('{number}' * number_count, n)
            number_dicts.append(iter_line)
    if len(number_dicts) > 0:
        return number_dicts
    else:
        return self.general_dicts
```

```python
def test_generate_general_dict():
    start_time = time.time()
    esd = EnumSubDomain('feei.cn')
    rules = {
        '{letter}': 26,
        '{letter}{number}': 260,
        '{letter}{letter}': 676,
        '{letter}{letter}{number}': 6760,
        '{letter}{letter}{number}{number}': 67600,
        '{letter}{letter}{letter}': 17576,
        '{letter}{letter}{letter}{number}{number}': 1757600,
        '{letter}{letter}{letter}{letter}': 456976,
        '{number}': 10,
        '{number}{number}': 100,
        '{number}{number}{number}': 1000,
    }
    for k, v in rules.items():
        esd.general_dicts = []
        dicts = esd.generate_general_dicts(k)
        print(len(dicts), k)
        assert len(dicts) == v
    print(time.time() - start_time)
```

```python
# 单字母26个
26      {letter}

260     {letter}{number}
676     {letter}{letter}
6760    {letter}{letter}{number}
67600   {letter}{letter}{number}{number}

# 三字母17576个
17576   {letter}{letter}{letter}

# 需要注意的是，四字母45万个，在效率上会指数级增长，但四字母也是最常用的子域
456976  {letter}{letter}{letter}{letter}

10      {number}
100     {number}{number}
1000    {number}{number}{number}
```

#### 同时最多10000个协程并行运行

当不对并发进行限制时，会导致内存暴涨和文件描述符用完的情况。 为了能够在普通PC上保持内存稳定的运行，将限制最大并发10000个协程。

```python
# https://github.com/FeeiCN/ESD/blob/master/ESD.py
@staticmethod
def limited_concurrency_coroutines(coros, limit):
    from itertools import islice
    futures = [
        asyncio.ensure_future(c)
        for c in islice(coros, 0, limit)
    ]
    async def first_to_finish():
        while True:
            await asyncio.sleep(0)
            for f in futures:
                if f.done():
                    futures.remove(f)
                    try:
                        nf = next(coros)
                        futures.append(
                            asyncio.ensure_future(nf))
                    except StopIteration:
                        pass
                    return f.result()
    while len(futures) > 0:
        yield first_to_finish()
async def start(self, tasks):
    for res in self.limited_concurrency_coroutines(tasks, 10000):
        await res
```

通过扫描`qq.com`，共`170083`条规则，找到`1913`个域名，耗时`100-160`秒左右，平均`1000-1500`条/秒，后续再引入多进程可跑满带宽。

## 主要问题

### 域名泛解析问题

通过DNS查询枚举子域名遇到的最大问题是域名泛解析问题，域名泛解析是厂商为方便维护解析记录，将域名所有情况都解析到同样服务器上。 比如电商网站都会给店铺提供自定义域名功能，域名必然是泛解析架构。

比如在域名服务商配置`*.feei.cn`的`A记录`到`103.21.141.30`，则不管是访问`papers.feei.cn`/`f.feei.cn`/`cobra.feei.cn`都将会解析到`103.21.141.30`，再由这台服务器上的NGINX来区分域名及对应的后端应用。

所以当拿着字典爆破泛解析域名的子域名时，不论域名是否真实存在，都将会存在解析结果。

目前最好的解决方式是通过先获取一个绝对不存在域名的响应内容，再遍历获取每个字典对应的子域名的响应内容，通过和不存在域名的内容做相似度比对，来枚举子域名，但这样的实现是以牺牲速度为代价。

但这样还是存在问题，比如蘑菇街的商家是有自定义子域名功能的，他可以配置`sports.mogujie.com`为他的店铺域名，而所有店铺的响应内容是相似的。 这样就会导致虽然这些店铺域名和不存在的域名的响应内容不相似，但在最终的域名集合中有非常多的店铺自定义域名，这些域名对于漏洞扫描来说只需要有一个即可。

若再次对所有子域名进行响应相似度比对的话，又会出现新的问题，部分系统设计时，如果未登录可能跳统一登录页，会导致大量误杀。 所以暂定为ESD仅处理子域名问题，至于子域名是否解析到一个应用相同响应内容则由后续的漏洞扫描器去解决。

### DNS缓存

各家DNS缓存结果和缓存失效策略各不相同，导致在取不存在的子域名来判断泛解析会存在问题。 比如`xxx.feei.cn`在`114.114.114.114`上`A记录`解析的结果是`103.21.141.30`，在`223.5.5.5`上结果为`103.21.141.20`(没及时更新缓存)。

只能通过多次查询来强制刷新DNS缓存，无疑增加了爆破效率。

### 出口线路

有些域名为增加网站的访问速度，会针对不同线路（电信、联通、移动、教育）解析不同IP。

这会导致两个问题

1. 单网络环境下无法枚举全所有线路的IP。 当我们将`ESD`部署在电信网络下，则只能枚举到域名针对电信机房的IP，而对于联通、移动、教育网的IP造成漏报。 这个问题目前还无法通过程序本身很好的去解决，理想状态是部署在多个线路机房中。
2. 不同DNS服务商对于网络出口的线路判定不一致，导致同一域名解析的IP不一致，进而引起对于泛解析域名的大量误报。

比如阿里云DNS（`223.5.5.5`/`223.6.6.6`）和114DNS（`114.114.114.114`）在同一网络环境（电信）下同一域名(`try.tuchong.com`)解析出来的域名不一样。

解决思路是在预扫描时，在判断一个不存在域名解析记录时，对所有需要用到的DNS都做一遍解析并比对解析结果，如果各个DNS对于同域名解析结果不一致则以`114.114.114.114`的为准（`114.114.114.114`目前解析效果上来看是几个中最优秀的）。

```bash
➜  ESD git:(master) ✗ dig try.tuchong.com @223.5.5.5
; <<>> DiG 9.10.6 <<>> try.tuchong.com @223.5.5.5
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 9900
;; flags: qr rd ra; QUERY: 1, ANSWER: 16, AUTHORITY: 0, ADDITIONAL: 0
;; QUESTION SECTION:
;try.tuchong.com.		IN	A
;; ANSWER SECTION:
try.tuchong.com.	15	IN	CNAME	all.tuchong.com.w.kunlunca.com.
all.tuchong.com.w.kunlunca.com.	15 IN	A	222.186.49.171
all.tuchong.com.w.kunlunca.com.	15 IN	A	58.218.203.138
all.tuchong.com.w.kunlunca.com.	15 IN	A	58.218.203.231
all.tuchong.com.w.kunlunca.com.	15 IN	A	180.101.150.30
all.tuchong.com.w.kunlunca.com.	15 IN	A	180.101.150.32
all.tuchong.com.w.kunlunca.com.	15 IN	A	222.186.49.166
all.tuchong.com.w.kunlunca.com.	15 IN	A	58.218.203.246
all.tuchong.com.w.kunlunca.com.	15 IN	A	222.186.49.168
all.tuchong.com.w.kunlunca.com.	15 IN	A	180.101.150.26
all.tuchong.com.w.kunlunca.com.	15 IN	A	180.101.150.29
all.tuchong.com.w.kunlunca.com.	15 IN	A	180.101.150.25
all.tuchong.com.w.kunlunca.com.	15 IN	A	58.218.203.228
all.tuchong.com.w.kunlunca.com.	15 IN	A	222.186.49.226
all.tuchong.com.w.kunlunca.com.	15 IN	A	58.218.203.226
all.tuchong.com.w.kunlunca.com.	15 IN	A	222.186.49.170
;; Query time: 21 msec
;; SERVER: 223.5.5.5#53(223.5.5.5)
;; WHEN: Wed Apr 18 11:36:17 CST 2018
;; MSG SIZE  rcvd: 314
```

```bash
➜  ESD git:(master) ✗ dig try.tuchong.com @114.114.114.114
; <<>> DiG 9.10.6 <<>> try.tuchong.com @114.114.114.114
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 61809
;; flags: qr rd ra; QUERY: 1, ANSWER: 7, AUTHORITY: 0, ADDITIONAL: 1
;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;try.tuchong.com.		IN	A
;; ANSWER SECTION:
try.tuchong.com.	14344	IN	CNAME	all.tuchong.com.w.kunlunca.com.
all.tuchong.com.w.kunlunca.com.	124 IN	A	180.163.155.11
all.tuchong.com.w.kunlunca.com.	124 IN	A	180.163.155.10
all.tuchong.com.w.kunlunca.com.	124 IN	A	180.163.155.12
all.tuchong.com.w.kunlunca.com.	124 IN	A	180.163.155.9
all.tuchong.com.w.kunlunca.com.	124 IN	A	180.163.155.7
all.tuchong.com.w.kunlunca.com.	124 IN	A	180.163.155.8
;; Query time: 75 msec
;; SERVER: 114.114.114.114#53(114.114.114.114)
;; WHEN: Wed Apr 18 11:34:31 CST 2018
;; MSG SIZE  rcvd: 181
```

## 对抗思路

> 虽然子域名本身就是公开的网络资产，但作为甲方安全得思考如何针对性的增大其收集子域名的难度。

### 使用泛解析

爆破泛解析是枚举子域名的难点，而泛解析的出现就是为了方便业务快速管理子域名，既然这样那企业采用泛解析的方式利大于弊。比如在新上线子域名时不用等待域名同步时间、对于一些不存在的域名能够handle 404页面、内部对于子域名统计更加方便。

### 人机识别

使用泛解析的方式仅仅是增大了时间成本，若面对的是定向攻击，攻击人员不在乎时间成本或是通过分布式的方式，则甲方需要解决人机识别，针对机器程序的页面相似度混淆。

## 评测子域名爆破工具

> 既然遇到了这么多问题，可以以这些问题来评测一款扫描器是否足够的优秀。

效率

- 普通纯基于字典爆破条件下的速度是多少？

覆盖率

- 是否支持基于字典的子域名爆破？字典的全面性及大小？
- 是否支持泛解析域名？泛解析域名的准确性？
- 是否支持从页面响应中、请求跳转、各种其他域名解析记录中收集域名？
- 是否支持HTTPS域名透明度？
- 是否支持DNS域传送漏洞？
- 是否支持搜索引擎子域名？

以上评估不好落地？那简单的话可以通过测试下面几个域名来判断

- 正常域名`qq.com`，看看扫描的最终域名数量和耗时
- 非正常域名`fliggy.com`、`plu.cn`、`taobao.com`，看看扫描的结果中是否存在大量误报

## 写在最后

作为渗透测试中最基础前置的点，枚举子域名需要做的事情太多，也只有各个点都覆盖到才能获取一份最接近真实全量的子域名。 项目命名为[ESD](https://github.com/FeeiCN/ESD)，最终实现已开源至[GitHub](https://github.com/FeeiCN/ESD)，欢迎参与一同维护。
