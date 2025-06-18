---
title: 优化Ebookstore项目
date: 2024-09-23 14:01:00
categories: 项目经验
tags:
  - 开发
index_img:
banner_img:
excerpt: " "
---

慢慢把学到的新技术栈都应用到 ebook 这个项目里来 记录一下过程

## “记住我”功能

大部分网站或 app 都会实现记住我功能 即一次登录后有很长一段时间可以免登录使用 这个功能无非两种方式实现 基于 session 或基于 token 下面分开讲讲细节

### 基于 session 的实现

- 我们知道 session 的原理就是在服务端存一个键值对的 Map 以此维护用户的状态 把无状态的 http 变为有状态的 那么记住我是怎么实现的呢？无非是作 session 的持久化

- 在服务器端 要持久化 session 可以在 yml 配置里设置 这样 spring 会把 session 持久化存储到硬盘 也就保持了服务端重启也不影响 session

- 在客户端 要持久化 session 需要知道 cookie 的一个属性 即 Expire 它区分了两种 cookie 一种叫会话 cookie 一种叫持久 cookie 会话 cookie 仅存在于浏览器打开时 也就是说 当你关闭浏览器后 会话 cookie 将不会再被浏览器发送给对应的 domain 持久 cookie 则有一个到期时间 到期之前都将有效

- 注意 cookie 里的 expire 属性指明的是浏览器的行为 而非服务器的 它告诉浏览器是否应该发送 cookie 给对应的域名

- 在 application.yml 文件里配置以下参数后 session 就持久化了 做到一次登录长期使用

### 基于 token 的实现

### 浏览器的缓存

- 我们知道一般网站要展示很多与用户信息相关的界面 比如个人主页 用户数据在一次会话内比较固定 我们希望把它缓存到客户端以此减少请求 加快响应 不过 假如存储在 sessionStorage 里 当你换一个页面浏览时 数据就将重置 所以这不是一个好的方法

- 因此我们把用户数据存放在 localStorage 里 这样将永不过期 什么时候清除它呢？自然是在获取数据 出现异常要求你重新登录时清除 这样就完成啦

- sessionStorage 则比较适合一个页面跳转到另外一个页面时的数据传递 比如填写表单时 上一页表单需要把数据保存好再填下一页
