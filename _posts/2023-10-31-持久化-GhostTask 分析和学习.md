---
layout: post
title:  "持久化-GhostTask-注册表创建计划任务-分析和学习"
date: 2023-10-22
categories: 持久化
---



# 文前漫谈	

最近一周刚放出来的一个持久化工具 [netero1010/GhostTask (github.com)](https://github.com/netero1010/GhostTask)。利用计划任务实现持久化。

号称能够对抗取证调查、利用Silver Ticket创建计划任务横向。

win32编程本身也有对 Task Scheduler的支持 [C/C++ Code Example Creating a Task Using NewWorkItem](https://learn.microsoft.com/en-us/windows/win32/taskschd/c-c-code-example-creating-a-task-using-newworkitem)。之前也写过一个调用api的创建计划任务的持久化工具，痛点就是一查就查出来了 ：)

不同的是GhostTask单纯从注册表的角度实现了计划任务的创建，还能够一定程度上实现计划任务的隐蔽。本文对其进行分析学习



# Usage Show

首先，需要 "**NT AUTHORITY/SYSTEM**" 权限（），command

```
GhostTask.exe localhost add demo "cmd.exe" "/c notepad.exe" LAB\Administrator weekly 14:12 monday,thursday
```

<img src="/img/image-20231101000141923.png" alt="image-20231101000141923" style="zoom:67%;" />

创建的计划任务名为demo，在事件查看器确实没有发现

<img src="/img/image-20231101000406482.png" alt="image-20231101000406482" style="zoom:50%;" />

回看注册表，依然能观察到创建的 demo

<img src="/img/image-20231101000629285.png" alt="image-20231101000629285" style="zoom:50%;" />



<img src="/img/image-20231101000652510.png" alt="image-20231101000652510" style="zoom:50%;" />

注册表读写

<img src="/img/image-20231101110726512.png" alt="image-20231101110726512" style="zoom: 50%;" />







# 代码分析

main函数逻辑很清晰，关键的两个函数 AddScheduleTask() 和 DeleteScheduleTask()

<img src="/img/image-20231101115749587.png" alt="image-20231101115749587" style="zoom:50%;" />





# Func AddScheduleTask()

我比较感兴趣的就是他如何实现有效隐藏计划任务，关键代码

<img src="/img/image-20231101123747790.png" alt="image-20231101123747790" style="zoom:50%;" />

原理是22 年的 Microsoft的一篇威胁报告

[Tarrask 恶意软件使用计划任务进行防御规避](https://www.microsoft.com/en-us/security/blog/2022/04/12/tarrask-malware-uses-scheduled-tasks-for-defense-evasion/)

<img src="/img/image-20231101124004490.png" alt="image-20231101124004490" style="zoom: 50%;" />

修改删除 SD 键值的value，计划任务并不会被 `schtasks /query` 命令或者 `任务计划程序` 显示。但是修改这个键值需要 System 用户的权限。

MicroSoft 还提出个建议，完全删除 Tree和Tasks 两个注册表项以及 `C:\\Windows\System32\Tasks` 中创建的 XML 文件，任务将继续根据定义的触发器运行，还能做到无文件持久化，直到系统重新启动或者任务关联的svchost.exe进程终止

<img src="/img/image-20231101141417823.png" alt="image-20231101141417823" style="zoom:80%;" />

还顺带嘲讽了一波。完



贴图后用

<img src="/img/0mQKUdDGJrdCldJbV.png" alt="img" style="zoom:67%;" />





# Refer

[Tarrask 恶意软件使用计划任务进行防御规避](https://www.microsoft.com/en-us/security/blog/2022/04/12/tarrask-malware-uses-scheduled-tasks-for-defense-evasion/)

