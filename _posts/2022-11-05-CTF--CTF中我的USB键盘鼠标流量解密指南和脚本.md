---
layout: post
title: CTF--CTF中我的USB键盘鼠标流量解密指南和脚本
author: P001water
date: 2022-11-05
categories: [流量分析, USB]
tags: [流量分析]
---

> 文章首发于[freebuf](https://www.freebuf.com/sectool/347971.html)

# 文前漫谈

<img src="/img/image-20230618192917421.png" alt="image-20230618192917421"  />



对ctf中的USB做一个了断吧，先摆上一些概念性的东西

USB是 `UniversalSerial Bus`（通用串行总线）的缩写，是一个外部总线标准，用于规范电脑与外部设备的连接和通讯，例如键盘、鼠标、打印机、磁盘或网络适配器等等。通过对该接口流量的监听，我们可以得到取键盘敲击键、鼠标移动与点击、存储设备的铭文传输通信、USB无线网卡网络传输内容等等。

# USB的几种用途
现在USB协议版本有`USB2.0, USB3.1,USB3.2, USB4`等，目前`USB2.0/3.0`比较常用，你在淘宝上买U盘的时候可能看见过很多`“USB 3.0”`。
闲谈哈，你可能听过`USB3.1`的概念，是某苹果为了装逼提出来的，跟`USB3.0`差别不大，但是人家是老大哥啊，所有慢慢都向大哥看齐，都流行说`USB3.1`

1. USB UART
> 异步串行通信口(UART)就是我们在嵌入式中常说的串口，它还是一种通用的数据通信协议。这种方式下，设备只是简单的将 USB 用于接受和发射数据，除此之外就再没有其他通讯功能了。

2. USB HID（Human Interface Device）
> HID（Human Interface Device，人机接口设备）是USB设备中常用的设备类型，是直接与人交互的USB设备，例如键盘、鼠标与游戏杆等。在USB设备中，HID设备的成本较低。另外，HID设备并不一定要有人机交互功能，只要符合HID类别规范的设备都是HID设备。

3. USB Memory
> USB Memory是数据存储
> 每一个 USB 设备（尤其是 HID 或者 Memory ）都有一个供应商 ID（Vendor ID） 和产品识别码（Product Id） 。 Vendor ID 是用来标记哪个厂商生产了这个 USB 设备。 Product ID 则用来标记不同的产品。


## lsusb命令
lsusb命令用于显示本机的USB设备列表，以及USB设备的详细信息。

<img src="/img/1666541348_63556724d69caa532b1d4.jpeg" alt="img"  />

> Bus 002：指明设备连接到哪条总线
> Device 002：表明这是连接到总线上的第二台设备
> ID：设备的ID
> VMware, Inc. Virtual Mouse：生产商名字和设备名


# Tshark常用参数
重点看下面的内容，这里就放几个处理流量包时常用的
```python
-r:设置tshark分析的输入文件
-T:设置解码结果输出的格式，包括fileds,text,ps,psml和pdml，默认为text
-e:导出协议字段
-Y:对应过滤器的过滤条件
```

# 键盘流量
以往接触到的，USB协议数据部分在`Leftover Capture Data`域中，数据长度为`8`个字节。
击键信息集中在第`3`个字节，每次击键都会产生一个数据包。所以如果看到给出的数据包中的信息都是` 8 `个字节，并且只有第 3 个字节不为 `0000`，那么几乎可以肯定是一个键盘流量了。

<img src="/img/1666541430_63556776eeea926f04c5d.jpeg" alt="img"  />



现在在比赛中常见的（2021国赛初赛有个例题)，是这种，`Leftover Capture Data`——>`HID Data`
这个改变需要注意什么呢，需要注意**tshark**导出数据时的过滤器字段，看这个对应的过滤器字段，去wireshark的英文文档里找。

```
tshark -r 数据包文件 -T fields -e 过滤器字段

tshark -r demo.pcap -T fields -e usb.capdata
#这个usb.capdata就是Leftover Capture Data的field name（字段名）
tshark -r demo.pcap -T fields -e usbhid.data
#同样的，这个usbhid.data就是HID data的field name 
```
<img src="/img/1666541579_6355680bb54c7afd75c48.jpeg" alt="img"  />



导出数据之后，就用python对应字典的数据写脚本解密就行，映射表是类似下面。

<img src="/img/1666541639_6355684762a980225540c.jpeg" alt="img"  />

## 数据部分详解：
> - 字节下标（我还没发现这个在哪儿）
>    - 0 : 修改键(组合键)
>    - 1 : OEM 保留
>    - 2~7 : 按键码
> - BYTE1 
>    - bit0: Left Control 是否按下，按下为 1
>    - bit1: Left Shift 是否按下，按下为 1
>    - bit2: Left Alt 是否按下，按下为 1
>    - bit3: Left WIN/GUI 是否按下，按下为 1
>    - bit4: Right Control 是否按下，按下为 1
>    - bit5: Right Shift 是否按下，按下为 1
>    - bit6: Right Alt 是否按下，按下为 1
>    - bit7: Right WIN/GUI 是否按下，按下为 1
> - BYTE2 - 暂不清楚，有的地方说是保留位
> - BYTE3-BYTE8 - 这六个为普通按键
> - 0b10（0x02） 和 0b100000（0x20）都是按下了shift键
>
> 例如： 键盘发送 02 00 0e 00 00 00 00 00，表示同时按下了 Left Shift + 'k'，即大写 K。
>
> 具体键位对应 [Universal Serial Bus HID Usage Tables - USB-IF](https://www.usb.org/sites/default/files/documents/hut1_12v2.pdf) 53页~59页	


## 我的键盘流量解密脚本
[may1as/UsbKbCracker: CTF中常见键盘流量解密脚本 (github.com)](https://github.com/may1as/UsbKbCracker)

根据wangyihang大佬的脚本改的，主要增加了个协议字段的选项
放在GitHub上了，用2021国赛初赛的举个例子吧

```python
python .\UsbKbCracker.py -f .\example\ez_usb.pcapng -e usbhid.data -Y "usb.src==2.8.1"
```
> 2023-5-27更新
>
> 有的比赛里，usb的数据位不一样，在源码里稍作修改就Ok

<img src="/img/1666541983_6355699f080910b589d61.jpeg" alt="img"  />

# 鼠标流量
鼠标流量同样，`Leftover Capture Data`——>`HID Data`可能是不同版本协议导致的（我还没去考证哈，别全信）



<img src="/img/1666542208_63556a8079fe23ca1faf4.jpeg" alt="img"  />



<img src="/img/1666542161_63556a515ffdc30d60a05.jpeg" alt="img"  />

## 数据部分详解：
与键盘击键的离散性不一样，鼠标移动时表现为连续性，不过实际上鼠标动作所产生的数据包也是离散的
鼠标数据包的数据长度为`4`个字节。

- 第一个字节代表按键，当取` 0x00` 时，代表没有按键、为 `0x01` 时，代表按左键，为 `0x02` 时，代表当前按键为右键。
- 第二个字节可以看成是一个 signed byte 类型，其最高位为符号位，当这个值为正（小于127）时，代表鼠标水平右移多少像素，为负（补码负数，大于127小于255）时，代表水平左移多少像素。
- 第三个字节与第二字节类似，代表垂直上下移动的偏移。
- 第四个是扩展字节，关于滚轮的操作记录
   - 0 - 没有滚轮运动
   - 1 - 垂直向上滚动一下
   - 0xFF - 垂直向下滚动一下
   - 2 - 水平滚动右键一下
   - 0xFE - 水平滚动左键单击一下

例如： 鼠标发送 00 01 fc 00，表示鼠标右移 01 像素，垂直向下移动 124 像素。

就这么多知识，剩下的就是写脚本了	
## 我的鼠标流量解密脚本
[may1as/UsbmiceCracker: CTF中常见鼠标流量解密脚本 (github.com)](https://github.com/may1as/UsbmiceCracker)

很多朋友使用wangyihang大佬的鼠标流量解密脚本，出现无法成功显示图片的问题。原因是tshark早前的版本导出数据带冒号，形如这样：`00:00:04:00:00:00:00:00`，而现在是`00000000ffff0000`，并不带冒号，导致处理数据时出现问题。 这里就处理了一下导出数据格式，增加了命令行里协议字段的选项

```python
 python .\UsbmiceCracker.py -f .\example\usb.capdata.pcapng -e usb.capdata -a LEFT
```
<img src="/img/1666542479_63556b8f5acb1c1f4e600.jpeg" alt="1666542479_63556b8f5acb1c1f4e600.png!small?1666542479895"  />

# 完

## 参考文章

[https://www.usb.org/document-library/device-class-definition-hid-111](https://www.usb.org/document-library/device-class-definition-hid-111)



