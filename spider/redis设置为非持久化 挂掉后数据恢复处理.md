##前提
因为一次大意把redis默认的持久化改成了非持久化配置，当时没有持久化的原因是redis保存一直报错，导致只能读取，不能写入操作，所以一时半会没找到解决方法，放着侥幸心里把redis
持久化配置干掉

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

