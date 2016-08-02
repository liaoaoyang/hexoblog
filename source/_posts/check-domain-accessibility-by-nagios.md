title: Nagios检查域名是否可连通
date: 2016-08-01 22:59:10
tags: [Nagios]
categories: 系统
---

# 概述

日常服务器使用中，可能存在某些服务器被意外的更改路由表等问题，在多张网卡的情况下，可能会出现Nagios可以发现服务器，但是实际上服务器对外连接不可用的情况。

这必然是对服务有着致命打击的，将其纳入监控势在必行。

# 分析

大多数时候，我们只需要要关心这个服务需要访问的域名的连通性，即主机的连通性，可以考虑通过nagios定时ping。如果是web服务，还可以通过curl判断访问情况。

思路确定的话，那么编写插件即变得很容易。针对Nagios的使用上，可以考虑定义自定义命令，通过nrpe调用客户端上的插件，通过主动检查的方式进行。

通过nrpe调用命令的设置上有一个很有趣的地方，客户端定义的指令和服务端定义的指令有一个位置的偏移，因为`服务端`的调用`check_nrpe`插件第一个参数`$ARG1`必定是`客户端`上的命令名称。所以服务端的定义的`$ARG2`将会被作为到客户端的`$ARG1`使用。

例如，服务端指定命令为：

```
define command {
    command_name                    check_domain_accessibility
    command_line                    $USER1$/check_nrpe -c $ARG1$ -a $ARG2$ $ARG3$ $ARG4$
    register                        1
}
```

客户端的命令为：

```
command[check_domain_accessibilty]=/path/to/plugins/check_domain_accessibility.sh -d $ARG1$ -t $ARG2$ -p $ARG3$
```
| 服务端 | 客户端  |
| --- | --- |
| $ARG1$ | check_domain_accessibilty |
| $ARG2$ | $ARG1$ |
| $ARG3$ | $ARG2$ |
| $ARG4$ | $ARG3$ |


可以看到的对应关系为：



# 插件代码

插件代码通过shell编写，按照Nagios插件的特定，只出现OK/CRITICAL/UNKNOWN三种情况。

考虑到内网请求是主要关注的点，连通性不佳（如ping丢包）也会被归类到CRITICAL之中，用于提醒SA关注。

```
#!/bin/sh

usage()
{
	echo "Usage: `basename $0` -d DOMAIN [-t TIMEOUT_S -p y|n]"
	exit 3
}

do_curl()
{
	domain=$1
	timeout=$2

	result=`curl -I -s --connect-timeout $timeout $domain -w %{http_code} | tail -n1`

	if [ "$result""x" = "200x" ];then
		return 0
	else
		return 1
	fi
}

do_ping()
{
	domain=$1
	timeout=$2
	package=4

	if [ ! -z $3 ];then
		if [ $3 -gt 0 ];then
			package=$3
		fi
	fi

	timeout=$(($timeout*$package))

	result=`ping -t $timeout -c $package $domain | egrep '\s0*\.?0%\spacket\sloss' | wc -l`

	if [ $result -eq 1 ]; then
		return 0
	else
		return 1
	fi
}


[ $# -eq 0 ] && usage

DOMAIN=""
TIMEOUT_S=1
PING_FIRST=""

while getopts ":d:t::p::" OPTION
do
    case $OPTION in
        d)
            DOMAIN=$OPTARG
            ;;
        t)
            TIMEOUT_S=$OPTARG
            ;;
        p)
            PING_FIRST=$OPTARG
            ;;
        \?)
            usage
            ;;
    esac
done

if [ -z $TIMEOUT_S ];then
    TIMEOUT_S=1
fi

if [ $PING_FIRST"x" = "nx" ];then
    PING_FIRST="n"
else
    PING_FIRST="y"
fi

if [ -z "$DOMAIN" ];then
    echo "You must specify DOMAIN with -d option"
    exit 3
fi

if [ $PING_FIRST = "y" ];then
	$(do_ping $DOMAIN $TIMEOUT_S)
else
	$(do_curl $DOMAIN $TIMEOUT_S)
fi

check_result=$?

if [ $check_result -ne 0 ];then
	if [ $PING_FIRST = "n" ];then
		$(do_ping $DOMAIN $TIMEOUT_S)
	else
		$(do_curl $DOMAIN $TIMEOUT_S)
	fi
	check_result=$?
fi

if [ $check_result -eq 0 ];then
	echo "$DOMAIN is reachable"
	exit 0
else
	echo "$DOMAIN not reachable"
	exit 2
fi
```


