---
title: "一个Golang RSS内容匹配器"
date: 2023-02-07T12:01:09+08:00
draft: false
categories:
    - golang
tags:
    - go-in-action
    - demo 
---
> go-in-action里的一个demo，给定需要匹配的字符串，在搜索源（如rss）中批量查询最新文章，返回结果（如相匹配的title和description）
> 
> 关键字：`goroutine并发`，`json/xml反序列化`，`网络请求` 
> 
> 项目地址：https://github.com/chajiuqqq/go-rss-matcher

效果：搜索“China”，发现了近期和China相关的文章，如中国气球误入美国:

    $ ./rss-matcher
        2023/02/04 12:04:33 title
        China says balloon spotted over U.S. is a 'civilian airship' that blew astray
        2023/02/04 12:04:33 description
        The State Department announced Secretary of State Antony Blinken will not go ahead with a planned trip to China, after the surveillance balloon was detected over U.S. airspace Thursday.
        2023/02/04 12:04:33 title
        Tensions continue to increase between the United States and China
        2023/02/04 12:04:34 HTTP Response Error: http://www.npr.org/rss/rss.php?id=43 code 404
        2023/02/04 12:04:34 HTTP Response Error: http://www.npr.org/rss/rss.php?id=1021 code 404
        2023/02/04 12:04:34 title
        Perspective: Jiang Zemin's passing marks the end of an era for China
        2023/02/04 12:04:34 description
        As China holds a memorial service for its late leader Jiang Zemin, an NPR correspondent who met Jiang reflects on the figure and his transforming country.
        2023/02/04 12:04:34 title
        Blinken postpones China trip after discovery of surveillance balloon
        2023/02/04 12:04:34 description
        Secretary of State Antony Blinken has postponed his trip to China after the discovery of what the Pentagon alleges to be a Chinese surveillance balloon. China's government says it's a weather balloon.
        2023/02/04 12:04:34 title
        China says balloon spotted over U.S. is a 'civilian airship' that blew astray
        2023/02/04 12:04:34 description
        The State Department announced Secretary of State Antony Blinken will not go ahead with a planned trip to China, after the surveillance balloon was detected over U.S. airspace Thursday.
        2023/02/04 12:04:34 description
        China's foreign ministry described the balloon as "a civilian airship" for meteorological research that had blown far off course by winds. The Pentagon suspects it's collecting sensitive information.

项目结构设计上支持多种搜索源，如rss等，只要设计对应的matcher就可以进行搜索匹配。

程序流程如下：

1. 读取data.json里的搜索源（feeds），并反序列化为对象供后面访问
2. 给每个feed寻找已注册的matcher匹配器，并开启goroutine匹配。匹配结果写入通道。
   1. 匹配过程首先请求feed的URL，获得XML
   2. 解析xml，反序列化到对象
   3. 对对象里的字段进行匹配
3. 循环读取通道内的结果，并打印
4. 如果匹配结束，关闭通道，主程序结束

几个设计上的亮点：

- 设计Matcher接口，所有的matcher都可以调用Search进行搜索
- 设计matcherMap，不同类型的Matcher注册后保存在这里，方便不同数据源匹配和数据源类型拓展
- 每个goroutine执行一个feed的搜索，并发执行，提高性能
- 查询结果后，goroutine通过无缓冲chan把数据发送主线程展示，同步获取并展示，利用了语言特性