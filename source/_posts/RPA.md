---
title: 使用TagUI实现RPA自动化流程
date: 2024-10-07 23:00:00
categories: WEB开发
tags:
  - RPA
index_img:
banner_img:
excerpt: "学习TagUI工具并实现基于RPA技术的发票数据自动化处理流程 《智能信息系统建模》课程作业"
---

《智能信息系统建模》课程作为《软工原理实践》的 plus 版 上课讲的东西一点用没有 只有大作业稍微能学到点东西

作为第一个大作业 我们将学习使用 TagUI 这个开源工具实现简单的 RPA 自动化流程 此博客作为学习记录

# TagUI

## 环境安装

第一步安装就遇到问题 尝试了很多次后下面是解决办法

1. 下载[旧版 TagUI 压缩包](https://github.com/kelaberetiv/TagUI/archive/master.zip) 然后去 github 下载[最新版 TagUI 压缩包](https://github.com/aisingapore/TagUI/releases/download/v6.110.0/TagUI_Windows.zip)

2. 将旧版压缩包解压后放到合适路径 并把目录下的 src 文件夹添加到环境变量 path 里

3. 把新版压缩包解压后 找到 src 文件夹下 unx、casperjs、phantomjs 几个文件夹复制到旧版的 src 中 另外记得安装一个 php

4. 然后应该就可以正常使用了 使用`tagui .\flows\samples\2_github.tag`测试一下即可

## 学习使用

### 启动参数

- `-d`会以创建 cmd 脚本的形式独立启动 `-h`会以不开启浏览器的形式启动

### 基础指令

- click：单击左键或右键 使用 html 的标记来定位元素 或者坐标点 或者也可以识别图像
- web：直接输入 url 即可访问
- type：进行文本输入
- read：读取元素的文本并存储给变量
- assign：给变量进行赋值

### 元素的定位

- DOM：使用 id、class 等 可能会有重复
- XPath：使用树状结构进行定位
- 坐标定位
- Image：图像识别定位

### 基本语法

- IF
- FOR
- functions：见http://www.tagui.com.cn/reference.html#helper-functions-reference
