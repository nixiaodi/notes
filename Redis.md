# Redis

## 基础

### 概述

- redis默认有16个数据库，可以通过select指令切换数据库
- redis是单线程的，为什么还这么快？
  - 误区
    - 高性能的服务器一定是多线程的？
    - 多线程(cpu上下文会切换)一定比单线程效率高？
  - 快的原因
    - **redis是将数据放在内存当中的，使用单线程效率就是最高的，多线程会切换cpu上下文**
    - **多路IO复用**
- redis是基于内存操作，CPU不是redis性能瓶颈，redis的瓶颈是根据服务器的内存和网络带宽

### 一般指令

#### 通过客户端连接

- 如果使用docker进入交互模式，则在窗口输入redis-cli -p 6379

  ```bash
  # redis-cli -p 6379
  127.0.0.1:6379> auth "nidi1995230"
  ```

#### 登录验证

- 进入redis交互模式时如果出现**(error) NOAUTH Authentication required**，则表明需要登录

- 在命令窗口输入auth “密码”即可登录

  ```bash
  127.0.0.1:6379> auth "nidi1995230"
  OK
  ```

#### 切换数据库

- select num(第几个数据库)

  ```bash
  127.0.0.1:6379> select 2
  OK
  ```

#### 查看数据库大小

```bash
127.0.0.1:6379[2]> dbsize
(integer) 0
```

#### 查看当前数据库所有的key

```bash
127.0.0.1:6379> keys *
1) "name"
```

#### 清空全部库

```bash
127.0.0.1:6379> FLUSHALL
OK
```

#### 清空当前库

```bash
127.0.0.1:6379> FLUSHDB
OK
```

## 基本数据类型

### redis-key

#### 指令

##### 判断key是否存在

```bash
127.0.0.1:6379> EXISTS name
(integer) 1
```

##### 移动key到另一个库

- move key num(第几个库)

```bash
127.0.0.1:6379> move name 1
(integer) 1
```

##### 设置key过期时间

- expire key seconds(多少秒)

```bash
127.0.0.1:6379[1]> EXPIRE name 10
(integer) 1
```

##### 查看key的剩余存活时间

```bash
127.0.0.1:6379[1]> ttl name
(integer) 6
```

##### 查看key的类型

```bash
127.0.0.1:6379[1]> type name
string
```

### 字符串

#### 概述

- 虽然存储的是字符串，但是如果是数字格式的字符串，照样可以按照数字的逻辑进行处理，比如递增、递减等

#### 指令

##### 新建k-v键值对

```bash
127.0.0.1:6379[1]> set age 18
OK
```

##### 获取key对应的value

```bash
127.0.0.1:6379[1]> get age
"18"
```

##### 追加字符串

- 返回的是追加之后字符串的长度，追加是直接将原来的内容和追加内容拼接
- 如果key不存在的话则新建这个k-v键值对

```bash
127.0.0.1:6379[1]> APPEND name nidi
(integer) 9
127.0.0.1:6379[1]> get name
"jiangnidi"
```

##### 获取字符串长度

```bash
127.0.0.1:6379[1]> STRLEN name
(integer) 9
```

##### 递增value的值(步长为1)

- 可以用来监控浏览量、播放量等

```bash
127.0.0.1:6379[1]> set views 0
OK
127.0.0.1:6379[1]> INCR views
(integer) 1
```

##### 递减value的值(步长为1)

```bash
127.0.0.1:6379[1]> DECR views
(integer) 0
127.0.0.1:6379[1]> get views
"0"
```

##### 指定步长递增value的值

- incrby key 步长

```bash
127.0.0.1:6379[1]> INCRBY views 5
(integer) 5
127.0.0.1:6379[1]> get views
"5"
```

##### 获取部分字符串和全部字符串

- 当截取范围为0 -1时表示全部字符串

```bash
127.0.0.1:6379[1]> GETRANGE name 0 3
"jian"
127.0.0.1:6379[1]> GETRANGE name 0 -1
"jiangnidi"
```

##### 替换字符串的一部分

- setrange key 替换的起始位置  替换的字符串(也就是从起始位置开始算，替换的长度就是替换字符串的长度)

```bash
127.0.0.1:6379[1]> SETRANGE name 2 xx
(integer) 9
127.0.0.1:6379[1]> get name
"jixxgnidi"
```

##### 新建k-v并设置过期时间

- setex(set with expire)
- setex key seconds value

```
127.0.0.1:6379[1]> SETEX dog 10 jiang
OK
127.0.0.1:6379[1]> ttl dog
(integer) 4
```

##### 如果不存在新建k-v

- 如果不存在才会创建，存在的话就会返回0(也就是false)
- **在分布式锁中常用**

```bash
127.0.0.1:6379[1]> setnx cat jiang
(integer) 1
127.0.0.1:6379[1]> get cat
"jiang"
127.0.0.1:6379[1]> setnx cat nidi 
(integer) 0
127.0.0.1:6379[1]> get cat
"jiang"
```

##### 同时新建多个k-v

```bash
127.0.0.1:6379[1]> mset k1 v1 k2 v2 k3 v3 
OK
```

##### 通过获取多个key的value

```bash
127.0.0.1:6379[1]> MGET k1 k2 k3
1) "v1"
2) "v2"
3) "v3"
```

##### 同时创建多个k-v且判断是否其中的k存在

- 由于k1已经存在，故返回false；同时k4并没有被设置
- **原子性操作，同一个事务**

```
127.0.0.1:6379[1]> MSETNX k1 v4 k4 v5
(integer) 0
127.0.0.1:6379[1]> MGET k1 k2 k3 k4
1) "v1"
2) "v2"
3) "v3"
4) (nil)
```

##### 通过字符串存储对象

在redis中:具有分层级的作用，通过mset设置user就类似于

```json
user：{
    1：{
    name:jiang,
    age:18
}
}
```

```bash
127.0.0.1:6379[1]> mset user:1:name jiang user:1:age 18
OK
127.0.0.1:6379[1]> mget user:1:name  user:1:age 
1) "jiang"
2) "18"
```

##### 获取并设置

- 先获取key的值再重新设置，得到的结果就是原有的value
- 第一次获取为null因为之前不存在

```bash
127.0.0.1:6379> getset db mysql
(nil)
127.0.0.1:6379> get db
"mysql"
127.0.0.1:6379> GETSET db redis
"mysql"
127.0.0.1:6379> get db
"redis"
```

### List

#### 概述

- 在redis中可以将list操作为栈、队列、阻塞队列

#### 指令

##### 向列表中存储值(从左侧)

- lrange 列表名 value
- 将一个或多个值插入到列表的头部

```bash
127.0.0.1:6379> LPUSH list one
(integer) 1
127.0.0.1:6379> LPUSH list two
(integer) 2
127.0.0.1:6379> LPUSH list three
(integer) 3
```

##### 从列表中取值(从左侧)

- lrange 列表名 范围；范围为0 -1时表示取出所有值
- 可以看出当指定范围为0 -1时获取到的是three two表示存储的数据遵循的原则是先进后出(类似于栈)
- 从列表的头部开始取值

```bash
127.0.0.1:6379> LRANGE list 0 -1
1) "three"
2) "two"
3) "one"
127.0.0.1:6379> LRANGE list 0 1
1) "three"
2) "two"
```

向列表中存储值(从右侧)

- 将一个或多个值插入到列表的尾部

```bash
127.0.0.1:6379> RPUSH list four
(integer) 4
127.0.0.1:6379> LRANGE list 0 -1
1) "three"
2) "two"
3) "one"
4) "four"
```

##### 移除元素(从左侧)

- 从左侧移除也就是从头部移除

```bash
127.0.0.1:6379> LPOP list
"three"
127.0.0.1:6379> LRANGE list 0 -1
1) "two"
2) "one"
3) "four"
```

##### 移除元素(从右侧)

- 从左侧移除也就是从尾部移除

```bash
127.0.0.1:6379> RPOP list
"four"
127.0.0.1:6379> LRANGE list 0 -1
1) "two"
2) "one"
```

##### 通过索引查找值(从左侧)

- 可以用来做阻塞队列，判断队列中是否有值

```bash
127.0.0.1:6379> LRANGE list 0 -1
1) "two"
2) "one"
127.0.0.1:6379> lindex list 1
"one"
```

##### 获取list长度

```bash
127.0.0.1:6379> LLEN list
(integer) 2

```

##### 移除指定数量的指定值

- lrem key 数量 指定值
- 原先有两个one，当数量为1时就从首部开始移除一个；如果为2就从首部开始移除两个

```bash
127.0.0.1:6379> LRANGE list 0 -1
1) "one"
2) "two"
3) "one"
127.0.0.1:6379> LREM list 1 one
(integer) 1
127.0.0.1:6379> LRANGE list 0 -1
1) "two"
2) "one"
```

##### 截取list

- 和lrange不同之处是：ltrim将原列表改变了，而lrange只是获取到了原列表的部分值

```bash
127.0.0.1:6379> LRANGE list 0 -1
1) "1"
2) "0"
3) "one"
4) "two"
5) "one"
127.0.0.1:6379> LTRIM list 0 1
OK
127.0.0.1:6379> LRANGE list 0 -1
1) "1"
2) "0"
```

##### 移除元素并移动到新列表(组合命令)

```bash
127.0.0.1:6379> LRANGE list 0 -1
1) "1"
2) "0"
127.0.0.1:6379> RPOPLPUSH list newlist 
"0"
127.0.0.1:6379> LRANGE list 0 -1
1) "1"
127.0.0.1:6379> LRANGE newlist 0 -1
1) "0"
```

##### 更新list中指定位置的值

- lset key 指定的index 需要更新的值
- 如果当前下标不存在则报错

```bash
127.0.0.1:6379> LRANGE list 0 -1
1) "1"
127.0.0.1:6379> LSET list 0 test
OK
127.0.0.1:6379> LRANGE list 0 -1
1) "test"
```

##### 在某个元素之前/之后插入值

- 如果判断为before时，就在元素之前插入(偏首部)；如果判断为after时，就在元素之后插入
- linsert key before/after 被插入的元素值 插入的值

```bash
127.0.0.1:6379> LRANGE list 0 -1
1) "test"
127.0.0.1:6379> LINSERT list before test yes
(integer) 2
127.0.0.1:6379> LRANGE list 0 -1
1) "yes"
2) "test"
```

### Set

#### 指令

##### 向set中插入值/获取set中的所有元素

- 从所有元素中可以看出set无序
- 无法添加重复元素

```bash
127.0.0.1:6379> sadd set hello
(integer) 1
127.0.0.1:6379> sadd set world
(integer) 1
127.0.0.1:6379> sadd set yes
(integer) 1
127.0.0.1:6379> SMEMBERS set
1) "hello"
2) "yes"
3) "world"
```

##### 判断set中是否存在某个元素

```bash
127.0.0.1:6379> SISMEMBER set hello
(integer) 1
```

##### 判断set元素个数

```bash
127.0.0.1:6379> SCARD set
(integer) 3
```

##### 移除set中的指定元素

```bash
127.0.0.1:6379> SREM set hello
(integer) 1
127.0.0.1:6379> SMEMBERS set
1) "yes"
2) "world"
```

##### 随机抽选出set中指定个数元素

- srandmemeber key 抽取元素个数
- 可以用来做抽奖程序调用

```bash
127.0.0.1:6379> SRANDMEMBER set 2
1) "yes"
2) "world"
```

##### 随机删除指定数量的元素

- spop key 删除元素个数

```bash
127.0.0.1:6379> SPOP set 1
1) "world"
127.0.0.1:6379> SMEMBERS set
1) "yes"
```

##### 移动指定元素从一个set到另一个

```bash
127.0.0.1:6379> SMOVE set newset yes
(integer) 1
127.0.0.1:6379> SMEMBERS set
(empty list or set)
127.0.0.1:6379> SMEMBERS newset
1) "yes"
```

##### 两个集合之间的差集

```bash
127.0.0.1:6379> sadd set1 1 2 3
(integer) 3
127.0.0.1:6379> SADD set2 2 3 4
(integer) 3
127.0.0.1:6379> SDIFF set1 set2
1) "1"
```

##### 两个集合之间的并集

- 可以用来做视频网站、微博的共同关注，做好友推荐

```bash
127.0.0.1:6379> SINTER set1 set2
1) "2"
2) "3"
```

##### 两个集合之间的并集

```bash
127.0.0.1:6379> SUNION set1 set2 
1) "1"
2) "2"
3) "3"
4) "4"
```

### Hash

#### 概述

- 适合进行对象的存储

#### 指令

##### 向hash中存储值并取出对应的值

```bash
127.0.0.1:6379> hset hash name jiang
(integer) 1
127.0.0.1:6379> HGET hash name
"jiang"
```

##### 向hash中存储多个值并取出多个值

- 从name属性中可以看出可以直接从hash中修改属性对应的值

```bash
127.0.0.1:6379> hmset hash name nidi age 18
OK
127.0.0.1:6379> HMGET hash name age
1) "nidi"
2) "18"
```

##### 从hash中取出所有的键和值

```bash
127.0.0.1:6379> HGETALL hash
1) "name"
2) "nidi"
3) "age"
4) "18"
```

##### 删除hash中的键

- 删除键之后对应的值也被删除

```bash
127.0.0.1:6379> HDEL hash name
(integer) 1
127.0.0.1:6379> HGETALL hash
1) "age"
2) "18"
```

##### 获取hash键值对的个数

```bash
127.0.0.1:6379> HLEN hash
(integer) 1
```

##### 判断hash中是否存在某个键

```bash
127.0.0.1:6379> HEXISTS hash age
(integer) 1
```

##### 获取所有的键/获取所有的值

```bash
127.0.0.1:6379> HKEYS hash
1) "age"
127.0.0.1:6379> HVALS hash
1) "18"
```

##### 让hash中指定的key对应的value自增或自减(指定步长)

```bash
127.0.0.1:6379> HINCRBY hash age 2
(integer) 20
127.0.0.1:6379> HINCRBY hash age -2
(integer) 18
```

##### 在判断key是否存在的情况下添加键值对

- 当键存在时则无法设置，返回false

```bash
127.0.0.1:6379> HSETNX hash age 32
(integer) 0
127.0.0.1:6379> HSETNX hash dog hello
(integer) 1
```

### Zset

#### 概述

- 对数据添加权重，比如排行榜应用、工资表排序
- 对消息队列添加权重，不同的权重消息优先级不同

#### 指令

##### 向zset中添加值并获取所有值

- zrange代表获取值同时按照升序排序
- zrevrange代表获取值同时按照降序排序

```bash
127.0.0.1:6379> ZADD zset 1 one 2 two 3 three
(integer) 3
127.0.0.1:6379> ZRANGE zset 0 -1
1) "one"
2) "two"
3) "three"
127.0.0.1:6379> ZREVRANGE zset 0 -1
1) "three"
2) "two"
```

##### 按照序号进行升序

- zrangebyscore key 序号的范围
- 关键字中的score代表的就是元素的序号
- -inf和+inf分别代表负无穷和正无穷
- 当后面添加withscore参数时就会把元素对应的序号带上
- 如果元素的序号在定义的序号范围之外则不会被取出

```bash
127.0.0.1:6379> ZRANGEBYSCORE zset -inf +inf
1) "one"
2) "two"
3) "three"
127.0.0.1:6379> ZRANGEBYSCORE zset -inf +inf withscores
1) "one"
2) "1"
3) "two"
4) "2"
5) "three"
6) "3"
127.0.0.1:6379> ZRANGEBYSCORE zset -inf 2 withscores
1) "one"
2) "1"
3) "two"
4) "2"
```

##### 按照序号进行降序排列

```bash
127.0.0.1:6379> ZREVRANGEBYSCORE zset +inf -inf withscores
1) "three"
2) "3"
3) "two"
4) "2"
5) "one"
6) "1"
```

##### 移除zset中的元素

```bash
127.0.0.1:6379> ZREM zset one
(integer) 1
```

##### 查看zset元素个数

```bash
127.0.0.1:6379> ZCARD zset
(integer) 2
```

##### 获取序号指定区间元素数量

```bash
127.0.0.1:6379> ZCOUNT zset 1 2
(integer) 1
```

### geospatial 地理位置

#### 概述

- 可以用于推算两地之间的距离，方圆几里之内的人(微信摇一摇)

- geo底层就是通过zset实现，故geo可以使用zset指令操作成员

  ```bash
  127.0.0.1:6379> ZREM china:city xian
  (integer) 1
  127.0.0.1:6379> ZRANGE china:city 0 -1
  1) "chongqing"
  2) "shenzhen"
  3) "hangzhou"
  4) "shanghai"
  5) "beijing"
  ```

#### 指令

##### 添加地理位置经纬度坐标

- china:city代表的是key，而之后的两个坐标分别是经度(longitude)和纬度(latitude),beijing代表成员(member)
- 地球两极无法添加
- 一般通过java程序读取配置文件导入(经度、纬度、名称)

```
127.0.0.1:6379> GEOADD china:city 116.371661 39.994241 beijing
(integer) 1
127.0.0.1:6379> GEOADD china:city 121.480066 31.236392 shanghai
(integer) 1
127.0.0.1:6379> GEOADD china:city 107.093737 29.858 chongqing
(integer) 1
127.0.0.1:6379> GEOADD china:city 114.064552 22.548457 shenzhen
(integer) 1
```

##### 指定成员获取对应的地理位置

```bash
127.0.0.1:6379> GEOPOS china:city xian
1) 1) "125.15537291765213013"
   2) "42.93330850721532244"
```

##### 查询两个成员之间的距离

- km代表指定距离的单位(unit)，可以是m、km

```bash
127.0.0.1:6379> GEODIST china:city xian hangzhou km
"1477.0258"
```

##### 查找在key中符合某个具体位置具体范围内元素

- key是china:city，也就是从china:key中的元素进行判断，半径为500km
- 可以通过这种方式应用搜索附近的人
- 后面添加withdist、withcoord还可以打印出对应的经纬度，直线距离等
- 后面添加count num可以指定取出多少个元素，从符合条件距离最短开始计算

```bash
127.0.0.1:6379> GEORADIUS china:city 120 30 500 km
1) "hangzhou"
2) "shanghai"
127.0.0.1:6379> GEORADIUS china:city 120 30 500 km withcoord withdist
1) 1) "hangzhou"
   2) "34.9603"
   3) 1) "120.2155110239982605"
      2) "30.2530820188523748"
2) 1) "shanghai"
   2) "197.4341"
   3) 1) "121.48006528615951538"
      2) "31.23639160652168556"
127.0.0.1:6379> GEORADIUS china:city 120 30 500 km withcoord withdist count 1
1) 1) "hangzhou"
   2) "34.9603"
   3) 1) "120.2155110239982605"
      2) "30.2530820188523748"
```

##### 以key中的元素为中心，查找出指定距离内key中符合的元素

```bash
127.0.0.1:6379> GEORADIUSBYMEMBER china:city xian 1000 km withcoord
1) 1) "xian"
   2) 1) "125.15537291765213013"
      2) "42.93330850721532244"
2) 1) "beijing"
   2) 1) "116.37165874242782593"
      2) "39.9942410243751425"
```

### Hyperloglog

#### 概述

- 用于统计许多元素中不重复元素的数量，可以用于统计网站的用户数量(同一个用户多次浏览重复不计)
- 无法获取结构中具体的元素值
- set也可以统计不重复元素，然而set占用内存空间较大；hyperloglog只统计无法取值

#### 指令

##### 向结构中添加元素

- 只能添加但无法取出

```bash
127.0.0.1:6379> PFADD key 1 2 3 4 5 6
(integer) 1
127.0.0.1:6379> PFADD key2 3 4 5 6 7
(integer) 1
```

##### 统计结构中的元素个数

```bash
127.0.0.1:6379> PFCOUNT key
(integer) 6
```

##### 合并两个结构中的数据并统计

- 合并时第一个结构代表的是目标结构，剩下的结构可以有很多表示源结构

```bash
127.0.0.1:6379> PFMERGE key3 key key1
OK
127.0.0.1:6379> PFCOUNT key3
(integer) 6
```

### Bitmaps 位存储

#### 概述

- 用于统计只有两种状态的数据，比如打卡、用户是否登录(用0和1表示)
- 操作二进制位来进行记录，只有0和1两种状态，内存消耗较小

#### 指令

##### 存储某个偏移量的位数值

- bit代表的是key，而0 1 2 3代表的是偏移量(offset)，可以形象比作周一、周二、周三等，后面的0和1代表的是位数值，可以比作打卡或未打卡

```bash
127.0.0.1:6379> PFCOUNT key3
(integer) 6
127.0.0.1:6379> setbit bit 0 1
(integer) 0
127.0.0.1:6379> SETBIT bit 1 0
(integer) 0
127.0.0.1:6379> setbit bit 2 0
(integer) 0
127.0.0.1:6379> SETBIT bit 3 1
(integer) 0
```

##### 对key进行位运算(统计)

- 类似于统计一周内的打卡数量

```bash
127.0.0.1:6379> BITCOUNT bit
(integer) 2
```

## 高级

### 事务

#### 概述

- redis单条命令保证原子性，但事务不保证原子性
- (一致性)redis事务本质就是一组命令的集合，要么同时成功，要么同时失败
- (顺序性)redis事务中的所有命令都会被序列化，在事务执行过程中，会按照顺序执行
- (排他性)redis事务没有隔离级别概念，不存在缓读、脏读，所有的命令在事务中，并没有直接执行，只有发起执行命令时，才会执行

#### 指令

##### 开启事务

```bash
127.0.0.1:6379> MULTI
OK
```

##### 命令入队

- 从返回值queue可以看出命令入队

```bash
127.0.0.1:6379> set k1 v1
QUEUED
127.0.0.1:6379> set ke v2
QUEUED
```

##### 提交事务

```bash
127.0.0.1:6379> exec
1) OK
2) OK
```

##### 放弃事务

```bash
127.0.0.1:6379> DISCARD
OK
```

##### 出现异常

- 如果事务中存在编译异常(命令有错)，事务中的所有命令都不会被执行
- 如果事务中存在运行时异常(1/0)，执行命令时其他命令是可以正常执行，错误命令抛出异常

### redis实现乐观锁

#### 概述

- 悲观锁:认为什么时候都会出问题，无论做什么都会加锁
- 乐观锁:认为什么时候都不会出问题，所以不会上锁；在更新数据期间判断在此期间是否有人修改过这个数据
- mybaits-plus实现乐观锁的方式是添加一个version字段，当更新数据时version也会进行更新

#### redis监视测试

- 前提条件

  - 同时开启两个终端，用来模拟并发操作
  - 设置原有存储money为100，原有消费out为0

- 其中一个终端监视(watch)money，开启事务，将余额减少10，消费增加10

  ```bash
  127.0.0.1:6379> set money 100
  OK
  127.0.0.1:6379> set out 0
  OK
  127.0.0.1:6379> watch money
  OK
  127.0.0.1:6379> MULTI
  OK
  127.0.0.1:6379> DECRBY money 10
  QUEUED
  127.0.0.1:6379> INCRBY out 10
  QUEUED
  ```

- 另一个终端先获取余额再修改余额

  ```bash
  127.0.0.1:6379> get money
  "100"
  127.0.0.1:6379> set money 1000
  OK
  ```

- 当第一个线程提交事务时会返回null

  ```bash
  127.0.0.1:6379> exec
  (nil)
  ```

- 测试结果

  - redis使用watch当做乐观锁进行操作
  - 由于第一个终端监视了money，当事务准备提交时发现money已经被修改，故执行失败(类似于version已经被修改)

- 当发现执行失败之后，放弃原有的监视，再重新监视，进而执行事务操作

  ```bash
  127.0.0.1:6379> UNWATCH
  OK
  127.0.0.1:6379> WATCH money
  OK
  ```

### Springboot整合redis

#### 相关资料

- [序列化配置规范](https://my.oschina.net/u/4394698/blog/4339964)
- [相关组件](https://www.codenong.com/cs106922950/)
- [Redis使用说明](https://cloud.tencent.com/developer/article/1497594)
- [ObjectMapper序列化问题](https://www.jianshu.com/p/c5fcd2a1ab49)
- [Jackson 注解详解](https://blog.csdn.net/sdyy321/article/details/40298081)
- [Jackson ObjectMapper配置](https://blog.csdn.net/Seky_fei/article/details/109960178)

#### JSR107缓存规范

##### 规范(核心接口)

###### CachingProvider

定义了创建、配置、获取、管理和控制多个CacheManager，单个应用在运行期间可以访问多个CachingProvider

###### CacheManager

定义了创建、配置、获取、管理和控制多个唯一命名的Cache，所有Cache存在于CacheManager的上下文中，一个CacheManager仅被一个CachingProvider所拥有的

###### Cache

类似于Map的数据结构并临时存储以key为索引的值，一个Cache仅被一个CacheManager所拥有的

###### Entry

存储在Cache中的key-value对

###### Expiry

每一个存储在Cache中的条目都有一个定义的有效期；一旦超过这个时间，条目为过期的状态；一旦过期，条目将不可访问、更新和删除；缓存有效期可以通过ExpiryPolicy设置

![20200406124803857](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/20200406124803857.png)

> 1、不同的CacheManager可以通过不同的厂商进行提供，例如Redis、MemCache等
>
> 2、不同的Cache可以对不同的缓存数据进行存储，例如对员工的缓存、对产品的缓存
>
> 3、CacheManager和Cache之间类似于连接池和连接之间的关系

#### Spring缓存抽象

Spring定义了`org.springframework.cache.Cache`和`org.springframework.cache.CacheManager`接口来统一不同的缓存技术

- Cache接口为缓存的组件规范定义，包含缓存的各种操作集合
- Cache接口下提供了各种xxxCache的实现，如RedisCache、EhCacheCache等

##### 核心接口

###### Cache

缓存接口，定义缓存操作；对应的实现包括RedisCache、EhCacheCache等

###### CacheManager

缓存管理器，管理各种缓存组件

###### @EnableCaching

开启基于注解的缓存

#### 缓存原理

> Springboot2.xx已经不再使用jedis，采用lettuce
>
> jedis：采用直连，多个线程操作不安全，如果需要安全使用就需要使用jedis pool
>
> lettuce：采用netty，实例可以在多个线程之间共享，不存在线程不安全(类似于NIO模式)

##### CacheAutoConfiguration

Springboot所有的配置类都有一个自动配置类，自动配置类都会绑定一个properties配置文件

在`spring.factory`中存在对应的CacheAutoConfiguration自动配置类

![image-20210120123636394](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20210120123636394.png)

首先看下CacheAutoConfiguration配置类做了哪些操作？

![image-20210120123831723](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20210120123831723.png)

> 1、首先通过@EnableConfigurationProperties绑定了CacheProperties配置类
>
> 2、通过@Import注解导入了CacheConfigurationImportSelector.class；而如果导入的是`ImportSelector`的实现类对象，就会通过`selectImports`方法返回的字符串数组向容器中注入对象

然后再分析下CacheConfigurationImportSelector内部类

```java
static class CacheConfigurationImportSelector implements ImportSelector {
    CacheConfigurationImportSelector() {
    }

    public String[] selectImports(AnnotationMetadata importingClassMetadata) {
        CacheType[] types = CacheType.values();
        String[] imports = new String[types.length];

        for(int i = 0; i < types.length; ++i) {
            imports[i] = CacheConfigurations.getConfigurationClass(types[i]);
        }

        return imports;
    }
}
```

以debug模式启动项目

1、`CacheType.values()`

![image-20210120141454048](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20210120141454048.png)

> 首先可以看出type数组中本质上就是对各种类型缓存的前缀进行记录
>
> 而CacheType实际上就是一个枚举类，枚举了所有的缓存类型

2、对type数组进行遍历，调用`CacheConfigurations.getConfigurationClass()`方法

![image-20210120141902682](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20210120141902682.png)

> 可以看出实际上就是向容器中导入了各种各样缓存的xxxCacheConfiguration类

那么就来针对性的分析`org.springframework.boot.autoconfigure.cache.RedisCacheConfiguration`

```java
@ConditionalOnClass({RedisConnectionFactory.class})
@AutoConfigureAfter({RedisAutoConfiguration.class})
@ConditionalOnBean({RedisConnectionFactory.class})
@ConditionalOnMissingBean({CacheManager.class})
@Conditional({CacheCondition.class})
class RedisCacheConfiguration {
    RedisCacheConfiguration() {
    }

    @Bean
    RedisCacheManager cacheManager(CacheProperties cacheProperties, CacheManagerCustomizers cacheManagerCustomizers, ObjectProvider<org.springframework.data.redis.cache.RedisCacheConfiguration> redisCacheConfiguration, ObjectProvider<RedisCacheManagerBuilderCustomizer> redisCacheManagerBuilderCustomizers, RedisConnectionFactory redisConnectionFactory, ResourceLoader resourceLoader) {
        RedisCacheManagerBuilder builder = RedisCacheManager.builder(redisConnectionFactory).cacheDefaults(this.determineConfiguration(cacheProperties, redisCacheConfiguration, resourceLoader.getClassLoader()));
        List<String> cacheNames = cacheProperties.getCacheNames();
        if (!cacheNames.isEmpty()) {
            builder.initialCacheNames(new LinkedHashSet(cacheNames));
        }

        redisCacheManagerBuilderCustomizers.orderedStream().forEach((customizer) -> {
            customizer.customize(builder);
        });
        return (RedisCacheManager)cacheManagerCustomizers.customize(builder.build());
    }
}
```

> 首先该配置类先进行了许多判断，比如是否存在RedisConnectionFactory组件，是否没有CacheManager组件；如果条件都成立，向容器中注入一个RedisCacheManager组件
>
> `从而说明只要自定义RedisCacheManager组件，该配置类就不会向容器中注入`

而容器中自动注入的RedisCacheManager组件使用的jdk序列化器，因此需要自定义RedisCacheManager

##### RedisCacheManager

```java
public class RedisCacheManager extends AbstractTransactionSupportingCacheManager {
	private final RedisCacheWriter cacheWriter;
    private final RedisCacheConfiguration defaultCacheConfig;
    private final Map<String, RedisCacheConfiguration> initialCacheConfiguration;
    private final boolean allowInFlightCacheCreation;
     public RedisCacheManager(RedisCacheWriter cacheWriter, RedisCacheConfiguration defaultCacheConfiguration) {
        this(cacheWriter, defaultCacheConfiguration, true);
    }
    private RedisCacheManager(RedisCacheWriter cacheWriter, RedisCacheConfiguration defaultCacheConfiguration, boolean allowInFlightCacheCreation) {
        Assert.notNull(cacheWriter, "CacheWriter must not be null!");
        Assert.notNull(defaultCacheConfiguration, "DefaultCacheConfiguration must not be null!");
        this.cacheWriter = cacheWriter;
        this.defaultCacheConfig = defaultCacheConfiguration;
        this.initialCacheConfiguration = new LinkedHashMap();
        this.allowInFlightCacheCreation = allowInFlightCacheCreation;
    }
```

> 可以看出构造RedisCacheManager需要在构造器中传入RedisCacheWriter和RedisCacheConfiguration

##### RedisCacheWriter

```java
public interface RedisCacheWriter {
	static RedisCacheWriter nonLockingRedisCacheWriter(RedisConnectionFactory connectionFactory) {
        Assert.notNull(connectionFactory, "ConnectionFactory must not be null!");
        return new DefaultRedisCacheWriter(connectionFactory);
    }
    static RedisCacheWriter lockingRedisCacheWriter(RedisConnectionFactory connectionFactory) {
        Assert.notNull(connectionFactory, "ConnectionFactory must not be null!");
        return new DefaultRedisCacheWriter(connectionFactory, Duration.ofMillis(50L));
    }
}
```

> 对于RedisCacheWriter可以看出构造器需要传入RedisConnectionFactory对象
>
> RedisCacheWriter实际就是对Redis进行读写，键值都是byte数组

##### RedisCacheConfiguration

```
public class RedisCacheConfiguration {
	private final Duration ttl;
    private final boolean cacheNullValues;
    private final CacheKeyPrefix keyPrefix;
    private final boolean usePrefix;
    private final SerializationPair<String> keySerializationPair;
    private final SerializationPair<Object> valueSerializationPair;
    private final ConversionService conversionService;
    public static RedisCacheConfiguration defaultCacheConfig() {
        return defaultCacheConfig((ClassLoader)null);
    }
    public RedisCacheConfiguration entryTtl(Duration ttl) {
        Assert.notNull(ttl, "TTL duration must not be null!");
        return new RedisCacheConfiguration(ttl, this.cacheNullValues, this.usePrefix, this.keyPrefix, this.keySerializationPair, this.valueSerializationPair, this.conversionService);
    }
     public RedisCacheConfiguration serializeKeysWith(SerializationPair<String> keySerializationPair) {
        Assert.notNull(keySerializationPair, "KeySerializationPair must not be null!");
        return new RedisCacheConfiguration(this.ttl, this.cacheNullValues, this.usePrefix, this.keyPrefix, keySerializationPair, this.valueSerializationPair, this.conversionService);
    }
```

> 这个配置类非常重要，可以配置ttl、CacheKeyPrefix、ConversionService等，可以通过链式操作进行构造
>
> 本质上就是自定义缓存策略

##### RedisAutoConfiguration

当引入`spring-boot-starter-data-redis`场景启动器后，那么`RedisAutoConfiguration`就开始起作用

```java
@ConditionalOnClass({RedisOperations.class})
@EnableConfigurationProperties({RedisProperties.class})
@Import({LettuceConnectionConfiguration.class, JedisConnectionConfiguration.class})
public class RedisAutoConfiguration {
    public RedisAutoConfiguration() {
    }

    @Bean
    @ConditionalOnMissingBean(
        name = {"redisTemplate"}
    )
    public RedisTemplate<Object, Object> redisTemplate(RedisConnectionFactory redisConnectionFactory) throws UnknownHostException {
        RedisTemplate<Object, Object> template = new RedisTemplate();
        template.setConnectionFactory(redisConnectionFactory);
        return template;
    }

    @Bean
    @ConditionalOnMissingBean
    public StringRedisTemplate stringRedisTemplate(RedisConnectionFactory redisConnectionFactory) throws UnknownHostException {
        StringRedisTemplate template = new StringRedisTemplate();
        template.setConnectionFactory(redisConnectionFactory);
        return template;
    }
}
```

> 首先判断是否存在RedisOperations类，然后导入了LettuceConnectionConfiguration配置类
>
> 最后向容器中导入了RedisTemplate和StringRedisTemplate组件，而且`都是在如果没有的前提下，也就是如果自定义就不会注入该组件`

RedisTemplate组件是对Redis操作进行封装，然而通常在Redis中缓存的都是对象，而RedisTemplate对对象进行序列化时默认使用的都是jdk序列化器，造成乱码

![image-20210120152206417](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20210120152206417.png)

如何知道默认序列化器使用的是JDK序列化器？

> 在RedisAutoConfiguration注入RedisTemplate组件时，首先通过`RedisTemplate<Object, Object> template = new RedisTemplate()`创建对象
>
> 而RedisTemplate类实现了`InitializingBean`接口，从而存在afterPropertiesSet方法

![image-20210120152838791](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20210120152838791.png)

> 这里可以看出在RedisTemplate组件创建时就会调用初始化方法，判断默认序列化器是否存在，如果没有就使用jdk序列化器

那么就需要自定义RedisTemplate组件且使用JSON序列化器

> 参考下容器注入的RedisTemplate，首先创建了一个RedisTemplate对象，然后调用set方法设置了参数中的RedisConnectionFactory
>
> 只是在此基础上不适用默认的序列化，使用自定义的JSON序列化器

#### 组件

##### RedisSerializer(Redis序列化器)

实现类

![image-20210120211257489](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20210120211257489.png)

> Jackson2JsonRedisSerializer序列化器是对JSON数据进行序列化，最为常用

[Jackson2JsonRedisSerializer官方API](https://docs.spring.io/spring-data/redis/docs/current/api/org/springframework/data/redis/serializer/Jackson2JsonRedisSerializer.html)

```bash
RedisSerializer that can read and write JSON using Jackson's and Jackson Databind ObjectMapper.
This converter can be used to bind to typed beans, or untyped HashMap instances. Note:Null objects are serialized as empty arrays and vice versa
```

```java
public class Jackson2JsonRedisSerializer<T> implements RedisSerializer<T> {
	public static final Charset DEFAULT_CHARSET = StandardCharsets.UTF_8;
	private final JavaType javaType;
	private ObjectMapper objectMapper = new ObjectMapper();
	public Jackson2JsonRedisSerializer(Class<T> type) {
		this.javaType = getJavaType(type);
	}
	@Override
	public byte[] serialize(@Nullable Object t) throws SerializationException {

		if (t == null) {
			return SerializationUtils.EMPTY_ARRAY;
		}
		try {
			return this.objectMapper.writeValueAsBytes(t);
		} catch (Exception ex) {
			throw new SerializationException("Could not write JSON: " + ex.getMessage(), ex);
		}
	}
	public void setObjectMapper(ObjectMapper objectMapper) {

		Assert.notNull(objectMapper, "'objectMapper' must not be null");
		this.objectMapper = objectMapper;
	}
```

> 可以看出serialize方法是对对象进行序列化，而setObjectMapper方法将参数中的ObjectMapper赋值给属性

##### ObjectMapper

官方解释：

```bash
ObjectMapper is a framework written in Swift that makes it easy for you to convert your model objects (classes and structs) to and from JSON
```

###### 对象序列化方法

![image-20210120213818259](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20210120213818259.png)

> 该系列方法将java对象序列化为json，并将json存储成不同的格式，常用writeValueAsString

###### 序列化信息配置

```java
 //在序列化时日期格式默认为 yyyy-MM-dd'T'HH:mm:ss.SSSZ
 mapper.configure(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS,false)
 //在序列化时忽略值为 null 的属性
 mapper.setSerializationInclusion(Include.NON_NULL);
 //忽略值为默认值的属性
 mapper.setDefaultPropertyInclusion(Include.NON_DEFAULT);
```

> 1、setSerializationInclusion可以传入`JsonInclude.Include`内部枚举类的对象来指定序列化时忽略哪些属性

##### RedisCache

```java
public class RedisCache extends AbstractValueAdaptingCache {

	private static final byte[] BINARY_NULL_VALUE = RedisSerializer.java().serialize(NullValue.INSTANCE);

	private final String name; // 缓存的名字
	private final RedisCacheWriter cacheWriter; //最终操作是委托给它的
	private final RedisCacheConfiguration cacheConfig; // cache的配置（因为每个Cache的配置可能不一样，即使来自于同一个CacheManager）
	private final ConversionService conversionService; // 数据转换
	
	// 唯一构造函数，并且还是protected 的。可见我们自己并不能操控它
	protected RedisCache(String name, RedisCacheWriter cacheWriter, RedisCacheConfiguration cacheConfig) {

		super(cacheConfig.getAllowCacheNullValues());

		Assert.notNull(name, "Name must not be null!");
		Assert.notNull(cacheWriter, "CacheWriter must not be null!");
		Assert.notNull(cacheConfig, "CacheConfig must not be null!");

		this.name = name;
		this.cacheWriter = cacheWriter;
		this.cacheConfig = cacheConfig;
		this.conversionService = cacheConfig.getConversionService();
	}

	@Override
	public String getName() {
		return name;
	}
	@Override
	public RedisCacheWriter getNativeCache() {
		return this.cacheWriter;
	}

	// cacheWriter会有网络访问请求~~~去访问服务器
	@Override
	protected Object lookup(Object key) {
		byte[] value = cacheWriter.get(name, createAndConvertCacheKey(key));
		if (value == null) {
			return null;
		}
		return deserializeCacheValue(value);
	}

	@Override
	@Nullable
	public ValueWrapper get(Object key) {
		Object value = lookup(key);
		return toValueWrapper(value);
	}

	// 请注意：这个方法因为它要保证同步性，所以使用了synchronized 
	// 还记得我说过的sync=true这个属性吗，靠的就是它来保证的（当然在分布式情况下 不能百分百保证）
	@Override
	@SuppressWarnings("unchecked")
	public synchronized <T> T get(Object key, Callable<T> valueLoader) {
		ValueWrapper result = get(key);
		if (result != null) { // 缓存里有，就直接返回吧
			return (T) result.get();
		}
	
		// 缓存里没有，那就从valueLoader里拿值，拿到后马上put进去
		T value = valueFromLoader(key, valueLoader);
		put(key, value); 
		return value;
	}


	@Override
	public void put(Object key, @Nullable Object value) {
		Object cacheValue = preProcessCacheValue(value);

		// 对null值的判断，看看是否允许存储它呢~~~
		if (!isAllowNullValues() && cacheValue == null) {
			throw new IllegalArgumentException(String.format("Cache '%s' does not allow 'null' values. Avoid storing null via '@Cacheable(unless=\"#result == null\")' or configure RedisCache to allow 'null' via RedisCacheConfiguration.", name));
		}
		cacheWriter.put(name, createAndConvertCacheKey(key), serializeCacheValue(cacheValue), cacheConfig.getTtl());
	}

	// @since 4.1
	@Override
	public ValueWrapper putIfAbsent(Object key, @Nullable Object value) {
	...
	}

	@Override
	public void evict(Object key) {
		cacheWriter.remove(name, createAndConvertCacheKey(key));
	}

	// 清空：在redis中慎用。因为它是传一个*去  进行全部删除的。
	@Override
	public void clear() {
		byte[] pattern = conversionService.convert(createCacheKey("*"), byte[].class);
		cacheWriter.clean(name, pattern);
	}

	public RedisCacheConfiguration getCacheConfiguration() {
		return cacheConfig;
	}
	// 序列化key、value   ByteUtils为redis包的工具类，Data Redis 1.7
	protected byte[] serializeCacheKey(String cacheKey) {
		return ByteUtils.getBytes(cacheConfig.getKeySerializationPair().write(cacheKey));
	}
	protected byte[] serializeCacheValue(Object value) {
		if (isAllowNullValues() && value instanceof NullValue) {
			return BINARY_NULL_VALUE;
		}
		return ByteUtils.getBytes(cacheConfig.getValueSerializationPair().write(value));
	}


	// 创建一个key，请注意这里prefixCacheKey() 前缀生效了~~~
	protected String createCacheKey(Object key) {

		String convertedKey = convertKey(key);

		if (!cacheConfig.usePrefix()) {
			return convertedKey;
		}

		return prefixCacheKey(convertedKey);
	}
	... 
}
```

#### 自定义RedisTemplate和RedisCacheManager

##### 自定义JSON序列化器

```java
@Bean
public RedisSerializer<Object> redisSerializer() {
    //创建JSON序列化器
    Jackson2JsonRedisSerializer<Object> serializer = new Jackson2JsonRedisSerializer<>(Object.class);
    ObjectMapper objectMapper = new ObjectMapper();
    objectMapper.setVisibility(PropertyAccessor.ALL, JsonAutoDetect.Visibility.ANY);
    //必须设置，否则无法将JSON转化为对象，会转化成Map类型
    objectMapper.activateDefaultTyping(LaissezFaireSubTypeValidator.instance,ObjectMapper.DefaultTyping.NON_FINAL);
    serializer.setObjectMapper(objectMapper);
    return serializer;
}
```

> 1、创建一个Jackson2JsonRedisSerializer序列化器，参数为Class对象
>
> 

### Redis配置文件

#### 概述

- redis配置文件对大小写不敏感

##### INCLUDES板块

![image-20201119004722802](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20201119004722802.png)

- 可以引入其他的配置文件，和该配置文件组合使用
- 类似于spring的@PropertiesSource注解引入外部配置文件

##### NETWORK板块

```bash
bind 127.0.0.1  # 绑定的ip地址，开启时只能通过指定ip访问
protected-mode yes # 保护模式
port 6379  # 端口
```

##### GENERAL板块

```bash
daemonize yes # 以守护进程方式运行，默认为no，需要开启，不然一退出进程就结束
pidfile /var/run/redis_6379.pid # 配置文件的pid文件，如果以后台方式运行，就需要指定一个pid进程文件

# Specify the server verbosity level.
# This can be one of:
# debug (a lot of information, useful for development/testing)
# verbose (many rarely useful info, but not a mess like the debug level)
# notice (moderately verbose, what you want in production probably)
# warning (only very important / critical messages are logged)
loglevel notice  # 日志级别

logfile "" # 日志文件保存位置
databases 30 # 默认数据库数量
always-show-logo yes # 是否在启动时显示log
```

##### SNAPSHOTTING板块(快照)

- 持久化：在规定的事件内，执行了多少次操作，则将持久化到文件

```bash
save 900 1  # 如果在900秒内进行了一次操作，则进行持久化操作
save 300 10
save 60 10000

stop-writes-on-bgsave-error yes # 如果持久化发生错误是否继续工作
rdbcompression yes  # 是否压缩rdb文件，消耗cpu资源
rdbchecksum yes  # 保存rdb文件时是否进行校验
dbfilename dump.rdb # rdb文件的保存位置
```

##### REPLICATION板块

##### SECURITY板块

```bash
requirepass nidi1995 # 是否需要设置密码
```

- 直接从终端设置密码

  ```bash
  127.0.0.1:6379> config set requirepass ""
  OK
  ```

##### CLIENTS板块

```bash
maxclients 10000 # 客户端的最大连接数
maxmemory <bytes> # 默认最大内存容量
maxmemory-policy noeviction # 当内存达到上限的处理策略
# 常用处理策略
# - noeviction：当内存使用达到阈值的时候，所有引起申请内存的命令会报错。
# - allkeys-lru：在所有键中采用lru算法删除键，直到腾出足够内存为止。
# - volatile-lru：在设置了过期时间的键中采用lru算法删除键，直到腾出足够内存为止。
# - allkeys-random：在所有键中采用随机删除键，直到腾出足够内存为止。
# - volatile-random：在设置了过期时间的键中随机删除键，直到腾出足够内存为止。
# - volatile-ttl：在设置了过期时间的键空间中，具有更早过期时间的key优先移除。 
##########################################
```

##### APPEND ONLY MODE板块(aof设置)

```bash
appendonly no  # 默认不开启aof模式，使用rdb模式
appendfilename "appendonly.aof"  # aof持久化文件
appendfsync everysec # 同步的执行策略，可以是always、everysec、no
```

### RDB

#### 概述

![img](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/NjYjYvF.png!web)

- 在指定的时间间隔内将内存中的数据集快照写入磁盘，实际操作过程是fork一个子进程，先将数据集写入临时文件，写入成功后，再替换之前的文件，用二进制压缩存储

#### RDB文件触发规则

- save规则满足条件下会自动触发rdb规则
- 执行flushall命令会触发rdb规则，但是flushdb命令并不会
- 退出redis也会生产rdb文件(通过shutdown命令，而不是直接kill进程或强制退出)

#### RDB文件的恢复

- 只要将rdb文件放在redis启动目录，redis启动就会自动检查dump.rdb文件恢复数据(也就是bin目录)

#### 优点

- RDB文件紧凑，全量备份，非常适合用于进行备份和灾难恢复
- RDB 在恢复大数据集时的速度比 AOF 的恢复速度要快
- 生成RDB文件的时候，redis主进程会fork()一个子进程来处理所有保存工作，主进程不需要进行任何磁盘IO操作

#### 缺点

- RDB快照是一次全量备份，存储的是内存数据的二进制序列化形式，存储上非常紧凑。当进行快照持久化时，会开启一个子进程专门负责快照持久化，子进程会拥有父进程的内存数据，父进程修改内存子进程不会反应出来，所以在快照持久化期间修改的数据不会被保存，可能丢失数据
- fork父进程时会占用一定内存空间，开启另外一条线程

### AOF

#### 概述

![img](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/u=17070361,1427234765&fm=15&gp=0.jpg)

- 将所有的命令记录下来，类似于history，恢复的时候把文件中的指令全部重新执行一遍
- redis会将每一个收到的写命令(没有读命令)都通过write函数追加到aof文件中，也就是日志记录；只需追加文件，并不是改写文件

#### 当aof文件出现错误

- 可以通过生成aof文件时redis提供的redis-check-aof工具进行修复

  ```bash
  127.0.0.1:6379> redis-check-aof --fix appendonly.aof
  OK
  ```

#### 优点

- AOF可以更好的保护数据不丢失，一般AOF会每隔1秒，通过一个后台线程执行一次fsync操作，最多丢失1秒钟的数据
- AOF日志文件没有任何磁盘寻址的开销，写入性能非常高，文件不容易破损
- AOF日志文件的命令通过非常可读的方式进行记录，这个特性非常适合做灾难性的误删除的紧急恢复

#### 缺点

- 对于同一份数据来说，AOF日志文件通常比RDB数据快照文件更大
- AOF开启后，支持的写QPS会比RDB支持的写QPS低，因为AOF一般会配置成每秒fsync一次日志文件，当然，每秒一次fsync，性能也还是很高的

### AOF对比RDB

![img](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/8326cffc1e178a82c532308ef2117b8ba977e8ae.jpeg)

https://baijiahao.baidu.com/s?id=1654694618189745916&wfr=spider&for=pc

https://segmentfault.com/a/1190000016021217

### Redis集群

#### 查看当前节点信息

```bash
127.0.0.1:6379> info replication
# Replication
role:master   # 可以看出角色为主节点
connected_slaves:0  # 没有连接的从机
master_replid:574e4f5ef88dae1875a1dfc80ca77893da88eb10
master_replid2:0000000000000000000000000000000000000000
master_repl_offset:0
second_repl_offset:-1
repl_backlog_active:0
repl_backlog_size:1048576
repl_backlog_first_byte_offset:0
repl_backlog_histlen:0
```

#### 使用不同的服务器搭建四个redis节点(默认都是自己的主节点)

![image-20201119103634306](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20201119103634306.png)

![image-20201119103709416](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20201119103709416.png)

#### Redis主从复制

##### 概述

- https://www.cnblogs.com/kismetv/p/9236731.html#t3

- 将一台redis服务器的数据，复制到其他的redis服务器；前者称为主节点(master/leader)，后者称为从节点(slave/follower)；**数据的复制是单向的，只能从主节点到从节点，Master以写为主，Salve以读为主**（80%的操作基本都是进行读操作）

  ![img](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/timg)

- ***默认情况下，每台redis服务器都是主节点***；且一个主节点可以有多个从节点(或没有从节点)，但一个从节点只能有一个主节点

##### 作用

- 数据冗余：主从复制实现了数据的热备份，是持久化之外的一种数据冗余方式
- 故障恢复：当主节点出现问题时，可以由从节点提供服务，实现快速的故障恢复；实际上是一种服务的冗余
- 负载均衡：在主从复制的基础上，配合读写分离，可以由主节点提供写服务，由从节点提供读服务（即写Redis数据时应用连接主节点，读Redis数据时应用连接从节点），分担服务器负载；尤其是在写少读多的场景下，通过多个从节点分担读负载，可以大大提高Redis服务器的并发量
- 高可用基石：除了上述作用以外，主从复制还是哨兵和集群能够实施的基础，因此说主从复制是Redis高可用的基础

#### 一主三从配置

##### 规定

- 阿里云的服务器作为主机，腾讯云和其他两个虚拟机作为从机

- slaveof 主机的ip和端口

  ```bash
  127.0.0.1:6379> SLAVEOF 123.57.66.144 6379
  OK
  ```

##### 查看阿里云和其他主机信息

- 腾讯云和虚拟机

  ![image-20201119104304842](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20201119104304842.png)

- 阿里云(**需要关闭密码验证**)

  ![image-20201119104939491](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20201119104939491.png)

- 如果不关闭密码验证，需要在从机配置文件的`REPLICATION板块`修改masterauth配置

  ![image-20201119110650485](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20201119110650485.png)
  - 或者在从机终端通过命令配置

    ```bash
    127.0.0.1:6379> CONFIG SET masterauth <nidi1995>
    OK
    ```

##### 从配置文件配置

- 从机的REPLICATION板块

  ```bash
  replicaof <masterip> <masterport>  # 代表主机的ip和端口
  masterauth <nidi1995>  # 代表主机的验证信息
  ```

- 服务器启动时就会作为从机启动(主机的地址固定，因为配置文件已经写死)

##### 测试

- 主机中添加一个k-v

  ```bash
  127.0.0.1:6379> FLUSHALL
  OK
  127.0.0.1:6379> SET k1 v1
  OK
  ```

- 从机中进行测试(从机中直接获得值)

  ```bash
  127.0.0.1:6379> GET k1
  "v1"
  ```

- 从机没有写的权限

  ```bash
  127.0.0.1:6379> set k2 v2
  (error) READONLY You can't write against a read only replica.
  ```

- 主机断开连接之后再重新连接，主机的写操作依然可以从从机获得对应的值

- 如果是通过终端配置的从机，当断开连接再重新连接时，该服务器就不会再作为从机存在，而是作为主机；通过配置文件配置的依然作为从机

- 即使在从机断开期间主机写入的数据，当从机重新连接并作为从机运行时，依然可以拿到断开期间写入的数据

#### 主从复制原理

- 主从节点之间的连接建立以后，便可以开始进行数据同步，该阶段可以理解为从节点数据的初始化。具体执行的方式是：从节点向主节点发送psync命令（Redis2.8以前是sync命令），开始同步
- 在数据同步阶段之前，从节点是主节点的客户端，主节点不是从节点的客户端；而到了这一阶段及以后，主从节点互为客户端。原因在于：在此之前，主节点只需要响应从节点的请求即可，不需要主动发请求，而在数据同步阶段和后面的命令传播阶段，主节点需要主动向从节点发送请求（如推送缓冲区中的写命令），才能完成复制
- 数据同步阶段完成后，主从节点进入命令传播阶段；在这个阶段主节点将自己执行的写命令发送给从节点，从节点接收命令并执行，从而保证主从节点数据的一致性
- 根据主从节点当前状态的不同，可以分为全量复制和部分复制：
  - 全量复制
    - 用于初次复制或其他无法进行部分复制的情况，将主节点中的所有数据都发送给从节点，是一个非常重型的操作
  - 部分复制
    - 用于网络中断等情况后的复制，只将中断期间主节点执行的写命令发送给从节点，与全量复制相比更加高效。需要注意的是，如果网络中断时间过长，导致主节点没有能够完整地保存中断期间执行的写命令，则无法进行部分复制，仍使用全量复制

### 哨兵模式

#### 概述

- 哨兵模式是一种特殊的模式，首先Redis提供了哨兵的命令，哨兵是一个独立的进程，作为进程，它会独立运行。其原理是**哨兵通过发送命令，等待Redis服务器响应，从而监控运行的多个Redis实例。**

![image-20201119113810036](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20201119113810036.png)

- 哨兵作用
  - 通过发送命令，让Redis服务器返回监控其运行状态，包括主服务器和从服务器
  - 当哨兵监测到master宕机，会自动将slave切换成master，然后通过**发布订阅模式**通知其他的从服务器，修改配置文件，让它们切换主机

- 多哨兵模式

  - 一个哨兵进程对Redis服务器进行监控，可能会出现问题，可以使用多个哨兵进行监控。各个哨兵之间还会进行监控，这样就形成了多哨兵模式

  ![img](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/11320039-3f40b17c0412116c.png)
  
  - 假设主服务器宕机，哨兵1先检测到这个结果，系统并不会马上进行failover过程，仅仅是哨兵1主观的认为主服务器不可用，这个现象成为**主观下线**；当后面的哨兵也检测到主服务器不可用，并且数量达到一定值时，那么哨兵之间就会进行一次投票，投票的结果由一个哨兵发起，进行failover操作；切换成功后，就会通过发布订阅模式，让各个哨兵把自己监控的从服务器实现切换主机，这个过程称为**客观下线**

#### 测试

##### 配置哨兵配置文件 sentinel.conf

- 在docker容器中找到redis哨兵启动程序

  - 位置为/usr/local/bin/redis-sentinel

- 在bin目录中创建配置文件sentinel.conf

- ```bash
  cat << EOF > sentinel.conf
  # sentinel monitor 监视器名称 被监控服务器的IP和端口号 哨兵多少次判断节点才算死亡
  > sentinel monitor mymaster 123.57.66.144  6379 1     
  > EOF
  ```

##### 使用redis哨兵启动程序以配置文件启动

```bash
ls    # bin目录下的所有文件
docker-entrypoint.sh  redis-benchmark  redis-check-rdb	redis-sentinel	sentinel.conf
gosu		      redis-check-aof  redis-cli	redis-server

redis-sentinel sentinel.conf  # 以配置文件启动
```

- 启动之后的界面

  ![image-20201119130557957](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20201119130557957.png)

- 端口为26379，监控器mymaster监控的ip为123.57.66.144，主机下面有三个从机

##### 当主机宕机之后

- 停止了阿里云docker容器之后，哨兵就会监视到主机宕机并重新选举

  ![image-20201119131153115](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20201119131153115.png)

- 检测到阿里云redis宕机之后，重新选举vm-1为新的主机

  ```bash
  127.0.0.1:6379> info replication
  # Replication
  role:master  
  connected_slaves:1
  master_replid:8ac6fb64ad4ab83adfc0c1a62ed04ab30345ad26
  master_replid2:0000000000000000000000000000000000000000
  master_repl_offset:0
  second_repl_offset:-1
  repl_backlog_active:0
  repl_backlog_size:1048576
  repl_backlog_first_byte_offset:0
  repl_backlog_histlen:0
  ```

##### 当主机重新连接之后

- 重连接之后就会作为新主机的从机

##### 优点

- 哨兵集群，基于主从复制模式，所有的主从配置优点，它都有
- 主从可以切换，故障可以转移，高可用性的系统
- 哨兵模式就是主从模式的升级，手动到自动，更加健壮

##### 缺点

- Redis不好在线扩容的，集群容量一旦到达上限，在线扩容就十分麻烦
- 哨兵模式的配置繁琐

##### sentinel.conf配置模板

https://www.icode9.com/content-4-650003.html

### Redis缓存穿透、击穿、雪崩(高可用问题)

#### 缓存正常流程

![img](Redis.assets/20180919143214712)

#### 缓存穿透(查不到)

##### 概述

用户发起请求的数据是缓存和数据库中都没有的数据，也就是缓存并没有命中，于是就向持久层查询；而用户不断发起请求，如发起为id为“-1”的数据或id为特别大不存在的数据，这时的用户很可能是攻击者，攻击会导致数据库压力过大

##### 解决方案

- 布隆过滤器

  - 布隆过滤器是一种数据结构，对所有可能查询的参数以hash形式存储，在控制层先进行校验，不符合则直接丢弃，从而避免对底层存储系统的压力

    ![img](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/20201105162325563.jpeg)



- 缓存空对象
  - 当存储层不命中后，即使返回的空对象也将其缓存起来，同时会设置一个过期时间，之后再访问这个数据将会从缓存中获取，保护了后端数据源
  - 出现的问题
    - 如果空值能够被缓存起来，这就意味着缓存需要更多的空间存储更多的键，因为这当中可能会有很多的空值的键
    - 即使对空值设置了过期时间，还是会存在缓存层和存储层的数据会有一段时间窗口的不一致，这对于需要保持一致性的业务会有影响

#### 缓存击穿(访问量大、缓存过期)

##### 概述

- 指一个key非常热点，在不停的扛着大并发，大并发集中对这一个点进行访问，当这个key在失效的瞬间，持续的大并发就穿破缓存，直接请求数据库，就像在一个屏障上凿开了一个洞
- 当某个key在过期的瞬间，有大量的请求并发访问，由于缓存过期，则会同时访问数据库来查询最新的数据，并且回写缓存，导致数据库瞬间压力过大
- 类似于微博热搜，许多用户同时访问该热搜

##### 解决方案

- 设置热点数据永远不过期

  - 从缓存层面来看，没有设置过期时间，故不会出现热点key过期现象

- 加互斥锁

  - 分布式锁：使用分布式锁，保证对于每个key同时只有一个线程查询后端服务，其他线程没有获得分布式锁的权限，故需要等待

    ![image-20201119140109620](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20201119140109620.png)

#### 缓存雪崩

##### 概述

- 缓存中数据大批量到过期时间，而查询数据量巨大，引起数据库压力过大甚至down机。和缓存击穿不同的是，缓存击穿指并发查同一条数据，缓存雪崩是不同数据都过期了，很多数据都查不到从而查数据库

![img](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/9c16fdfaaf51f3dec6606ae3a504f81938297953.jpeg)

- 双十一：停掉一些其他的服务

##### 解决方案

- redis高可用
  - 既然redis有可能挂掉，那么多增设几台redis，这样一台挂掉之后其他的还可以继续工作，其实就是搭建的集群
- 限流降级
  - 在缓存失效后，通过加锁或者队列来控制读数据库写缓存的线程数量。比如对某个key只允许一个线程查询数据和写缓存，其他线程等待
- 数据预热
  - 在正式部署之前，我先把可能的数据先预先访问一遍，这样部分可能大量访问的数据就会加载到缓存中。在即将发生大并发访问前手动触发加载缓存不同的key，设置不同的过期时间，让缓存失效的时间点尽量均匀