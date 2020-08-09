
## 背景
mongo修改默认数据库路径挂载点空间不够，需要修改mongo默认数据路径到空间足够项目使用的目录下。

## 修改MongoDB默认数据路径只需以下几步
1. 停止MongoDB
```
$ sudo systemctl stop mongod.service
```
2. 复制mongo至新位置
MongoDB默认数据路径为 /var/lib/mongo
```
$ sudo rsync -av /var/lib/mongo /mnt/database/mongodb/
```
这里， 一定是 /var/lib/mongo，而不是/var/lib/mongo/，多了个斜杠，rsync将转储目录复制到安装点，而不是转移成一个包含内容mongo的目录。

3. 备份原来文件
```angular2
$ sudo mv /var/lib/mongo /var/lib/mongo.bak
```
修改数据存储路径并服务重启成功后可删除。

4. 修改配置文件
```
$ sudo vi /etc/mongod.conf
```
将文件中的修改为dbPath
```
dbPath: /mnt/database/mongodb/mongo
```
并且注释掉bindIp，以使其他远程终端能连接MongoDB。

5. 启动MongoDB
```
$ sudo systemctl start mongod.service

```
6. 查看是否启动成功
```
$ sudo systemctl status mongod.service
```
若显示 active（running）则启动成功！或者
```
$ sudo cat /var/log/mongodb/mongod.log
```
如出现
```
[thread1] waiting for connections on port <port>
```
其中的默认为27017，在 /etc/mongod.conf中配置，则启动成功！


##参考链接：
https://blog.csdn.net/zqx1205/article/details/75330320