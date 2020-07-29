##前提
因为一次大意把redis默认的持久化改成了非持久化配置，当时没有持久化的原因是redis保存一直报错，导致只能读取，不能写入操作，所以一时半会没找到解决方法，放着侥幸心里把redis
持久化配置干掉
### redis配置文件怎样不持久化
```
-增加一条这个
save ""
-下面注释掉
#save 900 1
#save 300 10
#save 60 10000

就这么简单
```

## 出现的问题
但是运行一段时间应用都退出了，最终发现redis挂掉了，由于没有持久化导致redis启动后数据全无

## 出现了上述问题 恢复过程
### 恢复思路
爬虫主要有4部分信息是重要的：去重信息，待下载队列，待解析队列(前面这三种都在redis)，持久化部分（我这里是mongo）
由于前三种信息都在redis中导致这些信息全无，如果全部想恢复是不能恢复了（由于每次下载放入队列的url没有用日志打出，因为着实没必要打印这个）。所以
采用持久化内容做为恢复数据的根本，把持久化的信息url放入去重队列，然后在放入新的种子放入待下载队列。

### 恢复过程：
-url读取：
mongoexport --collection=scrapy_items --db=crawl_ask1 --fields=url --out=./url1.json

eg: mongo导出的的原始信息 分为三个数据库分别导出

```
db1:
{"_id":{"$oid":"5eba756968dc172742f085f6"},"url":"https://www.120ask.com/question/41691482.htm"}
...
...
{"_id":{"$oid":"5eba756968dc172742f085f7"},"url":"https://www.120ask.com/question/40605309.htm"}

db2:
{"_id":{"$oid":"5eba756a68dc172742f085fe"},"url":"https://www.120ask.com/question/11614458.htm"}
{"_id":{"$oid":"5eba756a68dc172742f085ff"},"url":"https://www.120ask.com/question/70591891.htm"}
...
...
//由于修改程序原因，下面url会变成这样
{"_id":{"$oid":"5f0de49e20a91a8ef13093b1"},"url":"62262866"}
{"_id":{"$oid":"5f0de49e20a91a8ef13093b2"},"url":"88166"}
{"_id":{"$oid":"5f0de49e20a91a8ef13093b3"},"url":"998743"}

db3:
{"_id":{"$oid":"5f0de49e20a91a8ef13093b4"},"url":"68225137"}
{"_id":{"$oid":"5f0de49e20a91a8ef13093b5"},"url":"18915738"}
...
{"_id":{"$oid":"5f0de49e20a91a8ef13093b9"},"url":"68351238"}
```

-awk分割（多字符分割）：
注意下awk可以多字符分割 一次分割出来 参考链接：https://blog.csdn.net/BigBirds911/article/details/54382682
```

cat url0.json| awk -F 'www.120ask.com/question/|.htm' '{print $2}'  

cat url1.json| grep "www.120ask.com/question" | awk -F 'www.120ask.com/question/|.htm' '{print $2}' >url1part1.url 
cat  url1.json| grep -v "www.120ask.com/question" | awk -F '"url":"|"}' '{print $3}' >url1part2.url
cat url1part* > url1.url

cat  url2.json| grep -v "www.120ask.com/question" | awk -F '"url":"|"}' '{print $3}' >url2.url

最终merge:
cat url0.url url1.url url2.url > urlall.url
```



### redis配置持久化，解决上次持久化问题。
--解决持久化保存失败问题，为什么redis在运行一段时间后会出现save失败(当时程序写redis错误时去redis命令行模式下去写入一样，然后想赶紧save吧，这是save一直失败)，
导致客户端只能只读不可写的问题。其实就在最近几天遇到的一个磁盘空间导致tab命令快捷键出问题，于是想到了可能是这个问题，就是redis保存的目录存储空间不够导致。

--看机器磁盘整体目录资源大小划分结构 df -h
```angular2
文件系统                                                   容量  已用  可用 已用% 挂载点
devtmpfs                                                    32G     0   32G    0% /dev
tmpfs                                                       32G   20K   32G    1% /dev/shm
tmpfs                                                       32G  3.2G   29G   10% /run
tmpfs                                                       32G     0   32G    0% /sys/fs/cgroup
/dev/sda2                                                   30G   24G  5.6G   82% /
/dev/sda1                                                  497M  109M  389M   22% /boot
//baikemypdfs1.file.core.chinacloudapi.cn/baikemypdfs1sh1 1000G  416G  585G   42% /data/repository
/dev/sdb1                                                   79G   57M   75G    1% /mnt/resource
tmpfs                                                      6.3G     0  6.3G    0% /run/user/0
/dev/sdc1                                                 1007G  225G  731G   24% /data
tmpfs                                                      6.3G     0  6.3G    0% /run/user/1003
tmpfs                                                      6.3G     0  6.3G    0% /run/user/1004
```
可以看到 挂载点/data目录空间比较大，决定redis dump文件保存在该目录下
于是修改了redis配置文件修改 dir
```
# The working directory.
#
# The DB will be written inside this directory, with the filename specified
# above using the 'dbfilename' configuration directive.
#
# The Append Only File will also be created inside this directory.
#
# Note that you must specify a directory here, not a file name.
dir /data/zhangzeguang/redis/
```
执行上面操作后，后续又发现问题->解决方法：
- redis 下面的配置文件修改后(修改dir)不生效 -> redis启动手动指定配置文件路径 src/redis-server ./redis.conf &
- 执行后save还是失败，因为保存的dump文件目录没权限 -> 需要sudo
- 但是sudo后启动了2个进程（两个线上kill很有意思的现象：kill掉sudo没问题，但是kill非sudo的就都会挂掉），linux sudo运行原来会启动两个进程，这块后期调研下原因


### 写一个python脚本把这些url信息写入redis去重队列
```
import re
import redis

#offline config
redis_ip = 'localhost'
redis_port = 6379
redis_password = 'test123'

pool = redis.ConnectionPool(host=redis_ip, port=redis_port, password=redis_password, decode_responses=True)   # host是redis主机，需要redis服务端和客户端都起着 redis默认端口是6379
redis_cli = redis.Redis(connection_pool=pool)
CRAWLED_KEY = "ask_crawl_dup"

if __name__ == '__main__':
    with open("/data/urlall.url", 'rt') as f:
        for line in f.readlines():
            if line:
                print(line.strip())
                redis_cli.sadd(CRAWLED_KEY, line.strip())

```
由于上面单线程操作redis保存了1400万url数据到redis,也需要一小时时间。


## 给自己留一手
### redis在线修改配置怎么办？



### redis save一次需要多久
redis现在内存信息：
- 执行： info memory
# Memory
used_memory:18929541016
used_memory_human:17.63G
used_memory_rss:18539913216
used_memory_rss_human:17.27G
used_memory_peak:19293846864
used_memory_peak_human:17.97G
used_memory_peak_perc:98.11%
used_memory_overhead:841462
used_memory_startup:791392
used_memory_dataset:18928699554
used_memory_dataset_perc:100.00%
allocator_allocated:18929816600
allocator_active:19857002496
allocator_resident:19957665792
total_system_memory:67540705280
total_system_memory_human:62.90G
used_memory_lua:37888
used_memory_lua_human:37.00K
used_memory_scripts:0
used_memory_scripts_human:0B
number_of_cached_scripts:0
maxmemory:0
maxmemory_human:0B
maxmemory_policy:noeviction
allocator_frag_ratio:1.05
allocator_frag_bytes:927185896
allocator_rss_ratio:1.01
allocator_rss_bytes:100663296
rss_overhead_ratio:0.93
rss_overhead_bytes:-1417752576
mem_fragmentation_ratio:0.98
mem_fragmentation_bytes:-389586776
mem_not_counted_for_evict:0
mem_replication_backlog:0
mem_clients_slaves:0
mem_clients_normal:49694
mem_aof_buffer:0
mem_allocator:jemalloc-5.1.0
active_defrag_running:0
lazyfree_pending_objects:0

- 登录redis客户端 执行save 故意save了2次（主要想看平均值）看下效果
127.0.0.1:6379> save
OK
(299.99s)
127.0.0.1:6379> save
OK
(283.63s)

从上面可以看到17.63G内容save需要近5分钟时间，也就是说一分钟可持久化近3.5g内容，如果是线上服务就很要命的(在save期间不允许读写操作，连接redis都不可以)。
这种手动调用是同步save，在配置文件中的save策略是异步策略。不会影响线上读写操作，但还是会浪费机器资源的。
但是保存在dump.rdb中会进行快照压缩，最终持久化文件大小为5.6G：
-rw-r--r--  1 root         root         5.6G 7月  29 10:45 dump.rdb
可见压缩率还是可观的

- redis设置为持久化后，redis重启后会自动加载持久化内容到硬盘，时间跟save时间等同
eg:redis启动后开始一段时间 执行任何命令结果：
127.0.0.1:6379> keys *
(error) LOADING Redis is loading the dataset in memory
(0.60s)
这是因为redis还没有启动起来：只有当redis启动日志出现 Ready to accept connections才算启动成功，如下：
31311:M 29 Jul 2020 11:20:03.923 * DB loaded from disk: 180.411 seconds
31311:M 29 Jul 2020 11:20:03.923 * Ready to accept connections

不过从上面可以看出redis启动加载到磁盘比保存在磁盘的save要快很多。一个近300秒一个180秒,
那是什么原因呢，难道是重新加载进来的会内存优化之类的，我们先去看看redis内存信息：
# Memory
used_memory:18929506352
used_memory_human:17.63G
发现没有变，确定了不是做内存优化，那原因其实就确定了，就是磁盘速度读取速度比写入速度快导致。

- 后台保存按理花费时间也很长，如果一次还没保存完，下一次保存的请求又来了怎么处理？
可以调研一下，我估计应该是如果有就取消保存任务，或者用一个消息队列，消息队列长度是1，后来的自动覆盖前面的保存消息，
应该是采用后面策略，前面直接取消不靠谱。






