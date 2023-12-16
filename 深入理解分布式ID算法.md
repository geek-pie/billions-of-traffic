# 深入理解分布式唯一ID
两个关键词: 分布式 唯一

## ID生成时需要考虑的内容
### ID长度带来的影响
### ID数据类型带来的影响
### ID是否有序带来的影响
- 连续递增
- 单调递增
- 趋势递增
### 安全性考虑

## 分布式场景下对ID的要求
### 全局唯一
### 高并发
### 高可用

## 最常见的ID生成方式
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

## 分布式场景的初代ID方案
### hash自增（步长自增）
### 分段自增
### redis自增

## 雪花算法和
### 雪花算法理论
#### 雪花Id的组成
#### workerId的分配
#### 问题和缺陷
##### 时钟回拨问题
##### Id数量和浪费

### 雪花算法的实现 之 百度UID
### 雪花算法的实现 之 美团Leaf
### 雪花算法的实现 之 Mybatis中的雪花算法

参考 
https://tech.meituan.com/2017/04/21/mt-leaf.html
