---
title: Redis的数据类型
date: 2021-01-26 20:54:57
categories: 缓存
tags:
- redis
---

[Redis 命令参考 - Redis 命令参考](http://redisdoc.com/)

# **Redis的数据类型 - string**

**string 字符串**

string: 最简单的字符串类型键值对缓存，也是最基本的

**key相关**

keys *：查看所有的key (不建议在生产上使用，有性能影响)

type key：key的类型

**string类型**

get/set/del：查询/设置/删除set rekey data：设置已经存在的key，会覆盖setnx rekey data：设置已经存在的key，不会覆盖

set key value ex time：设置带过期时间的数据expire key：设置过期时间ttl：查看剩余时间，-1永不过期，-2过期

append key：合并字符串strlen key：字符串长度

incr key：累加1decr key：类减1incrby key num：累加给定数值decrby key num：累减给定数值

getrange key start end：截取数据，end=-1 代表到最后setrange key start newdata：从start位置开始替换数据

mset：连续设值mget：连续取值msetnx：连续设置，如果存在则不设置

**其他**

select index：切换数据库，总共默认16个flushdb：删除当前下边db中的数据flushall：删除所有db中的数据

# **Redis的数据类型 - hash**

**hash**

hash：类似map，存储结构化数据结构，比如存储一个对象（不能有嵌套对象）

**使用**

hset key property value：> hset user name imooc> 创建一个user对象，这个对象中包含name属性，name值为imooc

hget user name：获得用户对象中name的值

hmset：设置对象中的多个键值对> hset user age 18 phone 139123123hmsetnx：设置对象中的多个键值对，存在则不添加> hset user age 18 phone 139123123

hmget：获得对象中的多个属性> hmget user age phone

hgetall user：获得整个对象的内容

hincrby user age 2：累加属性hincrbyfloat user age 2.2：累加属性

hlen user：有多少个属性

hexists user age：判断属性是否存在

hkeys user：获得所有属性hvals user：获得所有值

hdel user：删除对象

# **Redis的数据类型 - list**

**list**

list：列表，[a, b, c, d, …]

**使用**

lpush userList 1 2 3 4 5：构建一个list，从左边开始存入数据rpush userList 1 2 3 4 5：构建一个list，从右边开始存入数据lrange list start end：获得数据

lpop：从左侧开始拿出一个数据rpop：从右侧开始拿出一个数据

pig cow sheep chicken duck

llen list：list长度lindex list index：获取list下标的值

lset list index value：把某个下标的值替换

linsert list before/after value：插入一个新的值

lrem list num value：删除几个相同数据

ltrim list start end：截取值，替换原来的list

# **Redis的数据类型 - zset**

**sorted set：**

sorted set：排序的set，可以去重可以排序，比如可以根据用户积分做排名，积分作为set的一个数值，根据数值可以做排序。set中的每一个memeber都带有一个分数

**使用**

zadd zset 10 value1 20 value2 30 value3：设置member和对应的分数

zrange zset 0 -1：查看所有zset中的内容zrange zset 0 -1 withscores：带有分数

zrank zset value：获得对应的下标zscore zset value：获得对应的分数

zcard zset：统计个数zcount zset 分数1 分数2：统计个数

zrangebyscore zset 分数1 分数2：查询分数之间的member(包含分数1 分数2)zrangebyscore zset (分数1 (分数2：查询分数之间的member（不包含分数1 和 分数2）zrangebyscore zset 分数1 分数2 limit start end：查询分数之间的member(包含分数1 分数2)，获得的结果集再次根据下标区间做查询

zrem zset value：删除member