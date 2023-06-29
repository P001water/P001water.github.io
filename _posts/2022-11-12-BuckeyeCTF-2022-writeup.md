---
layout: post
title: CTF---BuckeyeCTF-2022-writeup
author: may1as
date: 2022-11-12
tags: [CTF,Writeup]
categories: [writeup]
---

# 文前漫谈

> 比赛地址：[https://pwnoh.io/challs](https://pwnoh.io/challs)


# Web

## buckeyenotes/beginner

> des：Note taking apps are all the rage lately but turns our they're harder to make than I thought :/. Even in development buckeyenotes has gotten some traction, Brutus signed up! I think his user name is brutusB3stNut9999. I wonder what kind of notes he writes 🤔 but I don't have his login....
> [https://buckeyenotes.chall.pwnoh.io](https://buckeyenotes.chall.pwnoh.io)

来看看国际友人的题目，登录界面，题目说了Brutus的用户名是brutusB3stNut9999
![image.png](https://cdn.nlark.com/yuque/0/2022/png/22999319/1668186755508-73ec6483-4391-4684-ac7d-fef64b81379e.png#averageHue=%23e1b77b&clientId=u17c9685a-5f1c-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=310&id=u225b7c18&margin=%5Bobject%20Object%5D&name=image.png&originHeight=465&originWidth=1400&originalType=binary&ratio=1&rotation=0&showTitle=false&size=31319&status=done&style=none&taskId=uf7d65e2c-9941-4fb9-a0fa-75fb5a322c4&title=&width=933.3333333333334)很自然想到sql注入了，于是对比一下输入回显，发现可能真的存在注入了
![image.png](https://cdn.nlark.com/yuque/0/2022/png/22999319/1668186975870-1b149cad-f59a-4633-8fd7-13fe20e1e939.png#averageHue=%23fdfdfd&clientId=u17c9685a-5f1c-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=221&id=u2a5d7bd6&margin=%5Bobject%20Object%5D&name=image.png&originHeight=331&originWidth=1013&originalType=binary&ratio=1&rotation=0&showTitle=false&size=12601&status=done&style=none&taskId=uf780ebaf-1808-4d85-a8d7-ef0211679fc&title=&width=675.3333333333334)
![image.png](https://cdn.nlark.com/yuque/0/2022/png/22999319/1668187027657-7d7551c4-cf4a-48ea-a46c-17fbd88be2d0.png#averageHue=%23fefefd&clientId=u17c9685a-5f1c-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=225&id=u6b27edc8&margin=%5Bobject%20Object%5D&name=image.png&originHeight=338&originWidth=1357&originalType=binary&ratio=1&rotation=0&showTitle=false&size=14058&status=done&style=none&taskId=uc0764901-6829-43f0-ab9a-7b386ec0707&title=&width=904.6666666666666)
然后一番折腾发现了这个，意思应该是过滤了等号。那就直接`1' or 1 -- -`
![image.png](https://cdn.nlark.com/yuque/0/2022/png/22999319/1668187138169-441068c0-df5d-4ff0-a4b1-8d0f1d08e046.png#averageHue=%23dcb373&clientId=u17c9685a-5f1c-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=170&id=u360c9431&margin=%5Bobject%20Object%5D&name=image.png&originHeight=364&originWidth=1162&originalType=binary&ratio=1&rotation=0&showTitle=false&size=44132&status=done&style=none&taskId=ucbe79d5e-d9a8-44b2-b22e-90fae6843f7&title=&width=544)![image.png](https://cdn.nlark.com/yuque/0/2022/png/22999319/1668217217370-280a5e5e-ce84-490b-a8b7-fae917fbd9e7.png#averageHue=%23fdfdfd&clientId=u095a0c98-f5b5-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=248&id=u1ff164a4&margin=%5Bobject%20Object%5D&name=image.png&originHeight=372&originWidth=1085&originalType=binary&ratio=1&rotation=0&showTitle=false&size=13157&status=done&style=none&taskId=u192e0464-cbff-4b23-b585-551368dd14c&title=&width=723.3333333333334)
应该是已经成功查询了，然后想着限制一下查询数量看看，也是试了不少次

```sql
' or 1 limit 1, 2-- -
过滤了等号
```

直接就出了![image.png](https://cdn.nlark.com/yuque/0/2022/png/22999319/1668187714872-57a5229b-b60d-469f-b95f-a2b5eded9cfd.png#averageHue=%23fdfdfc&clientId=u17c9685a-5f1c-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=229&id=ufb60dc2a&margin=%5Bobject%20Object%5D&name=image.png&originHeight=344&originWidth=979&originalType=binary&ratio=1&rotation=0&showTitle=false&size=16623&status=done&style=none&taskId=u579616bb-c797-47a8-a301-d4361daa81c&title=&width=652.6666666666666)

### View source 

![image.png](https://cdn.nlark.com/yuque/0/2022/png/22999319/1668219706177-39d02000-ec08-4ca4-b2b7-e9de0b8a2adc.png#averageHue=%232a2e40&clientId=u095a0c98-f5b5-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=295&id=ua800c5b3&margin=%5Bobject%20Object%5D&name=image.png&originHeight=443&originWidth=1656&originalType=binary&ratio=1&rotation=0&showTitle=false&size=70761&status=done&style=none&taskId=ud673b059-4f48-496d-beeb-becbdca199f&title=&width=1104)
![image.png](https://cdn.nlark.com/yuque/0/2022/png/22999319/1668219721836-0f42e7dc-94ab-4158-96cb-c29b43fb8901.png#averageHue=%232a2e40&clientId=u095a0c98-f5b5-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=211&id=u414b36bf&margin=%5Bobject%20Object%5D&name=image.png&originHeight=317&originWidth=1578&originalType=binary&ratio=1&rotation=0&showTitle=false&size=49409&status=done&style=none&taskId=u0145ddb0-a8a2-403c-994d-06aa3917dfe&title=&width=1052)
用的是sqlite数据库![image.png](https://cdn.nlark.com/yuque/0/2022/png/22999319/1668219859301-9c5a8940-2e95-4279-ba69-9abec10ecb71.png#averageHue=%23f9f8f8&clientId=u095a0c98-f5b5-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=232&id=uc0605c18&margin=%5Bobject%20Object%5D&name=image.png&originHeight=348&originWidth=1149&originalType=binary&ratio=1&rotation=0&showTitle=false&size=23405&status=done&style=none&taskId=u1603b8cc-5460-4d9e-ad2c-b9ae89d7355&title=&width=766)
代码逻辑要求sql语句中查询的结果是`brutusB3stNut9999`，如果是则`$authenticated = true;`得到flag，呜呜呜，要是有密码，直接输密码就行。索性这里只过滤了个=，我们只要limit限制查询结果就OK，也就是

```sql
' or 1 limit 1, 2-- -
```

但是不知道为什么`' or 1 limit 1, 1-- -`不行，这点暂时没想明白
![image.png](https://cdn.nlark.com/yuque/0/2022/png/22999319/1668220642017-b86f7179-0338-4956-a2e7-d0a6731d6a96.png#averageHue=%23fdfdfd&clientId=u095a0c98-f5b5-4&crop=0&crop=0&crop=1&crop=0.8379&from=paste&height=300&id=ud2835ba5&margin=%5Bobject%20Object%5D&name=image.png&originHeight=450&originWidth=1237&originalType=binary&ratio=1&rotation=0&showTitle=false&size=13192&status=done&style=none&taskId=u4d365121-db1a-4502-a3df-6108b93da3e&title=&width=825)

## pong/beginner

> des：I dug up my first ever JavaScript game, but this time, my AI is unbeatable!! Hah!
> [https://pong.chall.pwnoh.io](https://pong.chall.pwnoh.io/)

A  JavaScript game，我对JavaScript的印象就是直接在响应包中找base64加密过的flag，或者是console弹一段代码得到flag，这个题目还真是涨见识了

```javascript
<script>
const socket = io();
const canvas = document.getElementById("game");
const pl = .16;
const pw = .02;
const bs = .04;
var up = 0;
var down = 0;
var p1 = .5;
var p2 = .5;
var bx = .5;
var by = .5;
var bvx = 0;
var bvy = 0;
var spin = 0;
var bt = 0;

var s1 = 0;
var s2 = 0;

function reset(ctx) {
  ctx.resetTransform();
  ctx.translate(0, 0.5);
  ctx.lineWidth = 1;
}

function set() {
  bx = 0.5;
  by = 0.5;
  bvx = 0;
  bvy = 0;
  bt = 0;
  spin = 0;
}

function draw() {
  canvas.width = canvas.clientWidth;
  canvas.height = canvas.clientHeight;
  const w = canvas.width;
  const h = canvas.height;
  const ctx = canvas.getContext("2d");
  reset(ctx);
  ctx.fillStyle = "#FFFFFF";
  ctx.fillRect(0, 0, w, h);

  // field lines
  ctx.fillStyle = "#aaaaaa";
  for(var y = 0; y < 1; y += .1) {
    ctx.fillRect(w / 2 - 5, (y + .025) * h, 10, .05 * h);
  }

  // ball
  ctx.fillStyle = "#000000";
  ctx.translate(bx * w, by * h);
  bt += spin;
  while(bt < 0) bt += 360;
  while(bt > 359) bt -= 360;
  ctx.rotate(bt * Math.PI / 180);

  ctx.fillRect(-(bs * h) / 2, -(bs * h) / 2, bs * h, bs * h);
  reset(ctx);

  // paddles
  ctx.fillStyle = "#000000";
  ctx.fillRect(pw * w, (p1 - pl / 2) * h, pw * w, pl * h);
  ctx.fillRect((1 - 2 * pw) * w, (p2 - pl / 2) * h, pw * w, pl * h);

  // scores
  for(var x = 0; x < 20; x++) {
    ctx.beginPath();
    ctx.rect(x * .05 * w, 0, .05 * w, 0.01 * h);
    ctx.stroke();
    if(x < s1) ctx.fillRect(x * .05 * w, 0, .05 * w, 0.01 * h);
    if(x > 9 && s2 > 19 - x) ctx.fillRect(x * .05 * w, 0, .05 * w, 0.01 * h);
  }
}

function tick() {
  const w = canvas.width;
  const h = canvas.height;

  // controls
  if(p1 - up * .01 > pl / 2) p1 -= up * .01;
  if(p1 + down * .01 < 1 - pl / 2) p1 += down * .01;
  p2 = by;

  // ball
  if(bvx != 0) spin = bvy / bvx * 5;
  bx += bvx;
  by += bvy;
  if(by < 0 || by > 1) bvy *= -1; // v bounce
  if(bx < pw * 2) {
    // left paddle bounce
    if(by > p1 - pl / 2 && by < p1 + pl / 2) {
      let diff = by - p1;
      bvy = .015 * diff / (pl / 2);
      bvx = .015 - Math.abs(bvy);
    }
  }
  if(bx > 1 - pw * 2) {
    // right paddle bounce
    if(by > p2 - pl / 2 && by < p2 + pl / 2) {
      let diff = by - p2;
      bvy = .015 * diff / (pl / 2);
      bvx = -(.015 - Math.abs(bvy));
    }   
  }

  if(bx < -.1 || bx > 1.1) {
    socket.emit("score", bx);
  }

  draw();
}

function init() {
  draw();
  setInterval(tick, 13);
  document.addEventListener("keydown", (e) => {
    if(e.key == "w") up = 1;
    if(e.key == "s") down = 1;
    if(e.key == "p") {
      socket.emit("begin");
    }
  });
  document.addEventListener("keyup", (e) => {
    if(e.key == "w") up = 0;
    if(e.key == "s") down = 0;
  });
}

socket.on("alert", (msg) => alert(msg));
socket.on("begin", (params) => {
  bvx = params.bvx;
  bvy = params.bvy;
});
socket.on("set", (scores) => {
  set();
  s1 = scores.sx1;
  s2 = scores.sx2;
});
</script>
```

> Since JavaScript is a client-side language, we can use our console in the developer tool to control some variables.

So 大意就是我们可以在开发者工具的控制台中控制这些变量，尽管看不太懂js，但大概能知道s1和s2是分数，而s1是我的分数，s2是对手的分数，于是可以在控制台中修改变量。
但是没啥用啊。但是这样打开了思路，可以找其他变量试试，最后发现了`bx`这个变量，是ball在x坐标上的位置，于是我们可以自己控制球的位置
![image.png](https://cdn.nlark.com/yuque/0/2022/png/22999319/1668224732500-cfc4c798-5ef6-4c62-95e0-0b5bfd5e9b54.png#averageHue=%23202124&clientId=u095a0c98-f5b5-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=137&id=u6406f77a&margin=%5Bobject%20Object%5D&name=image.png&originHeight=206&originWidth=1044&originalType=binary&ratio=1&rotation=0&showTitle=false&size=4826&status=done&style=none&taskId=u3120ced5-366b-44d6-a985-e7e3034013c&title=&width=696)
最后令bx等于1.5，球就越过了对手的防线(超出了屏幕)，we win ！，就得到了flag
![image.png](https://cdn.nlark.com/yuque/0/2022/png/22999319/1668224705181-57d0a30f-11b1-4939-977c-c04bac2e13d1.png#averageHue=%23999998&clientId=u095a0c98-f5b5-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=173&id=u7e685b2b&margin=%5Bobject%20Object%5D&name=image.png&originHeight=260&originWidth=972&originalType=binary&ratio=1&rotation=0&showTitle=false&size=19431&status=done&style=none&taskId=u1134502d-58cf-4084-875f-7e7c0a2c155&title=&width=648)


# Misc

## what-you-see-is-what-you-git/beginner

> des：I definitely made a Git repo, but I somehow broke it. Something about not getting a HEAD of myself.

git相关的题目，关键还是logs日志文件，HEAD中有些线索
![image.png](https://cdn.nlark.com/yuque/0/2022/png/22999319/1668250367571-0ab3f7e3-071f-4737-a7fa-c70008d7ef67.png#averageHue=%2338352f&clientId=u381ece55-2730-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=375&id=ua3d81ad3&margin=%5Bobject%20Object%5D&name=image.png&originHeight=562&originWidth=1894&originalType=binary&ratio=1&rotation=0&showTitle=false&size=373020&status=done&style=none&taskId=u453a60c5-7dbf-4b88-b1bb-2444bdcc563&title=&width=1262.6666666666667)
然后恢复Hid the flag提交的暂存区

> git reset --hard 7ae8453a76a41d40bdfcc7992175390f70ba9fdf
> 注解	# 重置暂存区与工作区，与上一次commit保持一致

再

> git log -p [file]
> 注解# 显示指定文件相关的每一次diff

得到flag
![image.png](https://cdn.nlark.com/yuque/0/2022/png/22999319/1668252918384-917496f3-9664-4a97-8bd0-56d35a447bee.png#averageHue=%232d2a27&clientId=u381ece55-2730-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=564&id=ua92949ea&margin=%5Bobject%20Object%5D&name=image.png&originHeight=846&originWidth=1258&originalType=binary&ratio=1&rotation=0&showTitle=false&size=490677&status=done&style=none&taskId=ub1aea956-09c9-407b-a4eb-7b9a6122a08&title=&width=838.6666666666666)

## sus/easy

> des：Something about this audio is pretty _sus_...
> Hint: The crackling in the audio should tell you that something's wrong.

wav的LSB隐写，参加 [Medium blog](https://sumit-arora.medium.com/audio-steganography-the-art-of-hiding-secrets-within-earshot-part-2-of-2-c76b1be719b3)!

```javascript
import wave
song = wave.open("sus.wav", mode='rb')
# Convert audio to byte array
frame_bytes = bytearray(list(song.readframes(song.getnframes())))

# Extract the LSB of each byte
extracted = [frame_bytes[i] & 1 for i in range(len(frame_bytes))]
# Convert byte array back to string
string = "".join(chr(int("".join(map(str,extracted[i:i+8])),2)) for i in range(0,len(extracted),8))
# Cut off at the filler characters
decoded = string.split("###")[0]

# Print the extracted text
print("Sucessfully decoded: "+decoded)
song.close()
```

![image.png](https://cdn.nlark.com/yuque/0/2022/png/22999319/1668257881655-ef7555c3-23ca-4987-be82-07ebd661026c.png#averageHue=%23423e3a&clientId=u381ece55-2730-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=264&id=u02e3b1c6&margin=%5Bobject%20Object%5D&name=image.png&originHeight=396&originWidth=1099&originalType=binary&ratio=1&rotation=0&showTitle=false&size=347522&status=done&style=none&taskId=u6b8db589-3f22-4504-92c5-44d534e6ac0&title=&width=732.6666666666666)

## keyboardwarrior/medium

> des：I found a PCAP of some Bluetooth packets being sent on this guy's computer. He's sending some pretty weird stuff, you should take a look.
> Flag format: buckeyectf{}

_相对平时的蓝牙流量，是很新的题目（协议）了。HCI_EVT和ATT协议，关于协议懒得去深究了，反正和一般的键盘数据数据段相同。	_
![image.png](https://cdn.nlark.com/yuque/0/2022/png/22999319/1668233632166-827e07bf-6117-4547-b648-f0fb092efe8a.png#averageHue=%2388c89c&clientId=u095a0c98-f5b5-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=385&id=u7e5e41e8&margin=%5Bobject%20Object%5D&name=image.png&originHeight=577&originWidth=2518&originalType=binary&ratio=1&rotation=0&showTitle=false&size=167464&status=done&style=none&taskId=u0437f91d-8983-44fb-be29-3fa4ce0b53e&title=&width=1678.6666666666667)
一大堆数据中只有第三个字节不同，于是把以前写的脚本换个导出字段，改改就行
[https://github.com/may1as/UsbMiceDataexp](https://github.com/may1as/UsbMiceDataexp)
![image.png](https://cdn.nlark.com/yuque/0/2022/png/22999319/1668233934352-8ec22e62-ca7c-47bb-8fd5-dfba6ee69d47.png#averageHue=%23efedec&clientId=u095a0c98-f5b5-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=297&id=u86da962a&margin=%5Bobject%20Object%5D&name=image.png&originHeight=445&originWidth=1563&originalType=binary&ratio=1&rotation=0&showTitle=false&size=79974&status=done&style=none&taskId=ubedbf63e-3950-487d-84c7-c61eca18b9f&title=&width=1042)
那个value的过滤器字段如下，然后-e替换成btatt.value即可
![image.png](https://cdn.nlark.com/yuque/0/2022/png/22999319/1668234317623-55a54416-d43a-49b8-be81-f67060b610c1.png#averageHue=%23fefcfb&clientId=u095a0c98-f5b5-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=204&id=u3b7ec34a&margin=%5Bobject%20Object%5D&name=image.png&originHeight=306&originWidth=2179&originalType=binary&ratio=1&rotation=0&showTitle=false&size=45564&status=done&style=none&taskId=u236e7b85-66d0-42a4-be54-5d741f5b510&title=&width=1452.6666666666667)
_但是这里还有个坑，得把_`_[]_`_换成_`_{}_`_，_`_-_`_换成_`___`_，符号不同可能是字典不一致导致的，但是这里无伤大雅_
![image.png](https://cdn.nlark.com/yuque/0/2022/png/22999319/1668239199020-ad70732e-dc4a-4c4c-8fc0-3ea2258f0413.png#averageHue=%23312d2a&clientId=u095a0c98-f5b5-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=504&id=u97a08891&margin=%5Bobject%20Object%5D&name=image.png&originHeight=756&originWidth=1777&originalType=binary&ratio=1&rotation=0&showTitle=false&size=628350&status=done&style=none&taskId=u50a7d481-6318-4bab-8ed3-6539a106fef&title=&width=1184.6666666666667)

# 一些小工具

file command for windows

> [https://gnuwin32.sourceforge.net/packages/file.htm](https://gnuwin32.sourceforge.net/packages/file.htm)



# resource

[https://siunam321.github.io/ctf/BuckeyeCTF-2022/](https://siunam321.github.io/ctf/BuckeyeCTF-2022/)
