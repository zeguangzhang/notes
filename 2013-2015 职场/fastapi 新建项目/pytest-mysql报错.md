Pip  install pytest-mysql报错如下：
```
  exec(code, locals())
        File "<string>", line 155, in <module>
        File "<string>", line 49, in get_config_posix
        File "<string>", line 28, in find_package_name
      Exception: Can not find valid pkg-config name.
      Specify MYSQLCLIENT_CFLAGS and MYSQLCLIENT_LDFLAGS env vars manually
      [end of output]
``` 
尝试解决：
1. 更新python版本从3.9到3.10，与当前使用的python版本一致，还是报错 (brew方式安装感觉更好)
2. 搜索天工，给出针对mac的答案，完美解决：
  a. 如果您使用 Homebrew，首先安装 MySQL 客户端和 pkg-config：
```
brew install mysql-client pkg-config
export PKG_CONFIG_PATH="$(brew --prefix)/opt/mysql-client/lib/pkgconfig"
```

  b. 然后尝试安装 mysqlclient：
  ```
pip install mysqlclient
  ```