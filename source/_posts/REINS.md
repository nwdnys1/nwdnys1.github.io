---
title: 在SJTU-REINS的打工日记
date: 2024-10-05 21:19:00
categories: 实验室
tags:
  - 打工
index_img:
banner_img:
excerpt: "大三开学顺利和 chp 面谈并进组开始打工 这个博客用来记录一下每一份工作项目的经历和经验"
---

大三开学顺利和 chp 面谈并进组开始打工 这个博客用来记录一下每一份工作项目的经历和经验

# LIGHTHOUSE

灯塔项目是一个类似于海上管理船只位置的云系统的项目 只不过空间由海域变为上海市区 对象由船只变为无人机等移动物联网设备

简单来说 灯塔系统会收集处于三维空间内各个个体的数据 并存储在一个云边融合系统（CEDS）内 之后将数据整理封装为态势数据 将态势数据分发回给每个个体 让个体通过实时的态势数据进行任务的规划和调度 态势数据会分为两部分 一部分是以个体为中心 在单位时间内可以安全独占的空间范围 另一部分是以个体为中心 依据任务需要而划分的一块空间范围内所有其它个体的数据

我负责的部分主要是从已有的 CEDS 系统中提取需要的数据 并封装为合适格式的态势数据进行分发 比较简单

## CEDS

CEDS 是黄子昂学长的硕士毕业论文中提出的云边融合存储系统 利用了边缘节点本身少量的数据存储与计算能力以及其分布于移动物联网设备附近的特点 实现了基于数据的就近存储、查询任务的划分下推以及负载感知的数据迁移功能的存储系统 做到了传输带宽的减少以及查询的加速 具体实现可以见其论文 我简单讲一下我的理解

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

### 部署

跟随仓库中的指导 在自己的电脑上部署一下 CEDS

1. 用 IDEA 跑起中心节点 需连接本地的 MySQL 和 MongoDB 服务
2. 用容器部署 12 个 MongoDB 作为边缘节点的数据库 再用容器部署 12 个边缘节点（基于 python） 手动在本地运行一个 center 节点用于查询 会监听 8000 端口
3. 本地运行 parse_csv 解析数据集 load_data 将数据导入边缘节点 最后用 register 接口把固定的元数据上传到数据中心
4. 运行各个查询进行试验即可

## 工作内容

### 思路

我的任务很简单 只需要进行查询后封装为相应格式的态势数据即可 即学会如何查询与如何使用正确的函数进行封装

### 如何查询所需数据

- 首先需要注册元信息 告诉数据中心这个元数据组的字段都是什么样的

  CEDS 提供了注册接口`/metaData/register` 请求体形如下：

  ```
  {
        "id": "6701140529b28d4cd90959d0",
        "topicPrefix": "taxi",
        "collectionName": "taxi",
        "payloadType": "json",
        "timeField": "TIMESTAMP",
        "fields": {
            "TAXI_ID": {"name": "TAXI_ID", "type": "str"},
            "TRIP_ID": {"name": "TRIP_ID", "type": "str"},
            "LONGITUDE": {
                "name": "LONGITUDE",
                "type": "float",
                "min": "-180",
                "max": "180",
                "granularity": "0.001",
            },
            "SPEED": {
                "name": "SPEED",
                "type": "float",
                "min": "0",
                "max": "120",
                "granularity": "1",
            },
            "LATITUDE": {
                "name": "LATITUDE",
                "type": "float",
                "min": "-180",
                "max": "180",
                "granularity": "0.001",
            },
        },
  }
  ```

  后续我们应该要先定好数据的元信息 然后进行测试

- CEDS 提供了查询接口`/statistic/searchTables`来查询数据相关的分片信息 接口的请求体格式类似如下：

  ```
      {
          "topic": topic,
          "realtime": False,
          "startTime": t_start * 60,
          "endTime": t_end * 60,
          "filters": [
              {"field": "LONGITUDE", "op": "bet", "val1": lon_start, "val2": lon_end},
              {"field": "LATITUDE", "op": "bet", "val1": lat_start, "val2": lat_end},
          ],
      }
  ```

  topic 即资源路径 realtime 我查看了代码 是注释掉的 用不着 startTime 和 endTime 就是时间范围 filter 内是各个字段的范围 比如这里有经度和纬度 经度对应的范围 op 是运算符 目前的实现中字符串只支持 equal 数值只支持 bet

  响应的数据形如下：shards 是数据包含在哪些节点的哪些分表里 比如这里是节点 1 的 05 到 11 分表 parseConfig 就是之前此元数据组注册的摘要

  ```
      {
          "shards": {"node1": [118605, 118606, 118607, 118608, 118609, 118610, 118611]},
          "parseConfig": {
              "id": "6701140529b28d4cd90959d0",
              "topicPrefix": "taxi",
              "collectionName": "taxi",
              "payloadType": "json",
              "timeField": "TIMESTAMP",
              "fields": {
                  "TAXI_ID": {"name": "TAXI_ID", "type": "str"},
                  "TRIP_ID": {"name": "TRIP_ID", "type": "str"},
                  "LONGITUDE": {
                      "name": "LONGITUDE",
                      "type": "float",
                      "min": "-180",
                      "max": "180",
                      "granularity": "0.001",
                  },
                  "SPEED": {
                      "name": "SPEED",
                      "type": "float",
                      "min": "0",
                      "max": "120",
                      "granularity": "1",
                  },
                  "LATITUDE": {
                      "name": "LATITUDE",
                      "type": "float",
                      "min": "-180",
                      "max": "180",
                      "granularity": "0.001",
                  },
              },
          },
      }
  ```

  当你向 8000 端口的查询中心发送请求后 其会先向云端数据中心发送上述请求并得到子节点列表 然后进行查询下推 返回结果

- 向 8000 的查询中心发送请求可以进行查询 这一步就是用户做的了 或者说我要做的 进行查询后封装

  请求体如下：

  ```
  {
    "function": "chadingdan",
    "module": "centerTasks",
    "filter": {
        "topic": "taxi",
        "realtime": False,
        "startTime": 1372662000,
        "endTime": 1372669200,
        "filters": [{"field": "TRIP_ID", "op": "str_eq", "val1": trip_id}],
    },
    "args": {"no": "args"},
  }
  ```

  这个请求会查询出 TRIP_ID 等于某一个值的所有数据 即某一个订单的轨迹数据 可以看到 function 是写死的类型 只针对了 taxi 一种数据组 不方便解耦 以后可以在应用层级上拓展

  论文内还测试了另外两种查询 一个是指定地区内经过的所有车辆 id 一个是时间范围内所有超速 60 的车辆 id 后者没有找到测试代码 不过查询的逻辑应该同理很简单

- 针对态势数据需要的查询 我们可以仔细分析一下：
  1. 第一部分态势数据需要确定一个 robot 在接下来某一段时间内可以安全独占的空间范围 重点是如何计算这个范围 比如查询附近某一小范围的所有 robot 然后根据速度信息进行计算？
  2. 第二部分态势数据需要确定一个 robot 在当前时间某一个大范围内所存在的其他 robot 的数据 重点是如何确定范围以及如何获取最新数据

### 汇报

- 个人理解：

  各个物联网设备会不断上传注册好的元数据流到边缘节点 那么首先我们可能要定下一种用于测试的数据格式？

  接下来就需要实时根据数据进行查询 生成态势数据返回给各个设备 先查询出需要的数据 然后根据数据计算出态势数据 最后进行适当封装即可

  - 如何查询？

    - 对于独占空间 可能需要查询一定范围内的所有设备数据（包括位置 速度等） 然后根据这些数据计算出独占空间
    - 对于感知范围内的其他设备数据 由于 CEDS 没有提供查询时间戳最新的数据 因此如何获取最新数据可能需要思考一下

  - 如何计算？

    - 对于不同的元数据组 比如无人机和手机两个组 查询出来的数据格式不一样 要通过这些数据计算得到统一的态势数据格式 可能需要思考一下如何处理

    - 对于固定范围内的其他设备的态势数据 可以固定所有设备的某些字段是必须的 比如空间位置、速度 把不同元数据组的设备视为同一种空间物体 用统一的方式使用这些字段来计算态势数据

    - 对于可以独占的空间 除了选取附近的设备数据外 还需要考虑每个设备本身的属性 比如形状大小 可以把这些属性作为函数的参数传入 用统一的函数计算出独占空间

  - 如何获取某一个空间范围内所有设备的实时位置？
    - 如果从所有时间范围内的数据查询最新数据 首先需要确定哪些设备在这个范围内 这个就不现实 因为不可能把所有设备的数据都查询一遍 后续查询最新时间戳的数据也不现实 因为可能需要查找很多个时间分表
    - 不妨查询这个空间范围内、某一单位时间内的所有数据 比如查询 10s 内的 找到这些数据后再根据时间戳筛选出最新的数据 如果一个设备 10s 内都没有数据 已经可以认为这个设备失去连接或者延迟太大了 10s 外的数据就算有也失去了实时性
    - CEDS的查询接口的函数都写死了 比如chadingdan就是筛选某一个TRIP_ID的数据 tongjiche则是筛选经纬范围内的所有TAXI_ID 我先新写了一个chajingwei的函数 查询指定时间范围与经纬范围内的所有数据 然后再写了一个getNewest的接口 会使用chajingwei方法查询当前时间前10s内的指定经纬范围内的数据 然后将查询得到的数据遍历一遍 得到时间戳最大的数据即为最新数据
    

### 10.11 组会

- 强化学习进行独占空间的计算 需要输入哪些数据 目前看来需要与人的距离以及实时速度、加速度
- 态势数据累计到一定程度后 使用这些数据进行空间使用效能的分析 并根据这些分析找到可以提升效能的点 比如算法的改进 或是空间资源的调度
- 态势数据统一格式 设备的上传数据也需要屏蔽差异性

### 10.18 组会

- 元数据更新有6s的延迟 因此需要额外的信息来找到实时位置 大致思路是设备经过一个边缘节点时 边缘节点向中心发送进入的时间戳索引 离开时发送离开的时间戳索引 这样就可以通过这两个时间戳知道设备实时位于哪个边缘节点中 索引形如<设备ID, 边缘节点ID, 进入时间戳, 离开时间戳> 通过这个索引可以找到设备在哪个边缘节点中的数据分表 可以通过kafka的连接和断开事件来触发这个索引的发送