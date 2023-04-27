## XSStrike

XSStrike 是一个开源的 XSS 漏洞扫描器。作为一个扫描器，他宣传自己用了四个手写的 payload 生成器，强大的模糊搜索引擎（应该是用来匹配漏洞规则，找注入点的），支持 DOM-XSS 扫描，支持爬虫扫描，还支持 WAF 检测和绕过等功能。

[s0md3v/XSStrike: Most advanced XSS scanner. (github.com)](https://github.com/s0md3v/XSStrike)

我对这个软件的总结是

### 安装

> 版本：release v3.1.5

该软件依赖 python3 和 pip

```shell
git clone https://github.com/s0md3v/XSStrike.git

cd XSStrike

pip install -r requirements.txt
```

### 使用手册

先贴一份帮助手册

```shell
┌──(kali㉿kali)-[~/XSStrike]
└─$ python3 xsstrike.py -h

        XSStrike v3.1.5

usage: xsstrike.py [-h] [-u TARGET] [--data PARAMDATA] [-e ENCODE] [--fuzzer] [--update] [--timeout TIMEOUT] [--proxy] [--crawl] [--json] [--path]
                   [--seeds ARGS_SEEDS] [-f ARGS_FILE] [-l LEVEL] [--headers [ADD_HEADERS]] [-t THREADCOUNT] [-d DELAY] [--skip] [--skip-dom] [--blind]
                   [--console-log-level {DEBUG,INFO,RUN,GOOD,WARNING,ERROR,CRITICAL,VULN}] [--file-log-level {DEBUG,INFO,RUN,GOOD,WARNING,ERROR,CRITICAL,VULN}]
                   [--log-file LOG_FILE]

options:
  -h, --help            show this help message and exit
  -u TARGET, --url TARGET
                        url
  --data PARAMDATA      post data
  -e ENCODE, --encode ENCODE
                        encode payloads
  --fuzzer              fuzzer
  --update              update
  --timeout TIMEOUT     timeout
  --proxy               use prox(y|ies)
  --crawl               crawl
  --json                treat post data as json
  --path                inject payloads in the path
  --seeds ARGS_SEEDS    load crawling seeds from a file
  -f ARGS_FILE, --file ARGS_FILE
                        load payloads from a file
  -l LEVEL, --level LEVEL
                        level of crawling
  --headers [ADD_HEADERS]
                        add headers
  -t THREADCOUNT, --threads THREADCOUNT
                        number of threads
  -d DELAY, --delay DELAY
                        delay between requests
  --skip                do not ask to continue
  --skip-dom            skip dom checking
  --blind               inject blind XSS payload while crawling
  --console-log-level {DEBUG,INFO,RUN,GOOD,WARNING,ERROR,CRITICAL,VULN}
                        Console logging level
  --file-log-level {DEBUG,INFO,RUN,GOOD,WARNING,ERROR,CRITICAL,VULN}
                        File logging level
  --log-file LOG_FILE   Name of the file to log
```

介绍一些常用的命令：

* -u   指定 url
* --skip-dom    跳过 DOM 型XSS 检测
* --data    添加 Post 方法参数
* --headers    添加请求头，key 和 value 之间要加空格 比如：`--headers "Cookie: 123"`
* --crawl    爬取指定网页并进行 XSS 检测
* -l    爬取深度，默认2层，配合 --crawl 使用
* --fuzzer    测试指定网页的过滤器和 WAF
* -d    设置请求之间的延迟
* --update    更新 strike，如果有新版本可用，strkie将下载最新版并合并到当前目录


### 靶场试验

以 DVWA 和 pikachu 为测试靶场，以各种常见的攻击方式为样本，学习使用这个工具。

#### 反射型 Get 方法

从难度为 low 的 DVWA 反射型 XSS 攻击模块开始。

先随便输入点东西，点击 Submit，获取注入点的 url，接着使用 XSStrike（后面简称 strike）扫描这个 url，已知这个注入点是反射型 XSS，所以我们直接加上 `--skip-dom` 跳过 DOM型 XSS 的扫描。

```shell
┌──(kali㉿kali)-[~/XSStrike]
└─$ python3 xsstrike.py -u "http://192.168.127.200/DVWA/vulnerabilities/xss_r/?name=DonkeyQian#" --skip-dom

XSStrike v3.1.5
# 没有检测到在线的 WAF（正确）
[+] WAF Status: Offline
# 检测到注入点的参数为 name（正确）
[!] Testing parameter: name
# 检测出了一个反射型注入点（正确）
[!] Reflections found: 1
[~] Analysing reflections
[~] Generating payloads
# 生成了 1536 个 payload，是不是有点太多了
[!] Payloads generated: 1536
------------------------------------------------------------
[+] Payload: <HTmL%0aONPoINtEReNTER+=+(prompt)``%0dx//
# 这个应该是效率评分
[!] Efficiency: 92
# 这个应该是有效性评分
[!] Confidence: 10
------------------------------------------------------------
......
```

随便找了几个 payload 放进去，确实有效，就是这个 payload 也未免生成的太多了吧。

将难度依次从 low 调整到 impossible ，strike 生成的 payload 直到 impossible 才失效，但是 strike 在扫描 impossible 难度的 url 时，也显示检测出了注入点，在命令行中加上 cookie，结果还是一样。

使用 wireshark 抓包，发现 strike 的确往靶场服务器发送了很多 HTTP 请求，但是靶场返回的数据已经把 payload 使用编码成了 HTML 实体，所以的确是误报了。

![](../../image/Pasted%20image%2020230426182936.png)

***
#### 反射型 Post 方法

post 方法的XSS 注入可以使用 DVWA 存储型 XSS 模块和 pikachu 的 反射性xss(post) 模块。

使用 Get 方法检测漏洞的时候直接将参数放在 url 后面，strike 就能自动检测出来，但是 Post 方法的参数不是通过 url 传递的，需要我们手动添加。

从 DVWA 的存储型 XSS 模块开始，随便提交点什么东西，然后通过浏览器的开发者工具获取 url 和请求参数。

```shell
┌──(kali㉿kali)-[~/XSStrike]
└─$ python3 xsstrike.py -u "http://192.168.127.200/DVWA/vulnerabilities/xss_s/" --data "txtName=nick&mtxMessage=1&btnSign=Sign+Guestbook" --headers "Cookie: PHPSESSID=pq1qcitrhgdo0b9p7uj19vf21q; security=low" --skip-dom

[+] WAF Status: Offline
[!] Testing parameter: txtName
[!] Reflections found: 1
[~] Analysing reflections
[~] Generating payloads
[!] Payloads generated: 3071
```

由于是 post 方法，所以要记得加上 cookie，然后提醒一下各位，做这个实验的时候要及时使用 `ctrl+c` 终止程序，测几个 payload 就行了，不然几千个弹窗会把你的页面直接卡死，我最后只能上数据库删表了...

从 low 难度开始测试，同样是到 impossible 难度就不行了，依然是有误报，而且有点抽风。

```shell
┌──(kali㉿kali)-[~/XSStrike]
└─$ python3 xsstrike.py -u "http://192.168.127.200/DVWA/vulnerabilities/xss_s/" --data "txtName=nick&mtxMessage=1&btnSign=Sign+Guestbook" --headers "Cookie: PHPSESSID=pq1qcitrhgdo0b9p7uj19vf21q; security=impossible" --skip-dom

        XSStrike v3.1.5

[+] WAF Status: Offline
[!] Testing parameter: txtName
[!] Reflections found: 1
[~] Analysing reflections
[~] Generating payloads
[!] Payloads generated: 1536
 ~] Progress: 1536/1536
[!] Testing parameter: mtxMessage
[!] Reflections found: 770
[~] Analysing reflections
[~] Generating payloads
[!] Payloads generated: 1148740
 ~] Progress: 29/1148740
```

mtxMessage 这个参数发现了 770 个注入点，给我都整不自信了，查了一下是不是我给翻译错了，国民党报战功都不敢像你这么报。而且一百万的 payload 给我搁这搞 DOS 呢？

***
再测试一下 pikachu 这个靶场，使用的模块是 反射型xss(post) 和 存储型xss。

首先来看 反射型xss(post) 这个模块，要先登录上去，再手工随便填点东西提交，然后开发者工具获取 url 和请求参数和 cookie。

```shell
┌──(kali㉿kali)-[~/XSStrike]
└─$ python3 xsstrike.py -u "http://192.168.127.200:9000/vul/xss/xsspost/xss_reflected_post.php" --data "message=nobody&submit=submit" --headers "Cookie: ant[uname]=test; ant[pw]=1fb586718403e6b398655502d2114f5ac27badd1; PHPSESSID=pq1qcitrhgdo0b9p7uj19vf21q; security=impossible" --skip-dom

[+] WAF Status: Offline
[!] Testing parameter: message
[!] Reflections found: 1
[~] Analysing reflections
[~] Generating payloads
[!] Payloads generated: 3072

------------------------------------------------------------
[+] Payload: <d3v%09OnmOuseoVEr+=+confirm()>v3dm0s
[!] Efficiency: 91
[!] Confidence: 10
------------------------------------------------------------
......
 ~] Progress: 3072/3072
[!] Testing parameter: submit
[-] No reflection found
```

用 hackbar 插件提交post 请求，payload 基本都能用。

再来看看 存储型xss，照样是手工提交，获取 url、请求参数和 cookie。

```shell
┌──(kali㉿kali)-[~/XSStrike]
└─$ python3 xsstrike.py -u "http://192.168.127.200:9000/vul/xss/xss_stored.php" --data "message=%E7%95%99%E4%B8%AA%E7%9B%90&submit=submit" --skip-dom --headers "Cookie: PHPSESSID=pq1qcitrhgdo0b9p7uj19vf21q; security=impossible"

        XSStrike v3.1.5

[+] WAF Status: Offline
[!] Testing parameter: message
[!] Reflections found: 2
[~] Analysing reflections
[~] Generating payloads
[!] Payloads generated: 3071
------------------------------------------------------------
[+] Payload: <htMl%0dONPoIntERENTEr+=+(prompt)``>
[!] Efficiency: 100
[!] Confidence: 10
```

用 hackbar 提交 post 请求

![](../../image/Pasted%20image%2020230426221415.png)

新增了一个鼠标移上去就会弹窗的标签，注入成功。

![](../../image/Pasted%20image%2020230426221513.png)

#### DOM型XSS 检测

还是用的 pikachu 靶场，DOM型xss 模块

```shell
┌──(kali㉿kali)-[~/XSStrike]
└─$ python3 xsstrike.py -u "http://192.168.127.200:9000/vul/xss/xss_dom.php" --headers "Cookie: PHPSESSID=pq1qcitrhgdo0b9p7uj19vf21q; security=impossible"

        XSStrike v3.1.5

[~] Checking for DOM vulnerabilities
# 检测出了易受攻击的对象（正确）
[+] Potentially vulnerable objects found
------------------------------------------------------------
4   document.getElementById("dom").innerHTML = "<a href='"+str+"'>what do you see?</a>";
2   if('ontouchstart' in document.documentElement) document.write("<script src='../../assets/js/jquery.mobile.custom.min.js'>"+"<"+"/script>");
------------------------------------------------------------
# 但是没有参数测试
[-] No parameters to test.
```

注入点的 HTML 代码如下图所示，感觉是 strike 根据正则表达式匹配到了这个注入点，但是没有找到哪里会触发这段易受攻击的代码。所以没有给我们生成 payload。

Dom 型 XSS 检测算是成功了一半吧，一般能找出来这段代码我们就能顺藤摸瓜找到可能的注入点了。

![](../../image/Pasted%20image%2020230426233838.png)

#### 爬虫功能

strike 还提供了一个爬虫功能，官方文档只提到了他是多线程的，爬取速度惊人，emmmm，既然这样，那大家就不要对他的准确性抱太大的期待了。

还是使用 DVWA 和 pikachu 进行测试。

*** 
pikachu 检测结果如下，有点长，大家可以不用看完，看我的总结就行，扫描出了一些项目依赖的组件，jquery 和 bootstrap ，还给出了组件的版本和当前版本存在哪些漏洞，漏洞的等级，还有漏洞的 CVE 编号。

但是 XSS 检测就是未来可期了，感觉就是爬了这个路径下的一堆文件，然后用正则表达式匹配可能存在的漏洞，我还以为能给我扫出来一堆超链接呢。感觉期望值还是有点高了。但是 Dom型xss 还是扫出来了的。

加上 `--skip-dom` 参数跳过 DOM 型xss 扫描，果不其然，xss 漏洞一个都没扫出来。

```shell
┌──(kali㉿kali)-[~/XSStrike]
└─$ python3 xsstrike.py -u "http://192.168.127.200:9000/" --crawl -l 3

        XSStrike v3.1.5

[~] Crawling the target
------------------------------------------------------------
[+] Vulnerable component: jquery v2.1.4
[!] Component location: http://192.168.127.200:9000/assets/js/jquery-2.1.4.min.js
[!] Total vulnerabilities: 3
[!] Summary: 3rd party CORS request may execute
[!] Severity: medium
[!] CVE: CVE-2015-9251
[!] Summary: jQuery before 3.4.0, as used in Drupal, Backdrop CMS, and other products, mishandles jQuery.extend(true, {}, ...) because of Object.prototype pollution
[!] Severity: low
[!] CVE: CVE-2019-11358
[!] Summary: parseHTML() executes scripts in event handlers
[!] Severity: medium
[!] CVE: CVE-2015-9251
------------------------------------------------------------
------------------------------------------------------------
[+] Vulnerable component: bootstrap v3.3.6
[!] Component location: http://192.168.127.200:9000/assets/js/bootstrap.min.js
[!] Total vulnerabilities: 4
[!] Summary: XSS in data-template, data-content and data-title properties of tooltip/popover
[!] Severity: high
[!] CVE: CVE-2019-8331
[!] Summary: XSS in data-container property of tooltip
[!] Severity: medium
[!] CVE: CVE-2018-14042
[!] Summary: XSS in collapse data-parent attribute
[!] Severity: medium
[!] CVE: CVE-2018-14040
[!] Summary: XSS in data-target property of scrollspy
[!] Severity: medium
[!] CVE: CVE-2018-14041
------------------------------------------------------------
------------------------------------------------------------
[+] Vulnerable component: jquery v1.11.3
[!] Component location: http://192.168.127.200:9000/assets/js/jquery-1.11.3.min.js
[!] Total vulnerabilities: 3
[!] Summary: 3rd party CORS request may execute
[!] Severity: medium
[!] CVE: CVE-2015-9251
[!] Summary: jQuery before 3.4.0, as used in Drupal, Backdrop CMS, and other products, mishandles jQuery.extend(true, {}, ...) because of Object.prototype pollution
[!] Severity: low
[!] CVE: CVE-2019-11358
[!] Summary: parseHTML() executes scripts in event handlers
[!] Severity: medium
[!] CVE: CVE-2015-9251
------------------------------------------------------------
[+] Potentially vulnerable objects found at http://192.168.127.200:9000/
------------------------------------------------------------
2   if('ontouchstart' in document.documentElement) document.write("<script src='assets/js/jquery.mobile.custom.min.js'>"+"<"+"/script>");
------------------------------------------------------------
[+] Potentially vulnerable objects found at http://192.168.127.200:9000/vul/xxe/xxe_1.php
------------------------------------------------------------
2   if('ontouchstart' in document.documentElement) document.write("<script src='../../assets/js/jquery.mobile.custom.min.js'>"+"<"+"/script>");
------------------------------------------------------------
[+] Potentially vulnerable objects found at http://192.168.127.200:9000/vul/xss/xss_dom_x.php
------------------------------------------------------------
3   var str = window.location.search;
4   var txss = decodeURIComponent(str.split("text=")[1]);
5   var xss = txss.replace(/\+/g,' ');
6   //                        alert(xss);
8   document.getElementById("dom").innerHTML = "<a href='"+xss+"'>就让往事都随风,都随风吧</a>";
10  //试试：'><img src="#" onmouseover="alert('xss')">
Potentially vulnerable objects found at http://192.168.127.200:9000/vul/xss/xsspost/post_login.php
11  //试试：' onclick="alert('xss')">,闭合掉就行
2   if('ontouchstart' in document.documentElement) document.write("<script src='../../assets/js/jquery.mobile.custom.min.js'>"+"<"+"/script>");
------------------------------------------------------------
------------------------------------------------------------
2   if('ontouchstart' in document.documentElement) document.write("<script src='../../../assets/js/jquery.mobile.custom.min.js'>"+"<"+"/script>");
------------------------------------------------------------
[+] Potentially vulnerable objects found at http://192.168.127.200:9000/vul/xss/xss_dom.php
------------------------------------------------------------
4   document.getElementById("dom").innerHTML = "<a href='"+str+"'>what do you see?</a>";
2   if('ontouchstart' in document.documentElement) document.write("<script src='../../assets/js/jquery.mobile.custom.min.js'>"+"<"+"/script>");
------------------------------------------------------------
 !] Progress: 226/226ct.php
```

***
DVWA 的扫描结果如下，设置的扫描深度为三级，也是什么都没扫出来。

```shell
┌──(kali㉿kali)-[~/XSStrike]
└─$ python3 xsstrike.py -u "http://192.168.127.200/DVWA/vulnerabilities/" --crawl -l 3 --skip-dom

        XSStrike v3.1.5

[~] Crawling the target
------------------------------------------------------------
[+] Vulnerable component: jquery v1.10.2
[!] Component location: http://code.jquery.com/jquery-1.10.2.min.js
[!] Total vulnerabilities: 3
[!] Summary: jQuery before 3.4.0, as used in Drupal, Backdrop CMS, and other products, mishandles jQuery.extend(true, {}, ...) because of Object.prototype pollution
[!] Severity: low
[!] CVE: CVE-2019-11358
[!] Summary: 3rd party CORS request may execute
[!] Severity: medium
[!] CVE: CVE-2015-9251
[!] Summary: parseHTML() executes scripts in event handlers
[!] Severity: medium
[!] CVE: CVE-2015-9251
------------------------------------------------------------
 !] Progress: 52/52y.php
```

#### WAF 检测

作者说 strike 的 WAF 指纹是借鉴的 `sqlmap`，等以后有空了可以对比一下这两个软件的检测准确率，但是俺也没有 WAF 环境呀，只能说是下次一定，这次就当是熟悉一下感情吧。但是看博客说这个功能还能检测过滤器，那我给大伙试试。

拿 high 难度的 DVWA 反射型XSS 测试一下

```shell
┌──(kali㉿kali)-[~/XSStrike]
└─$ python3 xsstrike.py -u "http://192.168.127.200/DVWA/vulnerabilities/xss_r/?name=XSStrike#" --fuzzer

        XSStrike v3.1.5

[+] WAF Status: Offline
[!] Fuzzing parameter: name
[!] [filtered] <test
[!] [filtered] <test//
[!] [filtered] <test>
[!] [filtered] <test x>
[!] [filtered] <test x=y
[!] [filtered] <test x=y//
[!] [filtered] <test/oNxX=yYy//
[!] [filtered] <test oNxX=yYy>
[!] [filtered] <test onload=x
[!] [filtered] <test/o%00nload=x
[!] [filtered] <test sRc=xxx
[!] [filtered] <test data=asa
[!] [filtered] <test data=javascript:asa
[!] [filtered] <svg x=y>
[!] [filtered] <details x=y//
[!] [filtered] <a href=x//
[!] [filtered] <emBed x=y>
[!] [filtered] <object x=y//
[!] [filtered] <bGsOund sRc=x>
[!] [filtered] <iSinDEx x=y//
[!] [filtered] <aUdio x=y>
[!] [filtered] <script x=y>
[!] [filtered] <script//src=//
[!] [filtered] ">payload<br/attr="
[!] [filtered] "-confirm``-"
[!] [filtered] <test ONdBlcLicK=x>
[!] [filtered] <test/oNcoNTeXtMenU=x>
[!] [filtered] <test OndRAgOvEr=x>
```

有点给我整不会了，WAF 检测是 offline 没错，但是检测参数的过滤器的时候可以看出 strike 可能是输入了这些 payload 检测，但是检测结果你起码要给一个吧。。。


## Noxss

Noxss 是一个高效的 XSS 扫描器，这里贴一下官方文档的特性翻译：

* 快速，适合测试数以百万计的url
* 支持基于dom的xss(使用Chrome或Phantomjs)和反射xss
* 现在根据注射位置只使用8个有效载荷(不模糊，更准确，更快)
* 异步请求(使用gevent)和多处理
* 支持单个url，文件和来自Burpsuite的流量
* 基于接口的流量过滤
* 支持特殊报头(如referer,cookie，自定义令牌等)
* 支持通过id快速重新扫描

### 安装
>版本: v1.0-beta

装这个东西确实挺费劲的，可以说是一波三折。

```text
Ubuntu
1.apt-get install flex bison phantomjs
2.pip install -r requirements.txt
Centos
1.yum install flex bison phantomjs
2.pip install -r requirements.txt
MacOS
1.brew install grep findutils flex phantomjs
2.pip install -r requirements.txt
```

这是官方给的安装文档，我用的是 kali，跟 ubuntu 一样是 debian 的，所以我用的是 ubuntu 的安装方式。

*** 
*首先是第一个问题*

![](../../image/Pasted%20image%2020230427185259.png)

这个软件 apt 下载不了了，但是看文档这个软件应该是一个浏览器，官方文档说还支持 chrome，所以我就直接下了 chrome，应该问题不大。

接下来是第二个问题，软件依赖的是 python2.7，我用的kali 默认python 就是指向 3.10，搜了一下怎么切换版本，这个好解决。

这篇文章说的很详细，[Linux切换python版本 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/343538387)

简单的贴一下命令

```shell
# 1.查看有没有你需要的 python 版本的配置信息，没有的话就第二步添加
update-alternatives --display python

# 2.建立一个 python 组，并添加两个 python 的版本
sudo update-alternatives --install /usr/bin/python python /usr/bin/python2.7 2
sudo update-alternatives --install /usr/bin/python python /usr/bin/python3.6 1

# 3.切换版本
sudo update-alternatives --config python
```

个人建议，python 命令就用 pyhton2 的版本，python3 这个命令就用 python3 的版本，这样就不用切来切去的了。

***
*接下来是第二个问题*

`pip install -r requirements.txt` 这条命令报错，大意就是安装 gevent 这个依赖的时候出错了，但是跟 pip 没有什么关系。在网上搜了一下，发现应该是 python 版本切过来了，但是 pip 没有使用对应版本的 python，这个搜了一下也解决了，解决方式是修改 pip 的源文件。

/usr/bin/pip，把第一行的 python3.10 修改成你要使用的那个版本，我改成了 python2.7

```python
#!/usr/bin/python3.10
# -*- coding: utf-8 -*-
import re
import sys
from pip._internal.cli.main import main
if __name__ == "__main__":
    sys.argv[0] = re.sub(r"(-script\.pyw|\.exe)?$", "", sys.argv[0])
    sys.exit(main())
```

然后用 `pip --version ` 看成功没有，发现报错了，报错信息忘了记录，搜了一下原来是我的 python2.7 下面没有安装 pip。用下面这两条命令安装了

```shell
curl https://bootstrap.pypa.io/get-pip.py -o get-pip.py
python2.7 get-pip.py
```

解决。

***
*最后是第三个问题*

pip 版本切过来之后执行 `pip install -r requirements.txt` 安装依赖，这次安装成功了，全称没报错，于是使用 `python start.py --check` 这条命令检查安装成功没有。

又报错了，报错信息如下：

```shell
ValueError: greenlet.greenlet size changed, may indicate binary incompatibility. Expected 128 from C header, got 40 from PyObject
```

搜了一下，发现应该是 greenlet 这个依赖的版本问题。

![](../../image/Pasted%20image%2020230427191227.png)

执行 `pip install greenlet==0.4.16` 命令，切成低版本。

总算打开了，但还是有报错，意思是没有装浏览器的插件，但这个应该是用来检测 Dom XSS 的，应该问题不大。 

```shell
┌──(kali㉿kali)-[~/NoXss]
└─$ python start.py --check
/***
 *     _______           ____  ___
 *     \      \   ____   \   \/  /  ______ ______
 *     /   |   \ /  _ \   \     /  /  ___//  ___/
 *    /    |    (  <_> )  /     \  \___ \ \___ \
 *    \____|__  /\____/  /___/\  \/______>______>
 *            \/               \_/
 *                                                     #v1.0-beta
 */

[18:45:34] [INFO] Message: 'chromedriver' executable needs to be in PATH. Please see https://sites.google.com/a/chromium.org/chromedriver/home

/home/kali/.local/lib/python2.7/site-packages/selenium/webdriver/phantomjs/webdriver.py:49: UserWarning: Selenium support for PhantomJS has been deprecated, please use headless versions of Chrome or Firefox instead
  warnings.warn('Selenium support for PhantomJS has been deprecated, please use headless '
[18:45:34] [INFO] Message: 'phantomjs' executable needs to be in PATH.

[18:45:34] [WARNING] No browser is installed correctly!
```

![](../../image/Pasted%20image%2020230427192949.png)

