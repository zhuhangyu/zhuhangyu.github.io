---
layout: post
title:  "scrapy学习和尝试：续"
date:   2015-09-02 19:00:00
categories: python
tags: python scrapy selenium xvfb phantomjs
author: "zhuhangyu"
---

接着[上一篇](/python/scrapy)，终于成功抓取淘女郎图片。

首先是登录，采用了[immzz](https://github.com/immzz/zhihu-scrapy)登录知乎的代码，这种方式是使用selenium和xvfb作为headless browser，或者使用selenium和phantomjs作为headless browser，模拟人去登录网站，比之前具体去分析每个登录细节，然后纯粹用代码模拟登录要方便得多。

当然，遇见需要输入验证码的时候，还是需要有台gui的机器，linux桌面或者windows、mac都行，先把网页截屏，然后存入redis，然后在gui的机器上命令行连上redis，显示图片，人工输入验证码。

使用headless browser的另一个好处是二级页面可以把cookie带过去。

接着再说一下之前触发淘宝反爬虫策略的原因：scrapy默认有个设置，CONCURRENT_REQUESTS = 16，我把它设置为2，就不会再触发了。

最终目的是想爬取[淘女郎网站](https://mm.taobao.com/json/request_top_list.htm?page=1)上的每个mm主页(比如：[田媛媛](https://mm.taobao.com/687471686.htm))下的所有展示图片。不过最后涉及到淘宝登录以及触发反爬虫策略，并没有成功。

还有一个改进，自定义了一个中间件，也是从网上获取的，现在找不到出处了，汗。。。

```python
class PhantomJSMiddleware(object):
    # overwrite process request

    def process_request(self, request, spider):

        if request.meta.has_key('PhantomJS'):  # 如果设置了PhantomJS参数，才执行下面的代码
            logging.info('PhantomJS Requesting: '+request.url)
            try:
                driver = request.meta['driver']
                driver.get(request.url)
                content = driver.page_source.encode('utf8')
                url = driver.current_url.encode('utf8')
                return HtmlResponse(url, encoding='utf8',
                                    status=200, body=content)

            except Exception, e:  # 请求异常，当成500错误。交给重试中间件处理
                logging.error('PhantomJS Exception!')
                return HtmlResponse(request.url, encoding='utf8',
                                    status=503, body='')
        # else:
        #     logging.info('Common Requesting: '+request.url)
```

意思就是在登录淘宝后，抓取一级页面和二级页面都使用headless browser，可以传递cookie，保持登录状态；而最终抓取图片的时候，是另一个域名，不需要传递cookie，也无需调用headless browser，这样下载图片速度就很快。

还有一个问题是，在登录出现验证码的时候进行截屏，发现总是乱码，解决方法是：

sudo apt-get install ttf-wqy-microhei  #文泉驿-微米黑
sudo apt-get install ttf-wqy-zenhei  #文泉驿-正黑
sudo apt-get install xfonts-wqy #文泉驿-点阵宋体

或者(我用的是ubuntu 14.04，用的是上面的包，下面的是centos的包，遇到问题的朋友可以试试看)

yum install fonts-chinese

最后，这个爬虫其实是分布式的，可以多台机器一起跑，只需要一个gui机器就行，再次感谢[immzz](https://github.com/immzz/)