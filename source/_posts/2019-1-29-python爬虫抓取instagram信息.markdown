---
title:      "python爬虫抓取instagram信息"
description:   "python模拟网页操作和模拟网络请求实践"
date:       2019-1-29 12:00:00
author:     "安地"
tags:
      - python
---


## 介绍
这是帮朋友弄的一个小东西，之前用python只有模拟网络请求的经验，但觉得这种需求应该不难，就答应做了。

## 开始

获取ins的很多信息需要先登录，google相关的东西基本都是抓取图片的，可用性不大。
想了两种方案，一种抓请求获取cookie再模拟获取评论的请求，直接获取到数据；另一种是抓取网页的方法，全模拟网页操作，抓取网页信息。
第一种很熟悉，所以我选择第二种入手。

## 模拟网页操作获取信息

### 模拟登录

网页上模拟点击我用的selenium，通过find_element_by_xpath等类方法获取到需要的元素，然后模拟操作。

```python
import io
import sys
from selenium import webdriver
from bs4 import BeautifulSoup
import time
from time import sleep

from selenium.webdriver import Firefox
from selenium.webdriver.common.by import By
from selenium.webdriver.common.keys import Keys
from selenium.webdriver.firefox.options import Options
from selenium.webdriver.support import expected_conditions as expected
from selenium.webdriver.support.wait import WebDriverWait

login_url = "https://www.instagram.com/accounts/login/?source=auth_switcher"

def login(user):
    options = Options()
    options.add_argument('-headless')  # 无头参数
    driver = Firefox(firefox_options=options)  # 配了环境变量第一个参数就可以省了，不然传绝对路径
    wait = WebDriverWait(driver, timeout=10)

    driver.get(login_url)
    driver.implicitly_wait(0.5)

    sleep(2)
    print(driver)

    account = driver.find_element_by_xpath("//input[@name='username']")
    account.clear()
    account.send_keys(user.get_account())
    print("账号输入完成!"+user.get_account())

    passwd = driver.find_element_by_xpath("//input[@name='password']")
    passwd.clear()
    passwd.send_keys(user.get_password())
    print("密码输入完成!")

    print (driver.title)
    print driver.current_url

    button = driver.find_element_by_xpath("//button[@type='submit']")
    button.click()
    print("开始登陆!")

    sleep(2)

    print (driver.title)
    print driver.current_url
```

登录成功后driver的title会发生变化，执行操作后需要等待几秒，模拟网页操作比较慢，这个就类似有一个真的网页给你操作，只是没有显示界面而已。

### 获取关注用户

在上一步之后继续操作，进去周杰伦的主页，然后再点击关注者，find_element_by_xpath可以通过标签属性来找到元素。
根据网页上的信息找出规则，然后就可以拿到想要的数据了。代码如下：

```python
    driver.get("https://www.instagram.com/jaychou/")
    sleep(2)
    print (driver.title)
    print driver.current_url

    button = driver.find_element_by_xpath("//a[@href='/jaychou/followers/']")
    button.click()
    sleep(2)
    print (driver.title)
    print driver.current_url

    li = driver.find_elements_by_tag_name("li")

    f = open('f.txt','a')
    for i in li:
        try:
            a = i.find_element_by_tag_name("a")
            herf =  a.get_attribute("href")
            print herf

            datepat = re.compile('^https://www.instagram.com/[^/]+/$')
            if datepat.match(herf):
                print "add"
                f.write(herf)
                f.write('\n')
            else:
                print "filter"
        except:
            print "exception"
            print i
    f.close()
```

这里拿到list后，然后遍历节点，过滤出我们要的用户名后，写到文件里。

## 模拟请求获取数据

尝试获取评论里的用户，获取评论信息在一个小的弹窗里，一页只显示几十条条数据，再获取更多需要模拟弹窗的滑动，没有找到对应的方法。想想这里还是用模拟请求来获取数据。

### 获取cookie

在前面的模拟网页登录后后可以获取cookie信息

```python
    cookie = [item["name"] + "=" + item["value"] for item in driver.get_cookies()]
    cookiestr = ';'.join(item for item in cookie)
    print cookiestr
```
另外也可以通过抓包获取cookie

### 分析评论请求

F12网页开发模式下，进行操作请求后可以在network里看到相关请求，找到获取评论列表的请求，url如下：
https://www.instagram.com/graphql/query/?query_hash=2cc8bfb89429345060d1212147913582&variables=%7B%22shortcode%22%3A%22BrCsJRUH2kV%22%2C%22child_comment_count%22%3A3%2C%22fetch_comment_count%22%3A40%2C%22parent_comment_count%22%3A24%2C%22has_threaded_comments%22%3Afalse%7D
解析出来就是：
https://www.instagram.com/graphql/query/?query_hash=2cc8bfb89429345060d1212147913582&variables={"shortcode":"{BrCsJRUH2kV}","child_comment_count":3,"fetch_comment_count":40,"parent_comment_count":24,"has_threaded_comments":false}

在对比获取其它帖子的评论知道变化的信息只有shortcode，知道了根据shortcode就可以获取到帖子的评论详情，fetch_comment_count可以改变获取评论的数量。

shortcode很容易在网页信息里找到，这样我们就能批量快速获取评论信息了。

### 模拟获取评论请求

```python
import urllib2
import json

cookiestr = "rur=FTW;mid=XD3cGQAEAAGlLkr5xZhGKXK6kdRC;mcd=3;sessionid=9961092202%3ATHPRPTOYnzHmKp%3A11;ds_user_id=9961092202;csrftoken=mx3RYvRa54z842Br8qtWwMkv40sJ7Gvn;urlgen=\"{\"43.224.245.181\": 63855}:1gjOVb:CxD9xU20ZPDIJm6pEBP8db7WXnU\""
url = "https://www.instagram.com/graphql/query/?query_hash=2cc8bfb89429345060d1212147913582&variables=%7B%22shortcode%22%3A%22{shortcode}%22%2C%22child_comment_count%22%3A3%2C%22fetch_comment_count%22%3A40%2C%22parent_comment_count%22%3A24%2C%22has_threaded_comments%22%3Afalse%7D"
def getComments(shortcode):
    commetUrl = url.format(shortcode=shortcode)
    headers = {'cookie':cookiestr}
    req = urllib2.Request(commetUrl, headers = headers)
    try:
        response = urllib2.urlopen(req)
        text = response.read()
        jsonobj = json.loads(text)
        print text
        #getData(jsonobj)
    except:
        print "except"
getComments("BrCsJRUH2kV")
```

把前面获取到的cookiestr保存复制过来，请求的url的shortcode用{}包进占位符，然后就可以根据shortcode获取到评论数据了。

### 解析并保存数据

把前面的getData方法注释打开，getData方法如下:
```python
def getData(jsonobj):
    comments = jsonobj["data"]["shortcode_media"]["edge_media_to_comment"]["edges"]

    f = open('commentsUser.txt','a')
    for comm in comments:
        user = comm["node"]["owner"]
        name = user["username"]
        id = user["id"]
        f.write(name+":"+id)
        f.write('\n')
        print name
    f.close()
```
根据数据去解析json，然后把用户名和id保存在文件里


## 总结

做爬虫在于分析网页信息，模拟网页操作理论上是万能的，模拟请求就会被各种限制，但模拟请求更高效快捷，如果能破解还是用模拟请求方便。