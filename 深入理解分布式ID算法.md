# 深入理解分布式唯一ID
> 关键词: 分布式 唯一
> 目标: 可以自主选择、分析乃至设计的分布式唯一id方案

## 背景和引入
### 雪花算法原理简述
<img width="960" alt="image" src="https://github.com/geek-pie/Billions-Of-Traffic/assets/22070982/623cbfcb-1d75-4b28-af48-8144a51e230a">

Snowflake算法描述：同一时刻 & 指定机器 & 某一并发序列，是唯一的。据此可生成一个64 bits的唯一ID（long）。

## ID生成时需要考虑的内容
### ID长度带来的影响*
关键: 考虑缓冲区的索引场景空间有限

在MySQL中，主键的长度会影响到查询性能。这是因为InnoDB存储引擎使用B+树数据结构来存储数据库表中的数据，表的主键存储在B+树的叶子节点上，叶子节点上还存储着主键所关联的行记录数据¹。

如果主键过长，那么每个索引都需要存储这个值，这将导致以下几个问题：
1. **内存使用**：在数据量大，内存珍贵的情况下，MySQL有限的缓冲区，存储的索引与数据会减少。
2. **磁盘IO**：由于缓冲区中存储的索引和数据减少，磁盘IO的概率会增加。
3. **磁盘空间**：索引占用的磁盘空间也会增加。

因此，为了提高查询性能和节省存储空间，建议使用较短的主键。

> All indexes other than the clustered index are known as secondary indexes. In InnoDB, each record in a secondary index contains the primary key columns for the row, as well as the columns specified for the secondary index. InnoDB uses this primary key value to search for the row in the clustered index. ***If the primary key is long, the secondary indexes use more space, so it is advantageous to have a short primary key***.


### ID数据类型带来的影响*
在MySQL中，主键的选择需要根据具体的业务需求和系统性能要求来综合考虑。数值型主键和字符型主键各有其优点和适用的场景¹²。

**数值型主键**（如INT、BIGINT）的优点包括¹：
1. **空间效率高**：数值型占用的存储空间相对较小，比字符型更节省空间。
2. **查询效率高**：数值型主键在索引和排序操作上更高效，因为整数比字符型更容易进行比较和排序。
3. **简单易用**：数值型主键可以通过自增长序列来生成，简化了数据的插入和维护过程。

**字符型主键**（如CHAR、VARCHAR）的优点包括¹：
1. **可读性强**：字符类型主键能够直观地反映其所代表的实际含义，方便开发人员和用户理解。
2. **灵活性高**：字符类型主键可以包含更多的信息，如有时需要使用复合键来标识数据，字符类型可以较好地满足这个需求。
3. **数据复用性高**：字符类型主键可以尽可能利用数据库已有的字符数据，避免额外冗余存储。
【2/3两点由于引入了有语义的内容，在一定程度下需要考虑安全性问题。另外也要根据场景使用确认不会出现变更的语义信息来组成字符型主键】

> id中引入有语义的内容的缺点
>
> 上述2/3两点描述了引入有语义的内容到id生成规则中的优点，但其劣势也需要注意：
> - 由于带有语义信息，在一定程度下需要考虑安全性问题
> - 语义信息必然要结合场景，要找到业务语境内均有效的通用业务语义，可能会有一定工作量；项目切换时也可能难以复用
> - 结合上述信息，带有丰富语义的id反而较难“全局唯一”，难以适用于全局抽象理念下的设计。如liunx语境下一切皆为文件，k8s语境下一切皆为资源

以下是一些比较重要的考虑因素¹：
- **存储空间**：数值型主键相对较小，节省存储空间；而字符型主键根据具体长度而定，可能占据更多的存储空间。
- **查询效率**：数值型主键在索引和排序操作上更高效；而字符型主键在特定场景下可能效率更高，如需要根据字符串进行模糊搜索或排序。
- **可读性**：字符型主键更直观易懂，方便开发人员和用户理解；而数值型主键则需要额外的映射关系才能理解其含义。
- **数据复用性**：字符型主键可以利用已有字符数据来标识数据，避免冗余存储；而数值型主键则需要额外的关联表来实现类似功能。

### ID是否有序带来的影响*
ID顺序递增类型和带来的影响* ：
- 连续递增
- 单调递增
- 趋势递增 (描述一组数据整体上的增长趋势，而不是严格的每一项都比前一项大。在这种情况下，可能存在一些局部的下降，但是整体的趋势是向上的)

在MySQL中，主键是否递增对数据库的性能和数据一致性有重要影响：

1. **性能优化**：设计MySQL表时，我们一般会设置一个自增主键，从而让主键索引尽可能的保持递增的趋势，这样可以[避免页分裂，让MySQL顺序写入，大大提高MySQL的性能](https://segmentfault.com/a/1190000043870965)。

2. **热点数据判断**：参考LRU原则，有序情况下可以判断ID对应数据的相对生成时间远近，便于区分冷热数据

> 连续递增和单调递增在并发场景下难以保证。综合来看，想要通过主键顺序得到性能buff，做到趋势递增是一种折中选择

### 安全性考虑
- MAC/IP地址泄露
- 后续Id被提前预测
- 从Id数量被计算出期间Id数

## 常见的ID生成方式
### UUID
UUID(Universally Unique Identifier)的标准型式包含32个16进制数字，以连字号分为五段，形式为8-4-4-4-12的36个**字符**，示例：550e8400-e29b-41d4-a716-446655440000，到目前为止业界一共有5种方式生成UUID，详情见IETF发布的UUID规范 A Universally Unique IDentifier (UUID) URN Namespace。
#### 优点：
性能非常高：本地生成，没有网络消耗。
#### 缺点：
- 不易于存储：UUID太长，16字节128位，通常以36长度的字符串表示，很多场景不适用。
- 信息不安全：基于MAC地址生成UUID的算法可能会造成MAC地址泄露，这个漏洞曾被用于寻找梅丽莎病毒的制作者位置。
- ID作为主键时在特定的环境会存在一些问题，比如做DB主键的场景下，UUID就非常不适用：
  - MySQL官方有明确的建议主键要尽量越短越好，36个字符长度的UUID不符合要求。
  - 对MySQL索引不利：如果作为数据库主键，在InnoDB引擎下，UUID的无序性可能会引起数据位置频繁变动，严重影响性能。
### 数据库自增ID
#### 优点：
- 简单易用
#### 缺点：
- 性能低

## [重点]分布式场景下对ID的要求
### 全局唯一
- 数据错误
- 回滚代价
- 性能影响
### 高并发
### 高可用
### 多数条件下要避免业务语义
考虑到高可用依赖组件带来的资源成本，复杂度以及维护工作量，稳定且通用方案是我们期待的，因此需要尽量避免业务语义。

## 普通的id演进为分布式的思考路径
### 全局唯一 高性能 高可用
保障全局唯一的思路(多实例下正确分配):
- 单实例内部避免重复
- 多实例间避免重复 维持隔离协定/分布式算法
  - 维持隔离协定
    - 分段
    - 步长、hash
    - 单实例id
  - 分布式算法
    - 高性能分布式锁直接维护id
      - redis自增
    - 分布式算法维护单实例id(雪花算法)
      - zookeeper

提高性能的思路:
- 不同步落盘
- 预分配

提高可用性的思路(多实例下高效分配, 容忍单点故障):
- 依赖的协调组件能高可用部署
- 对协调组件弱依赖(容忍一定程度的不可用): 
  - 预协调
  - 持久化
- 避免实例增减对原有隔离协定的结果造成影响
  - 一致性hash

## 分布式场景下的ID生成方式
### 步长自增
穿插式生成id
#### 优点
- 离线确认步长和起始值，理想情况下仅需一次确认
- 不同实例可以完全离线地生成id
- 趋势递增
#### 缺点
- 实例内部需要持久化记录自己上一个生成的id，以便在重启发生后继续工作，效率受限
- 难以横向拓展
### 分段自增
各自为营式生成id
#### 优点
- 获取号段后，不同实例可以完全离线地生成id
- 趋势递增
#### 缺点
- 实例内部需要持久化记录自己上一个生成的id，以便在重启发生后继续工作，效率受限
- 号段用尽后需要协调下一个号段，需要一个高可用的协调方案
### redis自增
通通排队生成id
#### 优点
- 简单易实现，可从0开始，能做到号段不浪费
#### 缺点
- 考虑redis本身的性能上限
- 考虑与redis间通信的时延

## 实际生产过程中的权衡和考虑
- 成本与收益
    极低冲突概率 + 冲突后影响可接受 的情况下，往往可以根据成本考虑，不要求达到完全不可能重复
- 真实性能需求
- 考虑ID数量

## 雪花算法及其衍生实现
### 雪花算法理论
#### 雪花Id的组成
#### workerId的分配
```
|1|-------------------41--------------------|-----10---|-----12-----|

从左到右
[1]首位: 二进制下的符号位，0表示正数，1表示负数
[41]时间戳位: 毫秒级时间戳，放在高位，保障单实例内部毫秒级的递增、保障id的整体趋势递增
[10]机器Id位
[12]sequenceId,序列号位
```
- 时间戳位往往参考一个基准时间，用当前时间戳减去基准时间后的结果填入时间戳位
    - 增加可用id数
    - 避免直接被解析出真实时间戳
- 理论上最多支持的同时可用实例数: 2^10 台
- 单实例上每毫秒内最多能生成的id数: 2^12 个
> 雪花算法实现时的左移操作目的就是将上述序列移动至对应位置
#### 优势
##### 原理简单，高性能，易扩展
> 可以说实际上snowflake的本质原理依然可以归纳到步长+号段。最高位为毫秒时间戳，对于每一个worker而言，即自动维护了1ms的步长; 而高位的时间戳+workerId，就提前协定分配了每毫秒中可用的号段。雪花算法的精妙之处，是在于借助了计算机系统成熟通用的毫秒级时间戳来维护上述需求，无需额外定义概念和实现。

#### 问题和缺陷
##### ***重点问题*** 时钟回拨导致的问题
问题描述: 由于NTP的存在，当本地时钟(RTC)可能出现一定漂移，需要进行时钟同步。若是本地时间更新，而需同步时间早，此时的同步即为时钟回拨。时钟回拨是NTP协议下的正常行为，只是时钟回拨会导致雪花算法再次取到已经用过的时间戳，造成id重复。

应对方案:
- 给id生成任务指定专用实例，系统设置关闭NTP时钟同步。要确保该实例上运行的其他服务可接受影响，此外也要求服务器自身RTC的正常运行
- 内存记录上一次生成的id，每一次生成后进行校验，是否时间戳回拨，若是则丢弃当前生成Id，进行重试，直至时间戳增加至满足条件
> 内存记录的方案如果出现时钟回拨期间恰好机器重启怎么办
> - 一般情况下时钟回拨的时长往往要远小于服务器重启所需时长，多数情况下机器重启后，本地时钟已经超过了回拨前的最大值
> - 极高要求下，可以进行落盘记录。每一个时间戳都落盘记录是不合理的。可以选择定时同步，冗余丢弃的方案，如每X秒同步记录一次，到下一次记录前确保上一次记录成功，如果发生时钟回拨，当本地时间大于X时才启用功能。

##### ***次要问题*** Id数量和浪费
问题描述: 由于id长度一定，可用id的个数是有限的，在数据量较大的场景中，要关注到这点。此外，由于毫秒级时间戳在高位，低位段长度还剩22位，意味着系统每毫秒的流逝，将减少2^22个，未经使用的id资源将丢弃，且这种丢弃是被动的，无法主动控制。

> 当然，大多数的需求场景中都不至于因此而直接导致系统问题，并且我们考虑软件架构设计一般能满足理论上最多20年左右的使用已经很足够了。不过，理论上的足够的冗余总是一件好事。参考IPv4的设计，虽然一开始看起来觉得很难，但面对它即将耗尽的事实，考虑这个问题也是有意义的。

应对方案: 
- 基准时间的引入
> 41位的毫秒级时间戳理论上可以用约69年，其起始时间是1970。引入比1970更大的基准时间，将可以让我们用更长的时间
- 将一套id限制在一个业务语境内。比如用于表示订单的id和用于表示商品的id值相同，这是很常见的，全局唯一更多情况下只是需要分类后的范围内的全局唯一

### 雪花算法的实现 之 [美团Leaf](https://tech.meituan.com/2017/04/21/mt-leaf.html) (重点推荐阅读)
#### Leaf-segment
- 依赖数据库，维护并获取号段
- 提前获取号段，避免临时获取造成阻塞
- 依赖的数据库HA+DR方案
#### Leaf-snowflake
[关键代码](https://github.com/Meituan-Dianping/Leaf/blob/master/leaf-core/src/main/java/com/sankuai/inf/leaf/snowflake/SnowflakeZookeeperHolder.java)
```
|1|-------------------41--------------------|-----10---|-----12-----|
```
- 1+41+10+12 完全沿用snowflake方案的bit位设计
- 引入ZooKeeper，分配和记录workId，记录由worker定时上传的本地时间
- worker本机存储workId达到对ZooKeeper弱依赖
- ZooKeeper记录 + 关闭NTP/等待时钟 等方案解决时钟回拨问题

### 雪花算法的实现 之 [百度UID](https://github.com/baidu/uid-generator/blob/master/README.zh_cn.md)
**默认**的bit分配
```
|1|------------28--------------|-----------22---------|-----13------|

[1]sign(1bit) 固定1bit符号标识，即生成的UID为正数。
[28]delta seconds (28 bits)
当前时间，相对于时间基点"2016-05-20"的增量值，单位：秒，最多可支持约8.7年
[22]worker id (22 bits)机器id，最多可支持约420w次机器启动。内置实现为在启动时由数据库分配，默认分配策略为用后即弃，后续可提供复用策略。
[13]sequence (13 bits)每秒下的并发序列，13 bits可支持每秒8192个并发。
```
- 通过MySQL来维护workerId，由于加长了机器Id位，有条件支持用后即弃的策略，来降低workerId的同步协调成本
- **借用未来时间**：UidGenerator通过借用未来时间来解决sequence天然存在的并发限制。
- **RingBuffer缓存**：采用RingBuffer来缓存已生成的UID，实现UID的生产和消费的并行化。
- **避免伪共享**：通过CacheLine补齐，避免了由RingBuffer带来的硬件级「伪共享」问题。
- **自定义workerId位数和初始化策略**：UidGenerator以组件形式工作在应用项目中，支持自定义workerId位数和初始化策略，从而适用于docker等虚拟化环境下实例自动重启、漂移等场景。
- **高性能**：最终单机QPS可达600万。

> 基于Java实现的Snowflake算法，bits分参数均可通过Spring进行自定义，可以根据实际业务需求根据 最大并发数/期望使用时间/节点重启频繁度 等进行自定义。并且给出了官方的配置参考，可以与mybatis

#### [DefaultUid](https://github.com/baidu/uid-generator/blob/master/src/main/java/com/baidu/fsg/uid/impl/DefaultUidGenerator.java)
#### [CachedUid](https://github.com/baidu/uid-generator/blob/master/src/main/java/com/baidu/fsg/uid/impl/CachedUidGenerator.java)

### 雪花算法的实现 之 [Mybatis-Plus中的雪花算法](https://github.com/baomidou/mybatis-plus/blob/3.0/mybatis-plus-core/src/main/java/com/baomidou/mybatisplus/core/incrementer/DefaultIdentifierGenerator.java)
[默认实现的关键代码](https://github.com/baomidou/mybatis-plus/blob/3.0/mybatis-plus-core/src/main/java/com/baomidou/mybatisplus/core/toolkit/Sequence.java)

```
|1|-------------------41--------------------|--5--|--5--|-----12-----|

```
- 将机器Id拆分为datacenterId + workId，根据两套规则来生成，最终组合到一起。本质上，机器Id的可用量没有变化。
    - 作用1: 将生成 机器Id 的因子数增加
    - 作用2: 以多数据中心的角度实现了一种分组方案

> - 主要提供的是一个接口，基于框架不引入额外组件依赖的前提
> 
> - 其雪花算法的默认实现，没有用户手动指定的情况下只能尽可能让不同机器上[macId/IP]、不同进程的实例生成不同的workId
> > 需要注意的是，上述“尽可能”无法确保 datacenterId + workId 的不重复。因为即使是尽可能选取唯一标识来生成，但过了一次hash(匹配数据长度和类型)之后，有几率重复。此外，上述“唯一标识”在一些情况下反而是更加容易导致碰撞的
> > - 实现的缺陷。[datacenterId 默认值是固定的，通过mac地址/IP进行更新](https://github.com/baomidou/mybatis-plus/blob/8e270f797e79862f2845d741f10077b2e5b8bfaa/mybatis-plus-core/src/main/java/com/baomidou/mybatisplus/core/toolkit/Sequence.java#L130C20-L130C35)。一旦存在权限等问题无法获取到本机network，将都设置相同
> > - 运行环境因素。[workId 获取的是java程序的进程Id](https://github.com/baomidou/mybatis-plus/blob/8e270f797e79862f2845d741f10077b2e5b8bfaa/mybatis-plus-core/src/main/java/com/baomidou/mybatisplus/core/toolkit/Sequence.java#L111C19-L111C19)，但由于进程ID是顺序分配的，在容器化环境中，反而使得workId与其他实例的中的workId冲突概率更高
> 
> 分布式场景下需要用户自己考虑合适的workId方案去实现接口。

## 参考 (以下参考都是写出提纲和之后让GPT生成内容时，GPT摘抄来源的参考。其中质量参差不齐尚未筛选)

https://zhuanlan.zhihu.com/p/397680307

https://zhuanlan.zhihu.com/p/154480290

- ID长度带来的影响
> (1) 为什么MySQL设计主键时key不能过长？ - 简书. https://www.jianshu.com/p/31bfd4ac3e61.
>
> (2) 数据库，主键为何不宜太长？ - 知乎. https://zhuanlan.zhihu.com/p/336082481.
>
> (3) mysql的主键超过最大值会发生什么？ - zhengbiyu - 博客园. https://www.cnblogs.com/zhengbiyu/p/17301057.html.

- ID数据类型带来的影响
> (1) MySQL主键使用数值型和字符型的区别-CSDN博客. https://blog.csdn.net/xhaimail/article/details/132732033.
>
> (2) 数据库的主键应该选择什么数据类型比较好？ - 知乎. https://www.zhihu.com/question/30888980.
>
> (3) MySQL：VARCHAR 可以作为主键吗？ - 极客教程. https://geek-docs.com/mysql/mysql-ask-answer/425_mysql_can_i_use_varchar_as_the_primary_key.html.
>
> (4) MySQL: VARCHAR作为主键使用吗？|极客笔记. https://deepinout.com/mysql/mysql-questions/425_mysql_can_i_use_varchar_as_the_primary_key.html.

- ID是否有序带来的影响
> (1) 面试官：MySQL主键为什么不是连续递增的？-mysql设置主键自增长 - 51CTO. https://www.51cto.com/article/>743188.html.
>
> (2) 面试官：MySQL主键为什么不是连续递增的？-CSDN博客. https://blog.csdn.net/qq_33312725/article/details/>128462658.
>
> (3) MySQL数据库——MySQL AUTO_INCREMENT：主键自增长 - CSDN博客. https://blog.csdn.net/Itmastergo/article/>details/130260568.
>
> (4) 为什么 MySQL 的自增主键不单调也不连续 - 知乎. https://zhuanlan.zhihu.com/p/202960807.
>
> (5) 为什么主键索引建议顺序递增？ - 知乎. https://www.zhihu.com/question/588206465.

- ID顺序递增类型和带来的影响
> (1) 增区间和增函数的区别？ 单调递增区间和增区间的区别？ - 百度知道. https://zhidao.baidu.com/question/460078269.html.
> 
> (2) 5种全局ID生成方式、优缺点及改进方案 - 知乎. https://zhuanlan.zhihu.com/p/397680307.
> 
> (3) 系统架构：分布式ID那点事儿 - 知乎. https://zhuanlan.zhihu.com/p/107592567.
> 
> (4) 分布式ID_复杂分布式系统中,往往需要对大量的数据和消-CSDN博客. https://blog.csdn.net/baidu_38900596/article/details/114404037.
> 
> (5) 忘掉 Snowflake，感受一下性能高出 587 倍的全局唯一 ID 生成算法 - 知乎. https://zhuanlan.zhihu.com/p/154480290.
> 
> (6) http://code.flickr.com/blog/2010/02/08/ticket-servers-distributed-unique-primary-keys-on-the-cheap/.
> 
> (7) https://mp.weixin.qq.com/s/kZAnYz_Jj4aBrtsk8Q9w_A.
> 
> (8) https://www.jianshu.com/p/fac342e41fb6.
> 
> (9) http://instagram-engineering.tumblr.com/post/10853187575/sharding-ids-at-instagram.
> 
> (10) https://docs.mongodb.com/manual/reference/method/ObjectId/.
> 
> (11) https://github.com/baidu/uid-generator.