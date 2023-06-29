---
layout: post
title: CTF---Python-ctf赛事通知机器人（飞书版）
author: may1as
date: 2022-08-07
categories: [瞎折腾,比赛通知机器人]
tags: [瞎折腾]
---

> 用的ctfhub的接口，分两个模块，放在云服务器上定时运行
>
> 本来想美化一下赛事发布的消息，增加比赛简介什么的，没啥精力折腾了（可能也没啥人看）
>
> 有需要的话，可以整合一下ctftime的赛事通知

```python
import requests
import re 
import time
import json
import hashlib
import base64
import hmac
from datetime import datetime


# 获取即将到来的比赛ID
def getupcoming_id():
    upcoming_url = "https://api.ctfhub.com/User_API/Event/getUpcoming" #获取即将开始的比赛ID
    upcoming_payload = {
        'offset': 0,
        'limit': 5
    } #POST请求
    upcoming_request = requests.post(upcoming_url, json=upcoming_payload)
    upcoming_id = []
    for i in range(len(upcoming_request.json()["data"]["items"])):
        upcoming_id.append(upcoming_request.json()["data"]["items"][i]["id"])
    return upcoming_id #返回比赛ID

    # 获取正在进行的比赛ID
def getrunning_id():
    running_url = "https://api.ctfhub.com/User_API/Event/getRunning" #获取正在进行的比赛ID
    running_payload = {
        'offset': 0,
        'limit': 4
    } #POST请求参数
    running_request = requests.post(running_url, json=running_payload)
    running_id = []
    for i in range(len(running_request.json()["data"]["items"])):
        running_id.append(running_request.json()["data"]["items"][i]["id"])
    return running_id #返回比赛ID

    # 通过比赛ID获取详细信息
def getinfo(id):
    s = ''
    for i in id:
        info_url = "https://api.ctfhub.com/User_API/Event/getInfo"  #获取详细信息的网址
        info_payload = {
            "event_id": i,
        } #POST请求参数
        info_request = requests.post(url=info_url, json=info_payload)
        js = info_request.json() #把返回值转为json格式
        s +="\n"\
        + "📛名称：" + "[" + js["data"]["title"] + "]" +"\n" \
        + "🕳比赛类型：" + js["data"]["class"] + "\n" \
        + "🕳比赛形式：" + js["data"]["form"]+ "\n" \
        + "⏰开始时间：" + time.strftime("%Y-%m-%d %H:%M:%S", time.localtime(js["data"]["start_time"])) + "\n" \
        + "⏰结束时间：" + time.strftime("%Y-%m-%d %H:%M:%S", time.localtime(js["data"]["end_time"])) + "\n" \
        + "✅比赛链接：" + js["data"]["official_url"] + "\n"\
        +"\n"
    return s

def sendtobot(zengyan,msg):
    url = "xxxxx"
    msgContent = {
        "text": zengyan+"\r\n"+msg,
    }
    req = {
        "timestamp": timestamp,
        # "sign": gen_sign(timestamp=timestamp, secret="PyiPNYD2OrHj4LktOqmJMg"),
        "msg_type": "text",
        "content": json.dumps(msgContent)   
    }
    payload = json.dumps(req)
    headers = {
        "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/105.0.0.0 Safari/537.36 Edg/105.0.1343.53",
        # 'Authorization': 'Bearer xxx', # your access token
        'Content-Type': 'application/json'
    }

    #推送通知
    response = requests.request("POST", url, headers=headers, data=payload)
    print(response.headers['X-Tt-Logid']) # for debug or oncall
    print(response.content) # Print Response


if __name__ == '__main__':
    # 当前日期和时间
    now = datetime.now()
    timestamp = datetime.timestamp(now)
    #发送到飞书
    print("---正在进行的比赛---")
    print(getinfo(getrunning_id()))
    sendtobot(zengyan = "🚩--正在进行的比赛--🏆",msg = getinfo(getrunning_id()))
```

```python
import requests
import re 
import time
import json
from datetime import datetime


# 获取即将到来的比赛ID
def getupcoming_id():
    upcoming_url = "https://api.ctfhub.com/User_API/Event/getUpcoming" #获取即将开始的比赛ID
    upcoming_payload = {
        'offset': 0,
        'limit': 3
    } #POST请求
    upcoming_request = requests.post(upcoming_url, json=upcoming_payload)
    upcoming_id = []
    for i in range(len(upcoming_request.json()["data"]["items"])):
        upcoming_id.append(upcoming_request.json()["data"]["items"][i]["id"])
    return upcoming_id #返回比赛ID

# 通过比赛ID获取详细信息
def getinfo(id):
    s = ''
    for i in id:
        info_url = "https://api.ctfhub.com/User_API/Event/getInfo"  #获取详细信息的网址
        info_payload = {
            "event_id": i,
        } #POST请求参数
        info_request = requests.post(url=info_url, json=info_payload)
        js = info_request.json() #把返回值转为json格式
        s +="\n"\
             + "📛名称：" + js["data"]["title"] + "\n" \
             + "🕳比赛类型：" + js["data"]["class"] + "\n" \
             + "🕳比赛形式：" + js["data"]["form"]+ "\n" \
             + "⏰开始时间：" + time.strftime("%Y-%m-%d %H:%M:%S", time.localtime(js["data"]["start_time"])) + "\n" \
             + "⏰结束时间：" + time.strftime("%Y-%m-%d %H:%M:%S", time.localtime(js["data"]["end_time"])) + "\n" \
             + "✅比赛链接：" + js["data"]["official_url"] + "\n"\
            +"\n"
    return s

def sendtobot(zengyan,msg):
    url = "xxxxxxxx"
    msgContent = {
        "text": zengyan+"\r\n"+msg,
    }
    req = {
        "timestamp": timestamp,
        # "sign": gen_sign(timestamp=timestamp, secret="PyiPNYD2OrHj4LktOqmJMg"),
        "msg_type": "text",
        "content": json.dumps(msgContent)   
    }
    payload = json.dumps(req)
    headers = {
        "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/105.0.0.0 Safari/537.36 Edg/105.0.1343.53",
        # 'Authorization': 'Bearer xxx', # your access token
        'Content-Type': 'application/json'
    }

    #推送通知
    response = requests.request("POST", url, headers=headers, data=payload)
    print(response.headers['X-Tt-Logid']) # for debug or oncall
    print(response.content) # Print Response


if __name__ == '__main__':
    # 当前日期和时间
    now = datetime.now()
    timestamp = datetime.timestamp(now)
    #发送到飞书
    print("🚩--即将到来的比赛--🏆")
    print(getinfo(getupcoming_id()))
    sendtobot(zengyan = "🚩--即将到来的比赛--🏆",msg = getinfo(getupcoming_id()))

```
