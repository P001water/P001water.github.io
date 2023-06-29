---
layout: post
title: CTF---Pythonweb SSTI的payload构造思路研究(基础篇)
author: may1as
date: 2022-07-24
categories: [PythonSEC, SSTI]
tags: [PythonSSTI]
---

> 在freebuf上发过，这里的是精简入门版

<a name="Q472p"></a>
## 文前漫谈
是一篇基础篇嘛，给新同学入门用，从payload的角度看看SSTI
<a name="kIZVB"></a>

## 从原点出发
你会发现，大多数payload总是从下面这段代码出发，延伸出各种各样的花样。而这一步的目的是得到当前模块中所有可以利用的类。得到的结果跟当前模块（py文件）import的库也是有关系的。
```python
''.__class__.__base__.__subclasses__()
```
详细解释：

- `''`，是一个字符串对象，`type('')的结果是<class 'str'>`，在python中，一切皆对象，再比如`type([])的结果是<class 'list'>`，因为`[]`是数组的标志
- `__class__`，是这个对象所属的类，到这一步，`''.__class__`的结果仍然是`<class 'str'>`

`''.__class__`和`''`，区别在于前者是个类，后者是个对象

- `__base__`，是获得指定类的基类（父类），这里`''.__class__.__base__`的结果是`<class 'object'>`，因为在python3中，所有类都默认继承了Object类。
- `__subclasses__()` ，`__subclasses__()`是`Object`类的静态方法，获得指定类的所有子类，返回一个列表。因为是个方法，所以后面的括号不能忘了带。在这个列表里，拿到了所有继承Object类的子类，而在python3里，几乎所有的内建类都继承了Object类，所以在下一步的构造中几乎所有的内置类我们都能找到并利用。

![image.png](https://cdn.nlark.com/yuque/0/2022/png/22999319/1658027320103-c47a63d2-1126-4788-8dcb-e350d26f90e1.png#clientId=ufec6c6e4-c976-4&crop=0&crop=0&crop=1&crop=1&errorMessage=unknown%20error&from=paste&height=336&id=uf63bd69d&margin=%5Bobject%20Object%5D&name=image.png&originHeight=874&originWidth=1844&originalType=binary&ratio=1&rotation=0&showTitle=false&size=1318807&status=error&style=none&taskId=ufa0ff5ba-896e-4d9f-93e2-c4379f4a6b4&title=&width=708)

<a name="X4tvK"></a>
## 拿一个Payload研究
尝试直接在代码中（最好在flask环境下）<br />`print("".__class__.__base__.__subclasses__()[128].__init__.__globals__['popen']('whoami').read())`。<br />是不是成功执行`"whoami"`了。也有可能你那报错了。比如，这是因为你从`''.__class__.__base__.__subclasses__()`这一步拿到的子类列表，索引为`128`的类对应的不是`os._wrap_close类`，因为从这一步我们没有进入**os模块**所在的地址空间，就不能通过`__globals__`得到**os模块**已经加载的可利用类<br />![](https://cdn.nlark.com/yuque/0/2022/jpeg/22999319/1664075389705-4ff2fbe1-5760-4046-b68c-a83c1c2ba0d3.jpeg#clientId=ucdbc9d90-ae7f-4&crop=0&crop=0&crop=1&crop=1&errorMessage=unknown%20error&from=paste&id=u0f51aebb&margin=%5Bobject%20Object%5D&originHeight=142&originWidth=690&originalType=url&ratio=1&rotation=0&showTitle=false&status=error&style=none&taskId=u6d61a53f-0238-4251-8739-4f4b157f134&title=)
> 补充一下` __globals__`
> Python 中的每一个函数都拥有一个 __globals__ 属性，储存着当前模块全局可读的量。以一个字典的形式，建立了一种映射的关系。它必须由一个函数方法调用，即不论是test.__globals__，还是a.__globals__。只要是在一个模块（同一个py文件）内结果都是一样，因为打印的都是全局可读的量。
> ![](https://cdn.nlark.com/yuque/0/2022/jpeg/22999319/1664075408196-37266b0f-9e3f-4bd3-a2fc-c25ade391b60.jpeg#clientId=ucdbc9d90-ae7f-4&crop=0&crop=0&crop=1&crop=1&errorMessage=unknown%20error&from=paste&id=u3db1dc59&margin=%5Bobject%20Object%5D&originHeight=179&originWidth=690&originalType=url&ratio=1&rotation=0&showTitle=false&status=error&style=none&taskId=u052c8644-01de-4304-90d0-274b4dab973&title=)


```python
#payload = "".__class__.__base__.__subclasses__()[128].__init__.__globals__['popen']('whoami').read()

a= ""
b= "".__class__
c= "".__class__.__base__
d= "".__class__.__base__.__subclasses__()
e= "".__class__.__base__.__subclasses__()[128]
f= "".__class__.__base__.__subclasses__()[128].__init__
g= "".__class__.__base__.__subclasses__()[128].__init__.__globals__
h= "".__class__.__base__.__subclasses__()[128].__init__.__globals__['popen']('whoami')
i= "".__class__.__base__.__subclasses__()[128].__init__.__globals__['popen']('whoami').read()

print(a,b,c,d,e,f,g,h,i)
```
![](https://cdn.nlark.com/yuque/0/2022/jpeg/22999319/1664075444684-877e41fc-c24f-46f9-8fe4-e7d04d2646d8.jpeg#clientId=ucdbc9d90-ae7f-4&crop=0&crop=0&crop=1&crop=1&errorMessage=unknown%20error&from=paste&id=udb9f8819&margin=%5Bobject%20Object%5D&originHeight=142&originWidth=690&originalType=url&ratio=1&rotation=0&showTitle=false&status=error&style=none&taskId=ue783536e-842a-4528-89d7-200216a78a9&title=)<br />按照上面的字母开始介绍吧。a到d在前面已经介绍过。这里从e开始。

- `e="".__class__.__base__.__subclasses__()[128]`

`e="".__class__.__base__.__subclasses__()[128]`的结果是`<class 'os._wrap_close'>`，所以你应当了解，使用这个payload的时候，这个128是特殊的，它得跟随着**os._wrap_close类**的位置变化（从零开始数，第129个是os._wrap_close类），这个payload也正是利用的os._wrap_close这个类。到这一步，`"".__class__.__base__.__subclasses__()[128]`其实就相当于os._wrap_close这个类了。

- `f="".__class__.__base__.__subclasses__()[128].__init__`
- `g= "".__class__.__base__.__subclasses__()[128].__init__.__globals__`

这一段的依据主要是前面的`__globals__`属性了，为什么要先调用`__init__`，因为`__globals__`必须依靠方法才能调用，打印全局可读的量。从上面的截图看，这一步就得到os模块全局可读的量了。注意是os模块，不是其他模块。从下面截图里的一些细节不能看出，比如`'__name__': 'os'`，可以看见所处模块的名称。<br />![](https://cdn.nlark.com/yuque/0/2022/jpeg/22999319/1664075486011-ff664dff-013c-48b5-a7a0-358ec60b006f.jpeg#clientId=ucdbc9d90-ae7f-4&crop=0&crop=0&crop=1&crop=1&errorMessage=unknown%20error&from=paste&id=u8b3c76fe&margin=%5Bobject%20Object%5D&originHeight=338&originWidth=690&originalType=url&ratio=1&rotation=0&showTitle=false&status=error&style=none&taskId=uf144781c-bc03-431d-82f3-185a32445e7&title=)<br />分析一下这一步的结果。这一步得到是一个字典，像下面这样
```python
'_unsetenv': <function <lambda> at 0x0000021A10612438>, 'getenv': <function getenv at 0x0000021A106129D8>
```
大概就是一种映射关系，函数名与所在地址空间的映射，你可以尝试用keys()，打印键试试<br />![](https://cdn.nlark.com/yuque/0/2022/jpeg/22999319/1664075485941-3a546d96-40cc-4fb5-8f0d-1c616c1128bb.jpeg#clientId=ucdbc9d90-ae7f-4&crop=0&crop=0&crop=1&crop=1&errorMessage=unknown%20error&from=paste&id=u25cde21a&margin=%5Bobject%20Object%5D&originHeight=165&originWidth=690&originalType=url&ratio=1&rotation=0&showTitle=false&status=error&style=none&taskId=u662b2758-5376-4539-bc55-8f6ec532b4b&title=)<br />你看，可以利用的函数是太多了。这里我们利用的是`'popen()'`这个函数，但是只要是在上面的函数，都可以利用，`system()`，甚至一些**os库**的常见函数，都能进行调用。<br />举个例子，调用上面框出来的os这个库的system()函数，甚至更加简单。一些读，写的操作都可以借助os这个库实现。<br />![](https://cdn.nlark.com/yuque/0/2022/jpeg/22999319/1664075486026-4efd9fd5-1539-432c-855c-d88004393e12.jpeg#clientId=ucdbc9d90-ae7f-4&crop=0&crop=0&crop=1&crop=1&errorMessage=unknown%20error&from=paste&id=ud881e957&margin=%5Bobject%20Object%5D&originHeight=125&originWidth=690&originalType=url&ratio=1&rotation=0&showTitle=false&status=error&style=none&taskId=u93ae0b24-0056-451f-a854-447b0cddb0e&title=)
<a name="B846K"></a>
## 文末补充
前面说过`''.__class__.__base__.__subclasses__()`得到的继承Object类的一个列表，数量与`import`的库有关。再谈谈吧<br />![image.png](https://cdn.nlark.com/yuque/0/2022/png/22999319/1658063148544-66e6e00b-f77f-4ede-ab78-9f63ed9428e1.png#clientId=ua06a8262-0111-4&crop=0&crop=0&crop=1&crop=1&errorMessage=unknown%20error&from=paste&height=262&id=u9d1053e5&margin=%5Bobject%20Object%5D&name=image.png&originHeight=465&originWidth=990&originalType=binary&ratio=1&rotation=0&showTitle=false&size=123422&status=error&style=none&taskId=u16caa168-be95-4275-bdd2-ceb29a921a5&title=&width=557)<br />不导包的时候，或者导`python`内置模块的时候。它的数量不会变化，导入一个外置包的时候，继承Object类的子类就变多了。好在这里不会影响到我们内置类的顺序，拿这个payload来说<br />`"".__class__.__bases__[0].__subclasses__()[128].__init__.__globals__['popen']('whoami').read()`，如果第129（从零开始）个不是`os._wrap_close类`，那接下来的使用就会发生错误。你可以替换成`129`输出试试。<br />但是在有些题目环境中，重启环境可能会影响拿到类的顺序的。这也是你直接使用别人payload打不通的原因<br />![image.png](https://cdn.nlark.com/yuque/0/2022/png/22999319/1658063235095-d10da392-6cfa-4080-8545-70cc16fa8e77.png#clientId=ua06a8262-0111-4&crop=0&crop=0&crop=1&crop=1&errorMessage=unknown%20error&from=paste&height=251&id=ue9e27e0a&margin=%5Bobject%20Object%5D&name=image.png&originHeight=458&originWidth=1074&originalType=binary&ratio=1&rotation=0&showTitle=false&size=132807&status=error&style=none&taskId=udefa99bb-e680-4e4e-bf58-e629007d58c&title=&width=588)



