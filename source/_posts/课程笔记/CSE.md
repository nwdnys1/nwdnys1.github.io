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
- 进行 replication 解决了容量 备灾和性能问题
- Why Large Chunks（64MB）?
  - 减少 master 的负担
  - 减少 TCP 连接的数量
  - 可以把元数据放在 master 的内存里
- 新增了 append 和 snapshot 操作
- 对于并发写 GFS 保证最终 chunk 的一致性 但是不保证读数据一定是最新的
- GFS 会确定一个 primary chunk server 来确定每个写的顺序
- 2 阶段写
  1. 发送数据但不写入 获得复制的列表 chunkserver 会以链式的方式进行传输 比如 1 收到数据后传给 2 2 收到数据后传给 3
  2. 发送请求给 primary chunk server 进行顺序的写入
  - 这是一种数据流和控制流分离的设计 一阶段是数据流 二阶段是控制流
- GFS 的重点之一是 append 即在文件末尾追加数据 希望并发的追加操作不会产生冲突 互相覆盖
- GFS 的 naming 是单纯的 flat naming 没有目录结构

## LEC 7: Key-Value Store

### 键值系统

- 为什么不用文件系统？

  - 对于小数据 文件系统的开销太大 比如一次 GET 需要 OPEN+READ 另外文件系统的最小单位是 block 对于小于 block size 的数据会浪费空间
  - 为了解决上面两个问题 可以把多个键值对放在一个文件里 形成简单的键值存储系统

- 一个基于文件系统 API 的键值存储系统

  - 键作为字符串存储 值作为 json 格式字符串存储 键和值用逗号分隔 每个键值对用换行符分隔
  - UPDATE 操作选择 APPEND 而不是修改
    - 覆写需要磁盘进行随机访问 速度很慢
    - 磁盘的顺序写很快 比随机访问快一个数量级
  - INSERT 同理也是在文件末尾追加 查询时从后往前查询
  - DELETE 同理 用一个标记位来标记删除 比如 null
  - 这样的设计叫做 Log-Structured 即日志结构化的设计 特征是 append-only
  - GET 操作需要从后往前查询 直到找到对应的键值对 线性复杂度 非常慢
  - 使用 INDEX 来加速查询 记录 key 到文件 offset 的映射 可以使用 B+树或者哈希表 查询速度可以达到 O(logn)或 O(1)
  - 但是前提是 INDEX 要放在内存里 通常 INDEX 本身就会非常大 必须把 INDEX 放在磁盘里
  - 使用 Cuckoo Hashing 减少冲突 保证 O(1) 的查询时间 UPDATE 也是 O(1)的 但是 INSERT 需要驱逐 会涉及很多 IO
  - 无限制的增长会导致空间不够 使用 compaction 来合并相同键的值 回收过期的键值对

- 范围搜索：使用 B+ 树索引

  - B+树的叶子是相连的 因此可以找到下限节点后顺序读
  - B+树的 INSERT 和 UPDATE 是比较繁琐的 因此还是不宜使用

- LSM 树（Log-Structured Merge Tree）

  - SSTable：Sorted String Table
    1. 在 SST 中搜索一个 key 是很高效的 因为 SST 是有序的
    2. 支持范围搜索
    3. 合并 SSTable 是很高效的 使用归并排序即可
  - MemTable：内存中的表
    1. 溢出后全部写入 SSTable
  - Drawbacks

    1. 查找旧值很慢 需要查找多个 SSTable
    2. compact 很慢
    3. 范围搜索仍然不快

    解决方案：确保 SSTable 之间的 key 没有重叠 这样可以快速判断 key 是否存在这个 SSTable 中（判断 maxmin）

  - Crash Recovery：维护 memtable 的 WAL（Write-Ahead Log）
  - Write Stall：当写入一个值时 触发很多次 compaction 导致延迟变高
  - Write Amplification：写放大 实际对于磁盘的写入量远远大于数据量
  - 查找不存在的 key 很慢 因为要查找所有 SSTable 可以通过 Bloom Filter 来加速

## LEC 8: Distributed Storage: Sequential Consistency

### How to deploy a KVS?

- 以微信为例 数据会部署在服务器和客户端上 就和我暑假做的日程 APP 一样 最大的重点是写时的数据同步

### Consistency Model

- Strong Consistency
  - 所有数据只有一个版本
  - 读写等价于串行化的读写（线程之间的顺序可以不一样）
  - 根据等价于哪种串行化 分为：
    - Strict Consistency：串行化顺序符合发生时间顺序 没有人实现这种模型 因为会有网络延迟的区别 机器的时钟不同步等问题
    - Sequential Consistency：对于各个线程内的操作 保证串行化顺序和发生时间顺序一致 对于不同线程则不保证
    - Linearizability：如果 B 在 A 结束之后开始 串行化顺序中 B 一定在 A 之后
- Release Consistency
- Eventual Consistency：所有服务器的数据最终会达到一致

### 如何实现 Linearizability？

- local property：如果一个数据的所有操作是线性化的 多个数据的操作组合也是线性化的
- Primary-backup：对于写操作 primary 转发给所有 backup 全部写入后返回 最终在 primary 里写入 对于读 返回 primary 的数据 而不能读 backup 的数据 否则会产生不能线性化的情况
- Partitioning：把不同数据分区到不同的 primary 上 来解决只有一个 primary 导致的性能瓶颈问题

## LEC 9: Distributed Storage: Eventual Consistency

### 如何实现 Eventual Consistency？

- Write-write conflict：不同设备写入同一个数据产生冲突
  - 为了合并冲突 需要 append 操作 使用 update function 可以定义任意的数据库操作
  - 通过 Log 来统一不同设备间的操作顺序：在实际写入之前先写入 Log 同步 Log 之后排好序再执行实际写入
  - 可以通过时间戳来排序 如果时间戳一致则比较设备 ID
- 为了用户体验 需要在同步前先执行本地 UPDATE 这样会产生问题 如果本地更新失败了怎么办？可以使用 Rollback and Replay 来解决
- **Rollback and Replay（没听懂 10.24）**：本地的 UPDATE 会记录状态 同步前 把所有本地的 UPDATE 回滚 然后把所有的 UPDATE 重新执行一遍

  问题在于如果时间久了 会有很多 UPDATE 需要回滚和重放 需要把 log 中 stable 的写提前 apply 到数据库中 减少回滚和重放的次数

  如果上次同步时所有设备的时间戳都大于这个写的时间戳 则是 stable 的

- Commit Scheme：一台 primary 服务器 分配一个全局的提交顺序

- Causal Ordering：在一台机器上 A 发生早于 B 或者是 A 触发了 B 的操作（比如 A 是添加数据 B 是删除数据）则 A 和 B 有因果关系
- Lamport Clock：每个操作都有一个时间戳 这个时间戳会保证 如果 A 是 B 的因 则 A 的时间戳小于 B 的时间戳 如果两个事件无因果关系（我们称其为并发事件） 则时间戳可能相等或无序

  - 算法：

    1. 每个服务器保留一个时钟 T
    2. 随着时间的推移 T 会增加
    3. 收到一条消息的时间戳为 T’ 则更新 T 为 max(T,T’+1) 也即保证时间戳大于已经发生的事件

  - 问题

    1. 时间戳可能相等 可以通过设备 ID 来解决
    2. 事件 A 发生在事件 B 之前 => A 的时间戳小于 B 的时间戳 但是反之不一定成立 因为每个设备的时间戳是独立的 换言之 没有办法确定 A 和 B 是因果关系还是并发事件

- Vector Clock：用一个 n 维向量来表示时间戳 每个维度代表一个设备的时间戳 T[i]严格大于（即每个维度都大于）T[j] 则 i 发生在 j 之后 若无法比较 则是并发事件

  - 算法：

    1. 若有 n 个设备 则每个设备有一个 n 维的向量代表时间戳
    2. 每个设备的时间戳初始化为全 0 的向量
    3. 事件 A 在设备 i 上发生时 设备 i 的时间戳的第 i 位加一
    4. 设备 i 向设备 j 发送消息时 把自己的时间戳 W 发送给对方 设备 j 把第 k 位时钟 更新为 max(W[k],V[k] + 1)

## LEC 10: Distributed Storage: All-or-nothing Atomicity

### Consistency under single-machine faults

- 任何调用都应该是 all-or-nothing 的 即要么全部成功 要么全部失败 否则会很难处理错误
- 以 bank transfer 为例 数据的一致性即保持账户总额不变
- 保持原子性的方法

  - shadow copy：保证对于一个文件的修改是原子的 通过写入一个新文件 然后重命名的方式来实现 因为重命名是原子的 虽然不会导致数据不一致 但是文件系统有可能会不一致

    Drawbacks：不支持并发 难以操作多文件 拷贝开销大

  - journaling：上面的问题可以在 rename 之前写入 WAL 来解决 journaling 会导致极大的额外空间开销 所以可以只对 metadata 进行
    - 如果在提交 journal 时崩溃怎么办（假设 journal 大于 sector size 所以写入并不是原子的）？
  - Logging：可以见 AEA 事务笔记
    - 日志会无限增长 所以还是需要截断 通过 checkpoint 机制：
      1. 等到没有任何操作在进行（而不是等到所有事务结束）
      2. 把 CKPT 记录到日志中 CKPT
      3. 把所有内存中的数据写入磁盘
      4. 丢弃除了 CKPT 之外的日志
    - Undo-redo vs Redo-only vs Undo-only
      - redo 的 log 内容少 所以性能会更好 redo 时也只需要扫一遍日志
      - undo-redo 对于小内存比较差
      - undo-only 几乎不会用到

## LEC 11: Distributed Storage: Before-or-after Atomicity & 2PL

- race condition：多个线程同时访问一个数据 由于操作包含多个步骤 会导致数据竞争
- before-or-after atomicity：也叫 isolation 保证一组读写操作是原子的 不允许其他线程看到中间状态

### Use Locks to achieve Isolation

- 用全局锁可以严格保证原子性 但是会导致性能瓶颈 粒度太粗
- 对于每一个数据加锁来减小粒度 然而没法保证类似于 transfer 操作的原子性 中间状态会被看到 解决方法：获取锁时固定顺序获取 先全部获取再释放 相当于一把大锁
- 2PL（Two-Phase Locking）
  - 每个数据有自己的锁
  - 在需要操作数据之前再获取锁
  - 所有释放延迟到一个完整的 action 结束后 而不是在每个数据操作后立即释放
- 细粒度锁对于每一个数据加锁 但是通常数据量太大 可以把所有数据哈希到几个桶里 每个桶加一把锁

### Before-or-after Atomicity 的严格定义

- Serializability：一系列事务的操作等价于它们串行执行的结果
  - Final State Serializability：最终状态等价于串行化的结果
  - Conflict Serializability：最常用的
  - View Serializability
  - Confilct ∈ View ∈ Final State

### Confilct Serializability

- 2 opreations confilct if：

  1. 操作了同一个数据
  2. 至少有一个是写操作
  3. 它们属于不同的事务

- Conflict Serializability：如果 conflicts 的顺序和某一个串行化的顺序一致 则是 conflict serializable 的

- Confilct Graph：每个节点是一个事务 如果 T1 和 T2 有冲突 且冲突的第一步是 T1 的操作 则有一条边从 T1 指向 T2
  - 只要没有环 则进行线性化后 一个拓扑排序就是一个串行化的顺序 使得 schedule 是 conflict serializable 的
  - 反之 如果有环 则一定不是 conflict serializable 的
  - Conflict Equivalence：如果两个 schedule 的 conflict graph 是同构的 则是 conflict equivalent 的

### View Serializability

- View Serializability：如果最终的写状态和中间的读状态可以等价于某个串行化的结果 则是 view serializable 的