# 深入理解分布式唯一ID
> 关键词: 分布式 唯一
> 目标: 可以自主选择、分析乃至设计的分布式唯一id方案

## 背景和引入
### 雪花算法原理简述
<img width="960" alt="image" src="https://github.com/geek-pie/Billions-Of-Traffic/assets/22070982/623cbfcb-1d75-4b28-af48-8144a51e230a">

## ID生成时需要考虑的内容
### ID长度带来的影响 [关键: 考虑缓冲区的索引场景空间有限]
在MySQL中，主键的长度会影响到查询性能。这是因为InnoDB存储引擎使用B+树数据结构来存储数据库表中的数据，表的主键存储在B+树的叶子节点上，叶子节点上还存储着主键所关联的行记录数据¹。

如果主键过长，那么每个索引都需要存储这个值，这将导致以下几个问题²⁴⁵：
1. **内存使用**：在数据量大，内存珍贵的情况下，MySQL有限的缓冲区，存储的索引与数据会减少。
2. **磁盘IO**：由于缓冲区中存储的索引和数据减少，磁盘IO的概率会增加。
3. **磁盘空间**：索引占用的磁盘空间也会增加。

因此，为了提高查询性能和节省存储空间，建议使用较短的主键。
> (1) 为什么MySQL设计主键时key不能过长？ - 简书. https://www.jianshu.com/p/31bfd4ac3e61.
>
> (2) 数据库，主键为何不宜太长？ - 知乎. https://zhuanlan.zhihu.com/p/336082481.
>
> (3) mysql的主键超过最大值会发生什么？ - zhengbiyu - 博客园. https://www.cnblogs.com/zhengbiyu/p/17301057.html.

### ID数据类型带来的影响
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

以下是一些比较重要的考虑因素¹：
- **存储空间**：数值型主键相对较小，节省存储空间；而字符型主键根据具体长度而定，可能占据更多的存储空间。
- **查询效率**：数值型主键在索引和排序操作上更高效；而字符型主键在特定场景下可能效率更高，如需要根据字符串进行模糊搜索或排序。
- **可读性**：字符型主键更直观易懂，方便开发人员和用户理解；而数值型主键则需要额外的映射关系才能理解其含义。
- **数据复用性**：字符型主键可以利用已有字符数据来标识数据，避免冗余存储；而数值型主键则需要额外的关联表来实现类似功能。

> (1) MySQL主键使用数值型和字符型的区别-CSDN博客. https://blog.csdn.net/xhaimail/article/details/132732033.
>
> (2) 数据库的主键应该选择什么数据类型比较好？ - 知乎. https://www.zhihu.com/question/30888980.
>
> (3) MySQL：VARCHAR 可以作为主键吗？ - 极客教程. https://geek-docs.com/mysql/mysql-ask-answer/425_mysql_can_i_use_varchar_as_the_primary_key.html.
>
> (4) MySQL: VARCHAR作为主键使用吗？|极客笔记. https://deepinout.com/mysql/mysql-questions/425_mysql_can_i_use_varchar_as_the_primary_key.html.

### ID是否有序带来的影响
在MySQL中，主键是否递增对数据库的性能和数据一致性有重要影响：

1. **性能优化**：设计MySQL表时，我们一般会设置一个自增主键，从而让主键索引尽可能的保持递增的趋势，这样可以避免页分裂，让MySQL顺序写入，大大提高MySQL的性能。

2. **数据一致性**：在MySQL5.7之前，这个递增值是直接保存在内存里面的，当服务器重启后，MySQL会读取表里面的最大主键id，然后将最大值+1作为下次递增的值。这种设计可能会导致主键不连续，从而影响数据的一致性。

3. **并发事务处理**：为了提高事务的吞吐量，MySQL可以处理并发执行的多个事务，但是如果并发执行多个插入新记录的SQL语句，可能会导致主键的不连续。

> (1) 面试官：MySQL主键为什么不是连续递增的？-mysql设置主键自增长 - 51CTO. https://www.51cto.com/article/>743188.html.
>
> (2) 面试官：MySQL主键为什么不是连续递增的？-CSDN博客. https://blog.csdn.net/qq_33312725/article/details/>128462658.
>
> (3) MySQL数据库——MySQL AUTO_INCREMENT：主键自增长 - CSDN博客. https://blog.csdn.net/Itmastergo/article/>details/130260568.
>
> (4) 为什么 MySQL 的自增主键不单调也不连续 - 知乎. https://zhuanlan.zhihu.com/p/202960807.
>
> (5) 为什么主键索引建议顺序递增？ - 知乎. https://www.zhihu.com/question/588206465.

ID顺序递增类型和带来的影响：
- 连续递增
- 单调递增
- 趋势递增 (描述一组数据整体上的增长趋势，而不是严格的每一项都比前一项大。在这种情况下，可能存在一些局部的下降，但是整体的趋势是向上的)

结论：连续递增和单调递增在并发场景下难以保证。综合来看，想要通过主键顺序得到性能buff，做到趋势递增是一种折中选择

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
### 安全性考虑
- MAC/IP地址泄露
- 后续Id被提前预测
- 从Id数量被计算出期间Id数

## 常见的ID生成方式
### UUID
UUID(Universally Unique Identifier)的标准型式包含32个16进制数字，以连字号分为五段，形式为8-4-4-4-12的36个字符，示例：550e8400-e29b-41d4-a716-446655440000，到目前为止业界一共有5种方式生成UUID，详情见IETF发布的UUID规范 A Universally Unique IDentifier (UUID) URN Namespace。
#### 优点：
性能非常高：本地生成，没有网络消耗。
#### 缺点：
- 不易于存储：UUID太长，16字节128位，通常以36长度的字符串表示，很多场景不适用。
- 信息不安全：基于MAC地址生成UUID的算法可能会造成MAC地址泄露，这个漏洞曾被用于寻找梅丽莎病毒的制作者位置。
- ID作为主键时在特定的环境会存在一些问题，比如做DB主键的场景下，UUID就非常不适用：
  ① MySQL官方有明确的建议主键要尽量越短越好[4]，36个字符长度的UUID不符合要求。
All indexes other than the clustered index are known as secondary indexes. In InnoDB, each record in a secondary index contains the primary key columns for the row, as well as the columns specified for the secondary index. InnoDB uses this primary key value to search for the row in the clustered index.*** If the primary key is long, the secondary indexes use more space, so it is advantageous to have a short primary key***.
  ② 对MySQL索引不利：如果作为数据库主键，在InnoDB引擎下，UUID的无序性可能会引起数据位置频繁变动，严重影响性能。
### 数据库自增ID
#### 优点：
- 简单易用
#### 缺点：
- 性能低

## 分布式场景下对ID的要求
### 全局唯一
- 数据错误
- 回滚代价
- 性能影响
### 高并发
### 高可用
### 多数条件下要避免业务属性
带有业务属性的id必然意味着局限在一定的业务场景之内，通用性差，而考虑到其高可用依赖的资源成本，复杂度以及维护工作量，为不同的业务场景设计不同的带有业务属性的id在多数条件下是难以接受的。

## 普通的id演进为分布式的思考路径
### 全局唯一 高性能 高可用
保障全局唯一的思路(多实例下正确分配):
- 单实例内部避免重复
  > 做到单实例内单调自增
- 多实例间避免重复 维持隔离协定/分布式锁
  > 维持隔离协定
  > - 分段
  > - 步长、hash
  > - 单实例id
  > 分布式算法
  > - 高性能分布式锁直接维护id
  >   - redis自增
  > - 分布式算法维护单实例id(雪花算法)
  >   - zookeeper

提高性能的思路:
- 不同步落盘
- 预分配

提高可用性的思路(多实例下高效分配, 容忍单点故障):
- 依赖的协调组件能高可用部署
- 对协调组件弱依赖(容忍一定程度的不可用): 预协调，持久化
- 避免实例增减对原有隔离协定的结果造成影响
  - 一致性hash

## 分布式场景下的ID生成方式
### hash自增、步长自增
穿插式生成id
#### 优点：
#### 缺点：
### 分段自增
各自为营式生成id
#### 优点：
#### 缺点：
### redis自增
通通排队生成id
#### 优点：简单易实现，可从0开始，能做到号段不浪费
#### 缺点：性能上限低，组件强依赖
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
##### [0]首位: 二进制下的符号位，0表示正数，1表示负数
##### [1-42]时间戳位: 毫秒级时间戳，放在高位，保障单实例内部毫秒级的递增、保障id的整体趋势递增
时间戳位往往参考一个基准时间，用当前时间戳减去基准时间后的结果填入时间戳位
- 增加可用id数
- 避免直接被解析出真实时间戳
##### []实例id位
##### []序号位: 保障单实例内部毫秒级以下的递增
#### 问题和缺陷
##### 时钟回拨问题
##### Id数量和浪费

### 雪花算法的实现 之 百度UID
### 雪花算法的实现 之 美团Leaf
### 雪花算法的实现 之 Mybatis中的雪花算法

参考 
https://tech.meituan.com/2017/04/21/mt-leaf.html
https://zhuanlan.zhihu.com/p/397680307
