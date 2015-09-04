---
layout: post
title:  "用locust+selenium压力测试登录后的页面"
date:   2015-09-04 10:00:00
categories: python
tags: python locust selenium phantomjs
author: "zhuhangyu"
---

接着[上一篇](/mysql/last-query-cost)，已经把普通浏览页面压测完毕。

现在的问题是，需要压测登录后的页面。

上次没提到用来做压测的工具是`locust`，这是一个以前同事彪哥推荐的，感谢他，专业的大师级自动化测试高手。`locust`的主页在[locust.io](http://locust.io/)，是一个基于python开发的压测工具，使用比较简单。[文档请点这里](http://docs.locust.io/en/latest/quickstart.html)

文档中说的很清楚：`The HttpSession instance will preserve cookies between requests so that it can be used to log in to websites and keep a session between requests.`

代码示例片段：

```python
from locust import HttpLocust, TaskSet, task

class UserBehavior(TaskSet):
    def on_start(self):
        """ on_start is called when a Locust start before any task is scheduled """
        self.login()

    def login(self):
        self.client.post("/login", {"username":"ellen_key", "password":"education"})

    @task(2)
    def index(self):
        self.client.get("/")

    @task(1)
    def profile(self):
        self.client.get("/profile")

class WebsiteUser(HttpLocust):
    task_set = UserBehavior
    min_wait=5000
    max_wait=9000
```

比如说上面就是在locust启动的时候先进行登录操作，对表单中的url：`/login`进行post提交，传入username和password的值。然后就可以请求登录后的页面`/profile`

但是，这只限于简单的表单，现在基本上大部分网站的登录方式都不简单，所以根本行不通。

对于我们这个网站的登录，基于之前我学习到的分析网站登录步骤，用浏览器f12观察过程，大概方式如下：

* 当client填完用户名密码后，点登录按钮，会先去get一个url，得到一个public key，同一个账号获得的key是不变的(不同的账号的key是不是不同，我倒没试过)
* 然后client对这个key加密了，而且每次加密后的内容不同，接着post一个url，这时候server应该是用本身的private key解密了刚才client post过来的key，然后存进session
* 然后server又把已经解密的key再次用别的方式加密了，响应给client
* 然后client把得到的key解密，再次post了另一个url，我估计在这里server会去判断两次key是不是匹配，没问题的话就登录成功了


但问题是：

* 一般表单提交都有个post url，但是我找不到，估计是我水平不行，汗。
* 信息也加密了，而且同一个账号每次登录，加密内容都会变。
* 我跑去问开发人员，他说用的是第三方的开源包实现的，具体怎么加密解密要去看源码了，我看不懂，再次汗。

那么，我就想，还是采用headless browser去模拟登录算了，关键在于要把cookie传给locust。

最终代码如下：

```python
# -*- coding: utf-8 -*-
from locust import HttpLocust, TaskSet, task
from selenium import webdriver
import time

class userLogin(TaskSet):
    cookiejar = {}

    def on_start(self):
        driver = webdriver.PhantomJS()
        driver.get("http://www.xxx.com/login")
        time.sleep(2)
        u = driver.find_element_by_id("username")
        time.sleep(2)
        u.clear()
        u.send_keys("myuser")
        p = driver.find_element_by_id("password")
        time.sleep(2)
        p.clear()
        p.send_keys("mypass")
        driver.find_element_by_id("submit").click()
        time.sleep(2)
        for cookie in driver.get_cookies():
            self.cookiejar[cookie['name']] = cookie['value']

    @task
    def userCenter(self):
        self.client.request('get', "/profile", cookies=self.cookiejar)

class WebsiteUser(HttpLocust):
    task_set = userLogin
    min_wait = 5000
    max_wait = 9000
```

说明：

* driver登录后，使用`get_cookies`方法获取cookies，返回类型是list of dictionaries
* locust的get方法不能带cookie，那就改用request方法，传入cookie，这个cookie的类型要求必须是dict或者cookiejar对象
* 所以我们只需要把driver获取到的cookies取出每个cookie的name的值和value的值，然后组成一个dict

所有压测代码写完后，以后该考虑自动化压测了，继续努力吧。

**补充：这个方法如果模拟用户较多，会导致PhantomJS非常占用cpu，按网上说的：传入`service_args = ['--load-images=false', '--disk-cache=true']`，效果也很不理想。虽然实现了功能，但最后证实不是一个好办法:(**