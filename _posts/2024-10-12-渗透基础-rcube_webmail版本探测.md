---
layout: post
title:  "渗透基础 -- rcube_webmail邮服版本探测思路和代码实现"
date: 2024-10-12
categories: 渗透基础，武器化
---

# 简介

本文介绍了开源产品RoundCube webmail邮件系统的版本探测思路，并用go语言实现工具化、自动化探测。

> 文章首发于 [渗透基础-rcube_webmail版本探测 - 先知社区 (aliyun.com)](https://xz.aliyun.com/t/15765?time__1311=GqjxnQiQ0%3DdQqGNDQiKiI2R3ooi%3DKzqa4D)



# 正文



## 0x01 探测思路研究

探测系统版本，最理想的方法就是系统主页html代码中有特定的字符串，比如特定版本对应的hash在主页的html代码中等等。

对于rcube mail，我们先查看它的主页html代码

<img src="/img/image-20240927023732769.png" alt="image-20240927023732769" style="zoom:57%;" />

稍作分析，发现上面html正文中，`?s`后面的字符串可能符合要求

```
?s=1716107237
```

这里有个验证的小技巧，可以用 `1716107237` 作为fofa的语句搜索（body="1716107237"），看是否有足够多的资产的html代码中都有这串字符串，如果结果足够多，那很可能就可以依次判断。不过也只是可能，具体是否可以以此为依据还得从代码中看 `1716107237` 是如何生成的

比如下图，资产足够多，起码能排除是随机生成的可能

<img src="/img/image-20240927025528660.png" alt="image-20240927025528660" style="zoom:57%;" />

##  0x02 探测思路判断

由于rcube mail是个开源软件。我们即可直接从源代码中分析 `?s=` 后的值是如何得到的

全局搜索 `?s=` 后，很快定位到下面的函数

<img src="/img/image-20240927175742113.png" alt="image-20240927175742113" style="zoom:57%;" />

*\rcmail_output_html::file_mod*

参考代码可知，`?s=1716107237` -> filemtime()函数（如下图） -> 这串数字即是文件的最后修改时间。

<img src="/img/image-20240927211814934.png" alt="image-20240927211814934" style="zoom:57%;" />

理论上静态文件可能修改较少无法作为判断的依据，因为修改可能较少

实际测试发现rcube mail每个版本的文件最后修改时间都不同。可能是使用了Jenkins这种构建工具，每次重新生成代码导致每个发布版本的文件都会新建，导致即使文件内容不变动，文件的最后修改时间还是会变。

我们即可依据此作为判断



## 0x03 实现细节

 依据上面的思路，我们要实现版本检测，大致步骤如下

1. 下载全部版本的rcubemail
2. 计算 jquery.min.js 或其他静态资源文件的最后修改时间
3. 建立rcube mail版本和文件最后修改时间的映射
4. go语言实现发送请求从目标html代码中取得 `?s=` 后的字符串，以此判断

实现过程中下载全部版本的rcubemail较为麻烦，截至文章发布日最新版本是 v1.6.9，大致估略版本不少于70个，

肯定不能手动下载。

解决办法：

这里用了github的REST api，从下面api地址获取rcubemail的全部版本

```
https://api.github.com/repos/roundcube/roundcubemail/releases
```

解析api返回的json数据，提取对应版本压缩包地址然后下载即可，关键代码如下

```go
		// github releases API URL
		url := "https://api.github.com/repos/roundcube/roundcubemail/releases"

		if SocksProxy != "" {
			CustomhttpClient = httputils.NewHttpClient(
				httputils.WithSocks5Proxy(SocksProxy),
			)
		} else {
			CustomhttpClient = httputils.NewHttpClient()
		}

		JsonResp, err := httputils.HttpWithSocks(url, CustomhttpClient)
		if err != nil {
			fmt.Println("Error making request:", err)
			return
		}
		defer JsonResp.Body.Close()

		body, err := io.ReadAll(JsonResp.Body)
		if err != nil {
			fmt.Println("Error reading response body:", err)
			return
		}

		var releases []struct {
			Assets []Asset `json:"assets"`
		}
		if err := json.Unmarshal(body, &releases); err != nil {
			fmt.Println("Error parsing JSON:", err)
			return
		}

		gologger.Info().Msgf("Matching download URLs:")
		var wg sync.WaitGroup
		for _, release := range releases {
			for _, asset := range release.Assets {
				if !strings.Contains(asset.BrowserDownloadURL, "asc") && !strings.Contains(asset.BrowserDownloadURL, "framework") {
					wg.Add(1)
					go func(urlDownload, filename string) {
						defer wg.Done()
						fmt.Printf("Downloading %s...\n", filename)
						if err := downloadFile(filename, urlDownload); err != nil {
							fmt.Printf("Error downloading %s: %v\n", filename, err)
						} else {
							fmt.Printf("Downloaded %s successfully.\n", filename)
						}
					}(asset.BrowserDownloadURL, asset.BrowserDownloadURL[strings.LastIndex(asset.BrowserDownloadURL, "/")+1:])
				}
			}
		}
		wg.Wait()
```

然后从下载的源代码压缩包中导出每个版本的jquery.min.js文件，文件名带上rcube mail版本方便下一步使用。

关键代码如下

```go
		dir := "D:\\SEC_Note\\rcube_versions"

		files, err := os.ReadDir(dir)
		if err != nil {
			fmt.Println(err)
			return
		}

		for _, file := range files {
			if file.IsDir() {
				continue
				//fmt.Println("Folder: ", file.Name())
			} else {
				absPath, err := filepath.Abs(filepath.Join(dir, file.Name()))
				if err != nil {
					fmt.Println(err)
					continue
				}
				fmt.Println("File: ", absPath)
				mainDo(absPath)
			}
		}
```

最后用php计算文件的最后修改时间

```php
<?php
$directory = '.';
$files = scandir($directory);

foreach ($files as $file) {
    if ($file != "." && $file != "..") {
        $filePath = $directory . DIRECTORY_SEPARATOR . $file;
        if (is_file($filePath)) {
            $fileMTime = filemtime($filePath);
            echo $file . ' + ' . $fileMTime . PHP_EOL;
        }
    }
}

?>
```

建立映射，即可得到如下版本映射。然后用go实现请求目标获得html代码，匹配 `?s={{num}}` 即可

```go
		// Define version map
		versionMap := map[string]string{
			"1636751527": "roundcubemail-1.3.17-complete.tar.gz",
			"1636751547": "roundcubemail-1.3.17.tar.gz",
			"1612812581": "roundcubemail-1.4.11-complete.tar.gz",
			"1612812601": "roundcubemail-1.4.11.tar.gz",
			"1636753154": "roundcubemail-1.4.12-complete.tar.gz",
			"1636753175": "roundcubemail-1.4.12.tar.gz",
			"1640818035": "roundcubemail-1.4.13-complete.tar.gz",
			"1640818057": "roundcubemail-1.4.13.tar.gz",
			"1694896977": "roundcubemail-1.4.14-complete.tar.gz",
			"1694897047": "roundcubemail-1.4.14.tar.gz",
			"1697461030": "roundcubemail-1.4.15-complete.tar.gz",
			"1697461064": "roundcubemail-1.4.15.tar.gz",
			"1614281846": "roundcubemail-1.5-beta-complete.tar.gz",
			"1614281870": "roundcubemail-1.5-beta.tar.gz",
			"1625341161": "roundcubemail-1.5-rc-complete.tar.gz",
			"1625341187": "roundcubemail-1.5-rc.tar.gz",
			"1634503084": "roundcubemail-1.5.0-complete.tar.gz",
			"1634503112": "roundcubemail-1.5.0.tar.gz",
			"1637615532": "roundcubemail-1.5.1-complete.tar.gz",
			"1637615555": "roundcubemail-1.5.1.tar.gz",
			"1640816963": "roundcubemail-1.5.2-complete.tar.gz",
			"1640817081": "roundcubemail-1.5.2.tar.gz",
			"1656275218": "roundcubemail-1.5.3-complete.tar.gz",
			"1656275241": "roundcubemail-1.5.3.tar.gz",
			"1695024809": "roundcubemail-1.5.4-complete.tar.gz",
			"1695024837": "roundcubemail-1.5.4.tar.gz",
			"1697451371": "roundcubemail-1.5.5-complete.tar.gz",
			"1697451403": "roundcubemail-1.5.5.tar.gz",
			"1699175465": "roundcubemail-1.5.6-complete.tar.gz",
			"1699175495": "roundcubemail-1.5.6.tar.gz",
			"1716112649": "roundcubemail-1.5.7-complete.tar.gz",
			"1716112666": "roundcubemail-1.5.7.tar.gz",
			"1722763434": "roundcubemail-1.5.8-complete.tar.gz",
			"1722763449": "roundcubemail-1.5.8.tar.gz",
			"1725175135": "roundcubemail-1.5.9-complete.tar.gz",
			"1725175151": "roundcubemail-1.5.9.tar.gz",
			"1646598729": "roundcubemail-1.6-beta-complete.tar.gz",
			"1646598754": "roundcubemail-1.6-beta.tar.gz",
			"1655027980": "roundcubemail-1.6-rc-complete.tar.gz",
			"1655025289": "roundcubemail-1.6-rc.tar.gz",
			"1658607434": "roundcubemail-1.6.0-complete.tar.gz",
			"1658607455": "roundcubemail-1.6.0.tar.gz",
			"1674504194": "roundcubemail-1.6.1-complete.tar.gz",
			"1674504217": "roundcubemail-1.6.1.tar.gz",
			"1688210976": "roundcubemail-1.6.2-complete.tar.gz",
			"1688211001": "roundcubemail-1.6.2.tar.gz",
			"1694765308": "roundcubemail-1.6.3-complete.tar.gz",
			"1694765330": "roundcubemail-1.6.3.tar.gz",
			"1697448186": "roundcubemail-1.6.4-complete.tar.gz",
			"1697448209": "roundcubemail-1.6.4.tar.gz",
			"1699174738": "roundcubemail-1.6.5-complete.tar.gz",
			"1699174760": "roundcubemail-1.6.5.tar.gz",
			"1705745704": "roundcubemail-1.6.6-complete.tar.gz",
			"1705745675": "roundcubemail-1.6.6.tar.gz",
			"1716107237": "roundcubemail-1.6.7-complete.tar.gz",
			"1716107254": "roundcubemail-1.6.7.tar.gz",
			"1722764714": "roundcubemail-1.6.8-complete.tar.gz",
			"1722764729": "roundcubemail-1.6.8.tar.gz",
			"1725175896": "roundcubemail-1.6.9-complete.tar.gz",
			"1725175911": "roundcubemail-1.6.9.tar.gz",
		}
```

然后就是工具实现过程



## 0x04 工具实现和开源代码

工具最终的效果，相关代码随后开源在我的github地址 

https://github.com/P001water

```
P1rcubemail.exe getver -u http://192.168.110.142/roundcubemail-1.6.7/
```

<img src="/img/image-20240928023039417.png" alt="image-20240928023039417" style="zoom:57%;" />



# 小结

本文实现了rcube webmail的版本探测的go语言工具化，但是后续还有优化的空间，比如

1. 

后来发现从github api接口拿到的并不是全部版本，只是到v1.3.17，后面还可以加上前面的版本。所以匹配不到版本也就代表着版本小于v1.3.17

2. 

rcube mail系统访问很多情况需要挂代理，需要加上代理访问

后续随着使用需求更新
