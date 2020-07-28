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

##出现的问题
但是运行一段时间应用都退出了，最终发现redis挂掉了，由于没有持久化导致redis启动后数据全无

##出现了上述问题 恢复过程
### 恢复思路
爬虫主要有4部分信息是重要的：去重信息，待下载队列，待解析队列(前面这三种都在redis)，持久化部分（我这里是mongo）
由于前三种信息都在redis中导致这些信息全无，如果全部想恢复是不能恢复了（由于每次下载放入队列的url没有用日志打出，因为着实没必要打印这个）。所以
采用持久化内容做为恢复数据的根本，把持久化的信息url放入去重队列，然后在放入新的种子放入待下载队列。

###恢复过程：
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
1 redis 下面的配置文件修改后(修改dir)不生效 -> redis启动手动指定配置文件路径 src/redis-server ./redis.conf &
2 执行后save还是失败，因为保存目录不在用户目录 -> 需要sudo
3 但是sudo后启动了2个进程，kill掉一个就都kill掉了 因为当前redis执行文件目录在用户目录下，而redis dir参数是在/data/目录下，所以启动了两个 -> redis整个目录拷贝到非用户目录（/data）下ok


-写一个python脚本把这些url信息写入redis去重队列



