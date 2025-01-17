Mysql使用篇
mysql无密码时，用dbeaver连接不会成功，而navicat又需要权限，于是采用命令行模式导入：
命令行导入步骤：
1. 登陆mysql,因为端口时默认的3306 所以省略 -P 3306。
```
mysql -h localhost  -u root --skip-password
```
2. 创建数据库：
```
CREATE DATABASE MYDB
```
3. 在数据库中执行sql文件的内容：
source /路径/到/你的/sql文件.sql;
这种模式本来时可以的，但是也是因为没有设置密码的关系，不会成功：
```
mysql -u 用户名 -p mydatabase < /路径/到/你的/example.sql
```
