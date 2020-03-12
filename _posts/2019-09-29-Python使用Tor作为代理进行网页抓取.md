---
title: Python使用Tor作为代理进行网页抓取
date: 2019-09-13
categories: [网络架构]
tags: [Python, Tor]
seo:
  date_modified: 2020-03-12 12:37:36 +0800
---

## 前言
---

### 为什么要用代理
在网络抓取的过程中，我们经常会遇见很多网站采取了防爬取技术，或者说因为自己采集网站信息的强度和采集速度太大，给对方服务器带去了太多的压力，所以你一直用同一个代理IP爬取这个网页，很有可能IP会被禁止访问网页，所以基本上做爬虫的都躲不过去IP的问题,需要很多的IP来实现自己IP地址的不停切换，达到正常抓取信息的目的。

### 常用解决办法
使用ip代理池, 使用代理池的代理ip, 隐藏我们的实际ip, 从何起到绕过防爬技术的干扰。<br>
这里顺便推荐一个githup开源项目[https://github.com/jhao104/proxy_pool](https://github.com/jhao104/proxy_pool):该项目通过采集几个常用免费代理网站的代理ip, 构建自己的代理ip池。<br>

**今天我们讲方法不是使用ip代理池, 而是通过Tor(洋葱路由)进行匿名访问目标地址**

## 介绍
---
### 什么是Tor(洋葱路由)
Tor（The Onion Router）是第二代洋葱路由（onion routing）的一种实现。<br>
Tor专门防范流量过滤、嗅探分析，让用户免受其害。Tor在由“onion routers”（洋葱）组成的表层网（overlay network）上进行通信，可以实现匿名对外连接、匿名隐藏服务。<br>

### 代理实现思路
1. 运行tor
2. 在Python中使用Tor作为selenium的代理
3. 对一个目标网站发起请求
4. 重复步骤2和3

**实现代码**
```python
from stem import Signal
from stem.control import Controller
from selenium import webdriver
from selenium.webdriver.chrome.options import Options
from bs4 import BeautifulSoup

# 通过Tor切换ip
def switchIP():
	with Controller.from_port(port = 9051) as controller:
		controller.authenticate()
		controller.signal(Signal.NEWNYM)

# 获取代理的浏览器
def get_browser(PROXY = None):
	chrome_options = webdriver.ChromeOptions()
	if PROXY != None:
		chrome_options.add_argument('--proxy-server=SOCKS5://{0}'.format(PROXY)) # 代理
	chrome_options.add_argument('blink-settings=imagesEnabled=false') #不加载图片, 提升速度
	chrome_options.add_argument('--headless') #浏览器不提供可视化页面.
	executable_path='/Users/fewave/project/python/demo/chromedriver' #设置启动驱动
	return webdriver.Chrome(executable_path=executable_path, options=chrome_options)

def main():

	for x in range(5):
		switchIP()
		browser = get_browser('127.0.0.1:9050')
		browser.get('https://cip.cc')
		html = browser.page_source
		soup = BeautifulSoup(html, 'html.parser')
		print('======第%d次请求=======' % (x+1))
	    print(soup.find_all('pre'))
		browser.quit()


if __name__ == '__main__':
	main()
```

### 准备工作
运行代码前, 还需做一下准备工作:
1. 安装Tor, 因为我的本地电脑为mac, 因此直接通过brew安装 `brew install tor`, 安装完成后启动Tor服务, `brew services start tor`
2. 下载浏览器驱动, 因为我本地使用的Chrome, 因此可到[https://sites.google.com/a/chromium.org/chromedriver/downloads](https://sites.google.com/a/chromium.org/chromedriver/downloads)(需翻墙) 下载对应版本的驱动(驱动版本需与本机浏览器的版本对应)
3. 下载python依赖, 可执行命令`pip install selenium stem bs4`
4. 更新torrc文件并重新启动Tor，以便可以向Tor控制器发出请求。在mac上，您可以在`/usr/local/etc/tor`中找到torrc.sample文件。通过执行`mv`命令,将torrc.sample重命名为torrc
```
mv /usr/local/etc/tor/torrc.sample /usr/local/etc/tor/torrc
```
并且将torrc文件中的以下两行取消注释
```
ControlPort 9051
CookieAuthentication 1
```
重启Tor
```
brew services restart tor
```
### 代码介绍

```python
# 通过Tor切换ip
def switchIP():
	with Controller.from_port(port = 9051) as controller:
		controller.authenticate()
		controller.signal(Signal.NEWNYM)
```
这个方法让我们切换IP。它向Tor控制器端口发出一个信号(Signal.NEWNYM)，这告诉Tor我们需要一个新的电路来路由流量。这将给我们一个新的exit节点，这意味着我们的流量看起来像是来自另一个IP。

```python
# 获取代理的浏览器
def get_browser(PROXY = None):
	chrome_options = webdriver.ChromeOptions()
	if PROXY != None:
		chrome_options.add_argument('--proxy-server=SOCKS5://{0}'.format(PROXY)) # 代理
	chrome_options.add_argument('blink-settings=imagesEnabled=false') #不加载图片, 提升速度
	chrome_options.add_argument('--headless') #浏览器不提供可视化页面.
	executable_path='/Users/fewave/project/python/demo/chromedriver' #设置启动驱动
	return webdriver.Chrome(executable_path=executable_path, options=chrome_options)
```
该方法将selenium webdriver设置为在无可数化模式下使用Chrome浏览器，并使用Tor作为代理路由我们的请求。这确保了所有对selenium webdriver的请求都经过Tor。

```python
def main():
	print('开始程序')
	for x in range(5):
		switchIP()
		browser = get_browser('127.0.0.1:9050')
		browser.get('https://cip.cc')
		html = browser.page_source
		soup = BeautifulSoup(html, 'html.parser')
		print('======第%d次请求=======' % (x+1))
		print(soup.find_all('pre'))
		browser.quit()
```

最后这段代码只向[https://cip.cc](https://cip.cc)发送一个请求，这样我们就可以通过selenium webdriver检查请求的IP。打印出代理后的ip
Stem 是基于 Tor 的 Python 控制器库，可以使用 Tor 的控制协议来对 Tor 进程进行脚本处理或者构建。

### 执行结果
```
======第1次请求=======
IP	: 23.129.64.187
地址	: 美国  华盛顿州  西雅图
运营商	: emeraldonion.org

数据二	: 美国

数据三	: 美国

URL	: http://www.cip.cc/23.129.64.187

======第2次请求=======
IP	: 109.70.100.20
地址	: 奥地利  奥地利

数据二	: 奥地利

数据三	: 奥地利

URL	: http://www.cip.cc/109.70.100.20

======第3次请求=======
IP	: 185.220.101.5
地址	: 荷兰  北荷兰省  阿姆斯特丹
运营商	: torservers.net

数据二	: 荷兰

数据三	: 德国

URL	: http://www.cip.cc/185.220.101.5

======第4次请求=======
IP	: 23.129.64.194
地址	: 美国  华盛顿州  西雅图
运营商	: emeraldonion.org

数据二	: 美国

数据三	: 美国

URL	: http://www.cip.cc/23.129.64.194

======第5次请求=======
IP	: 162.244.81.196
地址	: 美国  纽约州  纽约
运营商	: serverroom.net

数据二	: 美国

数据三	: 美国纽约纽约

URL	: http://www.cip.cc/162.244.81.196

```
很明显我们的真实ip已经被隐藏了

## 总结
---
上述代码通过启动浏览器驱动， 通过浏览器驱动代理Tor, 从而隐藏我们的真实ip。 不过驱动的启动比较慢， 频繁的驱动重启会让网页的爬取效率大打折扣。因此使用上述方法时， 应该尽量减少浏览器驱动的重启次数。<br>

**ps:**
Selenium: 自动化测试工具。它支持各种浏览器，包括 Chrome，Safari，Firefox 等主流界面式浏览器，如果你在这些浏览器里面安装一个 Selenium 的插件，那么便可以方便地实现Web界面的测试。换句话说叫 Selenium 支持这些浏览器驱动。<br>
Beautiful Soup: 提供一些简单的、python式的函数用来处理导航、搜索、修改分析树等功能。它是一个工具箱，通过解析文档为用户提供需要抓取的数据，因为简单，所以不需要多少代码就可以写出一个完整的应用程序。<br>
Stem: 是基于 Tor 的 Python 控制器库，可以使用 Tor 的控制协议来对 Tor 进程进行脚本处理或者构建。