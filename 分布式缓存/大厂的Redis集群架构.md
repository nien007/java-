- [类 codis 架构](#--codis---)
  * [codis](#codis)
  * [阿里云](#---)
  * [百度 BDRP 2.0](#---bdrp-20)
  * [京东](#--)
- [基于官方 redis cluster 的方案](#-----redis-cluster----)
  * [官方 redis cluster](#---redis-cluster)
  * [AWS ElasticCache](#aws-elasticcache)
  * [百度贴吧的ksarch-saas：](#-----ksarch-saas-)

---
## 面试问题

![](https://img-blog.csdnimg.cn/20210330164808591.png)

##  redis 集群方案

业界主流的Redis集群化方案主要包括以下几个：

- 客户端分片

- 代理分片
 > codis 的架构，按组划分，实例之间互相独立；
- 服务端分片

 > 基于官方的 redis cluster 的方案；


##   大厂的 redis 集群方案

大厂的 redis 集群方案主要有两类，一是使用类 codis 的架构，按组划分，实例之间互相独立；
另一套是基于官方的 redis cluster 的方案；下面分别聊聊这两种方案；

## 类 codis 架构

![img](https://img-blog.csdnimg.cn/20181220143701631)

这套架构的特点：

- 分片算法：基于 slot hash桶；
- 分片实例之间相互独立，每组 一个master 实例和多个slave；
- 路由信息存放到第三方存储组件，如 zookeeper 或etcd
- 旁路组件探活

使用这套方案的公司：
阿里云： ApsaraCache, RedisLabs、京东、百度等

### codis

slots 方案：划分了 1024个slot， slots 信息在 proxy层感知； redis 进程中维护本实例上的所有key的一个slot map；

迁移过程中的读写冲突处理：
最小迁移单位为key；
访问逻辑都是先访问 src 节点，再根据结果判断是否需要进一步访问 target 节点；

- 访问的 key 还未被迁移：读写请求访问 src 节点，处理后访问：
- 访问的 key 正在迁移：读请求访问 src 节点后直接返回；写请求无法处理，返回 retry
- 访问的 key 已被迁移(或不存在）：读写请求访问 src 节点，收到 moved 回复，继续访问 target 节点处理

### 阿里云

AparaCache 的单机版已开源(开源版本中不包含slot等实现)，集群方案细节未知；[ApsaraCache](https://github.com/alibaba/ApsaraCache)

### 百度 BDRP 2.0

主要组件：
proxy，基于twemproxy 改造，实现了动态路由表；
redis内核： 基于2.x 实现的slots 方案；
metaserver：基于redis实现，包含的功能：拓扑信息的存储 & 探活；
最多支持1000个节点；

slot 方案：
redis 内核中对db划分，做了16384个db； 每个请求到来，首先做db选择；

数据迁移实现：
数据迁移的时候，最小迁移单位是slot，迁移中整个slot 处于阻塞状态，只支持读请求，不支持写请求；
对比 官方 redis cluster/ codis 的按key粒度进行迁移的方案：按key迁移对用户请求更为友好，但迁移速度较慢；这个按slot进行迁移的方案速度更快；

### 京东

主要组件：
proxy: 自主实现，基于 golang 开发；
redis内核：基于 redis 2.8
configServer(cfs)组件：配置信息存放；
scala组件：用于触发部署、新建、扩容等请求；
mysql：最终所有的元信息及配置的存储；
sentinal（golang实现）：哨兵，用于监控proxy和redis实例，redis实例失败后触发切换；

slot 方案实现：
在内存中维护了slots的map映射表；

数据迁移：
基于 slots 粒度进行迁移；
scala组件向dst实例发送命令告知会接受某个slot；
dst 向 src 发送命令请求迁移，src开启一个线程来做数据的dump，将这个slot的数据整块dump发送到dst（未加锁，只读操作）
写请求会开辟一块缓冲区，所有的写请求除了写原有数据区域，同时双写到缓冲区中。
当一个slot迁移完成后，把这个缓冲区的数据都传到dst，当缓冲区为空时，更改本分片slot规则，不再拥有该slot，后续再请求这个slot的key返回moved；
上层proxy会保存两份路由表，当该slot 请求目标实例得到 move 结果后，更新拓扑；

跨机房：跨机房使用主从部署结构；没有多活，异地机房作为slave；

## 基于官方 redis cluster 的方案

![img](https://img-blog.csdnimg.cn/20181220143701647)

和上一套方案比，所有功能都集成在 redis cluster 中，路由分片、拓扑信息的存储、探活都在redis cluster中实现；各实例间通过 gossip 通信；这样的好处是简单，依赖的组件少，应对400个节点以内的场景没有问题（按单实例8w read qps来计算，能够支持 200 * 8 = 1600w 的读多写少的场景）；但当需要支持更大的规模时，由于使用 gossip协议导致协议之间的通信消耗太大，redis cluster 不再合适；

使用这套方案的有：AWS, 百度贴吧

### 官方 redis cluster

数据迁移过程：
基于 key粒度的数据迁移；
迁移过程的读写冲突处理：
从A 迁移到 B;

- 访问的 key 所属slot 不在节点 A 上时，返回 MOVED 转向，client 再次请求B；
- 访问的 key 所属 slot 在节点 A 上，但 key 不在 A上， 返回 ASK 转向，client再次请求B；
- 访问的 key 所属slot 在A上，且key在 A上，直接处理；（同步迁移场景：该 key正在迁移，则阻塞）

### AWS ElasticCache

ElasticCache 支持主从和集群版、支持读写分离；
集群版用的是开源的Redis Cluster，未做深度定制；

### 百度贴吧的ksarch-saas：

基于redis cluster + twemproxy 实现；后被 BDRP 吞并；
twemproxy 实现了 smart client 功能；使用 redis cluster后还加一层 proxy的好处：

1. 对client友好，不需要client都升级为smart client；（否则，所有语言client 都需要支持一遍）
2. 加一层proxy可以做更多平台策略；比如在proxy可做 大key、热key的监控、慢查询的请求监控、以及接入控制、请求过滤等；

即将发布的 redis 5.0 中有个 feature，作者计划给 redis cluster加一个proxy。

---


##  史上最全面试宝典《Java面试红宝石》 免费领取

![](https://img-blog.csdnimg.cn/20210329213023227.png)

## 惊：书中题目，在面试中直接出现

![](https://img-blog.csdnimg.cn/2021032921325244.png)



![](https://img-blog.csdnimg.cn/202103292135075.png)
