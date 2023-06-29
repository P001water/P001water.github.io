---
layout: post
title:  "渗透技巧——用python简单起个https服务器"
date:   2023-06-12
categories: 渗透技巧
---



> 文章首发于[用python简单起个https服务器 - 先知社区 (aliyun.com)](https://xz.aliyun.com/t/12605)

# 前言

偶然碰见的很有意思的内容

平时我们经常有起个http服务器的需求，一般是 `python -m http.server port`，很明显，这样的文件下载时是明文传输的

访问本地起的http服务

<img src="/img/image-20230613140607581.png" alt="image-20230613140607581"  />

wireshark捕获下，http的内容是直接可见的

<img src="/img/image-20230613140735449.png" alt="image-20230613140735449" style="zoom: 67%;" />



这时候考虑加个网站ssl证书把http > https，也就是自签名，反正也是访问我们自己的服务，下面是python代码实现，小改动自，[Yicong – Medium](https://ohyicong.medium.com/how-to-create-a-https-server-with-one-liner-of-code-655e7e28ccd)

自己可以根据需求改动签名中的信息，比如甩锅啥的

```python
# Python 3
# Usage: python3 SimpleHTTPSServer.py

from OpenSSL import crypto, SSL
from socket import gethostname
from pprint import pprint
from time import gmtime, mktime
import os
import http.server, ssl
import argparse

parser = argparse.ArgumentParser(description="SimpleHTTPSServer \n Usage: python3 SimpleHTTPSServer.py 443")
parser.add_argument("port", type=int, default=443, nargs="?", help="Port Number, default is 443")
args = parser.parse_args()
CERT_PATH = "server.crt"
KEY_PATH = "server.key"
PEM_PATH = "server.pem"

def create_self_signed_cert(cert_path,key_path,pem_path):
    # create a key pair
    k = crypto.PKey()
    k.generate_key(crypto.TYPE_RSA, 1024)

    # create a self-signed cert
    cert = crypto.X509()
    cert.get_subject().C = "UK"
    cert.get_subject().ST = "London"
    cert.get_subject().L = "London"
    cert.get_subject().O = "Dummy Company Ltd"
    cert.get_subject().OU = "Dummy Company Ltd"
    cert.get_subject().CN = os.gethostname()
    cert.set_serial_number(1000)
    cert.gmtime_adj_notBefore(0)
    cert.gmtime_adj_notAfter(10*365*24*60*60)
    cert.set_issuer(cert.get_subject())
    cert.set_pubkey(k)
    cert.sign(k, 'sha1')
    if (not os.path.exists(cert_path)):
        open(cert_path, "wb").write(crypto.dump_certificate(crypto.FILETYPE_PEM, cert))
    if (not os.path.exists(key_path)):
        open(key_path, "wb").write(crypto.dump_privatekey(crypto.FILETYPE_PEM, k))
    if (not os.path.exists(pem_path)):
        open(pem_path, "wb").write(crypto.dump_privatekey(crypto.FILETYPE_PEM, k))
        open(pem_path, "ab").write(crypto.dump_certificate(crypto.FILETYPE_PEM, cert))
    
def create_https_server(cert_path,key_path,port):
    server_address = ('0.0.0.0', port)
    try:
        httpd = http.server.HTTPServer(server_address, http.server.SimpleHTTPRequestHandler)
        httpd.socket = ssl.wrap_socket(httpd.socket, server_side=True, certfile='server.pem', ssl_version=ssl.PROTOCOL_TLS)
    except:
        exit("[ERR] Port %d has been taken."%(port))
    print("Serving HTTPS on 0.0.0.0 port %d..."%(port))
    httpd.serve_forever()
    
if __name__ == "__main__":
    create_self_signed_cert(CERT_PATH,KEY_PATH,PEM_PATH)
    create_https_server(CERT_PATH,KEY_PATH,args.port)
```

用上面的代码起个https服务，同样访问文件，因为是自签名的，所以用浏览器访问依然会被浏览器标记为不信任，还是显示为http

<img src="/img/image-20230613145420949.png" alt="image-20230613145420949"  />

<img src="/img/image-20230613141549693.png" alt="image-20230613141549693" style="zoom: 80%;" />

但是用wireshark抓包看，流量都已被加密了

<img src="/img/image-20230613141909325.png" alt="image-20230613141909325" style="zoom: 67%;" />





有的时候我们会放些敏感文件托管在服务器上，直接起个http的在服务器上，很容易几天就被标成恶意ip了。可能就是因为明文传输过程中敏感文件中的特征引起的，这种起个自签名的https服务，流量是被加密过的，对IDS在流量层面的检测应该也有点作用

# Download file



由于是自签名的，下载文件时需要忽略证书检查，否则下载不成功

<img src="/img/image-20230613134029212.png" alt="image-20230613134029212"  />

比如wget

```
wget --no-check-certificate https://192.168.110.1:9999/demo.py
```

<img src="/img/image-20230613144331303.png" alt="image-20230613144331303"  />
