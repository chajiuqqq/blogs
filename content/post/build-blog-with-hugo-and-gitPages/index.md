---
title: "Hugo + GitPages博客搭建指南"
date: 2022-08-24T16:32:10+08:00
draft: true
categories:
    - 技术
---

应该算是第一篇比较正式的blog吧，前几天把bolg搭起来了，今天又实现了持续集成，可以实现一个仓库保存blogs和网站页面，并在更新的时候自动构建+部署网站。这种构建方式最大的好处是免费+资料掌握+自动部署，那下面就来介绍一下怎么实现。

## 使用流程

在多台设备上可clone一个仓库，手动在content/post下新建md文件攥写文章，完成后git提交并推送远程仓库，远程仓库会自动构建网站，用户等一会就可以看到网站更新了。编写环境只要有git即可完成编写+部署流程。

## 搭建逻辑

博客可以是静态的，也可以是动态的。静态的就是一堆由HTML和CSS、JS组成的文件直接被浏览器访问，没有后台程序提供业务支持，好处是不吃资源，访问只受网络带宽限制，坏处是没法做业务处理了；动态指的是由前端+后端组合支持的网站，好处是可以做自由的业务和管理，坏处是部署麻烦，对资源需求大，并发访问受到服务器性能限制。

为了便于维护，本文搭建的是静态网站。
![网站首页](site.png)

采用技术是
- Markdown：文章编写语法
- Hugo：静态网站编译工具
- GitHub Page：网页托管
- GitHub Action：自动构建服务

Hugo的使用逻辑是，我们编写markdown格式文章，并附加某种格式的数据，写好后交给Hugo工具“构建”生成静态的HTML和CSS、JS组成的文件，这些文件统一放在public文件夹下，此时部署public就可以看到你的网站了。其中一个缺点是每次需要Hugo的环境来“构建”

当结合Github Action时，可以把这个构建过程交给云端处理。每次主分支推送时都会触发这个“构建”，同时把public下的文件推送到`gh-page`分支，利用Github Page挂载你的网站，这样网站就更新好并可以浏览了。

## 搭建步骤

### 1、安装hugo


### 2、添加仓库

### 3、设置GitHub Page

### 4、设置Github Action

官方文档里有[说明](https://gohugo.io/hosting-and-deployment/hosting-on-github/#build-hugo-with-github-action),这里概述一下：

1. 在你的项目下创建`.github/workflows/gh-pages.yml`这个路径的文件
2. 复制这一段进去,注意下面两个`main`的地方，如果你用`master`，则改成`master`

```
name: github pages

on:
  push:
    branches:
      - main  # Set a branch to deploy
  pull_request:

jobs:
  deploy:
    runs-on: ubuntu-20.04
    steps:
      - uses: actions/checkout@v2
        with:
          submodules: true  # Fetch Hugo themes (true OR recursive)
          fetch-depth: 0    # Fetch all history for .GitInfo and .Lastmod

      - name: Setup Hugo
        uses: peaceiris/actions-hugo@v2
        with:
          hugo-version: 'latest'
          # extended: true

      - name: Build
        run: hugo --minify

      - name: Deploy
        uses: peaceiris/actions-gh-pages@v3
        if: github.ref == 'refs/heads/main'
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_dir: ./public
```

### 5、测试一下