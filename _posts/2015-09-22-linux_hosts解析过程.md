---
layout: post
title:  "HOSTS解析"
date:   2015-09-22
categories: linux
---

# /etc/hosts 解析过程
### 遇到的问题
今天做NTP同步（地址为域名）时发现同步特别慢，大约有4-5秒的卡顿延时。
`/usr/sbin/ntpdate ntp.xxx.com`
### 分析
用strace追踪后发现`/etc/hosts`文件里这个域名指定了多个IP，并且向每个IP都发起了ntp请求。

```
open("/etc/host.conf", O_RDONLY)        = 3
open("/etc/hosts", O_RDONLY|O_CLOEXEC)  = 3
open("/etc/gai.conf", O_RDONLY)         = 3
connect(3, {sa_family=AF_INET, sin_port=htons(123), sin_addr=inet_addr("172.16.18.73")}, 16) = 0
connect(3, {sa_family=AF_UNSPEC, sa_data="\0\0\0\0\0\0\0\0\0\0\0\0\0\0"}, 16) = 0
connect(3, {sa_family=AF_INET, sin_port=htons(123), sin_addr=inet_addr("192.168.6.5")}, 16) = 0
connect(3, {sa_family=AF_UNSPEC, sa_data="\0\0\0\0\0\0\0\0\0\0\0\0\0\0"}, 16) = 0
connect(3, {sa_family=AF_INET, sin_port=htons(123), sin_addr=inet_addr("8.8.8.8")}, 16) = 0
connect(3, {sa_family=AF_INET, sin_port=htons(123), sin_addr=inet_addr("172.16.181.73")}, 16) = -1 EINPROGRESS (Operation now in progress)
```
Google后发现`/etc/host.conf`下有个配置参数` multi `，Centos 6.5系统中默认开启了此项功能。

>multi
>Valid values are on and off.  If set to on, the resolv+ library will return all valid addresses for a host  that  appears  in  the  /etc/hosts  file,instead of only the first.  This is off by default, as it may cause a substantial performance loss at sites with large hosts files.

### 解决办法有两种：
1. 关闭`/etc/host.conf`中的**multi**配置`multi off`
2. 注释掉`/etc/hosts`中无用的IP地址

```
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
#192.168.6.5 ntp.xxx.com
172.16.18.73 ntp.xxx.com
#8.8.8.8 ntp.xxx.com

```
### 总结
`/etc/hosts`支持一个域名对应多个IP的解析，可以利用此功能提高服务高可用。当`multi`功能开启时*gethostbyname()*返回此域名对应的所有IP地址池，否则只返回对应的第一个IP地址。需要注意的是这个功能在 Centos 5.4默认是关闭的，而在CentOS 6.5上默认是开启的。
