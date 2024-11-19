---
title: 《应用系统体系结构》课程笔记
date: 2024-09-23 14:01:00
categories: 课程笔记
tags:
  - 架构
  - 后端
index_img:
banner_img:
excerpt: " "
---

## 预备知识：如何用 Docker 快速部署本地环境？

这门课程将会学习很多技术栈 同时也就势必需要配置很多环境 如果还像以前一样 前端用 npm 手动启动 后端用 IDEA 启动 数据库作为自启动服务 那么后续要装的环境只会越来越多 越来越难管理 不说系统盘会不会爆炸 启动一次完整的项目可能都要花半天 因此使用 Docker 来配置本地的环境 做到一键启动

### 从前端 Vite 项目开始

- 和服务器上的部署都类似 不过我们通常本地的操作系统都是 Windows 官方推荐安装一个图形化前端 Docker Desktop 链接放这里：https://docs.docker.com/desktop/install/windows-install/ 安装完后需要配置虚拟环境 网上文档里都有教学 我就不赘述了 本人是使用了 HyperV

- 还是编写 Dockerfile 和服务器上的部署不同 为了追求本地开发的效率 我们不能再通过先编译生成静态文件复制到容器内部来部署了 这样子部署 开发时无法热重载 代码更改看不到效果

  考虑到本地开发 源代码都是在的 不妨利用挂载来进行热重载 因此采用下面的 Dockerfile：

  ```
  FROM node:18-alpine

  WORKDIR /frontend

  # 只设置启动命令
  CMD ["npm", "start", "--", "--host"]
  ```

  也就是和本地一样直接启动 `--host`参数是用于暴露端口的 不设置的话宿主机无法访问这个端口

- 然后编写一下 docker-compose 文件 和服务器上部署区别不大 加上一个挂载即可

```
  frontend:
    image: ebook-frontend
    container_name: ebook-frontend
    ports:
      - 5173:5173
    networks:
      - my-network
    volumes:
      - ../frontend:/frontend
    command: sh -c "npm install" # 这条命令初次运行需要加上 因为linux环境有些依赖可能需要重新安装 之后就可以注释掉了 因为挂载实现了持久化 后续再有缺少依赖就重新加上
```

- 最后可以用命令行一键拉起 后续也可以在 Docker Desktop 里使用 GUI 来启动 里面还提供了终端、日志、浏览文件等功能 很方便

### 后端 Spring Boot

- 后端的思路是一样的 也是挂载后直接运行即可 注意 maven 的本地依赖也要挂载 要不然每次都需要重新下载 下面贴一下代码
  Dockerfile:

  ```
  FROM maven:3.8-openjdk-17

  WORKDIR /backend

  CMD ["mvn", "spring-boot:run"]

  EXPOSE 8080
  ```

  docker-compose.yml:

  ```
  backend:
  image: ebook-backend
  container_name: ebook-backend
  ports:
    - 8080:8080
  networks:
    - my-network
  volumes:
    - ../backend:/backend
    - E:/.m2/repository:/root/.m2/repository #挂载本地仓库
  env_file:
    - ../deployment/backend/.env
  ```

- ps：本人测试后发现容器内编译+启动大概需要十几秒 而 IDEA 启动却只需要 3-4 秒 不知道这其中的差别在哪 因此还是采用 IDEA 来启动项目

### MySQL 数据库

- 和服务器上部署无区别 另外提一嘴 既然服务器上已经部署了数据库了 其实就无需再在本地配置数据库容器了 直接连服务器方便又省心 Redis 也是同理的

## 9.23

### 谈谈应用架构

- 应用服务器首先为了安全考虑（比如数据库的端口不能暴露） 可以把应用和**数据库**、**文件服务器**等分开部署 通过内网访问 防止外网攻击

- 随后可以添加**分布式缓存服务器** 利用服务器的内存来增加读取速度 考虑到缓存需要少写 设计数据库时就可以把一张表拆分为只读数据和多写数据

- 请求一多 就需要**负载均衡服务器**来进行转发 比如 nginx 有一些新的问题就需要解决 比如分布式 session 的处理

- 数据一多 数据库就需要分布式部署 利用**主从复制**就可以做读的负载均衡 以及容灾备份 为了充分利用资源 可以轮流设为主节点 防止从节点过冷 为了保证主节点的可用性 还需要哨兵等中间件

- 图片音频一多 就需要 **CDN 服务器**来进行内容分发 为了单一访问 还需要一个反向代理服务器来代理这些 CDN 服务器

- 为了代码简单 可以把缓存、数据库、文件服务器的操作统一抽象成一个 API 形成**统一数据访问网关**

- 出现 NoSQL 数据库和搜索引擎后 数据访问的逻辑会更复杂 继续在网关里统一编写

- 为了处理异步通信 需要**消息服务器**

- 不同的应用服务器也可以提取出公共的服务 形成**微服务架构** 等待应用调用

### 服务的有状态和无状态

- http 协议是无状态的协议 即没有办法记住用户的状态

  有状态的服务是需要占用资源的 比如把实例放在内存里 但是不能无限扩大 如果要限制资源大小 又会因为 LRU 等策略导致缓冲池的抖动

  因此 要尽量使用无状态的服务 保留必需的有状态实例

- Spring Bean 的作用域（以一个 service 为例）

  - singleton：全局单例（默认）
  - prototype：每调用一次就创建一个新的实例
  - request：每进行一次 http 请求就创建一个
  - session：一次会话创建一个实例
  - application：一个 ServletContext 生命周期创建一个实例
  - websocket：一次 websocket 对话创建一个实例

- 数据库连接池=核心数\*2+有效硬盘数 与用户数无关
  ![](https://image.blog.nwdnysl.site/image-e386b0b13719a1472c0f8d15c2f006a4.png)

### 作业：会话计时器

![](https://image.blog.nwdnysl.site/20240925165306-d3f7c51e9b6c2b6c2526349dcb31357e.png)

- 首先为了作业要求 我需要把原本使用 Spring Security 默认的登录校验流程提取到自定义的 LoginController 里 研究了一下午后 总结流程为：在自定义的 Login 函数里 调用默认的 authenticate 函数进行登录校验 可以是外部注入的（那么就是自定义校验逻辑） 也可以使用默认的（使用 daoAuthenticationProvider 类）把用户信息放到 SecurityContextHolder 里 再把这个上下文里的数据放入 session 然后清除上下文 实际上后续任何需要验证的请求 都会遵循这三步（查找 session 放入上下文 在请求处理过程中使用上下文 清空上下文）

  至于 logout 函数就是失效一下 session 即可

- 然后我们创建 TimerService 来计时 实现很简单 详见 Ebook 仓库代码

- 问题的解答：
  在 Controller 和 Service 层都使用了 session 作用域的 scope。
  - Service 层使用 session 的原因：
    - 用户会话保持的时间是一个状态 需要有状态的服务 使用默认的单例的话 每个用户获取到的 service 都是同一个实例 无法为每一个用户维护不同的状态 所以显然不用单例模式 同理的 如果选用 prototype 同一个用户每一次调用会获取到不同的服务 每次都会获得一个新的计时器 自然也不符合要求
    - 而既然要求一个会话维护一个状态 那么 session 作用域是最好的选择 即一个用户如果有新的会话 调用的 service 就会新创建一个实例 做到一次会话对应一个计时器对应一个状态 实现为每一次会话记录持续时间
  - Controller 层使用 session 的原因：
    - 如果服务层使用了 session 作用域 而控制层却使用单例作用域 因为控制层只被创造一次 也就只进行一次依赖注入 也就是说控制层实际上只会调用同一个服务层实例 那么服务层的 session 作用域也就无效了 同理如果是 prototype 作用域 多个 controller 也只会调用同一个服务层实例 是没有必要的
    - 和服务层一样使用 session 作用域是最好的选择 同一个会话的所有请求对应控制层 这个控制层又会对应注入一个服务层

## 9.25

### 异步通信模型

- 为什么需要异步模型（而不用同步模型）：

  1. 上层和接口还是紧耦合的
  2. 缺少调用保障
  3. 软件声明
  4. 没有请求的 buffer
  5. 过于强调请求响应模型
  6. 交流是不可靠的
  7. 通俗理解：同步模型下如果有超过服务器处理能力的高并发请求 所有超过阈值的请求都会被抛弃 而异步模型就可以塞入消息队列慢慢处理 直接响应用户 告知请求一定会被处理

     如果请求数始终超过服务器能力 那么同步异步没有区别 都无法处理 只能提升资源

- 什么是异步模型：没有服务端和客户端之分 而是用消息中间件来处理异步信息 就好比你给室友发条 wx 叫他买早饭 你只需要发送出消息 不需要等待响应而阻塞 也不需要知道室友买早饭的过程

- kafka 中有生产者和消费者 生产者（比如控制层）把消息放入类似缓冲区的 topic 区中 等待消费者（比如服务层）取出进行处理

- 客户端如何知道异步处理的结果？

  1. ajax 重新发一个请求确认
  2. websocket 推送
  3. 消费者完成任务后 把结果放入 topic 等待原来的生产者取出进行处理

- java 提供一个 JMS 来进行消息处理 是异步且可靠的（一定会保障消息传输到）

- JMS 消息的格式：

  1. 一个消息头
     - 发送目标
     - 发送模式
     - 消息 ID
     - 发送时间戳
     - 相关消息 ID
     - 回复给谁
     - 有无转发
     - 消息类型
     - 过期时间
     - 优先级
  2. 消息属性（可选 可以作为消息头的自定义扩展）
     - 用户 ID
     - AppID
     - 转发数
     - 组 ID
     - 生产者和消费者的事务 ID
     - 接受的时间戳
     - 消息状态
  3. 消息体（可选）
     - Text：一个 String 对象
     - Map：一个键值对集合
     - Bytes：一个字节数组
     - Stream：一个流对象
     - Object：一个可序列化的对象
     - Message：上述五个消息类的父类

- 如何实现（JMS）

  1. 消息发布有两种类型 点对点和广播 一个通过 queue 一个通过 topic
  2. 向连接工厂获取一个连接 创建一个上下文和 JMS 服务器进行交互
  3. 通过上下文创建消息 进行发送和接受 发送时可以设置上述的 selector 接收异步的消息则需要创建一个 listener 来进行监听 在 onMessage 函数里编写处理操作
  4. 持久化订阅：创建一个持久化的消费者 订阅一个 topic 会把
     所有未过期且未收到的消息接收
  5. 消息浏览器：只浏览消息 不消费
  6. 对于下订单 也就是控制层向某一个 topic 发送订单信息 服务层监听这个 topic 并消费每个订单消息

### Apache Kafka

- kafka 基于日志来存储消息 是只追加的数据 即使用时间戳来标记删除和更新 写的操作速度快

- 由于 topic 的消息是在内存中存储的 每个用户会维护一个目前读到的消息在内存中的偏移量 来保证每个用户读消息不会乱

- topic 很大时可以用哈希分区 甚至可以形成集群 部署到不同服务器上 为了容灾还可以加复制因子参数 来进行备份

### 作业：使用 Kafka 中间件处理订单请求

- 配置 kafka 环境：使用容器即可 新版 kafka 无需再使用 zookeeper 只需要使用内置的 raft 即可 总共两行代码搞定 `docker pull apache/kafka:3.7.0` `docker run -d -p 9092:9092 apache/kafka:3.7.0`
- 在 Spring 里使用 kafka(可参考：https://blog.csdn.net/Eternal_Blue/article/details/125293622)
  1. 首先添加依赖`spring-kafka` 然后在 application.yml 文件里配置 bootstrap-servers（服务 url）
  2. 创建话题：随便创建个类 或者在启动类里 注册类型为 NewTopic 的 Bean spring 会自动读取并在 kafka 里进行创建 参数分别是 topic 名称、分区数和复制因子
  3. 发送消息：以下订单为例 在控制层先进行鉴权获取 uid（因为消息接受时上下文就没有了） 然后调用 kafkaTemplate 的 send 方法发送封装好的消息即可
  4. 监听消息：创建一个 Listener 类 在里面编写函数并打上`@KafkaListener`注解 这个函数会创建一个消费者实例进行消息的监听 注解参数有很多 主要是 topic 名称和 groupId 我的理解 一个组就是多个消费者并发的消费队列中的消息 可以提高效率 如果消息只有 1 个 那就会随机发给组内某一个消费者
  5. 完成操作：消费者接收到下订单的消息后 调用服务层的方法来处理订单 把订单处理的结果使用 websocket 进行推送 即可让用户知道结果了

## 9.30

### WebSocket

- ws 是一个全双工的应用协议 即双向工作对等 底层是 TCP 协议

- ws 应用会运行一个 endpoint 注册此 endpoint 的 uri 作为连接端点

- 连接建立包含两部分 握手和数据传输 使用 GET 向 endpoint 进行请求 会进行协议的升级 握手后协议升级为 ws

- 使用注解@ServerEndpoint 来注册一个 endpoint 作为服务端 OnOpen 函数中进行连接建立时的操作 OnMessage 函数进行收到消息时的操作 OnError、OnClose 同理

- 群发消息如果使用遍历是很慢的 因此需要把 sessionMap 开成并发安全结构 后续可以多线程处理

- ws 支持自定义编码器 对于文本和二进制消息有两个接口 实现接口中的 encode 方法后即可自定义如何编码 同样也有解码器接口

- 通过编写 Message 类的子类 可以实现多种自定义消息类 在 decode 时就可以根据类别对应处理 多个 endpoint 就可以使用一个 decoder （也就是说 比如一个聊天室的 endpoint 一个点赞消息通知的 endpoint 聊天室可以有两种消息 Text 和 Image 点赞可以有一种消息 LikeM 三种消息都编码成文本 在同一个 decoder 里解码 分别解成三种子消息 然后根据消息类型分别处理）

- 对于下订单服务 当 listener 收到订单处理完成的消息时 应该把消息封装后通过注入的 ws 发送给对应用户 用户在前端需要先注册好 ws 服务 可以再下订单之前建立连接 收到 ws 消息后关闭连接

- 总结
  - ajax 实现的是前端的异步处理 即用户点击发送请求后可以立即进行其他操作 收到响应后会自动处理
  - 消息队列实现的是后端的异步处理 让服务的处理可以异步实现 由于异步无法使用响应让用户知道结果 就使用 ws 来实时通知用户

### 作业：使用 WebSocket 实现订单处理通知

- 作业要求：在下订单服务中加入 WebSocket 通知功能 当订单处理完成后 通过 WebSocket 通知用户订单处理结果
- 实现思路：配置一个 WebSocket 服务端点 当 listener 监听到订单处理完成的消息时 调用 ws 服务的发送函数 发送消息给用户 前端页面需要先建立连接 然后监听消息即可
- 前端在何时建立连接：为了不浪费资源 可以在用户进入购物车页面时建立连接 在用户离开购物车页面时关闭连接

- 代码实现：在后端配置一个 websocket 的端点 这个端点不会收到消息 只需要封装一个 sendToUser 函数用于给前端的用户推送消息即可 sessionMap 采用 uid 做键

  实现端点后 在 finishOrder 的监听器中向用户发送消息 如果处理成功 发送“订单成功处理” 否则发送“订单处理失败+{异常信息}”

  前端在购物车页面加入 ws 使得进入购物车页面后与后端端点建立连接 关闭页面后关闭连接 避免不必要的资源浪费 为 onmessage 方法配置一个简单的 alert 用户将会在收到消息时看到一个提醒框

## 10.9

### Transaction

- 基本概念

  - 事务是一组操作的集合 保证这组操作要么全部成功 要么全部失败
  - 事务的四个特性：ACID

    - Atomicity：原子性 事务要么全部成功 要么全部失败
    - Consistency：一致性 事务前后数据的完整性保持一致
    - Isolation：隔离性 事务之间互不干扰
    - Durability：持久性 事务一旦提交 数据就会永久保存

- JTA 是 Java 提供的事务管理接口 使用@TransactionAttribute 注解来控制事务的传播属性

  - REQUIRED：（默认）如果当前线程有事务就加入 如果没有就新建一个事务
  - REQUIRES_NEW：无论如何都新建一个事务 这种属性可以让子方法的失败不影响父方法的事务
  - SUPPORTS：如果有事务就加入 如果没有就不用事务
  - NOT_SUPPORTED：不支持事务 遇到事务后挂起 当前方法执行完后再恢复
  - MANDATORY：必须有事务 如果没有事务就抛出异常
  - NEVER：不能有事务 如果有事务就抛出异常
  - Spring 中此注解变为@Transactional(propagation = Propagation.REQUIRED)

- 事务的隔离

  - 未隔离的事务可能会出现以下问题

    - 脏读：事务 A 进行写入 写入过程中事务 B 读取到了未提交的数据 事务 A 后续进行了回滚 此时事务 B 读取到的数据就是不一致的
    - 不可重复读：事务 A 读取第一次数据 之后事务 B 对同一个数据进行了写入 事务 A 再次读取数据时发现数据不一致
    - 脏写：事务 A 进行写入 事务 B 也进行了写入 导致操作被覆盖
    - 幻读：事务 A 需要查询满足某些条件的数据 在查询一次后 事务 B 插入了一条新的符合条件的数据 导致事务 A 再次查询时发现数据增加了新的“幻影”数据行

  - 使用@Transactional 的 isolation 属性来设置隔离级别（从上到下隔离级别逐渐增高 常用可重复读）
    - READ_UNCOMMITTED：允许读取未提交的数据 会出现脏读、不可重复读、幻读
    - READ_COMMITTED：只能读取已提交的数据 避免了脏读 但是会出现不可重复读、幻读
    - REPEATABLE_READ：确保同一事务内多次读取同一行数据是一致的（也即正在被读取的数据不能被修改）避免了脏读、不可重复读 但是会出现幻读
    - SERIALIZABLE：事务完全串行化（为各个数据项上锁 甚至为表上锁）避免了脏读、不可重复读、幻读 但是性能较差

- 多数据库的写入

  - 当存在多个数据源时 Tomcat 会自动使用分布式的事务 采用**两阶段提交** 第一阶段向所有数据库询问是否准备好 第二阶段若所有数据库都准备好则提交 否则回滚 此种处理仍会出现错误 但是概率较低 可以人工处理

- 锁机制

  - 乐观离线锁：数据会被用户 session 读取后离线编辑 在写入更新数据时先查询数据的版本号 然后在更新时判断版本号是否一致 如果不一致说明基于旧数据进行了修改 则不进行更新 否则更新数据 并将版本号+1 适用于写远少于读的场景
  - 悲观离线锁：加上写锁 保证只有一个线程可以写入数据 适用于写入操作频繁的场景
  - Coarse-Grained Lock：粗粒度锁 一次锁住多个数据
    - 乐观锁：给关联的对象加上共享的版本号（可以是内存也可以持久化） 如果对象的版本号被修改 其关联的对象将不能写入
    - 悲观锁：给版本号上锁 保证只有一个线程可以写入

- 注：@Transactional 注解只能对于资源管理器（比如数据库 消息队列）事务有效 对于非资源管理器事务（比如局部变量 x++）无效

### 作业：在下订单的服务中加入事务管理

- 作业要求：在保存订单的 saveOrder 方法与保存订单项的 saveOrderItem 方法中加入事务管理 使得两个保存方法要么同时成功 要么同时失败
- 实现比较简单 在 saveOrder 和 saveOrderItem 方法上加上@Transactional 注解即可 两个方法都会在父方法的事务中运行
- 显然不能使用 REQUIRES_NEW 因为这样会新建一个事务 导致回滚时只会回滚子事务 而不会回滚其他事务
- 注：使用 REQUIRES_NEW 时 saveOrder 加入父方法的事务 saveOrderItem 新建一个事务 而在插入订单项时 saveOrder 还未提交 外键约束会导致插入失败 然后 saveOrderItem 会回滚 但是 saveOrder 仍会提交

## 10.16

### 事务管理

- 事物的回滚和提交实际上是资源管理器通过 redo log、undo log 来实现的

#### 可串行化调度

- 调度：并发事务的执行顺序
- 串行调度：事务完全按照顺序执行
- 并发调度：事务可以并发执行
- 可串行化调度：并发执行的事务的结果和串行执行的结果等价 则称这个并发调度是可串行化的 可串行化调度太多 通常只找子集
- 等价交换：两个事务交换位置后结果不变 比如交换两个读、交换两个不同数据的读写
- 冲突可串行化调度：通过等价交换把一个并发调度转化为串行调度 即可证明这个并发调度是可串行化的
- 无冲突的调度可以使用等价交换转换为串行调度 有冲突的调度则需要使用冲突图来判断是否可串行化
- 冲突图：事务之间的冲突关系 用图表示 有向边表示冲突 无环则可串行化

#### 事务提交和回滚

- 数据库恢复机制：无非是单机多副本、主从复制、异地多机灾备等

- 策略

  - 原子性
    - 窃取：未结束事务可以将脏页落盘 占用内存少 但是影响原子性 需要 undo log
    - 非窃取：未结束事务不能将脏页落盘 不影响原子性 占用内存多
  - 持久性
    - 强制：已完成事务必须落盘 不存在持久性问题 但是 IO 开销大
    - 非强制：已完成事务可以不落盘 但是可能会丢失数据 需要 redo log

- 日志

  - redo 用于持久性 undo 用于原子性
  - 日志是磁盘文件 但是由于写入日志是不修改的 可以顺序写入 速度很快 比数据库的写入快很多
  - 日志记录了每一次数据库的操作 包括事务的开始和结束范围等
  - 按功能分类
    - Undo Log：格式为<事务 ID 数据项 数据旧值> 用于回滚 在数据库写入之前写日志 一般还包含一个日志序号 LSN（Log Sequence Number）用于记录日志的顺序 可以是逻辑日志或物理逻辑日志
    - Redo Log：格式为<事务 ID 数据项 数据新值> 用于重做 必须是物理日志
  - 按性质分类
    - 逻辑日志：记录事务中高层抽象的逻辑操作 相当于 SQL 语句 比如小明的账户减少了 100 元 不是幂等的 不能重复执行
    - 物理日志：记录数据库的具体物理操作 比如某个页面的某个 offset 的数据由 A 改为 B 是幂等的 可以重复执行
    - 物理逻辑日志：记录数据页面的物理信息 但是记录的是逻辑操作
  - 日志的性质
    - 幂等性：重复执行不会产生不同的结果 逻辑日志不满足
    - 失败可重做性：日志执行失败后 可以通过重做达成恢复 逻辑日志不满足 因为一条逻辑记录可能对应多项数据修改 比如数据表和索引
    - 操作可逆性：逆向执行日志可以撤销操作 物理日志不满足 因为数据页的偏移可能已经被修改 而物理逻辑日志可以通过页面前后指针来完成回滚

## 10.19

### 事务管理

- 数据库恢复方法
  1. 影子拷贝方法：所有写操作会在一个拷贝数据库上进行 提交事务时切换数据库指针 效率很低 难以支持高并发
  2. 基于 Undo Log 的恢复方法：恢复时找到所有未结束（需要回滚）的事务 反向扫描 undo log 恢复数据
  3. 基于 Redo Log 的恢复方法：恢复时找到所有已提交的事务 正向扫描 redo log 恢复数据 需要通过 checkpoint 机制来判断是否刷盘
  4. 补偿日志：Undo 日志的 Redo 日志 因为 Undo 是不具有幂等性的 不能重复执行 所以需要记录撤销动作 当 Undo 过程时又发生崩溃 保证 Undo 日志不会重复执行

### 多线程

- 如何创建一个线程实例（推荐使用第一种 因为 java 里一个类只能扩展一个类）

  1. 提供一个 Runnable 接口的实现类 重写 run 方法
  2. 扩展 Thread 类 重写 run 方法

- Thread 类的方法

  - sleep：让线程休眠一段时间 可以认为时间是准的
  - interrupt：中断线程 会抛出一个中断异常 interrupted 则是判断是否被中断
  - join：当前线程会阻塞的等待此线程执行完毕 参数是等待时间
  - wait：让线程等待 直到被唤醒 会释放锁
  - notify：唤醒一个等待的线程

- Synchronized：一个关键字 修饰的方法或者代码块带锁 只有一个线程可以执行

  - java 里其实每一个对象都有一个锁 sychronized 就是获取这个锁
  - 每一个类也有锁 所以调用类的 static 方法也是保证同步的
  - Reentrant Synchronized：可重入锁 一个线程可以多次获取同一个锁

- 原子性：java 规定 除了 long 和 double 以外的基本类型的读写是原子的 加上 volatile 关键字可以保证原子性

- Liveness：活性 一个线程不会永远等待下去

  - 死锁：两个线程互相等待对方释放锁
  - 饥饿：一个线程一直无法获取到锁
  - 活锁：两个线程因为不断响应而无法继续执行 并非阻塞

- Guarded Blocks

  - 单纯使用 while 循环来判断条件 确保线程间的协调 然而会浪费资源
  - 更有效的方法是使用 wait 和 notify 来进行线程间的通信
  - wait 会使当前线程等待 并释放所有锁 notifyAll 会唤醒所有等待的线程 并使得其中一个线程获得锁 以此进行线程间的通信 常见的例子是生产者消费者模型 当队列为空 消费者等待 等生产者生产后通知消费者

- Immutable Object：不可变对象 一旦创建就不能修改 适合多线程环境 因为不知道用户会如何读取数据 为了防止其他线程在读取时修改数据 只能把数据设为不可变 比如 final 和 private

## 10.21

### 高并发对象

- Lock Objects
- Executors：提供线程管理和创建 本质上就是一个线程池
- 并发安全的集合：比如 ConcurrentHashMap、CopyOnWriteArrayList
- Atomic Variables：原子变量 比如 AtomicInteger、AtomicLong
- ThreadLocalRandom：线程安全的随机数生成器
- Virtual Threads：轻量级线程 由于 OS 的线程数量有限 多个虚拟线程会复用一个 OS 线程 比如 server 就可以为每一个 client 连接创建一个虚拟线程

### 缓存

#### Memory Cache

- 为什么要手动管理
  - 数据库自带的缓存不可控制 而且不在同一个进程中 而且很难做分布式缓存
  - MemCached 可以缓存非数据库数据 比如一个请求的结果
- 比如 stock 这样多写的数据字段 不应该放到缓存里 应该从 book 类里拆分出来
- memcached 会把 chunk 根据大小不同分为很多等价类 选择合适大小的 chunk 来存储数据
- 分布式缓存首先节点间无需通信 只需使用哈希算法来确定数据位于哪个节点即可 需要用一致性哈希来进行数据分片 保证增加减少节点时数据的迁移量最小
- 缓存达到上限时 可以使用 LRU 直接删除缓存 也可以驱逐到磁盘上 因为可能是计算后的结果数据 所以还是比普通的请求要快的

#### Redis

- Redis 支持的值类型比较多 比如字符串、列表、集合、有序集合、哈希表 另外还支持主从复制和持久化
- Redis 还可以用于消息队列 把 Redis 当做一个发布订阅的中间件

### 作业：使用 Redis 缓存书籍数据

- 思路：

  我打算在 DAO 层使用缓存 首先判断 DAO 层里哪些接口的返回结果适合缓存到内存里

  缓存数据应该满足多读少写 像上课提到的 stock 字段属于多写 应该提取到另一个实体类里不作缓存 所以我新建了 BookDetails 实体类 包含了 stock、sales 等经常需要变动的字段

  另外 并不是所有接口都适合作缓存 比如搜索书籍时用到的 searchBooks 接口 如果全部作缓存 每一个关键词就会对应一堆数据 内存根本放不下 并且不同用户查询相同关键词的概率也不大 因此在这里不作缓存 即使后续要作缓存也应该是提取最热门的几个搜索关键词的结果作缓存

  最终在 BookDAO 层里 只对 getBookById 接口结果中的 Book 实体作缓存

  除了 BookDAO 我也在 UserDAO 里作了缓存 包括 getUserByUsername 和 getUserById 两个接口。前者经常用于其他接口的鉴权 可以放入缓存加快所有需要鉴权的接口的速度 后者则是用于展示某个用户的个人信息时需要用到 在用户量不大的情况下 也可以适当作缓存

- 具体实现：

  按照教程配置一下 RedisConfig 主要就是要自己注册一个 RedisTemplate 的 Bean 并自定义一下键值序列化器

  然后封装一个 RedisUtil 编写一些常用的接口 比如 set、get 和 delete

  为了区分不同的 DAO 层的缓存键 在前面加上前缀（比如“book::”） 并加上标识唯一键的属性（比如“id::”）

  为了每一个 DAO 层实例注入属于自己的 RedisUtil 需要声明 scope 为 prototype 这样也可以适当并发

  get 一个数据时 先去 redis 查找 如果有就返回 如果没有就去数据库查找 set 到 redis 里然后返回

  set 一个数据时 先 set 到数据库里 然后 set 到 redis 里（顺序不能反 否则 redis 里保存的数据可能是没有 id 的）

  delete 一个数据时 先 delete redis 里的数据 然后 delete 数据库里的数据

## 10.23

### Searching

- 文本的存储

  - char：固定长度的字符 不管几个字符 都占 100 个字节
  - varchar：可变长度的字符 只占用实际长度的字节 需要额外存储长度和偏移量
  - text：可变长度的字符 不限制长度 实际上是存储一个指针指向一个存储在其他地方的数据

- 用 like 进行模糊查询时 会导致全表扫描 会很慢 先把文本数据结构化 建立关键词到 id 的反向索引

#### Lucene

- Lucene 是一个 IR（Information Retrieval）工具包 提供了索引和搜索的功能 由 Apache 维护 适用于 Java 项目

- Lucene 会为文本数据建立索引文档 存储到磁盘上 查询时搜索索引来找到文本数据

- Lucene 的索引文件结构（注意 文档相当于数据库的一行记录）

  - .fnm：<字段名，是否索引，是否向量化> 向量化是为了计算文档的相似度 实现这个字段的模糊匹配
  - .tis：<字段名，记录值，文档频率> 记录值是字段的值 文档频率是这个记录值在多少文档中出现过 每一条记录值指向.frq 文件里的多条记录
  - .frq：<文档 ID，出现频率> 每一条记录值指向.prx 文件里的多条记录
  - .prx：<位置>

- 搜索的质量可以用召回率和准确率来衡量（recall & precision）

  - 召回率=结果中正确的文档数/所有正确的文档数
  - 准确率=结果中正确的文档数/结果中的文档数

- Index Flie Formats

  - 基本概念

    - Index：一组文档的集合
    - Document：一条记录 由多个 field 组成
    - Field：一个记录的某个字段 包含多个 terms
    - Term：一个词 不同字段下的同一个词是不同的 term

  - Inverted Index：倒排索引 指的是从 term 到 document 的映射 即一个 term 在哪些 doc 中出现过

  - 一个 field 可以被 stored 或 indexed 如果被 indexed 可以被作为搜索的条件 如果被 stored 则在搜索出这个文档时 可以带上这个字段的值

  - Types of Fields

    - TextField：文本字段 会被分词器分词
    - StringField：字符串字段 不会被分词器分词
    - IntPoint
    - LongPoint
    - FloatPoint
    - DoublePoint
    - SortedDocValuesField：用于排序的字段 以下四个都是
    - SortedSetDocValuesField
    - NumericDocValuesField
    - SortedNumericDocValuesField
    - StoredField：存储字段 不会被索引

  - Segments：index 太大时可以分段 分布式的存储到不同位置 比如可以按照某一个字段`time`来分段 查询时就可以剪枝 和数据库的分区类似

  - Document Number：文档的编号

- Query

  - TermQuery：单个字段、单个 term 的查询 不可以再分词
  - BooleanQuery：多个查询的组合
  - PhraseQuery：短语查询 匹配多个 term 的序列
  - MultiPhraseQuery：多个短语的查询
  - PointRangeQuery：范围查询 针对数值类型的查询
  - PrefixQuery：前缀查询
  - WildcardQuery：通配符查询
  - RegexpQuery：正则表达式查询
  - FuzzyQuery：模糊查询

- Scoring

  - 对于一次查询 可以对查询结果中的文档进行相关性评分 来展示最相关的文档 Lucene 支持自定义评分的模型 下面是几种默认的算法
    - BM25：基于词频和文档频率的评分算法
    - TF-IDF：词频-逆文档频率
      - 评分=(词频\*逆文档频率\*字段长度归一化\*字段权重)求和
      - 词频是指一个词在此文档中出现的次数 比如 TOM 在 BOOK 中出现了 3 次 而 AND 在 BOOK 中出现了 30 次
      - 逆文档频率是指一个词在所有文档中出现的次数的倒数 比如 BOOK 在所有的查询结果中一共出现了 1000 次 那么逆文档频率就是 1/1000
      - IDF 相当于这个关键词的权重 一个关键词在所有文档中出现的次数越多 说明其对于查询的重要性越低 保证了一些常见词的权重降低
      - 长度归一化（可选）是指对于不同长度的文档进行归一化 因为文档字段越长 容易出现关键词的次数越多 相当于词频变为了关键词出现的概率
      - 字段 boost 权重（可选）是用户人为对于某一个字段的权重进行调整 比如 title 字段比较重要 可以给 title 字段的权重增加一些
    - SimilarityBase：提供基类 用户可以继承这个类来实现自己的评分算法

- SOLR：对于 Lucene 的封装 提供一个 web 页面平台进行手动搜索

#### Elasticsearch

- 全文搜索 实时性好 字节目前在使用 原理是建立宽表 即关键字到 id 的反向索引

## 10.28

### Web Services

#### Overview

- Web 指 web 协议 Service 指无视 OS 和语言的服务 一个接口可以实现多种协议的服务

#### SOAP WS

- SOAP（Simple Object Access Protocol）：用于在网络上交换结构化的和类型化的信息 SOAP=XML+HTTP 即把 http 请求以 xml 格式发送

  ```xml
  <soap:Envelope xmlns:soap="http://www.w3.org/2003/05/soap-envelope">
    <soap:Header>
    </soap:Header>
    <soap:Body>
      <m:GetStockPrice xmlns:m="http://www.example.com/stock">
        <m:StockName>IBM</m:StockName>
      </m:GetStockPrice>
    </soap:Body>
  </soap:Envelope>
  ```

  其中的 Envelope 是必须的 Header 是可选的 Body 是必须的

- WSDL（Web Services Description Language）：描述 web 服务的接口和实现的 XML 文档 结构类似于下

  ```xml
  <definitions name="StockQuote"
    targetNamespace="http://example.com/stockquote.wsdl"
    xmlns="http://schemas.xmlsoap.org/wsdl/"
    xmlns:soap="http://schemas.xmlsoap.org/wsdl/soap/"
    xmlns:tns="http://example.com/stockquote.wsdl"
    xmlns:xsd="http://www.w3.org/2001/XMLSchema">
    <message name="GetLastTradePriceInput">
      <part name="body" element="xsd:string"/>
    </message>
    <message name="GetLastTradePriceOutput">
      <part name="body" element="xsd:string"/>
    </message>
    <portType name="StockQuotePortType">
      <operation name="GetLastTradePrice">
        <input message="tns:GetLastTradePriceInput"/>
        <output message="tns:GetLastTradePriceOutput"/>
      </operation>
    </portType>
    <binding name="StockQuoteSoapBinding" type="tns:StockQuotePortType">
      <soap:binding style="document" transport="http://schemas.xmlsoap.org/soap/http"/>
      <operation name="GetLastTradePrice">
        <soap:operation soapAction="http://example.com/GetLastTradePrice"/>
        <input>
          <soap:body use="literal"/>
        </input>
        <output>
          <soap:body use="literal"/>
        </output>
      </operation>
    </binding>
    <service name="StockQuoteService">
      <port name="StockQuotePort" binding="tns:StockQuoteSoapBinding">
        <soap:address location="http://example.com/stockquote"/>
      </port>
    </service>
  </definitions>
  ```

  其中的 message 是消息的定义 portType 是接口的定义 binding 是绑定的定义 service 是服务的定义

  - 系统 B 将一个接口通过 WSDL 文件暴露在网络上 其同时会生成一个代理 负责将 SOAP 消息转换为调用接口的方法 系统 A 先获取 WSDL 文件 然后生产一个代理类 其负责将方法调用转换为 SOAP 消息 并发送给系统 B 得到 SOAP 消息后 代理类会将其转换为方法的返回值
  - 通常 SOAP 消息是通过 HTTP 的 POST 请求发送的 请求体内是一个 XML 文件
  - 在 spring 中 可以使用`@WebService`注解来暴露一个服务 会在指定 url 暴露 WSDL 文件 消费者只需要 wsimport 即可在本地当做一个类来调用

#### RESTful WS

- REST（Representational State Transfer）：一种软件架构风格 设计风格而不是标准 通过 URI 来定位资源 通过 HTTP 方法来操作资源
  - Representational：资源有不同的表示形式 每个资源都有一个唯一的 URI
  - State：客户端的状态 即资源的表示形式 客户端自己维护
  - Transfer：客户端的表示随着 URI 的变化而转移
- How to Design RESTful API（CRUD）？
  - 用 method 来区分操作 用状态码来表示操作结果
  - Create：POST
  - Read：GET
  - Update：PUT
  - Delete：DELETE

#### Trade-offs

- Advantages of WS
  - 跨平台
  - 自描述性 使用 WSDL 可以很容易了解服务的参数和返回值
  - 模块化
  - 跨防火墙 通过 HTTP 和 HTTPS 可以穿透防火墙
- Disadvantages of WS
  - 低生产力 需要多写很多代码 不适合单机应用
  - 低性能 需要封装和解析 而且是纯文本传输 浪费带宽
  - 不安全 需要 HTTPS 等其他方法

## 10.30

### Micro Services

- 什么是微服务：将一个大型的应用拆分成多个小的服务 每个服务都可以拥有自己的数据库和接口 通过 HTTP 或者消息队列来通信 Spring Cloud 是最常用的微服务框架
- 为什么要使用微服务：高生产力、高可扩展性、高可靠性、高可维护性

#### Spring Cloud

- 组成

  - 服务发现：服务注册中心会把注册的服务信息保存 类似于<服务名，ip，端口>的键值对 使用网关来转发对应服务名的请求到对应的 ip 和端口
  - 负载均衡：同样使用网关来实现
  - 断路器：当某个服务不可用时 断路器会打开 避免请求堆积
  - 监控：监控各个服务的状态并收集日志 用链路跟踪来确定哪个服务出了问题
  - 消息队列（可选）：使用消息队列来实现客户端和服务端的通信 减少微服务的延迟带来的影响 一个服务对应一个 topic

- Eureka

  - 一个服务注册中心 用于注册和发现服务
  - 只需要加入依赖 配置端口 在服务模块里加上`@EnableEurekaClient`注解即可注册服务

- Spring Cloud Gateway

  - 一个网关 用于转发请求到对应的服务
  - 可以配置路由规则 对符合规则的路由进行各种处理 比如添加请求头、重定向等
  - 微服务框架中可以用于客户端请求转发给对应的服务 比如`/api/book/**`转发给`book-service/**` 而`book-service`会被解析为 ip 和端口

- Spring Cloud Function：一个用于构建无服务器应用的框架

  - 什么是 Serverless：无服务器计算 无需维护状态 是一种事件驱动的计算模型 FaaS（Function as a Service）是一种 Serverless 的实现

  我的理解就是把业务处理函数（比如图片处理）单独拆分出来作为一个微服务 让其他服务调用 如果是传统的服务模式 需要自己维护状态和调用（比如涉及到数据库的操作） 而无服务器模式则是无状态的 只需要传入参数和返回结果

  - 只需要在函数上加上`@Bean`注解即可将函数注册为一个服务

#### 作业：使用微服务部署作者搜索功能

- 具体实现

  1. 拆分微服务：将原本的项目转移到子模块里 命名为 main-service；新建 search-service 模块来实现新的功能；另外新建 shared 模块来存放各个服务需要共享的一些类 比如 entity 类和 DTO 类；新建 gateway 模块实现 gateway 的路由转发逻辑

  2. 编写新功能：同样建立控制层、服务层等软件包 然后简单的编写一个新接口 映射到`/api/search/author` 上 这个接口会根据输入的标题返回找到的书的作者名字

  3. 配置服务注册：使用 nacos 作为服务注册中心 先在 yml 配置里配置好服务名称、服务注册中心的 url 等 然后下载新版的 nacos 使用终端启动 最后启动各个微服务即可 访问 nacos 页面 可以发现三个服务都已注册

  4. 配置 gateway：gateway 作为一个网关 会处理所有请求并转发给对应的服务 同时还可以配置负载均衡 在这里我们配置 gateway 的端口为 8000 另外两个服务的端口分别为 8080 和 8081

     gateway 的路由配置可以在 application.yml 里静态配置 也支持在 nacos 的配置管理中进行动态配置 我选择先静态配置 把/api/search/\*\*的请求转发给 search-service 剩下所有其他请求转发给 main-service

  后续在服务器上部署时 用 nginx 把 8000 端口的请求反向代理为 http 的端口 80 后 用户就可以不指定端口来访问项目前端了

  注：其中鉴权机制仍可以正常运行 因为 spring cloud gateway 会把请求带有的 cookie 一起发送给服务 并且响应中的 set-cookies 也会带上

  另外 如果要涉及到微服务框架中的鉴权 有两种方案：

  1. jwt 因为是无状态的鉴权 微服务不需要做什么修改 只需要在各自服务内部进行鉴权即可
  2. session 因为是有状态的鉴权 要么把所有需要鉴权的请求集中发送给某一个服务 要么所有服务各自进行鉴权 但是需要使用 redis 等方式进行 session 的共享 保证不同服务可以共享用户的状态 在此就不赘述了

#### 作业：使用函数式服务部署计算总价功能

- 具体实现

  1.  创建新模块名为 price-service 端口为 8082 导入 spring-cloud-funciton 的依赖
  2.  为方便 在启动类编写一个计算总价的函数 Bean 接受一个 `Flux<String>`类型的参数 其格式是 json 数组 内容是 orderitem 的列表 遍历其中每一个 item 累加其数量与其价格之积 最终得到总价
  3.  在 gateway 中加入新路由 将/computePrice 的请求转发给部署在 8082 端口的 price-service 从而可以到网关访问此服务
  4.  前端需要计算总价时调用这个接口即可

## 11.4

### MySQL Optimization

#### Index

- 把数据表的某一个或多个字段的值作为键 数据项的指针作为值 存储在一个 B+ 树中 从而做到对数时间的查找
- 多了空间开销和时间开销（UPDATE 时需要维护） 建议使用在数据量大的表上
- 内存中也可以建立一些哈希索引
- InnoDB 对于全文索引使用了反向映射
- 聚簇索引：索引的实际位置和数据的实际位置一样 在 InnoDB 中主键索引就是聚簇索引
- 不建议在 NOT NULL 字段上建立索引 因为会增加空间开销 如果需要置空 可以人为使用一个特殊值来代替
- Index Prefix
  - 对于特别长的数据字段 只取数据的前 N 个字符做索引
  - row format
    - compact：索引的前缀是固定的
    - dynamic：索引的前缀是动态的
    - compressed：索引的前缀是压缩的
    - Redundant：索引的前缀是冗余的
- FULLTEXT Index：全文索引 适用于文本字段的搜索
- Spatial Index：空间索引 适用于地理位置的搜索
- MySQL 最多支持在 16 个字段上建立索引 查询时从左到右匹配索引
- 复合索引
  - 最左匹配原则：从左往右匹配 如果是等值查询可以继续 否则就停止

#### Foreign Key

- 如果对于不同字段的查询频率不一样 可以使用外键进行垂直分表

#### Database Structure

- 字段
  - 使用数值类型来存储数值可以减少空间开销
  - 尽量使用 NOT NULL
- Row Format
  - InnoDB 采用了各种方式来减少空间开销
- Index
  - 主键需要尽可能小 可以采用自增的方式 也可以采用 UUID 自增由于只能顺序写入 需要很大的开销保证唯一性 UUID 则可以分布式生成 但是会增加空间开销
  - 仅在多读少写、数据量大的表上建立索引
  - 索引键需要尽可能小才能保证速度 所以对于字符串字段可以使用前缀索引
- Join
  - join 的两列尽量使用相同的数据类型和字段名
- Normalization
  - 适当的范式化可以减少冗余 但是会增加 join 的开销
- Data Type
  - 尽量使用数值类型来存储数值
  - 小于 8KB 的数据可以使用 VARCHAR 大于 8KB 的数据应该使用 BLOB 因为如果数据太大会导致一次访问内存页面连一个数据都读不完

#### Many Tables

- table_open_cache：打开的表的数量
- max_connections：最大连接数

#### Limits on MySQL

- MySQL 对于数据库和表的数量没有限制 只限于操作系统的限制
- MySQL 一张表最多只有 4096 个列 然而 InnoDB 引擎的限制是 1017 个列
- 在声明 MySQL 中一张表的字段类型时 一行数据最多 65535 字节
  - 注意 BLOB 和 TEXT 类型的数据不会占用声明的字节数 而是 9-12 个字节的指针开销
  - VARCHAR 类型的数据还会有额外的开销存储元信息 通常是 2 字节
  - NULL 数据需要额外的 1 位来存储
- 在 InnoDB 引擎的实际存储中 一行数据最多存储不超过 innodb_page_size 的一半（实际会略小于一半）
  - 声明时 VARCHAR 类型的长度可以超过 innodb_page_size 的一半 因为不知道实际存储的数据大小 **在实际存储时如果超过了一半则会报错？？？**
  - CHAR 类型由于是定长 声明多少就会占用多少空间 所以总共加起来不能超过 innodb_page_size 的一半

## 11.6

### MySQL Optimization

#### Opimization for InnoDB Tables

##### Tables

- 一旦表的大小达到几百兆 可以使用 OPTIMIZE TABLE 指令来优化表 会把碎片整理到一起 并重建索引
- 一个长主键会浪费磁盘空间 因此建议创建一列自增列作为主键
- 尽量使用 VARCHAR 而不是 CHAR 因为前者可以压缩空间 除非确定这个字段的长度是固定的（比如学号）
- 对于具有很多重复文本或数字的字段 考虑使用 COMPRESSED 行格式（比如学号都具有相同的前缀）对于数值类型 会选定一个基准值 然后存储相对于基准值的差值
- 默认情况下 每一条 MySQL 语句会在一个事务里执行 因此会在每一条语句后落盘 会造成很大的额外开销 可以通过 SET AUTOCOMMIT=0 来关闭自动提交 然后使用 START TRANSACTION 和 COMMIT 指令来手动提交
- InnoDB 实际会对只读语句自动优化 关闭事务和自动提交
- 尽量避免在 INSERT、UPDATE、DELETE 大量的数据后执行事务的回滚 因此少做大事务
- 增加 buffer pool 的大小可以使得数据进行缓存 减少磁盘的读写 注意多个实例共享一个 buffer pool
- SET INNODB_CHANGE_BUFFERING=all 可以设置 UPDATE 和 DELETE 操作会被缓存 而不是直接写入磁盘 否则只有 INSERT 会被缓存
- 一条大数据量的语句会导致锁表 尽量不要在过程中执行 COMMIT
- 长时间的事务由于锁表还会导致 InnoDB 无法在其他事务中操作数据 除非降低隔离级别
- 在一次性导入大量数据时 可以暂时关闭自动提交 手动建立一次事务来导入数据 同理也可以通过 SET UNIQUE_CHECKS=0 和 SET FOREIGN_KEY_CHECKS=0 来暂时关闭唯一性检查和外键检查
- 使用多行 INSERT 语句可以减少客户端和服务器之间的通信次数
- 对于自增字段进行大量 INSERT 时 SET INNODB_AUTOINC_LOCK_MODE=2 而不是 1 此时会给每个线程分配一个自增值范围 而不是一个值 字段可能不会严格递增
- 因为主键会自动创建聚簇索引 所以尽量选择经常进行范围查询的字段作为主键
- 索引字段不要设置为 NULL
- 调整 INNODB_FLUSH_METHOD 参数为 O_DSYNC 会使得文件和文件元数据异步写入磁盘
- 调整 INNODB_FSYNC_THRESHOLD 参数 可以减少磁盘的写入次数
- 使用非旋转介质的磁盘可以提高随机访问性能 可以把数据文件放到 SSD 上 把日志文件放到 HDD 上
- 提高 IO 能力可以减少 backlog 的开销
- 使用 TRUNCATE TABLE 而不是 DELETE FROM TABLE 因为后者还是会遍历每一行数据 前者则是直接删除指针 如果有外键 需要注意顺序
- 主键尽量避免修改 并且在创建表时就确定好
- 内存表通常是用于临时存储数据的 非关键且只读少写的数据可以存在内存表里

##### Caching

- `--innodb_buffer_pool_size`：缓存池的大小 默认是 128MB 必须是实例数 \* chunk_size 的整数倍
- `--innodb_buffer_pool_instances`：缓存池的实例数 默认是 8 最多是 64
- 建立多实例缓存池后 同一张表就可以在多个缓存池里缓存
- 缓存池的替换策略不是 LRU 而是根据数据的热度来替换 会把新数据放到 3/8 的队列位置 如果是冷数据立马就会被替换 热数据在内存里被访问时则会被放到队尾 还会做一定的数据迁移
- 预读取会把其他数据读取到缓存里 可以顺序预读取 也可以随机预读取
- `innodb_page_cleaners`：将脏页落盘的清理线程数量 默认是 4 不会超过缓存池的实例数
- `innodb_max_dirty_pages_pct_lwm`：脏页的最大比例 默认是 10% 会触发脏页的清理
- 为了减少 warm-up 的时间 InnoDB 会把缓存池中最常用的页存到磁盘上 并在重启时直接 restore 到缓存池里 持久化的比例由`innodb_buffer_pool_dump_pct`决定 默认是 25%

## 11.11

### MySQL Backup and Recovery

#### Backup and Recovery Types

- Physical Backup：备份整个数据库的文件（.MYD、.MYI） 适合大型数据库 恢复速度快
- Logical Backup：备份数据库的逻辑结构 适合小型数据库 可以导入到其他 SQL 数据库中
- Online Backup：备份时数据库仍然在运行 会出现备份同时写入的问题 需要锁来保证一致性
- Offline Backup：备份时数据库停止运行 影响用户体验 一般数据量小
- Warm Backup：备份时数据库仍然在运行 但是禁止写入
- Local Backup：本地备份
- Remote Backup：远程控制机器进行备份
- Snapshot Backup：快照备份 增量备份的一种 通过 COW 来记录每一次修改 当改变达到阈值后进行一次全量备份 MySQL 本身不支持 必须使用 LVM 或者 ZFS 等文件系统
- Full/Increment Backup：启动 MySQL 的 binlog 功能 可以记录每一次修改的日志 通过备份时的 binlog 来进行增量备份

#### Database Backup Methods

- 建议选择企业版的 MySQL 默认是物理备份和热备份
- 如果使用社区版：
  - 逻辑备份可以用 mysqldump 来实现 使用`--single-transaction`参数可以在备份时不锁表 可读不可写
  - 物理备份时需要保证内存落盘 使用`FLUSH TABLES tbl list WITH READ LOCK`来落盘 同时上锁 可读不可写
  - 使用`--tab`参数可以额外导出一个 csv 文件 使用`SELECT * INTO OUTFILE 'file_name' FROM tbl_name`可以导出文本文件
  - MySQL 支持使用 binary log 来进行增量备份 当进行全量备份时 使用`FLUSH LOGS`来落盘之前的 binlog
  - 使用主从复制 老生常谈了 不赘述 主要是备份方法时切换到从库上进行备份
  - 如果使用 MyISAM 可以使用`REPAIR TABLE`来修复表

#### An Example of Backup and Recovery

- 遇到断电或操作系统宕机时 我们可以认为磁盘数据是安全的 此时 InnoDB 会自动对 log 进行恢复
- 文件系统崩溃或其他硬件问题 导致磁盘数据损坏 需要格式化磁盘 并且需要从备份中恢复
- 假如每周日的 1 点进行一次全量备份 使用`mysqldump --all-databases --master-data --single-transaction > backup_{date}.sql`指令 产生的文件就可以用于恢复 这个备份操作需要一个所有表上的全局读锁
- 使用`--log-bin`启用 binlog 功能 binlog 会定期截断 `--delete-master-logs`可以在全量备份时删除旧的 binlog
- 假如周三的 8 点遇到了灾难 先恢复到周日 1 点的状态 然后使用`mysqlbinlog gbichot2-bin.000001 gbichot2-bin.000002 | mysql`来恢复 假如一个 binlog 是一天 现在恢复到了周二 1 点的状态
- 最后使用`mysqlbinlog gbichot2-bin.000003 ... | mysql`来恢复到周三 8 点的状态 `...`表示周二 1 点到周三 8 点的所有 binlog 文件
- 日志和备份应该存储在和数据库不同的地方 保证安全
- 总结：
  - 若数据库未损坏 InnoDB 会自动使用 log 进行恢复
  - 若数据库损坏 使用以下策略防止数据丢失
    1. 启用 binlog 来进行增量备份
    2. 使用 mysqldump 来进行定期的全量备份
    3. 使用`FLUSH LOGS`来定期把 binlog 落盘

#### mysqldump for Backups

- dump 文件可以用于恢复数据库 也可以用于主从复制 也可以用于实验
- `--tab`会多生成一个 csv 文件
- 各种指令用法详见 ppt

#### Point-in-Time Recovery

- PITR 是指恢复数据库到某个时间点的状态
- 注意 mysqlbinlog 不应该起多个线程并行执行多个 binlog 内的语句之间可能会有依赖关系 正确做法是在一条指令内执行多个 binlog
- 使用`--start-datetime`和`--stop-datetime`来查看小时间段内的 binlog 找到需要的 position
- 使用`--start-position`和`--stop-position`来指定恢复的范围 可以跳过一部分 binlog 这样就可以跳过某些失误的操作

## 11.13

### MySQL partitioning

#### Partitioning Types

- Range Partitioning：根据某个字段的范围来分区 即连续值

  - RANGE 分区示例如下 创建了一个根据 joined 字段的 YEAR 来分区的表 范围规定必须为是单边开区间 方便后续添加新的分区和删除旧的分区进行合并 并且范围是连续升序的

  ```sql
  CREATE TABLE employees (
    id INT NOT NULL,
    fname VARCHAR(30),
    lname VARCHAR(30),
    hired DATE NOT NULL DEFAULT '1970-01-01',
    separated DATE NOT NULL DEFAULT '9999-12-31',
    job_code INT NOT NULL,
    store_id INT NOT NULL
  )
  PARTITION BY RANGE (YEAR(joined)) (
    PARTITION p0 VALUES LESS THAN (1991),
    PARTITION p1 VALUES LESS THAN (1996),
    PARTITION p2 VALUES LESS THAN (2001),
    PARTITION p3 VALUES LESS THAN MAXVALUE
  );
  ```

  - RANGE 也可以在多列上进行分区 比如`PARTITION BY RANGE COLUMNS (a, b, c)` 先比较 a 如果相等再比较 b 如果相等再比较 c
  - NULL 值会被放到最小的分区中

- List Partitioning：根据某个字段的值来分区 即离散值

  - LIST 分区示例如下 创建了一个根据 region 字段的值来分区的表

  ```sql
  CREATE TABLE employees (
    id INT NOT NULL,
    fname VARCHAR(30),
    lname VARCHAR(30),
    hired DATE NOT NULL DEFAULT '1970-01-01',
    separated DATE NOT NULL DEFAULT '9999-12-31',
    job_code INT NOT NULL,
    store_id INT NOT NULL,
    region VARCHAR(10)
  )
  PARTITION BY LIST (region) (
    PARTITION pNorth VALUES IN ('NORTH'),
    PARTITION pSouth VALUES IN ('SOUTH'),
    PARTITION pEast VALUES IN ('EAST'),
    PARTITION pWest VALUES IN ('WEST')
  );
  ```

  - `INSERT IGNORE INTO employees VALUES` 会忽略插入失败的数据 只插入属于某个分区的数据
  - 想要插入不属于任何分区的数据 可以使用`PARTITION pOther VALUES IN (DEFAULT)`来插入
  - NULL 会被当做某一个离散值来处理 如果没有对应的分区则会插入失败
  - RANGE 和 LIST 分区支持以下类型
    - 所有整数类型 包括 TINYINT、SMALLINT、MEDIUMINT、INT、BIGINT 不支持 FLOAT 和 DECIMAL 类型
    - 所有日期和时间类型 包括 DATE、DATETIME、TIMESTAMP、TIME、YEAR
    - 所有字符串类型 包括 CHAR、VARCHAR、BINARY、VARBINARY 不支持 TEXT 和 BLOB 类型 因为是指针

- Hash Partitioning：根据某个字段的哈希值来分区 由用户定义

  - HASH 分区示例如下 创建了一个根据 id 字段的哈希值来分区的表 哈希函数是对于分区数取模

  ```sql
  CREATE TABLE employees (
    id INT NOT NULL,
    fname VARCHAR(30),
    lname VARCHAR(30),
    hired DATE NOT NULL DEFAULT '1970-01-01',
    separated DATE NOT NULL DEFAULT '9999-12-31',
    job_code INT NOT NULL,
    store_id INT NOT NULL
  )
  PARTITION BY HASH (id)
  PARTITIONS 4;
  ```

  - 哈希函数也可以使用线性哈希函数 其扩展分区时需要迁移的数据较少 使用`LINEAR HASH`来指定
  - NULL 会被当做 0 来处理

- Key Partitioning：根据某个字段的哈希值来分区 由 mysql 定义

  - KEY 分区示例如下 创建了一个根据 id 字段的哈希值来分区的表 不指定列则会取主键

  ```sql
  CREATE TABLE employees (
    id INT NOT NULL,
    fname VARCHAR(30),
    lname VARCHAR(30),
    hired DATE NOT NULL DEFAULT '1970-01-01',
    separated DATE NOT NULL DEFAULT '9999-12-31',
    job_code INT NOT NULL,
    store_id INT NOT NULL
  )
  PARTITION BY KEY (id)
  PARTITIONS 4;
  ```

  - 也可以使用`LINEAR KEY`来指定线性哈希函数
  - NULL 会被当做 0 来处理

- Subpartitioning：在分区内再根据某个字段进行分区 示例如下 先按照年份分区 每个年分区里按照天数继续分区

  ```sql
  CREATE TABLE employees (
    id INT NOT NULL,
    fname VARCHAR(30),
    lname VARCHAR(30),
    hired DATE NOT NULL DEFAULT '1970-01-01',
    separated DATE NOT NULL DEFAULT '9999-12-31',
    job_code INT NOT NULL,
    store_id INT NOT NULL
  )
  PARTITION BY RANGE (YEAR(joined))
  SUBPARTITION BY HASH (TO_DAYS(joined))
  SUBPARTITIONS 4 (
    PARTITION p0 VALUES LESS THAN (1991),
    PARTITION p1 VALUES LESS THAN (1996),
    PARTITION p2 VALUES LESS THAN (2001),
    PARTITION p3 VALUES LESS THAN MAXVALUE
  );
  ```

  - 也可以直接声明子分区数量和名称 形如

  ```sql
  PARTITION BY RANGE (YEAR(joined))
  SUBPARTITION BY HASH (TO_DAYS(joined))(
    PARTITION p0 VALUES LESS THAN (1991)(
      SUBPARTITION s0,
      SUBPARTITION s1,
      SUBPARTITION s2,
      SUBPARTITION s3
    ),
    ......
  );
  ```

#### Partitioning Management

- `ALTER TABLE employees PARTITION BY RANGE (YEAR(joined)) PARTITIONS 4;`会重新分区 但是会把数据全部拷贝一遍
- RANGE 分区的范围应该尽量和未来的查询范围相匹配 使得查询可以在一个分区内完成
- `ALTER TABLE employees DROP PARTITION p0;`会删除分区以及其数据 分区的范围仍然会保留连续性
- `ALTER TABLE employees ADD PARTITION (PARTITION p0 VALUES LESS THAN (1991));`会添加一个新的分区 范围必须大于已有的分区 对于 LIST 则不能重复
- `ALTER TABLE employees REORGANIZE PARTITION p0 INTO (PARTITION p0 VALUES LESS THAN (1991), PARTITION p1 VALUES LESS THAN (1996));`会重新组织分区 可以分裂分区 也可以合并分区
- HASH 和 KEY 不允许删除分区 因为数据会被重新分布 但是可以合并分区 使用`ALTER TABLE employees COALESCE PARTITION 4;`来合并分区
- `ALTER TABLE employees EXCHANGE PARTITION p0 WITH TABLE employees_archive;`会用分区交换表 但是表的结构必须和分区的结构一样 也不能有外键约束 不能有分区 表内不能有分区外的数据 加上`WITHOUT VALIDATION`可以跳过分区范围的检查 但是会真的插入不合法的数据 需要后续手动检查
- 还有很多指令可以维护分区表
  - REBUILD PARTITION
  - OPTIMIZE PARTITION
  - ANALYZE PARTITION
  - REPAIR PARTITION
  - CHECK PARTITION

#### Partitioning Pruning

- MySQL 中的 query plan 会自动根据分区的范围来进行分区裁剪 从而减少查询的开销
- 不只 SELECT 语句 也包括 UPDATE 和 DELETE 等语句
- 因此分区逻辑应该尽量和查询逻辑相匹配 使得查询可以在一个分区内完成

## 11.18

### MongoDB & NoSQL

- 为什么需要 NoSQL
  - 数据量超过 10M 后 RDBMS 的性能会急剧下降 因为 B+树的深度会增加
  - RDBMS 无法处理非结构化数据 比如一篇文章的内容
- Big Data
  - 结构化数据 尤其是轻量级的数据 非常适合使用 RDBMS
  - 半结构化数据 比如 JSON XML 等
  - 非结构化数据 比如图片、视频、音频、文本等
- RDBMS
  - 数据量：在 GB 级别
  - 访问方式：交互式 批处理比较难
  - 结构：固定的 schema
  - 约束：强约束
  - 扩展：非线性的扩展性能

#### MongoDB

##### Basic Concepts

- Document：类似于 JSON 的数据结构 由键值对组成
  - 每一个文档都有一个唯一的 `_id` 字段+
  - 键的顺序会区分不同的文档
  - 键不能包含`.` `$` `null` 且不能是空字符串 且不能是`_`开头的保留字
  - 键值是类型敏感和大小写敏感的
- Collection：类似于表 由多个 Document 组成
  - schema-free：不对文档的结构做任何限制
  - 文档的分表需要遵循 data-locality 原则，即相关的文档应该存储在一起
  - Subcollection：类似于垂直分表 但是表之间不会有任何关联
- Database：类似于数据库 由多个 Collection 组成
  - 不同的数据库会被存储到不同文件
  - 保留数据库
    - admin：存储用户信息
    - local：存储本地数据 不会被复制
    - config：存储分片信息 用于剪枝 因为 sharding 是按`_id`来分片的
- 基本命令
  - `show dbs`：显示所有数据库
  - `use db_name`：切换数据库
  - `show collections`：显示所有集合
  - `db.collection_name.find()`：显示所有文档
  - `db.collection_name.find({key: value})`：显示符合条件的文档
  - `db.collection_name.insertOne({key: value})`：插入一个文档
  - `db.collection_name.insertMany([{key: value}, {key: value}])`：插入多个文档
  - `db.collection_name.updateOne({key: value}, {$set: {key: value}})`：更新一个文档
  - `db.collection_name.updateMany({key: value}, {$set: {key: value}})`：更新多个文档
  - `db.collection_name.replaceOne({key: value}, {key: value})`：替换一个文档
  - `db.collection_name.deleteOne({key: value})`：删除一个文档
  - `db.collection_name.deleteMany({key: value})`：删除多个文档
  - `db.collection_name.aggregate([stage1, stage2, ...])`：聚合查询 可以进行统计、分组、排序等操作
  - `db.collection_name.drop()`：删除一个集合
  - `db.dropDatabase()`：删除一个数据库

##### Spring with MongoDB

- 需要导入 spring-boot-starter-data-mongodb 依赖
- 需要在配置文件中配置 MongoDB 的连接信息：`spring.data.mongodb.uri=mongodb://localhost:27017/test`
- 需要在实体类上加上`@Document`注解
- 需要在 DAO 层继承 MongoRepository 接口
- 结合 JPA 时 将不需要 JPA 维护的字段加上`@Transient`注解

##### Indexing

- Normal Index：`db.collection_name.createIndex({key: 1})`：升序索引 `db.collection_name.createIndex({key: -1})`：降序索引
- Geospatial Index：`db.collection_name.createIndex({key: "2dsphere"})`：地理位置索引 值需要是含两个数值的数组

- 上述索引都可以在 Mongo Compass 中进行可视化创建
