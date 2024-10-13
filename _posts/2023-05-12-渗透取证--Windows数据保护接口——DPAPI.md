---
layout: post
title: 渗透取证 -- Windows数据保护接口——DPAPI
date: 2023-05-12
categories: [取证, windows DPAPI]
---

# 文前漫谈

>  DPAPI

```
Data Protection Application Programming Interface
数据保护应用程序编程接口
```

​	这是windows贴心的为开发者提供的针对用户身份加密的方案——DPAPI。win程序开发者可以方便的使用该API加密保护敏感数据，这种基于用户身份的加密使得该加密数据仅可被对数据进行加密的用户解密，它是Windows中保护敏感数据的首选API之一，被广泛应用于各种应用程序数据保护部分。

DPAPI提供了两种方法：基于用户和基于机器的加密。在基于用户的加密中，系统会根据用户的凭据生成一个唯一的主密钥，并将其存储在用户的配置文件中。此主密钥用于加密和解密用户的私密数据。在基于机器的加密中，系统会根据计算机的凭据生成一个唯一的主密钥，并将其存储在本地计算机上。此主密钥用于加密和解密系统级别的数据。

DPAPI针对用户身份加密的方案可以抽象理解为，不同用户拥有不同的`DPAPI master key`，一个用户也可能有不同的master key，这个`DPAPI master key` 相当于一个密钥，这个调用`DPAPI master key实施加密的过程由系统直接操作，相当于Windows给开发者封装了个加密算法，直接调用就能实现的敏感数据的加密解密保护



**我们渗透场景中的哪些应用采用了DBAPI** 

*  Internet Explorer, Google Chrome浏览器保存的登录凭据


*   Outlook, Windows Mai 等的邮箱账户密码
*   内部FTP管理器帐户密码
*   共享文件夹和资源访问密码
*   无线网络帐户密钥和密码
*   RDP远程桌面连接密码







# DPAPI中的重要概念

## Function CryptProtectData and CryptUnprotectData

分别是DPAPI的加密和解密函数，详细见文档 [CryptProtectData function (dpapi.h) - Win32 apps \| Microsoft Learn](https://learn.microsoft.com/en-us/windows/win32/api/dpapi/nf-dpapi-cryptprotectdata)

Win32 API中的函数，均在Dpapi.h头文件中







## DPAPI blob

在计算机术语中，"blob" 是 "Binary Large Object" 的缩写。它是指一种数据类型或格式，用于存储二进制数据。

在DPAPI中，"DPAPI blob" 是指使用DPAPI进行加密保护的二进制数据块。但是它实际是个结构体，其中包含了我们的密文

<img src="/img/dpapi3.png" alt="未记录的 DPAPI blob 结构"  />



## Master Key：	

DPAPI的核心

64字节，用于解密DPAPI blob，使用用户登录密码、SID和16字节随机数加密后保存在Master Key file中

Example：

```
19c05880b67d50f8231cd8009836e3cdc55610e4877f8b976abd5ca15600d0e759934324c6204b56f02527039e7fc52a1dfb5296d3381aaa7c3eb610dffa32fa

长度是128
```







## Master Key file：

二进制文件，可使用用户登录密码对其解密，获得Master Key。其所在目录受保护，即使你在资源管理器中开启了显示隐藏文件也看不了，或者cmd的dir命令，也无法看见。不过可以用powershell看，下面介绍。



分两种：

- 用户Master Key file，隐藏属性不可解，位于

> %APPDATA%\Microsoft\Protect\{SID}		//SID是当前用户的安全标识符

powershell -Force可以查看（-hidden也能看，不过winows低版本不支持-hidden的参数）

```
$path=$env:APPDATA+"\Microsoft\Protect\"+(((iex "whoami /user")|Out-String).Split(' '))[-1];Get-ChildItem -Force $path.trim()
```

<img src="/img/image-20230618192232242.png" alt="image-20230618192232242"  />



- 系统Master Key file，隐藏属性不可见，位于

> %WINDIR%\System32\Microsoft\Protect\S-1-5-18\User

powershell查看

```
Get-ChildItem -Force $env:windir\System32\Microsoft\Protect\S-1-5-18\User		
```

<img src="/img/image-20230618174621882.png" alt="image-20230618174621882"  />





## Preferred文件：

总是位于Master Key file的同级目录，显示当前系统正在使用的MasterKey及其过期时间，默认90天有效期







# 如何获得Master Key

## mimikatz从lsass.exe进程dump

```
mimikatz.exe privilege::debug sekurlsa::dpapi > dbapi.txt
```

<img src="/img/image-20230618181455536.png" alt="image-20230618181455536" style="zoom:80%;" />



另一种就是离线dump，也差不多就不写了，或许是好处是不需要在对面的机子上用mimikatz？





# C# 实现DPAPI加密和解密

从代码实现来说，我们作为开发者写工具的时候，肯定是对上面所说的master key是无感的。这意味着我们开发工具（比如解密浏览器凭据）的时候，是直接调用Windows api里的两个函数的，举个例子

```c#
using System;
using System.Security.Cryptography;
using System.Text;

namespace TestDemo
{
    class Program
    {
        static void Main(string[] args)
        {
            String en = Encrypt("pool");
            Console.WriteLine("Data encrypt by dpapi：\n" + en);
            String de = Decrypt(en);
            Console.WriteLine("Data decrypt by dpapi：\n" + de);

        }

        private static String Encrypt(string text)
        {
            // first, convert the text to byte array 
            byte[] originalText = Encoding.Unicode.GetBytes(text);

            // then use Protect() to encrypt your data 
            byte[] encryptedText = ProtectedData.Protect(originalText, null, DataProtectionScope.CurrentUser);

            return Convert.ToBase64String(encryptedText);
        }

        private static String Decrypt(string text)
        {
            // the encrypted text, converted to byte array 
            byte[] encryptedText = Convert.FromBase64String(text);

            // calling Unprotect() that returns the original text 
            byte[] originalText = ProtectedData.Unprotect(encryptedText, null, DataProtectionScope.CurrentUser);

            return Encoding.Unicode.GetString(originalText);
        }

    }
}

```

调用输出

<img src="/img/image-20230627170101035.png" alt="image-20230627170101035" style="zoom:67%;" />







# 后话

这是我当时遇见的一些问题

1. 我们可以使用用户登录密码对Master Key file解密获得Master Key。那么问题来了，只有NTLM Hash的情况下，能否对Master Key file文件解密得到Master Key

> 分情况，如果NTLM Hash能成功解密，拿到用户明文密码，就能通过解密Master Key file文件得到Master Key;
>
> 如果解密不了，就无法得到Master Key（但是如果能得到NTLM Hash，Master Key大概率也是能抓到的）。
>
> 为什么？因为最初的DPAPI设计中，解密Master key需要用户密码的 NTLM Hash，导致直接能够通过拿到SAM数据库中用户密码哈希，直接解密得到Master Key，进一步解密任何加密的DPAPI data blob，这个问题在DPAPI的第二版就被修复了，现在master key使用 SHA1 哈希进行加密，MS特意回避了能够通过NTLM Hash解密获得master key的情况。











