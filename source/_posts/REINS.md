---
title: SJTU-REINS
date: 2024-10-05 21:19:00
categories: REINS
tags:
  - 实习
  - 开发
  - 架构
  - 运维
  - 测试
index_img:
banner_img:
excerpt: "大三顺利进入REINS打工 记录一下项目经历和经验"
---

# PHAROS

灯塔项目是一个类似于海上管理船只位置的云系统的项目 只不过空间由海域变为上海市区 对象由船只变为无人机等移动物联网设备

简单来说 灯塔系统会收集处于三维空间内各个个体的数据 并存储在一个云边融合系统（CEDS）内 之后将数据整理封装为态势数据 将态势数据分发回给每个个体 让个体通过实时的态势数据进行任务的规划和调度 态势数据会分为两部分 一部分是以个体为中心 在单位时间内可以安全独占的空间范围 另一部分是以个体为中心 依据任务需要而划分的一块空间范围内所有其它个体的数据

灯塔的核心思想是流控分离 即中心会分发数据流给个体设备 但是不会干涉个体设备的任务规划和调度 只是给出了安全空间的限制 个体设备会根据态势数据自行调整任务的路径 和现有的大部分统筹任务路线规划是不同的

## CEDS

CEDS 即黄子昂学长的硕士毕业论文中提出的云边融合存储系统 利用了边缘节点本身少量的数据存储与计算能力以及其分布于移动物联网设备附近的特点 实现了基于数据的就近存储、查询任务的划分下推以及负载感知的数据迁移功能的存储系统 做到了传输带宽的减少以及查询的加速 具体实现可以见其论文 大概包含以下几个部分：

### 边缘数据就近存储机制

大部分云边融合系统会将数据存储于资源富足可扩展的云端数据库中 但这会带来很多不必要的传输开销 利用边缘节点本身的存储能力 可以将数据就近存储 大大减少传输延迟

然而数据的就近存储会带来其他问题 比如原本全都位于云端中心的数据 由于就近存储就会分布在不同的边缘节点上 针对此问题 CEDS 提出一种基于 Rosetta 过滤器的全局索引机制 简单来讲 这个过滤器能够实现一个数据表内任一字段的范围过滤 即给定一个字段的某一范围 在常数时间内返回这个表是否存在这个范围内的数据 基于此种过滤器 加上基于时间划分的数据分表 可以做到在很短的时间内定位到某一个时间范围内的数据分表 并快速判断哪些分表内存在本次范围查询相关的数据 这就是 CEDS 的全局索引机制 基于此种索引 CEDS 在云端仅需存储数据表的元信息摘要（包括过滤器的 bitarray） 同时这也将为后文的查询下推提供便利

### 查询任务拆分下推

大部分云边存储系统基于边缘节点的存储数据 每次查询会将相关的数据表全部聚合到云端后进行筛选 产生了很多不必要的传输开销 CEDS 基于任务划分 将查询和聚合任务下推给各个相关的边缘子节点 减少带宽开销

具体而言 一次查询会先向云端中心索要本次查询可能涉及到的所有数据分表 以及这些分表存储在哪些边缘节点上 其方法可以简单的由之前提到的过滤器实现

获取到节点列表后 只需要把子查询任务分发给这些节点即可 每个子节点会进行相关的范围查询（节点内存在常规的索引来加快查询）并把查询结果返回 最终在查询节点进行数据聚合任务 把结果返回给用户

### 负载感知数据迁移

数据就近存储定会存在不均衡现象 由于查询任务划分给子节点并发执行 查询的延迟显然由查询时间最大的子查询决定 如果某些节点上的数据查询次数特别多 导致超出边缘节点的承受范围 导致子查询阻塞 就会导致所有涉及的查询延迟显著变高 为了防止此种情况出现 CEDS 会监测并记录每段时间内边缘节点的查询资源开销 比如记录前 3 分钟内某一节点的内存占用 一旦超过 1G 就判定存在热点数据 并将热点数据迁移到云端来减缓此节点的压力

此种策略是折中的策略 理论上可以达到最均衡的方案肯定是为每一个边缘节点根据查询开销动态分配性能资源 但这在现实中肯定不可行 因此使用较为折中的策略来解决数据不均衡分布的问题

## 技术调研

### Redisearch

- 项目使用了 redisearch 建立设备数据的地理索引 实现实时数据查询的加速 即把经纬度作为 GEO 类型的元素编写索引
- 具体实现
  - 文档参考：https://redis.io/docs/latest/develop/interact/search-and-query/indexing/（redis不同类型的索引如何使用）https://redis.readthedocs.io/en/stable/examples/search_json_examples.html#Projecting-using-JSON-Path-expressions（将 JSON 数据添加到索引的示例代码）
  - redis 实例需要使用 redis-stack 项目中使用 python 因此安装 pip 库的 redis 即可
  - 具体代码见项目中的`realtime_map.py`文件 只需要先创建 schema 描述索引的字段类型 然后添加索引即可

### Apache Avro

- 项目使用了 apache avro 进行数据的压缩 包括：
  - 设备传输数据给 realtime_map 与 data_receiver
  - 节点计算态势数据时向其他节点发送`/query`请求查询 reatime_map
  - 节点通过消息队列传输态势数据给中心
- 具体实现
  - 文档参考：https://avro.apache.org/docs/
  - python 需要安装 avro 库 java 则通过 maven 导入 avro 依赖
  - 需要预先编写好数据的 schema 语法见文档 编写好后即可使用 avro 的序列化与反序列化方法进行数据的压缩与解压
  - python 中数据类型比较自由 使用 dict 类型即可操作 但是 java 的数据类型比较严格 avro 本身提供一个 record 类封装数据 但是我使用了 fastjson 库的 json 类型来操作数据 这就导致必须要编写一个转换函数进行转换 并且这个转换需要递归分类进行 一旦类型对应不上 序列化时就会报错

### 时空索引

- 时空索引是一种将时空数据进行编码以便于快速查询的技术 我们可以先简单认为时空索引需要解决这样的查询：查询某一个时间段内某个空间范围内所有的数据
- 先从空间索引开始说起 目前主流的空间索引可以分为三种：哈希、树以及空间填充曲线
  - 哈希思想主要是通过网格哈希索引来实现的 也就是把已有的地理空间划分为网格 每个网格相当于一个哈希桶 哈希索引里就会存储每一个网格内有哪些数据 假如需要查询某一个范围内的所有数据 只需要先根据范围找到对应网格 然后再在哈希桶内找到对应数据即可 这个方案偏理论 实际工程不太用 对于灯塔项目 一个网格可能会有很多数据 尤其是不同时间段的数据 会导致哈希桶内数据量过大 失去意义
  - 基于树的索引包括四叉树、R 树等
    - 四叉树仍然是一种网格索引 但是它是一种递归的网格索引 不断四划分空间直到触发阈值为止 对于范围查询 四叉树需要递归的找到所有被包含的节点
    - R 树是 B 树在高维空间的扩展 其结构很简单 适合静态空间对象的索引
      - 先引入几个概念 BB 即 Bounding Box 是一个包含了空间对象的矩形 MBB 则是最小的 BB
      - R 树的非叶子节点存放(BB,ptr)的键值对 叶子节点则存放(MBB,ptr)的键值对 叶子节点的 ptr 直接指向了空间对象数据的位置 其可以是质点 也可以是几何对象
      - 需要进行范围查询时 R 树和四叉树一样 需要递归寻找被范围包含的节点
  - 基于降维的索引通常使用空间填充曲线来实现 GM 部分已经详细讲过了 简单概括就是使用递归二分把高维数据映射到 bytearray 然后 Geohash 会通过 base32 编码得到字符串
- 加入时间维度后 有三种主流的实现方式 即基于时间分片、基于空间分片以及时空维度同时进行索引
  - 基于时间分片是目前最主流的时空索引方式 比如 ES 使用空间填充曲线、PG 使用 R 树作为空间索引等 然后数据按照时间进行分片 个人认为如果灯塔的历史数据分析不需要跨很大的时间段 是可以使用这种方式的
  - 基于空间分片即把空间网格化后分表 由于查询需求 很少被实际应用
  - 时空同时进行索引 在 GM 中的 Z3 索引中有实现 大致可以理解为把时间也进行二分后和空间数据一起编码为 bytearray 在时空尺寸不匹配时会有严重的空间放大问题 详见京东 JUST 的论文
  - 另外 轨迹数据比较适合空间填充曲线 R 树等基本可以不考虑 空间填充曲线若使用 B+树 可能仍存在写入性能差的情况 但是很多非关系型数据库采用的是 LSM 树结构 写入性能会好很多
- 综上 要么基于时间分片 要么时空一起编码
  - 实际上 时空一起编码的索引 还是需要按时间分片 否则会有严重的空间放大问题
  - 既然如此 就变成决定时间参不参与编码了
  - 从感性上讲 我觉得时间参不参与不太影响查询性能 对于一个时空查询 其先会按照时间被分到不同分片上 中间的分片都是占满了时间段的 因此时间编码失去意义 只有两端的分片是可以用时间编码剪枝的 如果我们的查询时间范围总是大于一个分片的时间范围 那么时间参与编码意义是不大的
  - 另外还有一个查询是直接 scan 还是先递归到基本分辨率后再 scan 的问题 我认为直接 scan 即可 因为磁盘上的随机访问仍然需要扫过未查询的数据 只不过省去了判断数据是否符合条件的过程 而递归查询范围这个过程可能本身就比较慢 JUST 是直接 scan 的
- 参考：https://zhuanlan.zhihu.com/p/663029637

#### GeoMesa

- GM 本身是一个类似于索引引擎的工具 首先需要选取合适的存储系统 要么 Redis 要么 HBase 等数据库 或者也可以直接通过 FS 存储 但是性能较差
- 现有的 redisearch 只支持 2d 数据的复合索引 要引入高度的话 可能只能遍历筛选 取决于独占空间部分需不需要 3d point
- 我们使用 GM 的目的是为了加速大量历史数据的查询 进行对历史数据的分析 而对于数据的写入与实时性并无要求 那么有两种方案
  - 先考虑如何使用 mongodb 存储数据 然后通过 GM 进行查询
  - 一种是保留现有的 mongodb 数据库 也就是近期数据仍向 mongodb 写入 然后定期（比如 1 天）把数据迁移到适合存储大量时空数据的数据库中（比如 HBase） 并且这个迁移过程通过 GM 实现（比如把 mongo 数据批量读到内存里 在通过 GM 写入 HBase）也就自动创建了时空索引 后续查询也就提高了性能 （代码量会小一点）
  - 另一种则是抛弃 mongodb 直接通过 GM 持续写入数据 大致流程就是通过 mqtt 接受消息 把消息里的数据解析为 GM 的 simplefeature 类 然后通过 writer 写入 HBase 里 自动建立索引 省去了 mongodb 的部分 但是代码量会大一点
- 方便切换其他数据库指的是最后存储历史数据的数据库 索引层可以固定
- GM 现有的 API 都是大型分布式数据库 如果要测试 可以先部署两个容器试试
- GM 支持空间索引和时空索引 其中时空索引支持 2d point+时间 或是 2d 非 point（polygon line 等）+时间 其中 POINT 类型不知道是否支持 3d redis 由于本身不支持 3d 数据类型 所以没法存储 HBase 还没研究
- 原理部分
  - GM 如何写入数据？（以 HBase 为例）
    - 参考：https://zhuanlan.zhihu.com/p/164645879
    - 插入一个 feature 时 进行以下几步操作
      1. 预处理 假如数据没有 id 生成 uuid 然后为数据添加合适的属性 方便插入 HBase 表 为了防止数据倾斜 最终对于 id 进行哈希 放入对应分区
      2. 计算索引值 GM 会找到数据的 geo 属性和 date 属性 提取出数据 计算索引值 其中时间数据为了适应 GeoHash 机制 采用了 BinnedTime 机制 即相对于 1970 年 1 月 1 号形成有限的 chunk 接下来开始计算索引 geo 数据为 double 类型 dtg 则为 long 类型 先进行标准化等处理后 最终通过编码得到索引值
      3. 索引值会被编码为 byte 数组 写入 HBase 表中
      4. 最后数据被序列化 写入 HBase 表中 默认使用 kryo 序列化 也可以使用 avro
  - GM 如何进行数据的序列化？
    - 参考：https://zhuanlan.zhihu.com/p/164647326
    - 分两步 分别是序列化 feature 和 type 为什么需要序列化 type？因为插入数据时需要判断是否共用连接 GM 会把 type 序列化后存放在缓存里 方便后续反序列化出来进行判断 相当于一个全局的 schema
    - ![](https://image.blog.nwdnysl.site/20250224205422-99fe17067b124be578cba66601944766.png)
    - 具体序列化过程略 可以视为和 avro 一样根据 schema 压缩为二进制数据
  - GM 如何进行数据的索引？
    - 参考：https://zhuanlan.zhihu.com/p/164748160
    - Z2 索引：将二维空间编码为一维 以经纬度数据为例 通过空间填充曲线决定了数据的唯一顺序 也就映射到了一维整数上（这个过程相当于分别不断二分经纬度） 从而支持 1d 的键值索引 当需要进行 2d 的范围查询时 递归的寻找所有被区域包含的字符串 然后通过二分查找快速找到所有数据 或者对编码作主键建立 B+树索引
    - Z3 索引：将二维空间和时间编码到一维 时间按照 time period 切分 取每一段内的 offset 作为二分的依据 然后和空间数据一起编码为 bytearray 这种方法具有显著的空间放大问题 后续会讲到
    - XZ2 索引：将二维非点对象编码为一维 与 Z2 类似 通过找到空间对象的 mbb 来进行编码 相当于找到 Z2 编码中 mbb 的左下角方块的编码
    - XZ3 索引：将二维非点对象和时间编码为一维 类似

#### TrajMesa

- JUST 的前身 基于 GM 实现的分布式 NoSQL 轨迹查询引擎
- 工作可以分为三部分：预处理、建立索引以及查询
- 预处理
- 建立索引与存储
  - 论文提到 传统的垂直存储（也就是一个点存一行）对于轨迹数据来说不适合 包括查询一个轨迹比较慢、IO 次数多等问题 因此 TM 采用了水平存储 一行存一个轨迹 包括轨迹的元数据（mbr、时间范围等）、点 list（经压缩）、签名（一个 4\*4 的编码来描述轨迹的形状）、其他属性
  - TM 存在两份轨迹数据的副本表 分别是 IDTI 和 SRI 即 IDTQ 的索引表和 SRQ 的索引表
    - IDTI 表的 Key 为 `shard（随机数）+ 设备 id+BinNum（距离 RefTime 的第几个 Bin）+ EleCode（Bin 内的时间戳）+ 轨迹 id`
    - SRI 表基于 GM 的 XZ2 索引实现 Key 为 `shard + PosCode（与签名类似的 2\*2 编码 细化轨迹形状）+ XZ2（XZ2 产生的编码）+ 轨迹 id` 注意签名是基于 mbr 的 而 PosCode 是基于 XZ2 空间的
- 查询
  - 支持 4 种查询：某一个设备在一段时间内所有轨迹的查询（IDTQ）、空间范围查询（SRQ）、相似性查询（SQ）以及 knn 查询（KNNQ）
  - IDTQ：查询窗口=`可能的shard+设备 id+与时间范围相交的BinNum+每个Bin的偏移范围` 然后通过并发扫描 并删去不符合条件的轨迹
  - SRQ：查询窗口=`可能的shard+与空间范围相交的XZ2编码+每一个XZ2子空间对应的PosCode`
  - SQ 和 KNNQ 讲的比较晦涩 暂时没看懂

#### JUST

- 参考：https://zhuanlan.zhihu.com/p/300606530
- 基于 TM 作为底层存储的完整数据引擎系统 我们主要看其更新的时空索引部分
- Z2T 索引
  - Z3 索引用于时空数据的编码 然而存在空间放大问题 即索引中时间的粒度过大 会涉及到很多不必要的数据 为了解决这个问题 JUST 提出了 Z2T 索引
  - Z2T 索引很简单 就是把时间分片 索引变为`Num(T)+Z2` 其中 Num(T) 是时间分片的编号 查询时相当于先按时间范围剪枝 然后扫描 Z2 索引的 min-max 范围
- XZ2T 索引
  - XZ2T 索引用于非点对象的编码 与 Z2T 类似 只是把 Z2 换成了 XZ2
  - 我其实不懂上述索引和按时间分片有什么区别
- 除此之外 JUST 还实现了 SQL 查询和存储引擎 包括 Plugin Table（预定义 schema 的表）、View Table（缓存查询中间结果为 DF）

#### JUST-Traj

- 结合了 JUST 和 TrajMesa 的优点 实现了一个轨迹数据管理系统
- 索引
  - XZ2+索引：即 TM 里 SRI 的索引 XZ2 再加上一个 PosCode
  - XZT 索引：即 TM 里的 IDTI 索引 BinNum+EleCode
  - XZ2+T 索引：按时间分片后的 XZ2+索引

#### 其他时空引擎

- GeoMesa：支持各种分布式数据库以及本地 fs
  - 基本原理如上 是把高维数据使用 Z 曲线编码为字符串 然后查询时使用递归 支持 2d+时间的索引
  - 尝试过 cassandra 和 redis 查过文档 没找到支持 3d POINT 的方法 其 srid 也只支持 4326 也许默认只能用于经纬度
  - 文档：https://www.osgeo.cn/geomesa/index.html
- PostGIS：是 PostgreSQL 的一个插件 支持空间索引 存储 3d 空间数据 甚至支持 GeoServer 接入
  - 原理：有三种索引 GIST BRIN 和 SP-GiST 如果后续决定使用 可以做性能测试
    - GiST 即通用搜索树 支持各种数据类型 PG 实际上是在这个基础上实现了 R 树
    - BRIN 是一种轻量级索引类型 专为处理非常大的表而设计 它通过存储数据块范围（block range）的摘要信息 而不是每个数据行的索引值 从而显著减少索引的存储空间和维护成本 简单来说 有点像 LSM 树 存储了每个分块的最值 从而可以快速定位到某个范围内的数据
    - SP-GiST 是一种支持分区搜索树的通用索引方法 意为空间分区的 GIST 适用多维空间数据 通过四叉树、kd 树等递归分割数据
  - 文档中写到：坐标可以包含可选的 Z 和 M 坐标值。 Z 坐标通常用于表示高程。 M 坐标包含一个度量值，该值可以表示时间或距离。 如果几何图形值中存在 Z 或 M 值，则必须为几何图形中的每个点定义这些值。 如果几何图形具有 Z 或 M 坐标，则坐标尺寸为 3D; 如果它同时具有 Z 和 M，则坐标尺寸为 4D。
  - 也就是说可以存储 4d 的 POINT（x,y,z,m） 并支持上述三种索引 我简单试了一下 是可以存储 4d 的 如果要使用 就考虑把时间数据进行分片 肯定不能存整个时间戳
  - 文档：http://postgis.net/docs/manual-3.5/
- Big Query：是 Google 的一个云数据仓库 支持 SQL 查询 支持空间数据类型
  - 和 redisearch 有点像 支持存储 GEO 类型数据 并且进行空间查询 但是不支持时空索引
  - 另外 google 的数据库区域基本都在欧美 而且要付费 甚至 api 可能还需要代理 基本排除这个方案
  - 文档：https://cloud.google.com/bigquery/docs?hl=zh-cn
- Snowflake：是一个云数据仓库 支持 SQL 查询 和 Big Query 类似 支持 GEO 类型数据 以及一些空间查询 不支持时空索引 并且也需要付费 也可以排除
  - 文档：https://docs.snowflake.com/
- RedShift：亚马逊的云数据仓库 亚马逊云账户需要绑卡 因此我没有做测试
  - 看了下文档 和 PG 一样支持 ZM 的 POINT 类型
  - 文档：https://docs.aws.amazon.com/redshift/latest/dg/welcome.html
- GeoLake：基于湖仓的空间数据层 没找到相关文档 看 github 介绍 应该是支持 ZM 的空间类型数据 但是没有提及时间数据
- GeoWave：和 GM 类似的索引软件 支持分布式数据库和 fs 等 支持 3 维空间数据和时间 并且可以选择多种索引 基本上 GW 对于分布式键值数据库就像是 PG 对于 PostgreSQL
  - 试着用了一下 bbox 查询语法不支持 3 维 大概率不支持三维空间查询 不知道文档里说的支持 3 维存储和索引是什么意思
  - 文档中介绍空间索引时也举了 3 维+时间的例子 原理和 GM 一样 递归分解 代码语法基本也和 GM 一样
  - 文档：https://locationtech.github.io/geowave/overview.html
  - python 接口：https://locationtech.github.io/geowave/latest/pydocs/
- Elastic Search：分布式搜索引擎 查看了文档 支持 geo 类型的空间数据 但是仅限于 2d 经纬度数据 可以通过复合条件查询实现 3d+时间的范围查询 不知性能如何
  - 文档：https://www.elastic.co/docs
- 总结
  - GM 和 GW 的索引原理差不多 GW 声称支持 3d 空间数据 但是还未找到用法 如果找到了则可以直接使用 另外 Z3 索引存在空间放大问题 可能需要测试性能
  - PostGIS 支持 4d 空间数据 可以把时间数据分片后作为 M 维存储 但是需要进一步测试性能 GL 应该和 PG 差不多 但是前者显然更完善成熟
  - 几个云数据库基本排除 都只支持空间索引
  - ES 支持时空数据的查询 但是不清楚具体查询用到的索引如何
  - 如果都不支持 自己实现 无论时间参不参与编码 都需要先把时间分片 我觉得时间参与编码意义不大 因为时间字段是可以自然有序的

#### GiST

- 参考：https://habr.com/en/companies/postgrespro/articles/444742/
- GiST 和 B+树类似 区别在于 GiST 的可扩展性 B 树只支持大于小于等于的比较操作 而 G 可以支持相对位置运算符（R 树的左侧右侧等） 亦或是 RD 树的交集等运算符 在某种意义上 我们可以认为 G 是一种接口 是各种索引实现的一个基础框架
  - 结构：平衡树 所有叶节点的深度相等 每个节点代表一个集合区间（也就是一个谓词条件）且节点之间可以有交集 根节点是全集合区间 越往下集合区间越小 在搜索数据时使用 consistent 函数逐级作 dfs 搜索 将所有满足搜索条件的节点返回
  - 基于 R 树：R 树将平面拆分为多个矩形 一个节点代表一个矩形 叶节点代表空间对象 一致函数则是判断矩形是否相交 如果对空间数据进行 G 索引 则采用 R 树作为基础
  - 基于 RD 树：这是 PG 对于全文搜索采用的索引结构

#### SP-GiST

- 参考：https://habr.com/en/companies/postgrespro/articles/446624/
- 从名字可以看出 SP-GiST 是对于 G 的一种扩展 SP 指的是空间分区 SP 索引适用于任何值域空间可以递归划分为非相交区域的数据
  - 结构：非平衡树 因为每个节点的区域都是不相交的 每个内部节点存储子节点的指针 以及一个前缀值（可以被视为是所有子节点都满足的谓词） 叶节点存储实际数据的指针以及数据的值 搜索过程仍然以 consistent 函数为基础 即递归判断节点的 prefix 是否满足搜索条件
  - 基于四叉树：四叉树递归划分二维平面 内部节点的 prefix 即为四叉树的区域中心 叶节点则存储实际数据 可以是链表
  - 基于 kd 树：kd 树递归划分多维空间 prefix 是 kd 树的划分线坐标 叶节点存储实际数据
  - 基数树：基数树用于对字符串进行索引 prefix 是字符串的前缀 叶节点存储实际数据
  - SP 索引不支持排序以及唯一性约束 不去不支持在多个列上建立

#### CODE

- 空间查询：https://postgis.net/docs/manual-3.5/using_postgis_query.html#using-query-indexes
- PG 的空间数据类型：https://postgis.net/docs/manual-3.5/using_postgis_dbmanagement.html#PostGIS_Geometry
- https://postgis.net/workshops/postgis-intro/3d.html
- 学会如何创建索引并进行查询了 其中 3d 空间查询需要使用 intersects 函数等来模拟 另外 没法使用 gevel 来查看索引表数据 不知道如何验证八叉树

## 论文研读

### 时空索引

- 结合时空关键字的轨迹范围查询混合索引结构：存储轨迹时 额外存储轨迹的文本信息 比如轨迹经过了 hospital、school 等地点 然后在关键字上做倒排索引 加快查询
- 除了使用 Z 曲线的 Geohash 以外 还有使用希尔伯特曲线的 Google S2 其先把球面投影到外切正方体上 然后在正方体的六个面上做希尔伯特曲线编码 S2 每一级之间的大小变化比 GH 要平缓许多 另外 似乎普遍认为希尔伯特的空间聚类效果更好
- 基于 LSM-OCTree 的时空流分布式调度和存储方案
  - 背景：时空数据流的索引更新和查询问题
  - 大致思路是先用八叉树预处理空间 LSM 树中一个数据块代表了一个或多个邻近的八叉树节点 合并时利用八叉树的空间性质进行合并 加快效率 查询时 由于内存中的 memtable 存放的是近期数据的八叉树 因此对于短时间的查询有较高的效率（相当于直接在内存的八叉树里进行查询）结论中提到 索引的更新效率大于普通八叉树索引（没有和 HBase 的时空索引对比 我觉得是比不过的） 查询效率比 HBase 的时空索引方案提升百分之二十左右 但是是在短时间范围内的查询 怀疑就是在内存中查询的 涉及到磁盘中 sst 的效率估计不如时空索引 总的来说没什么用 这个结合 LSM 和 OCTree 的思路
- 基于轨迹大数据时空分布的索引与查询方法
  - 背景：基于空间等分的空间索引通常会遇到数据分布不均匀的问题 比如 GH 将查询拆分为子查询 其中因为数据不均 子查询的耗时各不相同（不利于并发？另外我觉得这里的子查询的立方体是等大小的 而非像 GM 那样匹配到前缀就停止 所以后面才能解决数据不均的问题 要不然每一次查询使用的立方体实际上都是不一样的）并且文章提到 当查询空间远大于单位立方体时 子查询数量过多而影响查询性能
  - 为此作者提出一种基于历史数据分布规律进行合并分区的方法 具体而言 先导入合适的历史数据 设定一个阈值 按照 Z 曲线的编码顺序遍历每一个立方体 合并直至数据量大于阈值 然后进行下一个立方体的合并 然后建立从 GH 编码到分区编号的倒排索引 存储点数据时 分区编号和 GH 编码一起作为行键一部分 以论文的示例来说 查询时可以将子查询数量变少（因为某些区域合并）并且每个子查询的耗时差不多
  - 实验结论：随着阈值变大 查询耗时减小直至稳定 另外 建立索引分区之后 插入未来数据后 查询耗时基本不变 说明在相同时间尺度下 数据分布的不均匀特性是相似的（也就是每天轨迹数据分布相似）
  - 作者提到两个问题 一是 GH 固定查询分区大小 导致当空间较大时子查询数量过多 这一点 GM 通过前缀匹配解决 二是数据分布不均匀 这篇文章的方法应该是不适用于 GM 的查询逻辑的 因为每次查询的子查询大小都是不一样的
  - 虽然文章介绍的方法不适用 但让我想到一个思路：无人机的轨迹数据在每个时间段内的分布可能也是相似的 因此做历史数据分析时 用户可能也会倾向于频繁查询某一些特定空间范围的数据 虽然数据会随着时间变化 但是这个查询空间所生成的子查询窗口（也就是一维编码的数组）应该是固定的 如果维护一个表 记录次数最频繁的查询空间范围 就可以不用反复分解空间范围生成查询窗口了 这个类似于缓存的表也可以放在内存里加快效率
- 学习索引
  - Kraska 等人提出 使用神经网络代替传统数据结构构建索引 大致思路是让模型学习数据分布的规律 从而预测数据的位置 直接访问即可
  - RMI：递归模型索引 数据集被划分为多个子集 组织为树形结构 根模型是路由模型 将查询分配到子节点 而叶模型是最终的数据预测模型 训练需要大量时间资源 并且更新写入需要重新训练模型
  - FITing-Tree：用线性模型拟合底层数据分布 将 B 树索引的叶节点从数据改为模型参数
  - ALEX：支持高效插入数据
  - 其余：https://zhuanlan.zhihu.com/p/625142215
  - 参考：https://zhuanlan.zhihu.com/p/649563211
- 基于改进的 K-means 聚类分区均匀化空间学习索引
  - 背景：提出适用于空间数据的学习索引方法
  - 学习索引需要数据降维 作者注意到使用 Z 曲线降维后的空间数据在一维的分布总是可以被划分为多个近似均匀分布的区间 因此先对数据进行聚类 划分为各个线性区间 然后用线性回归进行拟合
  - 范围查询即通过模型找到上下限位置 然后扫描并二次过滤
  - 结论：使用分段线性函数替代神经网络拟合数据分布 结构简单且查询效率高
- Efficient Cost Modeling of Space-filling Curves
  - 背景：SFC 用于编码空间数据 但是现有的某一种特定的 SFC 往往在查询性能上不理想 希望基于成本函数以及数据分布特征来选择合适的 SFC 但是一般计算的开销较大
  - 作者提出了基于强化学习选择 sfc 的方法 其提出的成本算法是 On 的 计算时间快 并且定义一种特殊的 SFC 称为 BMC 可以生成不同的曲线类型 最终实验表明学习效率与选择出的 sfc 的查询性能都优于现有方法
- https://ieeexplore.ieee.org/document/10598151
  - 背景：缺少合适的云存储服务来存储时空数据 因此作者实现了一种基于云服务的轨迹数据管理系统
  - 存储架构：包括内存层 磁盘层 以及对象存储层（也就是云服务层）内存层中存放了所有数据的索引以及部分缓存块 实际数据则根据热度存放在磁盘层或对象存储层
  - 存储结构
    - 轨迹数据具有时空连续性 因此将轨迹数据基于时间进行 chunk 的划分 这同时使得连续的 chunk 往往在空间上也有局部性 后面会提到这个局部性是怎样被利用来提高查询性能的
    - 具体而言 一个 oid 对应一个类似于链表的 chunk 列表 其中包含一个头部 chunk 和剩余的不可变 chunk 头块是这个设备最新的轨迹数据 不可变块则是曾经的头块 所有 chunk 都有一个逻辑指针指向下一个 chunk 它们具有轨迹上的连续性 之所以是逻辑指针而非物理指针 是因为 chunk 可能被存储在不同的地方 因此作者实现另一个存储管理器来找到逻辑指针对应的物理存储位置 相当于一个映射表
    - 索引会完全存储在内存中 除此之外 头块以及部分最新的不可变块也会存储在内存中 以提高查询性能
  - 索引设计
    - head chunk
      - 头块包含了 oid 时间范围等元信息 一旦头块转换为不可变块 其索引会被删除
      - 头块的索引是一个空间网格索引 每一个网格维护了一个 postings list 其记录了哪些 oid 的头块轨迹数据落在这个网格内
      - 由于空间不均 列表的大小会有所不同 作者提出一个分裂方法 我暂时没有细看
    - immutable chunk
      - 不可变块的元数据包括时间范围 mbb oid 与 chunkid 指针
      - 不可变块的索引是一个 b 树变种 键是时间范围 一个叶节点存储了时间范围内的多个 chunk 索引还会维护一个指向最新叶节点的指针 从而可以快速插入下一个索引
      - 为了解决轨迹数据只占 mbb 的一部分而导致的假阳性的问题 采用一个空间位图来描述轨迹的形状
    - 索引全部位于内存中是这个系统可以高效进行查询的关键 作者进行了计算 得出 64GB 内存可以存储 16TB 的轨迹数据的索引
  - 查询过程
    - ID-T 查询：查询某个设备在一段时间内的所有轨迹数据
      - 判断头块是否符合时间范围
      - 访问不可变块的索引树 二分找到时间范围的下限叶节点 然后顺序遍历到第一个符合 oid 的节点
      - 从这个 chunkid 开始 沿着指针遍历 直到某个节点不再符合时间范围
      - 合并 chunkid 到实际存储中取出数据
    - ST-查询：查询某个时空范围内的轨迹数据
      - 访问头块的网格索引 找到所有符合空间范围的网格 取出 oids
      - 根据 oids 找到头块 根据头块的元信息进一步判断是否符合空间范围 取出所有符合的数据
      - 访问不可变块的索引树 先根据时间范围找到下限叶节点 然后顺序遍历叶节点
      - 遍历时通过空间范围和空间位图进一步判断是否符合条件 取出所有 chunkid 到实际存储中取出数据
  - 其它优化
    - 不可变块在超出阈值后会被持久化到磁盘 在存储持久化之前 会先将所有块按照 oid 和空间信息进行排序 这使得这一部分的 chunk 中 同一个 oid 的数据是连续的 保证了局部性
    - 访问磁盘的多个 chunk 时 管理器会利用 chunk 的局部性 也即会尽可能将多个 chunk 的随机访问变为一个顺序访问
    - 数据迁移基于 LRU 以及存储成本模型
- https://ieeexplore.ieee.org/document/10597771/
  - 基于键值存储的轨迹数据管理系统
- https://ieeexplore.ieee.org/document/10598038/

### 查询优化

- 基于机器学习的数据库技术综述

  - 论文介绍了机器学习在数据库中的应用 包括八个方面：数据库运维、​ 数据存储、​ 优化器与执行器、​ 查询优化、​ 负载管理、​ 安全与隐私、​ 自管理、​ 支持 ML 的数据库
  - 我主要关注查询优化这一块 包括 SQL 重写 索引推荐和自然语言查询
    - SQL 重写包括外部和内部重写 外部重写一般由人工完成 将 SQL 语句进行一定原则的重写 比如避免全表扫描操作 内部重写则是在优化器内完成的 对 SQL 树进行修改和替换
    - 索引推荐：常用的索引都是较通用的数据结构 没有对数据分布进行利用 也没有使用深度学习模型等
    - 视图推荐：从查询集合中提取高频的子查询 并将其存储为视图以提升查询性能 一般分为两步 先从不同语句中识别等价的子查询作为候选视图 然后对候选视图进行评估 选择合适的视图
  - 另外 广义的查询优化还包含了数据库内部的优化器和执行器
    - 基数估计：指的是估计一个查询结果的大小 用于估计查询的成本
    - 计划选择：计划选择器会生成不同的连接计划 计划数量对于表的数量是指数级的 因此找到一个最优计划是 NP 难问题 这一步可以使用机器学习来进行优化
    - 分布式协同：在集群上 数据划分和任务调度的均匀尤为重要 根据节点状态进行自动的划分是一个重要的研究方向
  - 基数估计
    - 数据库代价估计分为两部分：基数估计和代价模型 前者即使是在现在的数据库中仍存在较大误差
    - 传统基数估计方法
      - 直方图：将数据划分为多个桶 统计每个桶的大小 通过桶的大小来估计基数
      - 数据画像
      - 基于采样的方法
    - 机器学习方法
      - 面向查询的基数估计：比如使用 RNN 学习数据表、查询条件与连接条件之间的关系 预估数据库可能存在的基数
      - 面向执行计划的基数估计：使用图神经网络来学习执行计划的特征 进而估计基数？
  - 查询计划选择
    - 静态计划选择方法：使用确定的代价模型来选择最优的执行计划
      - 基于 DP 的方法：类似于 dp 计算矩阵乘法的代价 虽然去掉了冗余计算 但是仍在指数级的状态空间内进行搜索
      - 基于启发式的方法：基于启发式算法生成随机的执行计划 但是没有保证找到最优解
    - 机器学习方法：静态计划依赖于估计器的优劣 换言之 估计器评估出的最优解在实际执行情况下可能并不是最优的
      - 基于估计器的自适应性方法：这类方法认为计划的优劣由估计器限制 因此在执行计划时动态的更新估计器的参数 使得估计器的评估结果更接近真实值
      - 基于 RL 的方法：将 Join 条件作为动作空间 状态空间由所有连接树组成 执行的代价作为奖励函数 求解策略函数以最小化长时奖励 即最小化执行代价
  - 索引自动推荐
    - 索引生成：即学习型索引
      - 范围索引：使用神经网络来学习数据分布的规律 直接预测数据的位置
      - 点索引：哈希索引
      - 布隆过滤器：可以是学习一个二元分类器来判断数据是否存在 也可以是学习一个哈希函数使得键和非键的冲突率最小化
    - 索引选择：即索引推荐
      - 在线方法：工作负载分析 索引方案选择 索引方案实现
      - 离线方法：单查询优化 工作负载优化
      - 半自动化索引调整：结合在线和离线方法
      - 机器学习方法：ITLS 将学习分类器应用在索引推荐上 通过遗传算法来选择索引
  - 物化视图选择
    - 识别等价子查询：包括基于符号和基于逻辑语义的判断
    - 视图选择：基于查询的频率和代价来选择 包括基于 DAG 和基于 ILP 的方法
  - 基于 NLP 的查询技术：比如 LLM 用于自然语言查询

#### 一些可以 fix 的小细节

- realtime_map 中接收消息时的 time 和消息中的 time 有延迟 差不多是 2s 不知道为什么
- 节点刚好处于区域边界时可能会有点问题 暂时没做测试
- 断开连接时如何告知节点 目前是心跳计时实现的 考虑是否要更加实时
- 如果只是发给每个单独的设备 似乎没有必要汇聚到中心？以后如果 push 给应用 一个应用可能需要多个设备的数据 比如一个应用商只负责 taxi 的数据
- 目前数据缺少方向 因此先以设备为中心划分矩形范围 后续加入方向后再进行修改
- jedis 中取出的 Document 类中的 properties 不知道为什么是$={}的格式 导致必须要解析 string
- B 与 A 的通信需要发送独占空间 数据比较大 除了让 zhd 进一步压缩以外也可以使用 avro 进行压缩
- 合并大圈小圈数据很 sb 看看能不能把 B 的小圈数据格式改一下
- java 这边的 avro 由于我使用了 fastjson 导致需要先解析为 jsonobject 再进行使用 需要手动分类序列化 未来可以考虑直接使用 avro 的 record 类
- avro 的压缩 如何自动生成 schema 文件？如何定义统一的 realtime_map 的 schema？（环境变量）
- 节点内部的通信无需使用 mqtt 可以使用 redis 共享 比如 sm 和 ceds 之间的通信（但是除开共享以外 还需要通知）
- 现在态势数据没有 是因为 mergecircle 里会取大圈和小圈的交集 而现在小圈没有数据
- problem 冲突的设备处于不同区域时 需要共享数据 让节点 push 转发逻辑如何解决？首先要决定谁来计算冲突解决
- 目前索引层和查询层在不同进程 因此有数据共享上的问题 可以考虑将查询层改为 java 也可以等后续索引层确定后再进行改写

#### 性能需求

- 现在是中心主动 push 瓶颈可能在两个地方 一个是每 1s 计算态势数据来不来得及（主要是节点的强化学习） 另一个是中心需要并发发送态势数据给各个设备
- 一种是加快计算频率 看看到多少会出现问题
- 另一种是增加订阅设备的数量 看看到多少会出现问题

#### 代码整合

- 三个部分的代码 CEDS 八叉树空间 小圈计算 称为 A B C
- 节点收到请求开始计算：
  - 大圈
    1. 在 A 的 situ_cal 中 取出 redis 所有的设备
    2. 对于每个设备 使用 redisearch 的范围查询得到范围内的设备 即大圈数据
  - 小圈
    1. 在 A 的 situ_cal 中 直接给 B 通过 mqtt 发送异步请求 topic 为 `situation/send/{timestamp} 3. B 得到 C 返回的小圈后 将小圈编码为八叉树格式 然后根据存储的每个设备的特性（优先级等）以及形状进行冲突的解决 最后返回以八叉树格式表示的小圈数据给 A topic 为 situation/receive/{timestamp}on/receive/{timestamp}
- 只需要改为使用 redis 然后使用 mqtt 即可 计算独占空间的小圈只需要每个设备的位置数据 然后使用元数据计算 独占空间的大圈则需要发给 C

### 项目架构

#### 并发处理

- 节点计算态势数据用了线程池进行并发
- 向邻居节点发送请求也可以并发处理 不过由于 http 有长连接机制 请求应该不会重新建立连接 且一个设备的计算最多也就发送 3 个请求（大多数情况是 1 个） 所以并发处理的意义不大
- 中心对于消息的处理使用线程池并发
  - 本来的思路是使用 TPE（ThreadPoolExecutor）类初始化线程池 然后用一个 ConcurrentHashMap 来存储每个请求的线程对象 请求的第一个消息会被线程池随机分配一个线程对象 后续的消息取出 map 里的线程对象即可保证同一个请求的消息被同一个线程处理 但是我不知道如何使用线程池指定某一个线程处理...
  - 现在的实现比较简单 就是一个 List<ExecutorService> 里面存了 n 个单线程池 每次请求根据时间戳取模分配一个线程池处理 不知道可不可以保证均匀
  - 中心并发的意义在于高并发请求 对于串行请求无提升
  - **改为 push 后 这部分就没什么意义了**
- **中心发送态势数据给各个设备**：现在是 for 循环+publish 不知道 mqtt 的 publish 是不是异步的 可能需要多线程或者至少保证异步发送

#### 总体架构

##### 物联网设备

- sender.py: 模拟设备发送数据 通过 mqtt 向边缘节点实时发送数据

##### 边缘节点

- realtime_map.py: 在内存中维护两个结构 `realtime_map`（设备的实时数据）和 `online_clients`（在线设备列表）其中 realtime_map 存储在 redis 中 方便使用 `redisearch` 进行地理空间查询
  - 节点收到设备的数据后更新 `realtime_map` 与 `online_clients` 同时上传 realtime_index 给中心
  - 另起一个子线程判断设备是否离线 标准是每隔 2s 进行检查 如果上一次收到数据的时间距离现在超过 2s 就认为设备离线
  - 使用 flask 提供接口`/query` 用于其他边缘节点进行态势数据计算时 来查询范围内的设备数据
- situation_calculation.py: 计算态势数据的模块 连接到中心节点的 mqtt 并订阅`/situation/{node_id}`的 topic 接受到中心发来的消息（payload 为 timestamp）后开始计算态势数据 步骤为
  1. 从本节点的 redis 和邻近节点的 `/query` 接口中获取需要的实时数据 其中使用了 `shapely` 库来判断大圈与邻居是否相交
  2. 节点把每个设备 realtimeData 中的时间戳作为态势数据的时间戳放入 用于中心节点的合并去重
  3. 用这些数据计算每个设备的态势数据 使用线程池进行多线程并发提高性能
  4. 封装好最终的态势数据 放入中心消息队列的`/situation/center/{timestamp}`的 topic 中 消息格式如下：
     ```
     {
         "nodeId": 1,
         "taxi/0001": {
             "TIMESTAMP": 1372662000, # 用于合并去重
             "BIGCIRCLE": {...},      # 其它设备的态势数据
             "SMALLCIRCLE": {         # 独占空间
                  "smallArea": ... ,
                  "largeArea": ... ,
              },
          },
         ...
     }
     ```
     ```
      {
          "taxi/0001": {
              "TIMESTAMP": 1372662000, # 用于合并去重
              "BIGCIRCLE": {...},      # 其它设备的态势数据
          },
          ...
      }
     ```
- process_manager.py：进程管理器 负责管理数据采集进程的生命周期
- data_handler.py: 数据采集进程 定期存储数据到 mongodb

##### 中心

- 中心会在内存中维护一个名为 `situationData` 的结构 其包含两部分数据 一个是 map 存储态势数据 其结构类似于：
  ```
  {
      "timestamp": {                            # 发起这次态势数据计算时的时间戳
          "nodes": {                            # 所有节点是否已经收到消息
              1: False,
              2: False,
              ...
              12: False,
          },
          "data": {                             # 节点计算的态势数据
              "TAXI_ID": {...},
              "DRONE_ID": {...},
          },
          "cnt": 12,                            # 还未收到消息的节点数
          "done": False,                        # 是否已经完成
      },
      ...
  }
  ```
  另一个是 `devices` 维护了哪些设备订阅态势数据 记录他们的设备 id 以及这个设备的 region 参数
- 中心每隔 1s 进行一次态势数据的计算 先向 situationData 中插入新的键值对 随后向中心的消息队列的 `situation/{nodeID}` topic 中放入 payload 为`timestamp`的消息 告知各个节点开始计算
- 中心收到节点返回的消息后立马进行合并 根据时间戳判断最新态势数据 如果 `nodes` 中已经收到过此节点的数据 则说明消息重复了 不处理 否则置为 TRUE 然后 cnt--
- 当发现所有 node 都已合并（即 cnt==0）则将 `done` 字段置为 TRUE 并遍历 devices 将态势数据发送到`situation/app/{deviceID}` topic 中 等待设备自取
- scheduler 类中还设置了两个定时任务
  - checktimeout 任务会定期查看每个请求的时间戳是否已经过时 2s 如果过时 2s 就删除 这个主要是为了防止因为某一个节点挂掉而导致一个未完成的态势数据一直存在内存里的情况
  - persistData 任务定期将已经完成的请求的态势数据持久化到数据库中 以便后续进行数据分析 这里不做处理直接作为文档存入 MongoDB 中（测试时怕数据溢出 所以每 1 分钟就把这个表 drop 掉）

# FEDERATED LEARNING

- 背景与联邦学习相似 区别在于客户端上传的是扩散后的贝叶斯模型 由服务器进行数据生成和对齐 并训练模型
- 总体流程是
  1. 客户端通过本地数据训练一个贝叶斯模型 提取出了数据的特征
  2. 客户端将贝叶斯模型进行扩散处理 生成一个相似但是不同的贝叶斯模型
  3. 客户端将扩散后的贝叶斯模型上传到服务器
  4. 服务器用扩散后的贝叶斯模型进行数据生成和对齐
  5. 服务器将生成的数据进行训练
- 贝叶斯模型可以保留非线性的因果关系 而 PCA 只能表达线性关系
- 之所以不用 diffusion 直接生成数据 一是过于简单可能会被破解 二是直接 diffuse 数据可能会破坏表格数据的因果关系 而贝叶斯模型可以保留这点
- 现在是用 diffusion 作为 encoder 为 decoder 生成的 DAG 打分 形成 reward 调整参数 默认 diffusion 可以较好的保留数据的特征分布
- 重点是隐私保护 如何评估隐私保护的程度？
- 流程是
  1. 将数据压缩 比如原本为 10000 行 压缩为 100 行
  2. 输入数据 X 到 encoder（扩散模型）中 加噪降噪后生成新的数据 X'
  3. 使用 X' 输入 decoder 生成 DAG
  4. 计算 BIC 和无环性惩罚 得出 reward 其中这里的 BIC 输入的是原采样数据和新数据生成的 DAG
  5. 根据 reward 调整 Actor 的参数
- ![](https://image.blog.nwdnysl.site/03744f80dced0a051b1ca21c29172c9-3bdd788d91ba9703de92bf6e0bec5451.png)

## 背景知识

### FL

- 联邦学习的背景是数据隐私问题 由于数据隐私问题 客户端无法将数据上传到服务器进行训练 然而世界上绝大多数可用数据都是私人数据 许多模型的训练已经面临数据稀缺的问题 因此谷歌提出了联邦学习的概念 期望数据可以保留在本地进行训练 将模型参数上传到服务器进行聚合 服务器将聚合后的模型参数下发到客户端进行进一步训练 最终收敛得到一个全局模型
- 联邦学习的基本流程
  1. 服务器下发初始的全局模型参数到客户端
  2. 客户端使用本地数据进行训练一个周期
  3. 客户端将训练好的模型参数上传到服务器
  4. 服务器将所有客户端的模型参数进行聚合 一种简单的聚合方法是对所有客户端的模型参数进行平均
  5. 服务器将聚合后的模型参数下发到客户端
  6. 重复 2-5 直到收敛
- 联邦学习可以调整的参数
  - 客户端的选择：通常只会选择部分客户端进行训练
  - 客户端的训练超参数：比如学习率、训练周期等
  - 模型参数的聚合方式：平均、加权平均等
- 联邦学习本身并不能保证数据隐私 需要结合差分隐私等技术来保证数据隐私 比如对数据进行加噪声处理
- 联邦学习的类型
  - 横向联邦学习：客户端之间的数据特征相同 但是数据量不同 比如不同医院之间的病人数据
  - 纵向联邦学习：客户端之间的数据量相同 但是数据特征不同 比如同一批人在银行和医院的数据
  - 联邦迁移学习：客户端之间的数据量和特征都不同 需要迁移到同一特征空间进行训练

### GFlowNet

- GFlowNet 是一种新的生成模型 主要目的是采样出多样化且高奖励的样本
- GFN 解决的问题是过拟合 传统方法（比如 RL）往往追求一个最大值 这使得模型容易过拟合 在某些场景中 比如分子生成 会导致生成的分子不够多样化 而 GFN 生成的是一个分布 其中高奖励的分子具有高概率 保证了生成的分子多样性
- 采样的过程基于一个 DAG 节点代表状态 边代表动作 通过一个随机游走的方式来采样出一个分布 训练的目的是让汇点的流量与奖励函数相等 也就使得采样的分布与奖励函数成正比
- GFN 中的 Policy 是一个条件概率分布 代表了当前节点 s 采样到下一个节点 s'的概率 前向策略是从 s 采样到 s'的概率 反向策略是从 s'反推到 s 的概率 由此一条路径的采样概率可以表示为前向策略或反向策略的乘积
- 一般策略 P 用一个神经网络来参数化建模 输入是当前节点 s 输出是下一个节点 s'的概率分布 训练时 损失函数是基于流平衡条件的 也就是最终的策略会使得流量平衡 并且终态节点的流量与奖励函数成正比

### Diffusion

- Diffusion 是一种新的生成模型 其主要思想是通过对数据进行加噪声处理 然后再去噪声来生成数据
- 模型包括 2 个部分
  - 正向扩散：对数据进行加噪声处理 使得数据逐渐变为高斯分布
  - 反向扩散：对数据进行去噪声处理 使得数据逐渐变为真实数据
- 正向扩散一般使用固定的加噪流程 比如生成高斯分布的噪声逐步添加到数据中 使得数据逐渐接近纯噪声
- 加噪声实质上是对输入数值和一个正态分布的随机数进行加权平均 这样的线性组合使得去噪非常简单 因此反向扩散往往学习的是噪声 而不是完整图像 大大减少了模型的复杂度
- 反向扩散则需要训练一个神经网络来预测噪声 即输入 x*t 输出需要去除的噪声 然后从 x_t 中取出噪声得到 x*{t-1} 反复迭代直到得到 x_0
- 反向扩散的损失函数一般是实际噪声和预测噪声之间的均方误差或 KL 散度等
- 当模型训练完毕后 可以通过一个随机噪声向量作为输入 经过反向扩散的过程来生成数据 这些数据会和训练数据特征相似

### DALLE2

- DALLE2 是 OpenAI 提出的一个文本生成图像的模型 其主要使用了 CLIP 模型进行文本和图像的编码 然后使用 diffusion 模型进行图像的生成
- CLIP 是一种对比学习模型 其输入文本图像的 pair 包括正例和负例 通过对比学习的方式来学习文本和图像的共同表示 使得正确的文本和图像在特征空间中的距离更近 反之则更远
- 本质上有点像使用 CLIP 监督 prior 模型 来实现文本转图像向量 最后再用 diffusion 模型来实现图像的生成

### GAN

- GAN 是一种生成对抗网络 其主要由生成器和判别器组成 生成器负责生成图像 判别器负责判断图像的真假 通过对抗训练的方式来优化生成器和判别器的参数
- 具体而言 生成器输入一个随机噪声向量 通过一系列的卷积层和反卷积层来生成图像 判别器输入生成器生成的图像和真实图像 做一个二分类来判断图像的真假 生成器的目标是骗过判别器 使得生成的图像被判别器判断为真实图像 判别器的目标是正确判断图像的真假
- 缺点包括
  - 由于需要训练两个网络 训练过程不稳定
  - 生成的图像多样性不足

### Auto-Encoder

- Auto-Encoder 是一种无监督学习模型 其主要由编码器和解码器组成 编码器负责将输入数据编码为一个低维的表示 解码器负责将低维的表示解码为原始数据 由于模型只是在自训练 所以被称为自编码器
- DAE 是一种变种的自编码器 在编码前对输入数据进行加噪声处理 模型不容易过拟合 因为把冗余的信息去掉了
- VAE 是一种变种的自编码器 其编码器不再生成一个特征 而是生成一个高斯分布 使得模型可以进行采样 进行图片的生成

### RL

- 强化学习属于机器学习中与监督学习和无监督学习并列的分支 其核心思想是在缺少标签数据的情况下 通过与环境的交互来学习一个策略 使得模型可以在给定的状态下选择一个最优的动作（或策略）
- 马尔可夫决策过程（MDP）是强化学习的基础模型 核心观念是马尔可夫性 也就是当前状态只与前一个状态有关 与之前的状态无关
- RL 的基本要素
  - 状态空间 S：表示所有可能的状态
  - 动作空间 A：表示所有可能的动作
  - 策略 π：表示在给定状态下选择的动作 是一个条件概率分布
  - 奖励函数 R：表示在给定状态下选择的动作所获得的奖励
  - 回报函数 G：奖励随时间的积累 其中折扣因子 γ 用于控制未来奖励的权重
- RL 的基本流程
  1. 给定当前状态 s_t
  2. agent 根据策略 π 选择一个动作 a_t
  3. agent 执行动作 a_t 环境根据动作 a_t 转移到下一个状态 s_t+1 并返回奖励 r_t（需要注意 环境的转移可能是一个概率分布）
  4. agent 根据奖励 r_t 和下一个状态 s_t+1 更新策略 π
  5. 重复 1-4 直到达到终止状态
- RL 的目标是找到一个最优的策略 π 使得在这个策略下 回报的期望值最大化 这个期望值被称为价值函数 V_π(s) 即状态 s 下 之后持续执行 π 策略所获得的期望回报
- 除了状态价值 V 以外 还有状态动作价值 Q_π(s,a) 也就是在状态 s 下 执行动作 a 后 之后持续执行 π 策略所获得的期望回报
- Bellman 方程
  - 任何状态下的回报期望都可以拆解为当前状态的即时奖励和下一个状态开始累积的未来奖励 这样的递归关系被称为 Bellman 方程
  - V(s) = E[R_t + γV(s_t+1)|s_t=s]
  - Q(s,a) = E[R_t + γQ(s_t+1,a')|s_t=s,a_t=a]
- 寻找最优策略的方法
  - 基于价值函数的方法：学习出准确的价值函数 然后让 agent 始终选择最大价值的策略 比如 Q-learning SARSA 等
  - 基于策略的方法：直接学习策略本身 使得策略的长期回报最大化 比如 REINFORCE TRPO PPO 等
  - 如何选择行动：前者使用 ε-贪婪策略来选择动作（前期探索 后期利用） 后者则直接按照动作的概率分布来选择
  - 如何更新策略：前者基于贝尔曼方程更新价值函数 后者则会将对应策略的概率进行调整（比如获得正奖励就增加概率 负奖励就减少概率）
  - 其中价值方法中更新策略包括 MC 方法和 TD 方法
    - MC 方法：计算多个回合的平均值来估计价值函数 这使得模型方差较大 但是偏差较小
    - TD 方法：使用即时奖励和 2 个估计值来估计价值函数 这使得模型方差较小 但是偏差较大
- 优化算法的超参数
  - 更新频率：更新策略的频率 比如可以一回合更新一次 也可以每个时间步都更新
  - 传播深度：更新策略时 向后传播多远 比如可以只传播到当前状态 也可以传播到更远的状态
  - 优化公式：比如价值算法可以用 TD 也可以用 MC

#### Q-Learning

- Q-learning 是一种基于价值函数的方法 其主要思想是通过学习状态价值函数 Q(s,a) 来选择最优的动作
- 使用 ε-贪婪策略来选择动作
- 使用 Bellman 方程来更新 Q(s,a) 即
  - Q(s,a)' = Q(s,a) + α[R_t + γmaxQ(s_t+1,a') - Q(s,a)] 其中 α 是学习率 γ 是折扣因子
  - 可以看到 QL 假设下一步选定的是下一步状态中 Q 值最大的动作 并以此计算 Q 值的误差
  - 这样的策略也被称为 off-policy 策略 也就是实际执行的策略和用于学习的策略不一致

#### DQN

- DQN 是 Q-learning 的一种变种 其主要思想是使用深度神经网络来近似 Q(s,a) 的值 而非使用表格来存储 Q(s,a) 的值
- DQN 包含三个部分
  - Q 网络：输入状态 s 输出 Q(s,a) 向量 目的是学习实际 Q 值
  - 经验回放：一个表格 存储 agent 过去的经验（s,a,r,s'） 通过随机采样来打破数据之间的相关性
  - 目标网络：和 Q 网络结构相同 目的是学习下一个状态后的折扣 Q 值
- 具体流程
  1. 生成训练数据：
     - agent 根据 ε-贪婪策略在 Q 网络的输出中选择一个动作 a_t
     - 执行动作 a_t 观察到奖励 r_t 和下一个状态 s_t+1
     - 将 (s_t,a_t,r_t,s_t+1) 存入经验回放表格中
     - 重复一定次数 得到批量的训练数据
  2. 训练 Q 网络：
     - 随机从经验回放表格中采样一批数据(s,a,r,s')
     - 输入状态 s 计算 Q 网络的输出 选取对应动作的 Q 值 即 Q(s,a)
     - 输入状态 s' 计算目标网络的输出 选取最大的 Q 值 即 maxQ(s',a')
     - 计算损失 L = MSE(Q(s,a),r+γmaxQ(s',a')) = Σ(Q(s,a)-r-γmaxQ(s',a'))^2 / batch_size
     - 使用反向传播算法来更新 Q 网络的参数
  3. 更新目标网络：
     - 重复 1、2 一定次数后 再将 Q 网络的参数复制到目标网络中
- 可以看到 DQN 基本遵循了原始的 Q-learning 的思想 其中
  - 经验回放的作用是打破数据之间的相关性（可以满足独立同分布）并且神经网络需要批量数据来训练
  - 目标网络的作用是避免 Q 函数更新过快 导致模型不稳定

#### SARSA

- SARSA 和 QL 的唯一区别在于
  - SARSA 是 on-policy 策略 也就是实际执行的策略和用于学习的策略一致
  - 也即 会在真正选择了下一步的动作后 才会更新上一步的 Q 值

#### 策略梯度算法

- 策略梯度算法是基于策略的方法 其本质是梯度上升法
- 目标函数是：J(θ) = E[R|π_θ] 也就是在当前策略下的期望回报 在实际中 我们对这个策略进行多次采样得到批量的轨迹 τ 然后计算 J(θ)的梯度
- 策略梯度定理告诉我们可以用采样的轨迹来估计 J(θ)的梯度 具体推导比较复杂 公式如下：
  ```
  ∇J(θ) = E[∑∇logπ_θ(a|s)Q(s,a)] = E[∑∇logπ_θ(a|s)(r+γV(s'))]
  ```
- 计算出梯度后即可更新策略：
  - θ' = θ + α∇J(θ) 其中 α 是学习率

#### Baseline

- 我们注意到整个梯度的计算中包含两项
  - 第一项是 ∇logπ_θ(a|s) 也就是当前策略的梯度
  - 第二项是 Q(s,a) 也就是当前状态下的动作价值函数
- 如果我们可以缩减某一项的尺度大小 就相当于给整个乘积值做了一个缩放 自然就可以减少方差
- Baseline 方法的核心思想就是将第二项减去一个基准值 b(s) 使得整体的方差变小 这个基准值需要保证无偏性 一般可以选择 Q(s,a)的均值作为基准值 也就是 V(s)

#### REINFORCE

- REINFORCE 是一种基于策略的方法 其主要思想是通过采样的轨迹来估计 J(θ) 的梯度
- 核心思想是
- 具体流程
  1. agent 根据当前策略 π_θ 采样多条轨迹 τ = (s_0,a_0,r_0,s_1,a_1,r_1,...,s_T,a_T,r_T)
  2. 对于每条轨迹的每一个时间步 t 计算 G_t = ∑γ^k-t \* r_k (k=t+1,...,T)
  3. 更新策略 θ' = θ + α∇J(θ) = θ + α∇logπ_θ(a|s)G_t
  4. 重复 1-3 直到收敛
- 可以看到 REINFORCE 的核心思想是使用 Monte Carlo 方法来估计 J(θ)的梯度 也就是通过采样的轨迹来估计 Q(s,a) 也就是说 轨迹的实际回报值 G_t = ∑γ^k-t \* r_k (k=t+1,...,T) 代表着当前状态下的动作价值函数 Q(s,a)
- 由于 G_t 是一个随机变量 因此 REINFORCE 的方差较大 但是偏差较小 可以引入 baseline 方法来减少方差

#### Actor-Critic

- Actor-Critic 是一种结合了基于价值函数和基于策略的方法 其主要思想是使用一个价值函数来估计当前策略的价值 然后使用这个价值函数来更新策略
- baseline 方法提到过如何通过引入一个基准值来减少方差 而 A-C 算法中这个 baseline 就是状态价值函数 V 这使得梯度公式中的 Q(s,a)可以被替换为 Q(s,a)-V(s) 这个函数也被称为优势函数 A(s,a) 也就是在当前状态下 选择动作 a 的优势
- 我的理解：首先 V 是 Q 在不同动作下的期望值 因此 Q-V 不会改变整个期望值 然后 减去这个均值使得梯度中的后一项（Q(s,a)-V(s)）的绝对值变小 这也就使得整个梯度项的方差变小了
- Actor-Critic 包含两个部分
  - Actor：负责选择动作的策略网络 其输入状态 s 输出动作的概率分布 π_θ(a|s)
  - Critic：负责估计当前策略的价值函数 其输入状态 s 输出价值 V(s)
- 具体流程
  1. Agent 执行一个时间步 t 得到(s,a,r,s')
  2. Critic 根据当前状态 s 和动作 a 计算 V(s) 根据下一个状态 s' 和当前策略的输出计算 V(s')
  3. Critic 计算出 TD 误差 δ = r + γV(s') - V(s)
  4. Critic 使用损失函数 L = MSE(δ) 来更新神经网络的参数
  5. Actor 更新策略 θ' = θ + α∇J(θ) = θ + α∇logπ_θ(a|s)A(s,a) 其中 A(s,a) = Q(s,a)-V(s) = r + γV(s') - V(s) = δ (实际上是无偏估计)

#### TRPO

- TRPO 是一种基于策略的方法 其核心思想是对模型的更新进行约束 从而保证模型性能单调不减 防止出现学习率过大导致模型性能下降的情况
- 信任域即对于模型参数更新的约束 在 TPRO 算法中 这个约束是基于 KL 散度的 即更新前后的策略 π 和 π' 之间的 KL 散度不能超过阈值 δ 本质上也就限制了策略不能变化太多
- TRPO 将参数的优化转换为一个最优化问题 其约束就是上面提到的信任域 而目标函数则是 L(θ) = E[π'_θ(a|s) \* A(s,a) / π_θ(a|s)] 根据证明可以得到 只要在约束内 L(θ)有所提升 那么模型性能 J(θ)必定单调不减 因此只需要用这个优化的 θ' 来替换 θ 即可
- 实际应用中 上述最优化问题过于复杂 通常使用线性化的方法来简化 具体过程我也细看 在此不赘述

#### PPO

- PPO 是对于 TRPO 的一种改进 其主要思想是改进目标函数 使得计算复杂度降低
- 有两种类型的 PPO 它们都丢弃了 KL 散度的约束 对目标函数 L 进行了一定的修改
  - PPO-Clip：使用一个剪切的目标函数 使得模型的更新不会超过一个阈值
  - PPO-Penalty：使用一个惩罚项来约束模型的更新
- PPO-Clip 为目标函数中的 r*t(θ) = π'*θ(a|s) / π_θ(a|s) （也被称为重要性采样比率）添加了一个剪切项
  ```
  L_CLIP(θ) = E[min(r_t(θ)A(s,a),clip(r_t(θ),1-ε,1+ε)A(s,a))]
  ```
  这使得如果策略分布变化过大 clip 会限制目标函数 L 的增长 从而梯度不会过大 也就限制了 θ 的更新
- PPO-Penalty 则是直接在目标函数中添加一个 KL 散度的惩罚项
  ```
  L_PENALTY(θ) = E[r_t(θ)A(s,a)] - β * D_KL(π_θ(a|s),π'_θ(a|s))
  ```
  其中 β 是一个动态变化的超参数 当 D_KL 过大时 β 会增大 反之减小 也就限制了 θ 的更新
- 两者的本质都是在限制目标函数 L 不会增长过大 从而限制 θ 的更新幅度
- 其中 PPO-Clip 算法在实际应用中效果更好 实现更简单

## 论文研读

### 联邦学习

- Causal Discovery with Reinforcement Learning
  - 背景：从一组变量中发现因果结构是大量研究正在进行的工作 一大类的方法是基于评分的 通过为每一个有向图 G 指定一个评分 S(G) 并在 NPH 的搜索空间中寻找最优图 为了避免空间过大 某些方法依赖局部启发式方法强制无环性 论文提出使用强化学习搜索出最佳评分的 DAG 其中使用编码-解码模型从数据生成 DAG 图 并计算包含预定义和两个无环性惩罚的 reward 函数 最终通过策略梯度和随机化方法来优化模型
  - 模型定义
    - 核心公式为：xi = fi(xpa(i), θ) + εi
    - 其中 xi 是变量 i 的值 xpa(i) 是 i 的父节点的值 θ 是参数 εi 是噪声（一般是高斯分布）
    - f 可以是线性的（比如矩阵）或者非线性的（比如神经网络）
    - 这里可以理解变量 i 是数据的一个特征或者一个属性
    - 论文需要做的事情本质上就是通过数据集 X（包含多个 x 向量 代表采样的数据）来学习出 DAG 图的结构 以及每个节点的函数 fi
  - 模型结构
    - 从 X 中采样 n 个样本 Xl 重塑为 S 其中每一个向量代表了 Xl 中一列的值 我们希望通过 S 生成一个二元邻接矩阵 A 使得 A 是 DAG 且有最佳评分
    - 编码器采用 Transformer 的编码器结构 记输出的向量为 enc_i
    - 解码器是单层的 g_ij(W1,W2,u)=u^T\*tanh(W1\*enc_i + W2\*enc_j) 其中 u 是可训练的 将 gij 输入 sigmoid 中 根据伯努利分布采样概率 得到 Aij 其中忽略所有的对角线元素
    - 论文提到解码器还可以选择 Transformer 解码器等其他模型 但是单层解码器效果最佳 推测是因为编码器已经学习到了足够的信息
  - 强化学习
    - 使用 Actor-Critic 的方法来优化模型
    - 预定义的评分函数采用了 BIC 评分 包含了两项 第一项是模型对于数据的似然函数 第二项是参数复杂度的惩罚项
    - 无环性采用 h(A) = trace(e^A)-d 当 h=0 时 A 是无环的 另外因为 h(A)极小 为了避免使用极大的惩罚权重 添加另一个惩罚项来平衡
    - 最终的 reward=-[S(G+λ1I(G 不属于 DAGs)+λ2h(A))] 其中 λ1 和 λ2 是可调的超参数
    - 论文提到 新的 reward 作为优化 不一定等价于不带无环性惩罚的最大化评分函数的优化 为此 论文证明了当 λ1 和 λ2 符合某个条件时 两者是等价的 在实际应用中需要选取合适的 λ1 和 λ2
    - Actor 包括了编码器和解码器 输入是采样的数据 S 输出是 DAG 其中编码器的输出 enc_i 作为解码器的输入
    - Critic 是一个简单 2 层的前馈神经网络 输入为编码器的输出 输出是预测的 reward
    - 训练时 目标函数 J 是即时奖励 即当前采样数据下此策略网络的期望奖励 可以认为折扣因子 γ=0
    - 状态 s 可以理解为随机采样的数据 而动作 a 则是生成的 DAG
    - Actor 的优化基于 REINFORCE 算法 具体而言 对 actor 作多次采样生成批量的 DAG 矩阵 然后计算 reward 也就可以估计出梯度 进行反向传播更新
    - Critic 的优化则是使用均方误差来更新 也就是预测 reward 和实际 reward 之间的均方误差
  - 最终遍历所有生成的 DAG 计算 reward 选择最优的 DAG 而非找到最优的策略网络
  - 为什么论文不直接使用梯度上升法？因为奖励函数不可导 本质上是用 critic 来拟合奖励函数（注意是针对某一个完整数据集 X 拟合 reward 函数 而不是泛化到所有数据集生成的 DAG 可以理解为记住了数据集的特征来评估 DAG）
  - 计算 reward 的耗时非常大 论文提到过记录已有的 reward 并且对 BIC 进行分解计算
- GANBLR: A Tabular Data Generation Mode
  - 背景：GAN 是一种生成模型 但是在表格数据生成上存在可解释性不足等问题 论文将 GAN 的 ANN 架构替换为经典贝叶斯模型
  - 具体而言 将生成器和判别器都替换为 BN
- FedTS（LZH）
  - 背景：联邦学习中 边缘设备的计算资源往往存在差异 为了充分利用高计算资源 客户端需要引入不同复杂度的模型 也就导致了模型的异构性 fedts 通过知识蒸馏帮助大模型辅助小模型训练 解决了模型异构性的问题 现有研究基本上关注的是数据特征的异构性 并且传统蒸馏需要师生模型共享数据 违反了隐私保护的原则 就算是针对于模型异构性的研究 也需要额外数据集来蒸馏 带来了额外的计算开销
  - 场景建模：k 个客户端 分为教师 T 和学生 S 其中 T 的计算资源较强 S 的计算资源较弱 模型为神经网络 进一步拆分为特征提取器 f 和分类器 g 师生共享相同的分类器结构 而在内部分别共享相同的特征提取器 目标是最小化所有客户端的损失函数之和
  - 传统知识蒸馏的目标函数为师生模型分类结果分布的 KL 散度 问题在于需要共享数据集
  - 解决问题的核心思想是在客户端之间共享生成器 G（代表了全局蒸馏出的知识） 具体而言 G 将标签 y 映射到特征向量 z 可以看做是分类器 g 的逆过程 但是是全局的
  - 生成器 G 在聚合阶段（客户端上传模型参数之后）由服务器进行训练 从数据集中随机采样出 c 作为训练数据 目标函数是最小化所有客户端的 g 对于 G 输入 c 后输出的 z 与真实标签 c 的损失函数 简单理解就是让 G 尽可能泛化的拟合所有客户端 g 的逆过程 其中权重参数 w 控制 G 向不同客户端的学习程度 与数据集中的客户端样本数量成正比 并且通过 rk 控制师生模型差异 在迭代初期 教师模型占比较大 随着迭代进行 学生模型占比逐渐增大 最后 有研究表明高置信度的客户端有更高的学习价值 它们又往往具有较低信息熵 因此权重参数的最后一项是信息熵的倒数的 softmax 归一化
  - 生成器 G 训练完后 其参数被下发给客户端 客户端在本地训练阶段将 G 用于知识库指导训练 f 和 g 本地客户端 k 的损失函数包括 4 项
    1. 本地数据集的损失函数 代表本地模型对数据集预测的误差
    2. 本地模型 f 与全局生成器 G 的损失函数 代表 f 与 G 的相似度
    3. 本地模型 g 与全局生成器 G 的损失函数 代表 g 与 G 的相似度 其中标签数据 ys 是全局随机采样的 保证 g 的泛化性
    4. 本地模型 f 输出的特征向量的正则项 用于获得更紧凑的特征空间
    - 训练过程相当于利用了全局蒸馏的知识加速了本地模型的训练 一方面使得 f 可以生成正确的特征 另一方面使得 g 获得更好的泛化性
    - 第 2 项中 如果本地数据集只包含某些类别的标签 那么其 f 模型应该是无法学习到其他类别的特征的
  - 本地客户端训练需要下发 3 个模型参数 f、g 和 G 其中 G 模型由上一步训练完成 而 f 和 g 需要进行全局聚合
    - 对于 f 直接采用数据集大小比例作为权重进行加权平均 分别对 T 和 S 聚合出 ft 和 fs（为何 f 可以直接加权？）
    - 对于 g 若客户端之间的标签分布不均匀 会导致 g 的训练过拟合 此时不能简单按照数据集比例 pk 进行加权 调整后的权重 ak 除了需要接近 pk 以外 还需要考虑客户端的 g 与其他所有客户端的 g 的平均余弦距离 这使得和其他客户端差异越大的 g 权重越小 反之则越大
  - 整体流程如下
    1. 初始化模型参数 f、g 和 G
    2. 选取样本客户端 k 进行本地训练
    3. 服务器通过客户端上传的 g 训练 G
    4. 服务器聚合 f 和 g 并下发给客户端
    5. 重复步骤 2-4 直到收敛
- EFFICIENT DIVERSITY-PRESERVING DIFFUSION ALIGNMENT VIA GRADIENT-INFORMED GFLOWNETS
  - 背景：扩散模型已经被广泛应用于各种生成任务 现如今大部分扩散模型的规模巨大 因此人们往往希望基于已有的预训练模型进行微调 通常使用奖励函数进行 然而传统 RL 存在收敛慢 多样性不足的问题 即使使用 GFN 也仍有优化空间 论文提出一种基于奖励函数梯度信息与预训练模型和微调模型的残差信息的损失函数 实现了高效的微调
  - GFN 的 DB 条件决定了损失函数 此外 GFN 和扩散模型天然具有可结合性 只需要将采样过程改为按时间采用即可
  - 论文提出的第一个损失函数基于奖励函数的梯度信息 通过对原 DB 条件进行求导 可以得到前向和后向的 δ-DB 条件
  - 仅仅通过 δ-DB 条件进行微调 可能会过度优化奖励 从而忽略了预训练模型的先验知识 若将微调模型和预训练模型的 δ-DB 条件相减 会消去相同的后向策略函数 可以得到残差 δ-DB 条件
  - 论文还进一步用前向预测技巧优化了流残差函数 最终得到损失目标
  - 可以看到 当前许多论文的研究方向都是找到一个合适的损失函数
- DAGNN
  - GNN 是针对图输入的神经网络 而 DAGNN 则特化于 DAG 图
  - 背景：MPNN 是基于消息传递的神经网络 其每一层输入的特征向量依赖于上一层输出的特征向量 以及对节点邻居进行聚合后的特征向量 DAGNN 在 MPNN 的基础上进行了改进 聚合操作的输入将依赖于当前层的父节点 这要求节点是可以拓扑排序的 符合 DAG 的特性 对于源点 聚合算子的结果是 0 另外 全局特征向量是每一层对于所有汇点进行池化后的输出
  - DAGNN 同样融入了注意力机制 也就是会计算节点之间的注意力权重作为聚合时的加权系数
  - DAGNN 引入双向处理 正向处理完毕之后 将 dag 反向 然后再对特征向量做一遍输入 得到的特征向量中 对汇点分别进行池化操作 然后拼接 相当于原图的源点和汇点
  - 拓扑批处理 使得 DAGNN 可以通过并行加速训练

# TODO

## 组会汇报

## PHAROS

- 基于 GM 先写一个索引层 只需要 2d+时间索引即可 先基于点存储查询写 以后可以拓展到轨迹数据
  - 还需实现查询接口 需要结合之前的查询下推
  - 表名不可以带斜杠 需要考虑一下如何修改
  - 目前 GM 的 schema 是固定为 taxi 的 后续加入转换逻辑适用于不同设备 转换逻辑可以是启动时发送请求获取 schema 然后存在内存里待用
  - GM 的时间索引可能不如遍历筛选 可能需要测试
- 将 tyc 的项目结合进灯塔
- 后续可能有篇论文可以发 预计暑假开始搞

## FL

- 要做的
  - 会生成有环图 但是可以保存生成过的 dag 进行数据增强
  - VAE 问题
    - loss 基本从第一个 epoch 后就收敛了
    - 对于离散特征列可能需要使用独热编码 并且针对每一个列训练单独的 decoder
    - 如果实在不行 也可以用 Transformer Encoder
    - mse 重构损失对于顺序敏感 不适合表格数据 因此我改用 stats+mmd 重构出来的数据看起来比较合理
  - tabppdm 问题
    - 把特征向量的列作为 label 进行训练 最终用条件变量控制生成不同的列特征向量
  - critic
    - 目前选择 gat 自动归一化且带 baseline 的模型进行预训练 并随机生成图来保证泛化性 可以把 vae 得到的特征向量作为节点的向量输入
- 目前有 2 种方案
  1. 还是用 dag 为输入 用一些其他库的方法 比如 bnlearn
     - 流程：先采样样本 样本输入 bnlearn 生成一批 dag 输入给 critic 输出预测 reward 然后计算 dag 的真实 reward 计算损失函数
  2. 用 enc_i 作为输入 要么输入的是原始数据采样 要么是用 ganblr 生成的相似数据 X'
     - 批量从原数据采样 形成[batch_size, max_length, n_samples]形状的 enc_i 输入给 decoder 生成 dag 进行训练 其中的 n_samples 是超参
- critic 和 actor 的 diffusion 先预训练 这两部分先并行做起来 看看效果 最后可以对比三种效果：完全交替训练、完全分阶段训练、先预训练再交替训练


## Other

- 查论文 谷歌学术 图书馆网站的电子数据库 VLDB ICDE SIGMOD ICDM EDBT 等会议 一是空间索引相关 二是查询优化 queryplan 清华李国良 ai 优化查询计划
- 后续可能要参与的工作
  - SQL 生成 类似的 灯塔的索引层也可以通过自然语言生成查询 最终返回结果 其中转换过程自己定 比如 NL 转换为时空具体范围 然后进行查询 或者转化为具体的查询窗口 另外比较重要的是 要根据数据的存储进行优化 这个是比较抽象的 具体方式可以用本地的 LLM
  - 发票的图神经网络 对于边敏感的神经网络可以用于识别行贿违法的 pattern 然后用模型学习并推理出哪些是异常的 另外还有个知识图谱
  - kfm 在做查询下推相关 我可以看看相关论文 结合到索引层
