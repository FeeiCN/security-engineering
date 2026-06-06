---
slug: /api-security
title: API安全
icon: plug-connected-icon
---

应用间HTTP接口互相调用非常常见，期间会存在哪些安全风险？如何保证API接口调用时的安全性，防止数据窃取、未授权访问、请求被篡改以及请求被重放等风险？

<!-- truncate -->

## API安全风险与解决方案

### 传输窃取 -> HTTPS

接口需要通过HTTPS防止请求在传输过程中被窃取。

- 让HTTP无法直接访问，会强制跳转到HTTPS。
- 除了公网，办公网内嗅探窃取数据更为容易，也应该全站HTTPS。

### 未授权访问 -> API密钥（Access/Secret Key）

API接口没有做任何安全措施的话，能够被任何人调用到存在未授权访问风险。因此需要给每一个调用此API的应用在发起请求时带上一个事先约定好不容易被猜到的密钥（Secret Key），如果是多个调用方，还需要是谁调用的，因此还需要约定一个调用方的**唯一标示** （Access Key或App ID）。

如果将AK/SK每次都带在请求中，容易被窃取或泄漏，因此调用方和被调用方约定好一套算法，调用方使用SK作为密钥按照约定算法计算出一个值并和AK一同发送至被调用方，被调用方根据AK找到对应的SK，也拿着SK和约定的算法计算一个值，比对这两个值是否一致。

### 请求篡改 -> 请求签名（Request Sign Token）

当请求被人截获时数据能被篡改造成请求篡改风险，因此调用方在发起请求时需要对请求所有字段进行签名，在被调用方进行验签操作。

签名过程中如果实现不正确，会导致各种绕过的可能。

以下为一些历史真实案例，通过一些手法实现数据篡改，引发各种严重漏洞。

##### 签名绕过一、签名时直接将参数值进行字符串拼接

当`order=123&status=0&sign=xxx`，签名（sign）值为`hash(order.value + status.value)`，如何实现`status`为TRUE（>=0）？

这种就是典型的签名实现错误，签名所拼接的字符串并没有进行字段区分，导致最终拼接的参数值只要为`1230`均能通过验证。

因此，可以通过`order=12&status=30&sign=xxx`来篡改参数值，实现通过签名验证。

修复方式：在进行hash时，每一个参数值使用分隔符进行连接。同时确保该分隔符仅在后端处理，不透出到前端。上面签名可改为`hash(order.value + 'fengefu*&%' + status.value)`。

##### 签名绕过二、签名时将用户可控参数拼接为JSON/XML字符串

`{"title:" + param.title.value + ", "amount":100}`，假设参数title外部用户可控，参数amount为固定值，如何实现将amount改为0.01？

当`param.title.value`=`1,"amount":0.01}//`时，即可将amount篡改为0.01。

通过可控参数，实现构造之后参数名和值，并闭合JSON字符串。通过`//`注释原有参数值，避免该JSON串后续解析时出错。

修复方式：禁止手动拼接JSON/XML等格式字符串。使用数组等对象转成JSON/XML字符串。

##### 签名绕过三、空数据签名并返回URL

假设`/filename?sign=xxx`中的`filename`用户可控（传参不可为空），填入正确的`filename`时会生成正确的`sign`即可访问下载文件。此外，可通过`/?sign=yyy`查看根目录文件列表。如何实现控制`filename`实现查看根目录文件列表？

`filename`是在uri的path中，只需要让签名的时候path为空，同时filename又有值，且不影响后续的链接访问。可以通过`filename`=`?a=b`方式实现查看根目录文件。

各大OSS厂商曾经都在这个点上出现过问题，导致可以枚举任意文件。

### 请求重放 -> 加随机数的请求签名（Request Sign Token with Timestamp）

虽然不能改变请求参数中的数据了，但整个请求还是可以重新再发起一遍的，假如请求涉及到转账之类的操作，会存在请求重放风险。

因此需要在请求中增加随机数，一个请求只能在一定时间内使用。但这样会导致需要维护一个随机数存活缓存，成本较大。

请求时可以将时间戳带上，服务端验证时和当前时间戳对比，如果超过一定时间（比如十分钟）则请求失效。

## API安全实现细节

- 应用A需要通过HTTPS调用应用B中一个保存信息的接口
- 应用A给调用方应用B分配了一个Access Key（123456789）和Secret Key（34b9a295d037d47eec3952e9dcdb6b2b）

### 调用方

原始的请求只有a、k1、k2三个参数。

```
GET /api/save-information?k1=233&k2=.cn&a=feei
```

为了防止重放攻击，我们需要在每个请求中增加timestamp，同时为了识别调用方需要增加Access Key或App ID，因此此刻的请求参数为。

```
GET /api/save-information?k1=233&k2=.cn&a=feei&timestamp=852048000&appid=123456789
```

先将需要提交的参数按照Key Name进行排序，此处排序是为了避免不同顺序提交时导致签名和验签结果不一致。

```
# 排序前顺序
{
    'k2': 233,
    'k1': '.cn',
    'a': 'feei',
    'timestamp': 852048000,
    'appid': 123456789
}

# 排序后顺序
{
    'a': 'feei',
    'appid': 123456789,
    'timestamp': 852048000,
    'k1': '.cn',
    'k2': 233,
}
```

对排序后的值进行拼接，拼接过程中可以增加特定分隔符来增加密钥泄漏时被人猜到算法从而伪造签名的风险。

```
'feei' + '#%$' + '123456789' + '#%$' + '852048000' + '#%$' + '.cn' + '#%$' + '233'
```

拼接完对数据进行Hash一下得到预签名（pre-sign）

```
pre_sign = sha1('feei#%$123456789#%$852048000#%$.cn#%$233')
```

拿到预签名值后，我们需要让预签名值和SK再Hash一次得到最终的签名（last-sign）。

```
last_sign = sha1(pre_sign + secret_key)
```

最终发出去的请求是

```
GET /api/save-information?k1=233&k2=.cn&a=feei&timestamp=852048000&appid=123456789&sign=LAST_SIGN
```

### 被调用方

- 判断AppID是否存在，并取到对应Secret Key
- 获得当前系统的timestamp和参数中timestamp对比，时间差超过10分钟则抛异常提示
- 对参数按照Key Name进行排序，拿到排序后的Value和分隔符进行拼接
- 对拼接的结果进行Hash得到Pre Sign
- 拿着Pre Sign和Secret Key进行Hash，和参数中的sign比对是否一致，一致则完成验签

## API安全代码参考

**Python签名实现**

```
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
    'timestamp': 852048000,
    'appid': 123456789
}
sign = generate_sign(params)
assert sign == 'e937d3da60b5dfe7189aea9aeb5fceec2cfc02ab'
```

**Java签名实现**

```
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

## API安全实践

- 研发实践
  - 将整个请求签名过程包装在请求基类中，上层业务无需感知
- 进阶安全
  - 针对每个调用方进行频率限制和监控
  - 对所有签名失败的记录进行监控
- 密钥安全
  - 不要将API密钥写在代码中（包括硬编码代码或配置文件），密钥可以放在环境变量、配置管理中心或KMS中，能减少API密钥和代码一起泄漏的风险。
  - 如果一定要将API密钥写在配置文件中，不要将配置文件放在版本管理系统中（Git/SVN），很容易在某一天将代码托管在公共源代码管理平台（比如GitHub）中时导致密钥泄漏。
  - 删除不需要的API密钥，降低为使用的密钥泄漏导致的风险。
  - 设定定期更新API密钥机制，定期的更换密钥，开启新密钥替换后老密钥在一定时间内失效。
