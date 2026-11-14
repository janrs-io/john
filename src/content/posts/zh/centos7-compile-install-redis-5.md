---
title: "CentOS7下编译安装redis-5.0.9"
author: "John Yang"
lang: zh
pubDatetime: 2021-04-03T07:26:00Z
tags:
  - Linux
  - Redis
description: "最后出现如下代码表示成功，路径根据自己的环境可能不同 复制配置文件到/etc/下 复制redis执行文件到/usr/bin/下"
---

## CentOS7下编译安装redis-5.0.9

#### **下载redis**
```shell
#code start

wget https://download.redis.io/releases/redis-5.0.9.tar.gz

#code end
```

#### **解压**
```shell
#code start

tar -zxvf redis-5.0.9.tar.gz

#code end
```

#### **安装**
```shell
#code start

cd redis-5.0.9

make && make install

#code end
```
最后出现如下代码表示成功，路径根据自己的环境可能不同
```shell
#code start

Hint: It's a good idea to run 'make test' ;)

make[1]: Leaving directory `/docker_share/redis-5.0.9/src'
cd src && make install
make[1]: Entering directory `/docker_share/redis-5.0.9/src'
    CC Makefile.dep
make[1]: Leaving directory `/docker_share/redis-5.0.9/src'
make[1]: Entering directory `/docker_share/redis-5.0.9/src'

Hint: It's a good idea to run 'make test' ;)

    INSTALL install
    INSTALL install
    INSTALL install
    INSTALL install
    INSTALL install
make[1]: Leaving directory `/docker_share/redis-5.0.9/src'

#code end
```
#### **配置**
复制配置文件到/etc/下
```shell
#code start

cp redis.conf /etc/

#code end
```

新建日志文件地址
```shell
#code start

cd /var/log

mkdir redis

cd redis

touch redis.log

#code end
```

复制redis执行文件到/usr/bin/下
```shell
#code start

cd src

cp redis-cli redis-benchmark redis-server /usr/bin

#code end
```
修改默认配置文件
```shell
#code start

vim /etc/redis.conf

#修改默认客户端最长连接时间。修改为30分钟
timeout 1800

#修改以守护进程的方式启动。默认为no
daemonize yes

# 设置日志文件的地址，跟上面的设置地址一样
logfile "/var/log/redis/redis.log"

#允许其他机器上的客户端连接当前redis
protected-mode no

#修改redis密码
requirepass redisPassword

#code end
```
#### **设置开机启动**
```shell
#code start

vim /usr/lib/systemd/system/redis.service

#启动代码
[Unit]
Description=Redis persistent key-value database
After=network.target
After=network-online.target
Wants=network-online.target

[Service]
ExecStart=/usr/bin/redis-server /etc/redis.conf
ExecStop=/usr/bin/redis-cli shutdown
#Restart=always
Type=forking
#User=redis
#Group=redis
RuntimeDirectory=redis
RuntimeDirectoryMode=0755

[Install]
WantedBy=multi-user.target

#code end
```
#### **设置开机启动并启动redis**
```shell
#code start

#开机启动
systemctl enable redis

#启动redis
systemctl start redis

#code end
```

#### **启动redis**
```shell
#code start

systemctl start redis

#code end
```

#### **查看redis日志**
```shell
#code start

cd /var/log/redis

cat redis.log

#code end
```

显示日志内容如下
```shell
#code start

8118:C 03 Apr 2021 04:38:52.898 # oO0OoO0OoO0Oo Redis is starting oO0OoO0OoO0Oo
8118:C 03 Apr 2021 04:38:52.898 # Redis version=5.0.9, bits=64, commit=00000000, modified=0, pid=8118, just started
8118:C 03 Apr 2021 04:38:52.898 # Configuration loaded
                _._
           _.-``__ ''-._
      _.-``    `.  `_.  ''-._           Redis 5.0.9 (00000000/0) 64 bit
  .-`` .-```.  ```\/    _.,_ ''-._
 (    '      ,       .-`  | `,    )     Running in standalone mode
 |`-._`-...-` __...-.``-._|'` _.-'|     Port: 6379
 |    `-._   `._    /     _.-'    |     PID: 8119
  `-._    `-._  `-./  _.-'    _.-'
 |`-._`-._    `-.__.-'    _.-'_.-'|
 |    `-._`-._        _.-'_.-'    |           http://redis.io
  `-._    `-._`-.__.-'_.-'    _.-'
 |`-._`-._    `-.__.-'    _.-'_.-'|
 |    `-._`-._        _.-'_.-'    |
  `-._    `-._`-.__.-'_.-'    _.-'
      `-._    `-.__.-'    _.-'
          `-._        _.-'
              `-.__.-'

8119:M 03 Apr 2021 04:38:52.901 # WARNING: The TCP backlog setting of 511 cannot be enforced because /proc/sys/net/core/somaxconn is set to the lower value of 128.
8119:M 03 Apr 2021 04:38:52.902 # Server initialized
8119:M 03 Apr 2021 04:38:52.902 # WARNING you have Transparent Huge Pages (THP) support enabled in your kernel. This will create latency and memory usage issues with Redis. To fix this issue run the command 'echo never > /sys/kernel/mm/transparent_hugepage/enabled' as root, and add it to your /etc/rc.local in order to retain the setting after a reboot. Redis must be restarted after THP is disabled.
8119:M 03 Apr 2021 04:38:52.902 * Ready to accept connections

#code end
```
