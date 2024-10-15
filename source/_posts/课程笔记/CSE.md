---
title: 《计算机系统工程》课程笔记
date: 2024-09-19 14:06:41
categories: 课程笔记
tags:
  - 分布式
index_img:
banner_img:
excerpt: " "
---

## LEC 2: The Case for A Distributed System

- 现代互联网应用具有的特征：
  1. 高请求率 每日数以百万计的请求
  2. 大量的数据 包括图片视频
  3. 对用户透明：用户不会知道服务器的状态 即无感知
  4. AI 带来的新需求：需要高算力（GPU）

### Case Study：以淘宝电商平台为例

- 旧时代的单主机部署：LAMP（Linux+Apache+MySQL+PHP）这种架构无法扩展 单主机的资源始终是有限的

- 如何扩展架构性能？

  1. 分解应用：做出专门定制的硬件 比如专门用于存储数据的服务器 将架构分解为应用、文件系统、数据库三台服务器 通过内网连接

  2. 缓存：为了避免频繁的数据访问 进行缓存 加快处理速度 和 1 同理 也可以做出专门作缓存的内存特化服务器

     后来也就产生了几个分布式缓存系统 比如 Redis 和 Memcached 而分布式带来的问题是 键的哈希方法 如果用传统的取模哈希 增加新的服务器时就会需要大量的重写 于是采用了 Consistent hashing 的方法（后面会细讲）

  3. 更多的应用服务器：无状态的应用服务器是很容易扩展的 因为只需要一模一样的复制即可 而有状态的应用（比如用户的会话状态）比较复杂 需要在分布式中进行数据的同步

     对于分布式的应用服务器 需要负载均衡 可以使用 http 重定向、反向代理等方式，轮询、随机、哈希等策略来实现

  4. 扩展数据库：两种方式 一是做主从复制 二是分库分表

  5. 分布式文件系统：比如 NFS、GFS 等 和分布式数据库比较类似

  6. 使用 CDN：通过内容分发尽可能的减少网络延迟 是一种变相的缓存

  7. 分离不同的应用：比如阿里分为了支付宝和淘宝 可以使用 k8s 进行集群部署

- 分布式系统的 fault 会很多 所以我们希望分布式系统有较好的容灾能力 比如个别集群内的服务器宕机 整个系统仍然可以正常使用

- 分布式系统需要实现高可用性

  - 复制备份：复制一样的系统做同样的事 来作为当前系统的备份 困难是如何维护一致性 比如写完 A 后 还没有复制到 B 就崩溃了
  - Retry 重试：发一次请求失败后再发一次 如果能得到正常响应即可视为正确

- 分布式系统的 CAP 理论

  - Consistency、Availablity、Partition 三者不可能兼得
  - 举例：（不严谨 实际上有更严谨的论文证明）

    假如有两个分区 S1 和 S2 它们之间的连接完全断开 为了符合 P 用户 C 将仍能正常使用 S1 和 S2 接下来 C 给 S1 发送一条`A=V1`的请求 为了符合 A S1 必须立即响应请求给 C（否则就相当于有一段时间系统不可用）如果这时 S1 和 C 的连接断开 C 将只能向 S2 进行查询 此时获取的数据就是和之前不一致的了（因为 S1 没办法和它保持一致）

  - 因此我们必须要做权衡 P 是比较必要的 因为不能指望单主机可以做到可靠 因此在 A 和 C 里进行二选一
  - 淘宝这样的应用就会选择 A 尽管可能会产生数据不一致（比如一本书卖个了两个人） 带来的损失也不大（退款呗）
  - 支付宝这样的应用就要选择 C 毕竟转账支付不能产生不一致 要不然会被钻漏洞或者起诉 这也是为什么支付宝转账会比较久（在同步数据）
  - 当然 如果没有网络故障 CAP 还是可以达到的 理论只是告诉你一个分布式系统不可能永远保持 CAP

- 分布式系统需要哪些属性（审视系统的五个维度）？
  1. 可扩展性
  2. 性能
  3. 故障容忍度
  4. 一致性
  5. 易用性

## LEC 3: File System 1

### iNode-based File System（单机文件系统）

- 一个文件有两个属性：持久化且有名字

- 磁盘会提供最简单的读写数据两个 API 而驱动会抽象成文件系统需要使用的更复杂的 API

- 文件为了可扩展性 需要设计成分布式的存储 即不连续的 sector 那就得使用元数据来存储这个 sector 属于哪个文件

- L1：Block 层

  - block 是文件系统中数据管理的基本单元 包含连续的多个 sector 可以调整大小
  - block 大小不能太大 会有空闲 不能太小 会浪费效率
  - 第一个 block 叫做 super block 用于存储文件系统的元数据 比如 block 大小、空闲的 block 数等 后面跟着一个 bitmap 存储所有 block 是否空闲 不过这样要查找一个空闲块就需要遍历了

- L2：File 层

  - inode（index node）会存储文件的元数据 包括这个文件有哪些块 inode 对于大文件来说会很大 所以会做多层映射（类似多级页表）

- L3：inode Number 层

  - 一个 inode table 存储 inode 的位置

- L4：File Name 层

  - 提供文件名到 inode num 的映射 便于根据文件名查找文件
  - 这个映射存储在一个文件里 名为 directory（目录）

- L5：Path Name 层

  - 即使用`/`区分的文件路径

- L6：Absolute path name 层

  - 引入根目录`/` 根目录的 inode 为第 1 个

- L7：Symbolic Link 层（软链接）

  - 硬链接：
    - 即为长路径文件创建快捷方式 相当于创建指向同一个 inode number 的指针
    - 为了防止两个快捷方式指向同一个 inode 而导致的影响 有一个 refcnt 记录 inode 有多少硬链接引用
    - 为了防止 link 成环带来的影响 dr 指令删除文件夹是必须为空的 且文件夹不能指向文件夹
    - `.`和`..`都是硬链接
  - 为了实现跨文件系统（比如多个硬盘）引入软链接：
    - 实际上软链接记录了路径这个字符串 而不是实际的指针 因此文件即使不存在也可以创建软链接

- Rename

  - 重命名时实际上是创建新文件后删除原文件 防止程序崩溃导致丢失

- Summary
  - 文件名不是文件的一部分 而是存在 directory 里的一个字符串 正因此 重命名实际上只能通过创建新文件来实现
  - 硬链接是等价的 不存在先后主次
  - directory 存储了各个文件名 是很小的 不像电脑里展示的那样 文件夹大小包含文件

## LEC 4: File System 2

### File System API

- OPEN/READ/WRITE

  - inode 中包含的文件元数据除了之前提到的 还包括属于哪个用户/组 可读写执行 三个最后修改的时间戳（读 写 链接）

  - 打开一个文件需要：

    - check 用户权限
    - 更新访问时间戳
    - 返回一个 fd 描述符（fd 也可以表示其他硬件 比如键盘显示器等）

  - 为什么返回 fd 而不是其他的选择？

    - 如果返回一个指针 用户态就可以访问内核态的结构 并且可以通过偏移量访问其他未检查的文件 十分不安全
    - fd 将会完全控制用户打开文件的过程和权限

  - Flie Cursor

    - 记录这个 fd 当前读到的位置

  - fd table 和 flie table

    - 每一个进程都有自己的 fd table 所有进程共享一个 file table（打开文件表）子进程会继承父进程的 fd 表 因此共享 cursor 位置

  - 由于每次 read 都会导致对于 inode 的写（要修改 atime）linux 提供参数 no-atime 使得当最后关闭文件时才进行修改

  - 由于数据泄露的问题 写时顺序应该采用更新 block bitmap、更新 inode、写入新数据 虽然此种顺序在极端情况下会产生硬盘浪费 但是可以通过扫描磁盘来恢复

- SYNC

  - 为了保证数据落盘 必须提供这个用于同步的 API

- 删除一个打开的文件

  - 在 linux 系统下 文件的 inode 将会在 refcnt 归零后进行删除 也就是延迟到关闭文件后删除

- MALH
  - 模块化
  - 抽象化
  - 分层化
  - 层级化

## LEC 5: Remote Procedure Call

分布式的数据存储 形如让几十台电脑的所有硬盘空间汇聚到一块 然后所有电脑通过远程连接的方式访问文件系统 这样就实现了高效的数据存储利用 这也是 RPC 的产生背景

### An Example

- 本地调用函数会变成对于远端服务器上函数的调用

- RPC Stub（把底层的代码进行封装 从而对于高层的应用就透明了）

  - client stub
    - 放入参数
    - 发送请求
    - 等待响应
  - server stub
    - 监听消息
    - 获取参数
    - 调用过程
    - 将结果放入响应
    - 发送响应
  - 通过 stub 我们在不修改原来函数的情况下实现了远程调用

- A Message May Contain：

  - 服务 ID
  - 服务的参数
  - 序列化与反序列化

- RPC request：

  - Xid：事务 id
  - call/reply
  - rpc version
  - program number：哪个二进制文件
  - program version
  - procedure number：哪个函数
  - auth stuff：权限
  - arguments

- RPC reply
  - Xid：事务 id
  - call/reply
  - accepted：是否接受请求 根据版本号、权限认证等
  - auth stuff
  - success：是否请求成功 根据文件号和过程号是否合法
  - results

### How to Pass the Data？

- Parameter passing

  - client 把内存中的对象序列化为无指针的结构
  - 发送数据
  - server 反序列化为内存中的对象并建立指针

- 不同机器间的数据形式会有很多差别

  - 大端小端
  - 32 位 64 位
  - 浮点数的格式
  - 字符集 utf 或其他
  - 对齐方式

- 标准化的编码

  - textual：json、xml、csv 等 存在二义性 且不适用于二进制数据
  - binary：更快更小 但是不可读 对于用户不友好 会通过 IDL 来定义结构

- 传输协议
  - TCP：可靠 但是慢
  - UDP：快 但是不可靠

### When RPC Meets Failure

- semantics（语义）

  - at most once：保证请求最多执行一次
  - at least once：保证请求至少执行一次
  - exactly once：保证请求只执行一次

- idempotent

  - 一个函数如果多次执行结果是一样的 就是幂等的
  - RPC 系统会保证幂等性 使得错误处理可以用简单的 retry 来解决

## LEC 6: Distributed Filesystem

- Distributed File Service Types
  - Upload/Download：比如 FTP 通过上传下载文件来实现 浪费资源
  - Remote Access：比如 NFS 通过远程访问文件来实现 用 RPC 接口

### NFS with RPC

- NFS 即 Network File System
- NFS 将大部分的接口替换为了 RPC 接口 但是没有 OPEN 和 CLOSE 接口 READ 和 WRITE 接口则多了一个 offset 参数 MKDIR 返回的多了 fh（file handle）参数
- NFS Protocol Steps:
  1. Mount：客户端通过 mount 命令将远程文件系统挂载到本地 形如`mount 192.168.1.1:/home /mnt`
  2. Read a file：应用调用 OPEN("f",O) NFS 会调用 LOOKUP(dirfh,"f") 把结果填到 fd 中返回给应用 应用调用 READ(fd,buf,n) NFS 会调用 READ(fh,offset,n) 返回结果给应用 最后应用调用 CLOSE(fd)
- 为什么 NFS 不提供 OPEN 和 CLOSE 接口？

  - 因为 OPEN 是有状态的 会创建类似于文件描述符的数据 而 LOOKUP 是无状态的 有状态会导致服务器重启后客户端的状态丢失 我们希望把状态保存在客户端 这也是为什么 READ 和 WRITE 多了一个 offset 参数 本质上就是把状态保存在客户端的 offset 中

- 什么是 fh（file handler）？
  - 本质上就是一个 remote file 的标识符
  - 不能使用 fd 因为 fd 是有状态的
  - 不能使用文件路径名 可能出现这种情况：客户端 1 打开文件 客户端 2 进行重命名 产生了不一致
  - 不能使用 inode number 可能出现这种情况：客户端 1 打开文件 客户端 2 删除文件后创建文件 分配的 inode number 一样 客户端 1 就会访问到错误的文件 原本在单机文件系统中由于文件打开 inode number 不会释放后分配给新文件 这种情况就不会发生
  - 因此 NFS 在 inode number 的基础上加了一个 generation number 相当于版本号 保证了文件的唯一性 以此作为 fh
- READ 和 WRITE 为什么要用 offset？

  - 为了实现幂等性
  - 为了保证无状态的服务端 把状态保存在客户端

- Performance Overheads

  - 通常比本地文件系统慢 但不是绝对 和服务器性能与网络速度有关
  - 优化
    - 缓存到客户端 减少远程操作 或者服务端缓存到内存里 不过如何检测本地的缓存是否过期是个问题
      - close-to-open consistency：关闭文件时写回到服务器 重新打开时再读取
      - read/write consistency：读写时检查文件是否过期
    - Read-ahead：预读取 读取一个文件时预读取后面的文件

- Drawbacks
  - Capacity：只能在单个服务器上放硬盘 无法扩展
  - Reliability：服务器宕机后 NFS 无法使用
  - Performance：单个服务器的带宽有上限

### 如何把单机文件系统变成分布式文件系统？

- Step 1：Distributed block layer
  - 把 block_id 改为(block_id,mac_id)的形式
  - 如何找到空闲的 block？用一个 master server 记录哪些 block 是空闲的 可以存在内存里
- Step 2：Distributed file layer
  - 几乎不用改 因为 block layer 已经解决了大部分问题
- Step 3：Distributed inode number layer
  - 同理 用 master server 记录 inode number 的分配情况
- 后续层几乎不用改
- 可以看到我们使用一个 master server 几乎解决了所有问题 但是这样会导致单点故障 性能瓶颈等问题
- 使用主从复制 可以解决单点故障问题和读性能的提升

### Case Study：GFS(Google File System)

- GFS 采用了 master/slave 架构 一个 master 多个 chunk server
- Why Large Chunks?
  - 减少 master 的负担
  - 减少TCP连接的数量
  - 可以把元数据放在master的内存里

