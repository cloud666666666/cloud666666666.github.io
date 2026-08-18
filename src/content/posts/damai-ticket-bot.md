---
author: Joker.Yun
pubDatetime: 2026-08-19T12:00:00.000Z
modDatetime: 2026-08-19T12:00:00.000Z
title: 16 星的抢票工具:写给谦友的 Selenium 自动化实战
slug: damai-ticket-bot
featured: false
draft: false
tags:
  - python
  - selenium
  - automation
description: 一个为大麦抢票而生的 Selenium 小工具:扫码登录、Cookie 持久化、轮询抢票。16 个 star 背后是真实的粉丝需求与工程取舍。
---

## 起因

薛之谦"天外来物"巡演开票即秒空。买不到票的谦友不止一个,
所以有了这个 175 行的 Python 脚本——[damai](https://github.com/cloud666666666/damai),
16 个 star,全部来自真实用户。

## 技术方案:Selenium 浏览器自动化

没有走纯协议逆向,选了 Selenium + ChromeDriver:

```python
from selenium import webdriver
from selenium.webdriver.chrome.options import Options
from selenium.webdriver.chrome.service import Service

chrome_options = Options()
service = Service("chromedriver.exe")
self.driver = webdriver.Chrome(service=service, options=chrome_options)
```

**为什么不用 requests 直接打接口?**

| 方案 | 优点 | 缺点 |
|---|---|---|
| 纯协议逆向 | 快(毫秒级) | 登录态/签名/风控频繁变化,维护成本高 |
| **Selenium** | 稳定,不碰加密参数,行为像真人 | 慢(秒级),资源占用高 |

对个人工具来说,"开票时能跑起来"比"理论上快 100 倍"重要。
协议方案每次大麦改版都要重新逆向,Selenium 只要页面结构不大改就能继续用。

## 三个核心设计

### 1. Cookie 持久化:一次扫码,长期复用

扫码登录一次,把 Cookie 序列化到本地,以后直接注入:

```python
# 保存
pickle.dump(self.driver.get_cookies(), open('cookies.pkl', 'wb'))

# 恢复(domain 必须正确,否则是假登录)
for cookie in cookies:
    cookie_dict = {
        'domain': '.damai.cn',
        'name': cookie.get('name'),
        'value': cookie.get('value'),
    }
    self.driver.add_cookie(cookie_dict)
```

坑:`domain` 写错会导致 Cookie 注入后依然未登录——这个注释
(`# 必须要有的, 否则就是假登录`)是踩过之后的记录。

### 2. 轮询等状态:状态机驱动

```python
self.status = 0  # 当前执行到哪一步
```

用 `status` 标记流程进度,每一步用 `while + sleep` 轮询页面标题/元素,
而不是固定 sleep 一堆时间——页面加载快时少等,慢时等够。

### 3. 弹窗处理

开票页会弹"购票须知"等弹窗,用 XPath 检测元素存在性,
存在就点掉再继续——不处理弹窗,后面的按钮点不到。

## 诚实的局限

1. **速度上限**:Selenium 是秒级操作,抢热门场次和毫秒级的协议脚本没法比——它适合"开票后几分钟内还能捡漏"的场次,不是所有场次
2. **反爬风险**:大麦有风控,Selenium 特征明显,使用频率高可能被验证码拦截
3. **维护性**:页面结构一变,XPath 要跟着改

## 写在最后

这个项目的意义不在技术多深,在于**真实场景 + 真实用户**:
16 个 star 的每个都是"我要去看老薛"的谦友。愿世界和平,愿大家都有票。

[项目地址](https://github.com/cloud666666666/damai)
