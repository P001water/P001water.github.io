---
layout: post
title:  "7.25 —— Hello!  Java SEC"
date: 2023-07-25
categories: 
---



>  这两天好像出鬼了，对于Java我好像突然顿悟了。以前死看不懂的Java代码突然理解了。架构，模式在脑子里都更清晰了，看代码终于是有头绪了。（可能是最近给我的国外研究生学生上课，蹭了他们JavaSEC Spring的课件、加上又学了遍Java SE）



说起来，最近都在摸.net和Java，接触下来的感觉就是这两门语言的思想和语法都十分相似。

.net是为了理解内网的一些工具学的，Java则是大势所趋（Java这一套快赶上”苹果的生态“了）

学习Java反序列化，我的建议是也得先重点看看**Java的IO流**，这块是真的麻烦，在反序列化的理解中也挺重要。并且IO流这一块也是其他很多语言没有突出的地方，很多语言把IO流封装的很好，Java是把这块儿甩给开发者了。



> 谨以此文，记录我的Java学习历程

* CC1-7 链

我看过很多遍了，这两天才tm真懂了。后面准备多看看链子了。

[CC链 1-7 分析 - 先知社区 (aliyun.com)](https://xz.aliyun.com/t/9409#toc-0)



* 反序列化流程分析

我本身正是想跟一遍下面的内容，看看怎么从`inputStream.readObject();`到类里Override重写的`readObject()`的

```
ObjectInputStream inputStream = new ObjectInputStream(new FileInputStream("filename"));
inputStream.readObject();
```

正好碰见panda师傅文章了，写的简单详实

[序列化流程分析总结 - panda \| 热爱安全的理想少年](https://www.cnpanda.net/sec/893.html)

[反序列化流程分析总结 - panda \| 热爱安全的理想少年](https://www.cnpanda.net/sec/928.html)





