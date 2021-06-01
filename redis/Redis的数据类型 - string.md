# 

# Redis的数据类型 - string

### string 字符串

string: 最简单的字符串类型键值对缓存，也是最基本的

### key相关

keys *：查看所有的key (不建议在生产上使用，有性能影响)

type key：key的类型

### string类型

get/set/del：查询/设置/删除
set rekey data：设置已经存在的key，会覆盖
setnx rekey data：设置已经存在的key，不会覆盖

set key value ex time：设置带过期时间的数据
expire key：设置过期时间
ttl：查看剩余时间，-1永不过期，-2过期

append key：合并字符串
strlen key：字符串长度

incr key：累加1
decr key：类减1
incrby key num：累加给定数值
decrby key num：累减给定数值

getrange key start end：截取数据，end=-1 代表到最后
setrange key start newdata：从start位置开始替换数据

mset：连续设值
mget：连续取值
msetnx：连续设置，如果存在则不设置

### 其他

select index：切换数据库，总共默认16个
flushdb：删除当前下边db中的数据
flushall：删除所有db中的数据



# Redis的数据类型 - hash

### hash

hash：类似map，存储结构化数据结构，比如存储一个对象（不能有嵌套对象）

### 使用

hset key property value：
-> hset user name imooc
-> 创建一个user对象，这个对象中包含name属性，name值为imooc

hget user name：获得用户对象中name的值

hmset：设置对象中的多个键值对
-> hset user age 18 phone 139123123
hmsetnx：设置对象中的多个键值对，存在则不添加
-> hset user age 18 phone 139123123

hmget：获得对象中的多个属性
-> hmget user age phone

hgetall user：获得整个对象的内容

hincrby user age 2：累加属性
hincrbyfloat user age 2.2：累加属性

hlen user：有多少个属性

hexists user age：判断属性是否存在

hkeys user：获得所有属性
hvals user：获得所有值

hdel user：删除对象

# Redis的数据类型 - list

### list

list：列表，[a, b, c, d, …]

### 使用

lpush userList 1 2 3 4 5：构建一个list，从左边开始存入数据
rpush userList 1 2 3 4 5：构建一个list，从右边开始存入数据
lrange list start end：获得数据

lpop：从左侧开始拿出一个数据
rpop：从右侧开始拿出一个数据

pig cow sheep chicken duck

llen list：list长度
lindex list index：获取list下标的值

lset list index value：把某个下标的值替换

linsert list before/after value：插入一个新的值(左为前)

​	before c c1 ==> ...,c1 ,c,... 

​	after c c1 ==>    ...,c,c1,...

lrem list num value：删除几个相同数据

ltrim list start end：截取值，替换原来的list

