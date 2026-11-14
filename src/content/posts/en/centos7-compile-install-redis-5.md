---
title: "Compiling and Installing Redis 5.0.9 on CentOS 7"
author: "John Yang"
lang: en
pubDatetime: 2021-04-03T07:26:00Z
tags:
  - Linux
  - Redis
description: "Step-by-step guide to downloading, compiling, configuring, and running Redis 5.0.9 as a systemd service on CentOS 7."
---

## Download Redis

```shell
wget https://download.redis.io/releases/redis-5.0.9.tar.gz
```

## Extract the Archive

```shell
tar -zxvf redis-5.0.9.tar.gz
```

## Compile and Install

```shell
cd redis-5.0.9
make && make install
```

When the build succeeds, you will see output similar to the following (paths may vary depending on your environment):

```shell
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
```

## Configuration

Copy the default configuration file to `/etc/`:

```shell
cp redis.conf /etc/
```

Create the log directory and log file:

```shell
cd /var/log
mkdir redis
cd redis
touch redis.log
```

Copy the Redis binaries to `/usr/bin/`:

```shell
cd src
cp redis-cli redis-benchmark redis-server /usr/bin
```

Update `/etc/redis.conf`:

```shell
vim /etc/redis.conf

# Set idle client connection timeout to 30 minutes (default: 0)
timeout 1800

# Run Redis as a background daemon (default: no)
daemonize yes

# Set log file path to match the file created above
logfile "/var/log/redis/redis.log"

# Allow connections from remote clients
protected-mode no

# Set the Redis authentication password
requirepass redisPassword
```

## Configure `systemd` Auto-Start

Create `/usr/lib/systemd/system/redis.service`:

```shell
vim /usr/lib/systemd/system/redis.service

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
```

## Enable and Start Redis

```shell
# Enable service on boot
systemctl enable redis

# Start Redis
systemctl start redis
```

## Check Redis Logs

```shell
cd /var/log/redis
cat redis.log
```

You should see startup output similar to:

```shell
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
```
