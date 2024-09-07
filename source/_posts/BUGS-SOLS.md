---
title: 遇到过的bugs以及solutions
date: 2024-09-07 23:42:00
categories: WEB开发
tags:
  - 前端
  - 后端
  - 部署
index_img:
banner_img:
excerpt: " "
---

# 自己写项目遇到过的 bug

- antd 的 form 组件的 initialValues 属性 只会在初次渲染时挂载 也就是说 useeffect 引起的重新渲染不会导致表单数据重新初始化 遇到需要渲染初始数据的情况（比如记住用户名密码） 用 form.setFieldsValue 来实现初始化渲染

- content type 为 formdata 的请求 在 fetch 中不能手动声明类型 否则 boundary 会消失 需要自动让 fetch 识别并生成

- 将后端改为使用 https 但前端仍为 http 时 会导致跨域问题 具体而言 服务器通过响应 body 里的 set-cookie 项来提供 cookie 其中有一个 samesite 属性用于告诉浏览器（客户端）这个 cookie 在什么时候需要发送 如果响应体里没有 samesite 属性 浏览器默认设置其为 lax 这会导致后端只能在不跨站（前端后端 url）的情况下发送 cookie 解决方法是后端显式设置 samesite 为 none 即不管何时都发送

- 顺便说一下跨域和跨站的区别：

  - 跨域=不同源 同源=协议+域名+端口相同
  - 跨站：先说明 eTLD+1 是什么东西 TLD（顶级域名）是记录在一个列表中的所有最高级别域名 比如常见的 com eTLD 则是有效顶级域名 是 TLD 再往前加一个点 +1 显然就是再加一层域名 eTLD+1 部分的域名相同=不跨站

- rn 框架的 fetch API 和 react 的不一样 rn 框架下 fetch 的请求体 body 无法接受 urlsearchparam 类型的对象为参数 导致后端 security 无法解析 react 的 fetch 则会自动解析成字符串 解决方法是手动 stringfy 一下

- 默认启动 springsecurity 的 csrf 防护时 不对 get 做限制 但是会要求其他类型的请求带上 csrftoken（就是在前端表单里隐藏提交的那个东西） 因此会导致服务间发送请求被拒 关闭 csrf 即可 有 ip 限制就够了

- ORM 框架中如果父子设置了 orphan delete 子集合的更新就不能直接使用 set 函数 而需要先 clear 后 addall 这是由于引用改变导致了孤儿被删除？

- react native navigation 的 navigate 逻辑是压入栈中 原本的旧页面的 useeffect 等函数还会生效未销毁 导致 alert 等冲突 而且还可以通过返回回到旧页面 解决方法是使用 replace

- 因为懒得使用外部中间件实现负载均衡 所以自定义一层 lbservice 使用 openfeign 自带的 ribbon 进行负载均衡 然后涉及到了 openfeign 如何在请求里带上 cookie 以及响应中的 setcookie 如何返回的问题 解决方法是配置一个 config 类继承 springdecoder 这样才能保留原来的自动选择 decoder 的特性 否则没法选择合适的 decoder 比如 json 和 text 需要的 decoder 就不一样

- rn 强制竖屏：见https://blog.csdn.net/qq_40759232/article/details/133883285

- rn 的默认字体颜色应该是跟随系统的 也就是说如果系统深色模式下的默认字体颜色是白色 就会变成白色 解决方法：强制浅色模式 或者使用不同的样式进行适配（也就是写死样式）

- 沟槽的 openfeign：如果要使用 requestparam、pathvariable 注解 必须显式声明 value 否则无法解析

- 在线生成 event 时 ai 有时候不返回 subtask 或者 reminders 有时候缺少 pro

- js 中的对象赋值都是按引用传递的 这意味着如果你不进行显式的深拷贝 就会莫名其妙改变父组件的参数 注意赋值时用复制对象而不是简单的=即可

- rn 项目的 app 图标 如果使用 png 转换 就会留出白边 原因未知 换成 jpg 就解决了

- redis 的配置

  - linux 环境下的编译：见https://blog.csdn.net/wangwang12345678910/article/details/118856459
  - 外网访问：由于在容器内无法用 localhost 访问 redis 所以采用外网 ip 首先要记得开放服务器安全组 然后 bind 0.0.0.0 监听所有端口 最后关闭保护模式或者设置密码（推荐后者）
  - 系统服务配置与 linux 防火墙配置见：https://blog.csdn.net/weixin_71901717/article/details/137911126
  - windows 上配置的修改要注意 service.conf 文件是系统服务启动的配置文件 另一个则是手动点击 exe 启动的配置

- java8 的 localdate 类型默认不支持序列化 需要自己配置一下 https://blog.csdn.net/qq_42435861/article/details/139305956

- 沟槽的 springcache 注解 因为 aop 要走 spring 框架代理的原因 如果在同一个 service 里拆分两个函数 一个函数写 cache 注解 另一个函数进行调用 这样的方式是不走代理的 也就不会生效 只能重新写一个 service 或者提取到 controller 里

- 沟槽的 redis 如果从缓存中获取的对象带有集合属性 就会报懒加载错误 具体原因我不懂 解决方法是覆写 hibernate 的方法 见https://stackoverflow.com/questions/54722546/sprng-boot-jpa-redis-lazyinitializationexception

- nacos 的注册逻辑：nacos 是自动帮你把服务的 ip 进行发现并注册 并不是随便选一个 ip 然后把服务放到这个 ip 上 因此实际上 注册过程相当于 nacos 找到这个服务的 ip（可能是优先内网） 然后加入到服务列表 这样其他服务调用这个服务时 就会向服务列表里记录的 ip 发送请求 https://blog.csdn.net/weixin_48359973/article/details/132378494

- 因为之前一直没搞懂 nacos 的注册逻辑 所以有些问题：

  - 两台服务器之间的容器的通信：在不同主机上的容器注册到 nacos 上后 发现无法互相调用 原因涉及到容器和宿主机的网络通信 由于容器和宿主机默认是以桥接模式形成内部局域网 而容器会部署到 172 开头的内网 ip 上 因此在容器内部 ping172 网段 实际上都是本宿主机内的其他容器 ip 而 nacos 自动注册时会把容器内的服务优先注册到这个 172 的网段 ip 这就导致了如果另一个主机内的容器调用这个服务 肯定是在 ping172 网段 实际上是在发送请求给本宿主机内的容器 肯定是不行的
  - 解决方法：（https://cloud.tencent.com/developer/article/1962232）

  1. 把桥接改为 host 模式 容器将会使用宿主机的 ip 注册时也就注册了宿主的 ip 其他容器去 ping 其他宿主机是可以的 因为 192 网段或者外网肯定是未被占用的
  2. 手动配置注册 ip 为宿主机的 ip（内网或者外网都可以） 端口映射都不用变 这样同理也可以 ping 通

  - 使用 Python-nacos 时 发现注册后服务总是自己挂掉 查看参数才知道 原来服务需要心跳机制来告诉 nacos 中心自己是否健康 springboot 中无需显式配置 会自动发送心跳 而 Python 需要显式设置参数 一般为 5s

- ELF 结合 springboot：https://www.cnblogs.com/toutou/p/SpringBoot_elk.html#_label1
