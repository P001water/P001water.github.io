---
layout: post
title: My CheatSheet
author: may1as
date: 2022-08-10
categories: [瞎折腾,CheatSheet]
tags: [瞎折腾]
pin: true


---

# 正则表达式速查卡

<img src="/img/FixclxAUUAAxJoX (1).png" alt="FixclxAUUAAxJoX (1)" style="zoom: 50%;" />


# Website collect

[User Agent String.Com](https://useragentstring.com/) —— 收集各种UA头

[Civitai | Stable Diffusion models, embeddings, hypernetworks and more](https://civitai.com/) —— ai图片模型

<img src="img/image-20230410104735171.png" alt="image-20230410104735171" style="zoom: 50%;" />

# Docker Error 

```
ph@ubuntu:~/Desktop$ docker-compose up -d
ERROR:Couldn't connect to Docker daemon at http+docker://localunixsocket - is it running?
If it's at a non-standard location,specify the URL with the DOCKER_HOST environment variable.
```

solution

```
sudo usermod -aG docker $USER
```

# Can not kill redis Process

solution

```python
/etc/init.d/redis-server stop
```

# Linux command Shortcut

```
vim ~/.bashrc
修改alias vol = "python vol.py"
source ~/.bashrc
```

不同用户之间不生效，之后在命令行中`vol`的效果等同于`python vol.py`

# Common port on network

```sql
21 ftp        ftp的端口号20、21的区别一个是数据端口，一个是控制端口，控制端口一般为21
69 TFTP       (简单文件传输协议) 
22 SSH 
23 Telnet
80 web
80-89 web
443 https   SSL心脏滴血
445 SMB     ms17-010永恒之蓝
873 Rsync未授权
1433 MSSQL
1521 Oracle    这玩应记不住？记不住？
3306 MySQL
3389 远程桌面
5432 PostgreSQL
5900 vnc   目前常用的协议有VNC/SPICE/RDP三种 、小巧，支持客户端和服务器端的直接拷贝粘贴，缺点：速度最慢
6379 redis未授权
7001,7002 WebLogic默认弱口令，反序列
8080 tomcat/WDCP主机管理系统，默认弱口令
8080,8089,9090 JBOSS
Jboss通常占用的端口是1098，1099，4444，4445，8080，8009，8083，8093这几个，
		默认端口是8080
		在windows系统中：
1098、1099、4444、4445、8083端口在/jboss/server/default/conf/jboss-service.xml中
8080端口在/jboss/server/default/deploy/jboss-web.deployer/server.xml中
8093端口在/jboss/server/default/deploy/jms/uil2-service.xml中。
8000-9090 都是一些常见的web端口
27017,27018 Mongodb未授权访问
28017 mongodb统计页面
50070,50030 hadoop默认端口未授权访问
161 SNMP
389 LDAP
512,513,514 Rexec
1025,111 NFS
2082/2083 cpanel主机管理系统登陆 （国外用较多）
2222 DA虚拟主机管理系统登陆 （国外用较多）
2601,2604 zebra路由，默认密码zebra
3128 squid代理默认端口，如果没设置口令很可能就直接漫游内网了
3312/3311 kangle主机管理系统登陆
4440 rundeck 参考WooYun: 借用新浪某服务成功漫游新浪内网
5984 CouchDB http://xxx:5984/_utils/
6082 varnish 参考WooYun: Varnish HTTP accelerator CLI 未授权访问易导致网站被直接篡改或者作为代理进入内网
7778 Kloxo主机控制面板登录
8083 Vestacp主机管理系统 （国外用较多）
8649 ganglia
8888 amh/LuManager 主机管理系统默认端口
9200,9300 elasticsearch 参考WooYun: 多玩某服务器ElasticSearch命令执行漏洞
10000 Virtualmin/Webmin 服务器虚拟主机管理系统
11211 memcache未授权访问
50000 SAP命令执行
```


# Python requests General Headers

```python
headers={
    'Accept':'text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8',
    'Cache-Control':'max-age=0',
    'Connection':'close',
    'Referer':'http://www.baidu.com/',
    'User-Agent':'Mozilla/5.0 (Windows NT 6.1; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/53.0.2785.104 Safari/537.36 Core/1.53.4882.400 QQBrowser/9.7.13059.400'
}
```

# Windows powershell 配置文件位置

> 在 Windows PowerShell 中可以有四个不同的配置文件。配置文件按加载顺序列出。较特定的配置文件优先于较不特定的配置文件（如果它们适用）。
>
> - **%windir%\system32\WindowsPowerShell\v1.0\profile.ps1**
>   此配置文件适用于所有用户和所有 shell。

> - **%windir%\system32\WindowsPowerShell\v1.0\ Microsoft.PowerShell_profile.ps1**
>   此配置文件适用于所有用户，但仅适用于 Microsoft.PowerShell shell。

> - **%UserProfile%\My Documents\WindowsPowerShell\profile.ps1**
>   此配置文件仅适用于当前用户，但会影响所有 shell。

> - **%UserProfile%\My Documents\WindowsPowerShell\Microsoft.PowerShell_profile.ps1**
>   此配置文件仅适用于当前用户和 Microsoft.PowerShell shell。


# xss get cookie 

<img src="/img/1668786596594-5d604f91-441b-46d8-b8ea-4f43f87f74ab.jpeg" alt="1668786596594-5d604f91-441b-46d8-b8ea-4f43f87f74ab" style="zoom:50%;" />



# Tcpdump 抓包和保存



```shell
tcpdump -i any port 80 -A -w dump.pcap

这是一个使用tcpdump命令的示例，用于在网络接口上捕获所有使用HTTP协议（端口号为80）的数据包，并将它们保存到名为“dump.pcap”的文件中。其中：

-i any 表示监听所有可用的网络接口。
port 80 指定只捕获目标端口为80的数据包。
-w dump.pcap 表示将捕获的数据包保存到名为“dump.pcap”的文件中，以二进制形式存储，而不是将其转换为ASCII文本。
```

