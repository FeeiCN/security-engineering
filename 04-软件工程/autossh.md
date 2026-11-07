---
slug: /autossh
title: 免密自动登录 SSH
icon: terminal-icon
---



开发过程中频繁用到SSH，而所有服务器密码都是自动生成的，每次手动输入过滤繁琐，效率上也有较大损失。

<!-- truncate -->

## 最佳实践

SHELL中通过`except`可以实现自动化和交互式的工作，因此可以通过`expect`来操作本机的SSH，并将预先内置好的密码输入进去。

**安装依赖**

```bash
yum install expect
apt-get install expect
```

```bash
#!/bin/bash

# AUTO SSH - Auto Login SSH Server (expect-based)
#
# @category Main
# @package  AutoSSH
# @author   Feei <feei@feei.cn>
# @license  Copyright (C) 2015 Feei. All Rights Reserved
# @link     http://github.com/FeeiCN/autossh
# @install
#
# $ git clone https://github.com/FeeiCN/autossh.git
# $ sudo cp autossh/autossh /usr/local/bin/
# $ 'server name|192.168.1.1|root|password|22|0' > ~/.autosshrc
#
# @usage
# $ autossh   // List Server
# $ autossh 1 // Login Num n Server

# 一些基本配置存放的路径
AUTO_SSH_CONFIG=`cat ~/.autosshrc`

# 输出有颜色的提示
BORDER_LINE="\033[1;31m############################################################ \033[0m"
echo -e $BORDER_LINE
echo -e "\033[1;31m#                     [AUTO SSH]                           # \033[0m"
echo -e "\033[1;31m#                                                          # \033[0m"
echo -e "\033[1;31m#                                                          # \033[0m"
i=0;

# 判断是否存在配置文件
if [ "$AUTO_SSH_CONFIG" == "" ]; then
	echo -e "\033[1;31m#              Config(~/.autosshrc) Not Found              # \033[0m";
	echo -e "\033[1;31m#                                                          # \033[0m"
	echo -e "\033[1;31m#                                                          # \033[0m"
	echo -e $BORDER_LINE
else
  	# 存在则输出配置文件中的机器列表
	for server in $AUTO_SSH_CONFIG; do
		i=`expr $i + 1`
		SERVER=`echo $server | awk -F\| '{ print $1 }'`
		IP=`echo $server | awk -F\| '{ print $2 }'`
		NAME=`echo $server | awk -F\| '{ print $3 }'`
		LINE="\033[1;31m#"\ [$i]\ $SERVER\ -\ $IP':'$NAME
		MAX_LINE_LENGTH=`expr ${#BORDER_LINE}`
		CURRENT_LINE_LENGTH=`expr "${#LINE}"`
		DIS_LINE_LENGTH=`expr $MAX_LINE_LENGTH - $CURRENT_LINE_LENGTH - 9`
		echo -e $LINE"\c"
		for n in $(seq $DIS_LINE_LENGTH);
		do
		    echo -e " \c"
		done
		echo -e "# \033[0m"
	done
	echo -e "\033[1;31m#                                                          # \033[0m"
	echo -e "\033[1;31m#                                                          # \033[0m"
	echo -e $BORDER_LINE

	# 获取需要登录的服务器编号
	if [ "$1" != "" ]; then
	    no=$1
	else
	    no=0
	    until [ $no -gt 0 -a $no -le $i ] 2>/dev/null
	    do
	        echo -e 'Server Number:\c'
	        read no
	    done
	fi
fi

i=0
for server in $AUTO_SSH_CONFIG; do
	i=`expr $i + 1`
	if [ $i -eq $no ] ; then
    		# 根据编号找到服务器基本信息（IP、名字、密码、端口、是否是堡垒机）
		IP=`echo $server | awk -F\| '{ print $2 }'`
		NAME=`echo $server | awk -F\| '{ print $3 }'`
		PASS=`echo $server | awk -F\| '{ print $4 }'`
		PORT=`echo $server | awk -F\| '{ print $5 }'`
		ISBASTION=`echo $server | awk -F\| '{ print $6 }'`
    		# 将这些信息输出成一个expect的临时脚本
		FILE='/tmp/.login.sh'
		if [ "$PORT" == "" ]; then
      		# 端口为空时的默认端口
			PORT=10022
		fi
		# 动态将组合的脚本内容输出到临时脚本文件中
		# expect脚本, -f从文件读取命令
        	echo '#!/usr/bin/expect -f' > $FILE
        	# 超时时间30秒
        	echo 'set timeout 30' >> $FILE
        	# 解决窗口大小变化导致的显示偏差问题
			echo 'trap {' >> $FILE
            echo '    set rows [stty rows]' >> $FILE
            echo '    set cols [stty columns]' >> $FILE
            echo '    stty rows $rows columns $cols < $spawn_out(slave,name)' >> $FILE
            echo '} WINCH' >> $FILE
            	# 调取ssh
        	echo "spawn ssh -p$PORT -l "$NAME $IP >> $FILE
        	if [ "$PASS" != "" ]; then
        		# 判断是否出现需要输入密码提示
	        	echo 'expect "password:"' >> $FILE
	        	# 发送密码
	        	echo 'send   '$PASS"\r" >> $FILE
        			# 自动切sudo
				if [ "$2" == 'sudo' ]; then
					# 以@符号来判断是否登录成功
					echo 'expect "@"' >> $FILE
					# 输入sudo su
					echo 'send   "sudo su\r"' >> $FILE
					# 判断是否出现需要输入密码提示
					echo 'expect "password for"' >> $FILE
					# 输入密码
					echo 'send   '$PASS"\r" >> $FILE
				else
					# 是否是堡垒机
					if [ "$ISBASTION" == 1 ] && [ "$2" != "" ]; then
							# 以>符号来判断登录堡垒机成功
							echo 'expect ">"' >> $FILE
							# 发现需要登录的目标服务器 IP
							echo 'send   '$2"\r" >> $FILE
							# 判断是否出现需要输入密码提示
							echo 'expect "password:"' >> $FILE
							# 输入密码
							echo 'send   '$PASS"\r" >> $FILE
							# 切换sudo
							if [ "$3" == 'sudo' ]; then
								echo 'expect "@"' >> $FILE
								echo 'send   "sudo su\r"' >> $FILE
								echo 'expect "password for"' >> $FILE
								echo 'send   '$PASS"\r" >> $FILE
							fi
					fi
				fi
			fi
          	# 退出交互控制
        	echo 'interact' >> $FILE
    		# 将配置好的文件加上可执行权限
		chmod a+x $FILE
    		# 执行该临时文件
		$FILE
    		# 清空该临时文件
		echo '' > $FILE
		break;
	fi
done
```

## 2 配置使用

**将脚本拷贝到Path目录中**

```bash
$ sudo cp autossh/autossh /usr/local/bin/
```

**配置文件**

```bash
vim ~/.autosshrc
# 服务器名称|IP|账号|密码|端口|是否是堡垒机
wufeifei|192.168.1.1|root|password|22|1
```

**查看机器列表**

```bash
$ autossh
############################################################
#                     [AUTO SSH]                           #
#                                                          #
#                                                          #
# [1] 192.168.1.110:feei                                   #
# [2] 10.11.2.103:root                                     #
# [3] 103.21.140.84:root                                   #
#                                                          #
#                                                          #
############################################################
Server Number:(Input Server Num)
```

**自动登录第一台服务器**

```bash
$ autossh 1
```

**自动登录第一台服务器并sudo**

```bash
$ autossh 1 sudo
```

**通过编号为 1 的堡垒机登录到 10.12.0.123**

```bash
$ autossh 1 10.12.0.123
```

**通过编号为 1 的堡垒机登录到 10.11.0.123 并切换到 sudo**

```bash
$ autossh 1 10.11.0.123 sudo
```

已开源在GitHub：[autossh](https://github.com/FeeiCN/autossh)。
