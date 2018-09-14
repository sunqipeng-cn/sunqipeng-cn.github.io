---
layout: post
title:  "redis中的数据类型和用法"
date:   2018-05-13 23:06:05
categories: 分布式
tags: redis
author: sqp
---

* content
{:toc}

# redis中的数据类型和用法

>Redis是一个远程内存数据库,不仅性能强劲,还具有复制特性和为解决问题而生的独一无二的数据模型。

## String

### SETEX key seconds value

将值 value 关联到 key ，并将 key 的生存时间设为 seconds (以秒为单位)。  
如果 key 已经存在， SETEX 命令将覆写旧值。  
这个命令类似于以下两个命令：  

``` shell
SET key value
EXPIRE key seconds  # 设置生存时间

redis> SETEX cache_user_id 60 10086
redis> GET cache_user_id  # 值
redis> TTL cache_user_id  # 剩余生存时间

# key 已经存在时，SETEX 覆盖旧值
redis> SET cd "timeless"
redis> SETEX cd 3000 "goodbye my love"
redis> GET cd
redis> TTL cd
```

### PSETEX key milliseconds value  

这个命令和 SETEX 命令相似，但它以毫秒为单位设置 key 的生存时间，而不是像 SETEX 命令那样，以秒为单位。

``` shell
redis> PSETEX mykey 1000 "Hello"
OK

redis> PTTL mykey
(integer) 999

redis> GET mykey
"Hello"
```

### SETNX key value

将 key 的值设为 value ，当且仅当 key 不存在。  
若给定的 key 已经存在，则 SETNX 不做任何动作。  
SETNX 是『SET if Not eXists』(如果不存在，则 SET)的简写。  
设置成功，返回 1 。  
设置失败，返回 0 。  

``` shell
redis> EXISTS job                # job 不存在
(integer) 0
redis> SETNX job "programmer"    # job 设置成功
(integer) 1
redis> SETNX job "code-farmer"   # 尝试覆盖 job ，失败
(integer) 0
redis> GET job                   # 没有被覆盖
"programmer"
```

### MSETNX key value [key value ...]

同时设置一个或多个 key-value 对，当且仅当所有给定 key 都不存在。  
即使只有一个给定 key 已存在， MSETNX 也会拒绝执行所有给定 key 的设置操作。  
MSETNX 是原子性的，因此它可以用作设置多个不同 key 表示不同字段(field)的唯一性逻辑对象(unique logic object)，所有字段要么全被设置，要么全不被设置。  
返回值：  
当所有 key 都成功设置，返回 1 。  
如果所有给定 key 都设置失败(至少有一个 key 已经存在)，那么返回 0 。  

``` shell
# 对不存在的 key 进行 MSETNX
redis> MSETNX rmdbs "MySQL" nosql "MongoDB" key-value-store "redis"
redis> MGET rmdbs nosql key-value-store
# MSET 的给定 key 当中有已存在的 key
redis> MSETNX rmdbs "Sqlite" language "python"  # rmdbs 键已经存在，操作失败
redis> EXISTS language                          # 因为 MSET 是原子性操作，language 没有被设置
redis> GET rmdbs                                # rmdbs 也没有被修改
```

### INCR key

将 key 中储存的数字值增一。  
如果 key 不存在，那么 key 的值会先被初始化为 0 ，然后再执行 INCR 操作。  

### INCRBY key increment

将 key 所储存的值加上增量 increment 。  
如果 key 不存在，那么 key 的值会先被初始化为 0 ，然后再执行 INCRBY 命令。

### DECR key

将 key 中储存的数字值减一。  
如果 key 不存在，那么 key 的值会先被初始化为 0 ，然后再执行 DECR 操作。

### DECRBY key decrement

将 key 所储存的值减去减量 decrement 。  
如果 key 不存在，那么 key 的值会先被初始化为 0 ，然后再执行 DECRBY 操作。

### GETSET key value

将给定 key 的值设为 value ，并返回 key 的旧值(old value)。  
当 key 存在但不是字符串类型时，返回一个错误。  
当 key 没有旧值时，也即是， key 不存在时，返回 nil 。  

``` shell
redis> GETSET db mongodb    # 没有旧值，返回 nil
redis> GET db
redis> GETSET db redis      # 返回旧值 mongodb
redis> GET db
```

## Hash

### HSET key field value

将哈希表 key 中的域 field 的值设为 value 。  
如果 key 不存在，一个新的哈希表被创建并进行 HSET 操作。  
如果域 field 已经存在于哈希表中，旧值将被覆盖。  
如果 field 是哈希表中的一个新建域，并且值设置成功，返回 1 。  
如果哈希表中域 field 已经存在且旧值已被新值覆盖，返回 0  

``` shell
redis> HSET website google "www.g.cn"       # 设置一个新域
(integer) 1

redis> HSET website google "www.google.com" # 覆盖一个旧域
(integer) 0
```

### HGET key field

返回哈希表 key 中给定域 field 的值。  
时间复杂度：O(1)  
返回值：给定域的值。  
当给定域不存在或是给定 key 不存在时，返回 nil 。  

``` shell
# 域存在
redis> HSET site redis redis.com
(integer) 1
redis> HGET site redis
"redis.com"
# 域不存在
redis> HGET site mysql
(nil)
```

### HMSET key field value [field value ...]

同时将多个 field-value (域-值)对设置到哈希表 key 中。  
此命令会覆盖哈希表中已存在的域。   
如果 key 不存在，一个空哈希表被创建并执行 HMSET 操作。  
时间复杂度：O(N)， N 为 field-value 对的数量。  
返回值：如果命令执行成功，返回 OK 。  
当 key 不是哈希表(hash)类型时，返回一个错误。  

``` shell
redis> HMSET website google www.google.com yahoo www.yahoo.com
OK
redis> HGET website google
"www.google.com"
redis> HGET website yahoo
"www.yahoo.com"
```

### HMGET key field [field ...]

返回哈希表 key 中，一个或多个给定域的值  
如果给定的域不存在于哈希表，那么返回一个 nil 值。  
因为不存在的 key 被当作一个空哈希表来处理，所以对一个不存在的 key 进行 HMGET 操作将返回一个只带有 nil 值的表。  
返回值：一个包含多个给定域的关联值的表，表值的排列顺序和给定域参数的请求顺序一样。  

``` shell
redis> HMSET pet dog "doudou" cat "nounou"    # 一次设置多个域
OK
redis> HMGET pet dog cat fake_pet             # 返回值的顺序和传入参数的顺序一样
1) "doudou"
2) "nounou"
3) (nil)                                      # 不存在的域返回nil值
```

### HKEYS key

返回哈希表 key 中的所有域。  

``` shell 
# 哈希表非空
redis> HMSET website google www.google.com yahoo www.yahoo.com
OK
redis> HKEYS website
1) "google"
2) "yahoo"
# 空哈希表/key不存在
redis> EXISTS fake_key
(integer) 0
redis> HKEYS fake_key
(empty list or set)
```

### HVALS key

返回一个包含哈希表中所有值的表。  
时间复杂度：O(N)， N 为哈希表的大小。  
当 key 不存在时，返回一个空表。  

``` shell
# 非空哈希表
redis> HMSET website google www.google.com yahoo www.yahoo.com
OK
redis> HVALS website
1) "www.google.com"
2) "www.yahoo.com"
# 空哈希表/不存在的key
redis> EXISTS not_exists
(integer) 0
redis> HVALS not_exists
(empty list or set)
```

### HDEL key field [field ...]

删除哈希表 key 中的一个或多个指定域，不存在的域将被忽略。  

``` shell
# 测试数据
redis> HGETALL abbr
1) "a"
2) "apple"
3) "b"
4) "banana"
5) "c"
6) "cat"
7) "d"
8) "dog"
# 删除单个域
redis> HDEL abbr a
(integer) 1
```

### HINCRBY key field increment

为哈希表 key 中的域 field 的值加上增量 increment 。  
增量也可以为负数，相当于对给定域进行减法操作。  

### HINCRBYFLOAT key field increment

为哈希表 key 中的域 field 加上浮点数增量 increment 。  
如果哈希表中没有域 field ，那么 HINCRBYFLOAT 会先将域 field 的值设为 0 ，然后再执行加法操作。  

### HEXISTS key field

查看哈希表 key 中，给定域 field 是否存在。  
如果哈希表含有给定域，返回 1 。  
如果哈希表不含有给定域，或 key 不存在，返回 0 。  

### HGETALL key

返回哈希表 key 中，所有的域和值。在返回值里，紧跟每个域名(field name)之后是域的值(value)，所以返回值的长度是哈希表大小的两倍。  

### SCAN cursor [MATCH pattern] [COUNT count]

SCAN 命令是一个基于游标的迭代器（cursor based iterator）： SCAN 命令每次被调用之后， 都会向用户返回一个新的游标， 用户在下次迭代时需要使用这个新游标作为 SCAN 命令的游标参数， 以此来延续之前的迭代过程。  
当 SCAN 命令的游标参数被设置为 0 时， 服务器将开始一次新的迭代， 而当服务器向用户返回值为 0 的游标时， 表示迭代已结束。以下是一个 SCAN 命令的迭代过程示例：  

``` shell
redis> scan 0
1) "7"
2)  1) "group"
    2) "park"
    3) "4"
    4) "7"
    5) "3"
    6) "8"
    7) "17"
    8) "1"
    9) "15"
   10) "14"

redis> scan 7
1) "0"
2) 1) "sst"
   2) "stock"
   3) "stock180824"
```

在上面这个例子中， 第一次迭代使用 0 作为游标， 表示开始一次新的迭代。  
第二次迭代使用的是第一次迭代时返回的游标， 也即是命令回复第一个元素的值 —— 7 。  
从上面的示例可以看到， SCAN 命令的回复是一个包含两个元素的数组， 第一个数组元素是用于进行下一次迭代的新游标， 而第二个数组元素则是一个数组， 这个数组中包含了所有被迭代的元素。  
在第二次调用 SCAN 命令时， 命令返回了游标 0 ， 这表示迭代已经结束， 整个数据集（collection）已经被完整遍历过了。
以 0 作为游标开始一次新的迭代， 一直调用 SCAN 命令， 直到命令返回游标 0 ， 我们称这个过程为一次完整遍历（full iteration）。  

SCAN 命令返回的每个元素都是一个数据库键。  
SSCAN 命令返回的每个元素都是一个集合成员。  
HSCAN 命令返回的每个元素都是一个键值对，一个键值对由一个键和一个值组成。  
ZSCAN 命令返回的每个元素都是一个有序集合元素，一个有序集合元素由一个成员（member）和一个分值（score）组成。  

### DUMP key

序列化给定 key ，并返回被序列化的值，使用 RESTORE 命令可以将这个值反序列化为 Redis 键。
序列化生成的值有以下几个特点：  
它带有 64 位的校验和，用于检测错误， RESTORE 在进行反序列化之前会先检查校验和。
值的编码格式和 RDB 文件保持一致。  
RDB 版本会被编码在序列化值当中，如果因为 Redis 的版本不同造成 RDB 格式不兼容，那么 Redis 会拒绝对这个值进行反序列化操作。  

``` shell
redis> SET greeting "hello, dumping world!"
OK
redis> DUMP greeting
"\x00\x15hello, dumping world!\x06\x00E\xa0Z\x82\xd8r\xc1\xde"
redis> DUMP not-exists-key
(nil)
```

## List

### LPUSH key value [value ...]

将一个或多个值 value 插入到列表 key 的表头  
如果有多个 value 值，那么各个 value 值按从左到右的顺序依次插入到表头： 比如说，对空列表 mylist 执行命令 LPUSH mylist a b c ，列表的值将是 c b a ，这等同于原子性地执行 LPUSH mylist a 、 LPUSH mylist b 和 LPUSH mylist c 三个命令。  
如果 key 不存在，一个空列表会被创建并执行 LPUSH 操作。  

### LRANGE key start stop

返回列表 key 中指定区间内的元素，区间以偏移量 start 和 stop 指定。  
下标(index)参数 start 和 stop 都以 0 为底，也就是说，以 0 表示列表的第一个元素，以 1 表示列表的第二个元素，以此类推。  
你也可以使用负数下标，以 -1 表示列表的最后一个元素， -2 表示列表的倒数第二个元素，以此类推。  

``` shell
# 加入单个元素
redis> LPUSH languages python
(integer) 1
# 加入重复元素
redis> LPUSH languages python
(integer) 2
redis> LRANGE languages 0 -1     # 列表允许重复元素
1) "python"
2) "python"
# 加入多个元素
redis> LPUSH mylist a b c
(integer) 3
redis> LRANGE mylist 0 -1
1) "c"
2) "b"
3) "a"
```

### LPOP key

移除并返回列表 key 的头元素。

### RPOP key

移除并返回列表 key 的尾元素。

``` shell
redis> RPUSH mylist "one"
(integer) 1
redis> RPUSH mylist "two"
(integer) 2
redis> RPUSH mylist "three"
(integer) 3
redis> RPOP mylist           # 返回被弹出的元素
"three"
redis> LRANGE mylist 0 -1    # 列表剩下的元素
1) "one"
2) "two"
```

### LINDEX key index

返回列表 key 中，下标为 index 的元素。  
下标(index)参数 start 和 stop 都以 0 为底，也就是说，以 0 表示列表的第一个元素，以 1 表示列表的第二个元素，以此类推。你也可以使用负数下标，以 -1 表示列表的最后一个元素， -2 表示列表的倒数第二个元素，以此类推。如果 key 不是列表类型，返回一个错误。  

### LSET key index value

将列表 key 下标为 index 的元素的值设置为 value 。

``` shell
redis> EXISTS list
(integer) 0

#对空列表进行lset
redis> LSET list 0 item
(error) ERR no such key

redis> LPUSH job "cook food"
(integer) 1

redis> LRANGE job 0 0
1) "cook food"

#对非空列表进行lset
redis> LSET job 0 "play game"
OK

redis> LRANGE job 0 0
1) "play game"


redis> LLEN job
(integer) 1

#index超出范围的lset
redis> LSET job 3 "out of range"
(error) ERR index out of range
```

### LREM key count value

根据参数 count 的值，移除列表中与参数 value 相等的元素。  
count 的值可以是以下几种：  
count > 0 : 从表头开始向表尾搜索，移除与 value 相等的元素，数量为 count 。  
count < 0 : 从表尾开始向表头搜索，移除与 value 相等的元素，数量为 count 的绝对值。  
count = 0 : 移除表中所有与 value 相等的值。  

时间复杂度 O(n)， n为列表的长度。

### LTRIM key start stop

对一个列表进行修剪(trim)，就是说，让列表只保留指定区间内的元素，不在指定区间之内的元素都将被删除。  
举个例子，执行命令 LTRIM list 0 2 ，表示只保留列表 list 的前三个元素，其余元素全部删除。
下标(index)参数 start 和 stop 都以 0 为底，也就是说，以 0 表示列表的第一个元素，以 1 表示列表的第二个元素，以此类推。  
你也可以使用负数下标，以 -1 表示列表的最后一个元素， -2 表示列表的倒数第二个元素，以此类推。  

时间复杂度 O(n)， n为被移除的元素的数量。

## Set

### SMEMBERS key

返回集合 key 中的所有成员。  
不存在的 key 被视为空集合。  

时间复杂度 O(N)， N 为集合的基数。

### SADD key member [member ...]

将一个或多个 member 元素加入到集合 key 当中，已经存在于集合的 member 元素将被忽略。  
假如 key 不存在，则创建一个只包含 member 元素作成员的集合。  
当 key 不是集合类型时，返回一个错误。

### SINTER key [key ...]

返回一个集合的全部成员，该集合是所有给定集合的交集。  
不存在的 key 被视为空集。  
当给定集合当中有一个空集时，结果也为空集(根据集合运算定律)。

### SPOP key

移除并返回集合中的一个随机元素。  
如果只想获取一个随机元素，但不想该元素从集合中被移除的话，可以使用 SRANDMEMBER 命令。  

### SREM key member [member ...]

移除集合 key 中的一个或多个 member 元素，不存在的 member 元素会被忽略。  
当 key 不是集合类型，返回一个错误。  

``` shell
redis> SMEMBERS languages
1) "c"
2) "lisp"
3) "python"
4) "ruby"

# 移除单个元素
redis> SREM languages ruby
(integer) 1

# 移除不存在元素
redis> SREM languages non-exists-language
(integer) 0

# 移除多个元素
redis> SREM languages lisp python c
(integer) 3

redis> SMEMBERS languages
(empty list or set)
```

## SortedSet

### ZADD key score member [[score member] [score member] ...]

score 值可以是整数值或双精度浮点数。  
时间复杂度:O(M*log(N))， N 是有序集的基数， M 为成功添加的新成员的数量。  
返回值:被成功添加的新成员的数量，不包括那些被更新的、已经存在的成员。  

``` shell
# 添加单个元素

redis> ZADD page_rank 10 google.com
(integer) 1


# 添加多个元素

redis> ZADD page_rank 9 baidu.com 8 bing.com
(integer) 2

redis> ZRANGE page_rank 0 -1 WITHSCORES
1) "bing.com"
2) "8"
3) "baidu.com"
4) "9"
5) "google.com"
6) "10"


# 添加已存在元素，且 score 值不变

redis> ZADD page_rank 10 google.com
(integer) 0

redis> ZRANGE page_rank 0 -1 WITHSCORES  # 没有改变
1) "bing.com"
2) "8"
3) "baidu.com"
4) "9"
5) "google.com"
6) "10"


# 添加已存在元素，但是改变 score 值

redis> ZADD page_rank 6 bing.com
(integer) 0

redis> ZRANGE page_rank 0 -1 WITHSCORES  # bing.com 元素的 score 值被改变
1) "bing.com"
2) "6"
3) "baidu.com"
4) "9"
5) "google.com"
6) "10"
```

### ZRANGE key start stop [WITHSCORES]

返回有序集 key 中，指定区间内的成员。其中成员的位置按 score 值递增(从小到大)来排序。  
下标参数 start 和 stop 都以 0 为底，也就是说，以 0 表示有序集第一个成员，以 1 表示有序集第二个成员，以此类推。你也可以使用负数下标，以 -1 表示最后一个成员， -2 表示倒数第二个成员，以此类推。

``` shell
redis > ZRANGE salary 0 -1 WITHSCORES             # 显示整个有序集成员
1) "jack"
2) "3500"
3) "tom"
4) "5000"
5) "boss"
6) "10086"

redis > ZRANGE salary 1 2 WITHSCORES              # 显示有序集下标区间 1 至 2 的成员
1) "tom"
2) "5000"
3) "boss"
4) "10086"

redis > ZRANGE salary 0 200000 WITHSCORES         # 测试 end 下标超出最大下标时的情况
1) "jack"
2) "3500"
3) "tom"
4) "5000"
5) "boss"
6) "10086"

redis > ZRANGE salary 200000 3000000 WITHSCORES   # 测试当给定区间不存在于有序集时的情况
(empty list or set)
```

### ZREM key member [member ...]

移除有序集 key 中的一个或多个成员，不存在的成员将被忽略。

### ZREMRANGEBYRANK key start stop

移除有序集 key 中，指定排名(rank)区间内的所有成员。下标参数 start 和 stop 都以 0 为底，也就是说，以 0 表示有序集第一个成员，以 1 表示有序集第二个成员，以此类推。你也可以使用负数下标，以 -1 表示最后一个成员， -2 表示倒数第二个成员，以此类推。

### ZREMRANGEBYSCORE key min max

移除有序集 key 中，所有 score 值介于 min 和 max 之间(包括等于 min 或 max )的成员。

### ZSCORE key member

返回有序集 key 中，成员 member 的 score 值。  
如果 member 元素不是有序集 key 的成员，或 key 不存在，返回 nil 。

### ZINCRBY key increment member

为有序集 key 的成员 member 的 score 值加上增量 increment 。
返回值:member 成员的新 score 值，以字符串形式表示。

