---
title: 从'一'开始优化Ebookstore
date: 2024-09-23 14:01:00
categories: WEB开发
tags:
  - 前端
  - 后端
  - 部署
index_img:
banner_img:
excerpt: "从大三开学开始 慢慢把学到的各种技术栈都应用到ebook这个项目里来 记录一下"
---

从大三开学开始 慢慢把学到的各种技术栈都应用到 ebook 这个项目里来 记录一下

# 基于 Token 管理用户状态

# “记住我”功能

大部分网站或 app 都会实现记住我功能 即一次登录后有很长一段时间可以免登录使用 这个功能无非两种方式实现 基于 session 或基于 token 下面分开讲讲细节

## 基于 session 的实现

- 我们知道 session 的原理就是在服务端存一个键值对的 Map 以此维护用户的状态 把无状态的 http 变为有状态的 那么记住我是怎么实现的呢？无非是作 session 的持久化

- 在服务器端 要持久化 session 可以在 yml 配置里设置 这样 spring 会把 session 持久化存储到硬盘 也就保持了服务端重启也不影响 session

- 在客户端 要持久化 session 需要知道 cookie 的一个属性 即 Expire 它区分了两种 cookie 一种叫会话 cookie 一种叫持久 cookie 会话 cookie 仅存在于浏览器打开时 也就是说 当你关闭浏览器后 会话 cookie 将不会再被浏览器发送给对应的 domain 持久 cookie 则有一个到期时间 到期之前都将有效

- 注意 cookie 里的 expire 属性指明的是浏览器的行为 而非服务器的 它告诉浏览器是否应该发送 cookie 给对应的域名

- 在 application.yml 文件里配置以下参数后 session 就持久化了 做到一次登录长期使用

## 基于 token 的实现

## 浏览器的缓存

- 我们知道一般网站要展示很多与用户信息相关的界面 比如个人主页 用户数据在一次会话内比较固定 我们希望把它缓存到客户端以此减少请求 加快响应 不过 假如存储在 sessionStorage 里 当你换一个页面浏览时 数据就将重置 所以这不是一个好的方法

- 因此我们把用户数据存放在 localStorage 里 这样将永不过期 什么时候清除它呢？自然是在获取数据 出现异常要求你重新登录时清除 这样就完成啦

- sessionStorage 则比较适合一个页面跳转到另外一个页面时的数据传递 比如填写表单时 上一页表单需要把数据保存好再填下一页

# 使用 Kafka 消息队列处理订单操作

kafka 相关知识可以见[应用系统体系结构课程笔记](https://blog.nwdnysl.site/2024/09/23/AEA/#Apache-Kafka)

- 今天想把服务器上的 kafka 也给配置了 结果出现个小问题 花了我半天才解决 沟槽的 下面讲讲这个问题

- 在上面的链接里可以知道 由于目前 kafka 集成了 kraft 来代替之前需要单独部署的 zookeeper 所以在本地配置 kafka 是很快捷的 只需要拉取镜像并运行即可

- 然而使用容器部署会产生一个问题 我之前没有注意到：kafka 有一个代理的 broker 这个 broker 的连接地址是在`kafka/config/server.properties`文件中的`advertised.listeners`来配置的 默认配置是`PLAINTEXT://localhost:9092` 如果你的后端也部署在容器里 显然这个 localhost 会导致连接不上（因为容器内的 localhost 指的不是宿主机）那你会说 很简单 我们改一下配置不就行了 确实 我们接下来要做的事都只是为了修改这么一个小小的配置：

  1. docker 有两种方式改变容器内的配置文件 一种是在 Dockerfile 里写 COPY 指令 一种是使用`docker cp`指令 详细操作见：https://blog.csdn.net/CatchLight/article/details/139526724
     然而 我本人试过很多遍 这个方法不生效 原因大概是 kafka 的镜像在启动时会自动生成一个配置文件 也就导致你复制进去的配置都会被默认配置覆写
  2. kafka 还提供了另一条路来修改配置 即启动参数来添加环境变量 所以我们只需要在 docker-compose 里添加这一行即可：

  ````
  environment:
     - KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://ebook-kafka:9092```
  ````

  解决了吗？并没有！如果你启动容器 kafka 会报错说缺少 zookeeper 的连接配置 不对啊 我们之前说过现在使用了 kraft 代替了它 怎么又要我配置呢？

  原因是这样的：kafka 如果设置了启动参数 会把所有环境变量写到一个空的配置文件里 而并不会先生成一个默认的配置文件后进行修改 所以在原来配置里的`process.roles=broker,controller`这一项就消失了 这一项是用来告诉 kafka 我要使用 kraft 的 同理的 还有很多配置都是必须的 而现在全没有了 自然会报错

  3. 走到这一步花了我半天的时间 然后我终于知道该如何配置沟槽的 kafka 了：把默认的配置文件找出来 把所有的键值对都作为环境变量配置到 docker-compose 里！是的 这样配置很蠢 但是我找不到更好的办法了 TT 如果你也遇到同样的问题需要解决 你可以查看我 Ebook 仓库的 dockercompose 文件找到所有应该注入的环境变量 希望这能帮到你
