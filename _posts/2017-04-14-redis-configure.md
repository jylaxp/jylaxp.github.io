---
layout: post
title: Redis 安装和配置
---

## 1.安装redis
### 环境:  
- 系统:centos6.8

### 准备工作
由于采用源码编译安装redis，所以在编译之前先做一些准备工作  
- redis源码是c语言，所以编译redis需要gcc
```shell
yum install -y gcc
# 或者安装开发工具包
yum groupinstall -y "Development Tools"
```
- 下载redis源码并加压
```shell
wget http://download.redis.io/redis-stable.tar.gz
tar zxvf redis-stable.tar.gz
cd redis-stable
```
### 编译安装
```shell
make
make test
make install
```
### 遇到的问题
- 执行make的时候报下面的错
```shell
zmalloc.h:51:31: error: jemalloc/jemalloc.h: No such file or directory
```
解决办法：  
```
make MALLOC=libc
```
或
```
make distclean
```
然后再make
- make成功以后，需要make test。在make test出现异常
```shell
couldn't execute "tclsh8.5": no such file or directory
```
解决办法
```shell
yum install -y tcl
```   


## 2. 启动和停止Redis

|命令|说明|
| :----------: | :----: |
|redis-server|redis服务|
|redis-cli|redis命令行客户端|
|redis-benchmark|redis性能测试工具|
|redis-check-aof|aof文件修复工具|
|redis-check-rdb|rdb文件修复工具|
|redis-sentinel|sentinel服务器|

### 2.1 直接启动
```shell
redis-server
```
终端里输出一下内容：
```shell
2142:C 04 Apr 19:35:23.300 # Warning: no config file specified, using the default config. In order to specify a config file use redis-server /path/to/redis.conf
2142:M 04 Apr 19:35:23.303 * Increased maximum number of open files to 10032 (it was originally set to 1024).
                _._
           _.-``__ ''-._
      _.-``    `.  `_.  ''-._           Redis 3.2.8 (00000000/0) 64 bit
  .-`` .-```.  ```\/    _.,_ ''-._
 (    '      ,       .-`  | `,    )     Running in standalone mode
 |`-._`-...-` __...-.``-._|'` _.-'|     Port: 6379
 |    `-._   `._    /     _.-'    |     PID: 2142
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

2142:M 04 Apr 19:35:23.322 # WARNING: The TCP backlog setting of 511 cannot be enforced because /proc/sys/net/core/somaxconn is set to the lower value of 128.
2142:M 04 Apr 19:35:23.322 # Server started, Redis version 3.2.8
2142:M 04 Apr 19:35:23.323 # WARNING overcommit_memory is set to 0! Background save may fail under low memory condition. To fix this issue add 'vm.overcommit_memory = 1' to /etc/sysctl.conf and then reboot or run the command 'sysctl vm.overcommit_memory=1' for this to take effect.
2142:M 04 Apr 19:35:23.323 # WARNING you have Transparent Huge Pages (THP) support enabled in your kernel. This will create latency and memory usage issues with Redis. To fix this issue run the command 'echo never > /sys/kernel/mm/transparent_hugepage/enabled' as root, and add it to your /etc/rc.local in order to retain the setting after a reboot. Redis must be restarted after THP is disabled.
2142:M 04 Apr 19:35:23.323 * DB loaded from disk: 0.000 seconds
2142:M 04 Apr 19:35:23.323 * The server is now ready to accept connections on port 6379

```

输出中有四个警告，现在来解决这四个警告
- 第一个警告
``` shell
no config file specified, using the default config. In order to specify a config file use redis-server /path/to/redis.conf
```

- 第二个警告
```shell
The TCP backlog setting of 511 cannot be enforced because /proc/sys/net/core/somaxconn is set to the lower value of 128.
```
临时修改，本次会话有效
```shell
echo 511 > /proc/sys/net/core/somaxconn
```
修改配置文件/etc/sysctl.conf, 重启后生效
```shell
vim /etc/sysctl.conf
```
在末尾追加
```shell
net.core.somaxconn = 512
```
保存后重启
- 第三个警告
```shell
overcommit_memory is set to 0! Background save may fail under low memory condition. To fix this issue add 'vm.overcommit_memory = 1' to /etc/sysctl.conf and then reboot or run the command 'sysctl vm.overcommit_memory=1' for this to take effect
```
这个警告中说的已经很清楚了
修改配置文件 /etc/sysctl.conf， 在文件加入vm.overcommit_memory = 1 重启生效；  
命令行中输入一下命令，本次会话有效
```shell
sysctl vm.overcommit_memory=1
```
- 第四个警告
```shell
you have Transparent Huge Pages (THP) support enabled in your kernel. This will create latency and memory usage issues with Redis. To fix this issue run the command 'echo never > /sys/kernel/mm/transparent_hugepage/enabled' as root, and add it to your /etc/rc.local in order to retain the setting after a reboot. Redis must be restarted after THP is disabled.
```
命令行中输入一下命令，本次会话有效
```shell
echo never > /sys/kernel/mm/transparent_hugepage/enabled
``` 
将上述命令加入到/etc/rc.local 文件， 重启后生效

### 2.2 通过初始化脚本启动Redis
在Redis源代码目录的utils文件夹中有一个名为redis_init_script的初始化脚本文件,内容如下：
```shell
#!/bin/sh
#
# Simple Redis init.d script conceived to work on Linux systems
# as it does use of the /proc filesystem.

REDISPORT=6379
EXEC=/usr/local/bin/redis-server
CLIEXEC=/usr/local/bin/redis-cli

PIDFILE=/var/run/redis_${REDISPORT}.pid
CONF="/etc/redis/${REDISPORT}.conf"

case "$1" in
    start)
        if [ -f $PIDFILE ]
        then
                echo "$PIDFILE exists, process is already running or crashed"
        else
                echo "Starting Redis server..."
                $EXEC $CONF
        fi
        ;;
    stop)
        if [ ! -f $PIDFILE ]
        then
                echo "$PIDFILE does not exist, process is not running"
        else
                PID=$(cat $PIDFILE)
                echo "Stopping ..."
                $CLIEXEC -p $REDISPORT shutdown
                while [ -x /proc/${PID} ]
                do
                    echo "Waiting for Redis to shutdown ..."
                    sleep 1
                done
                echo "Redis stopped"
        fi
        ;;
    *)
        echo "Please use start or stop as first argument"
        ;;
esac

```

为了让脚本正常工作，需要做一些配置
- redis配置文件   
    上述脚本中有这一行代码
```shell
CONF="/etc/redis/${REDISPORT}.conf"
```
这个脚本启动的时候回读取 /etc/redis/6379.conf 这个配置文件。在redis的源码中有一个 redis.conf 的模板配置文件，
我就以这个配置文件为模板来进行配置。
```
mkdir /etc/redis
cp redis.conf /etc/redis/6379.conf
```
这一步已经完成，redis.conf 配置文件的具体配置项，后面有详细的配置。
- 开机启动
    将上述脚本文件copy到 /etc/init.d目录中
```
cp redis_init_script /etc/init.d/redis
```
在 /etc/init.d/redis 脚本靠前的位置加入
```shell
# chkconfig:   2345 90 10
# description:  Redis is a persistent key-value database
```
设置开机启动
```
chkconfig redis on
```
修改redis启动方式， 修改配置文件/etc/redis/6379.conf的daemonize项，改为yes

```shell
################################# GENERAL #####################################

# By default Redis does not run as a daemon. Use 'yes' if you need it.
# Note that Redis will write a pid file in /var/run/redis.pid when daemonized.
daemonize yes
```
至此，脚本开启启动配置完成。


## 3. 复制
### 3.1 环境:  
准备三台机器，ip地址如下:
- node1: 192.168.10.10
- node2: 192.168.10.11
- node3: 192.168.10.12

### 3.2 配置
#### 3.2.1 规划：  
master: 192.168.10.10  
slave: 192.168.10.11, 192.168.10.11   

#### 3.2.2修改配置
redis主从复制配置很简单，只需配置在从节点的配置文件项即可
```shell
# Master-Slave replication. Use slaveof to make a Redis instance a copy of
# another Redis server. A few things to understand ASAP about Redis replication.
#
# 1) Redis replication is asynchronous, but you can configure a master to
#    stop accepting writes if it appears to be not connected with at least
#    a given number of slaves.
# 2) Redis slaves are able to perform a partial resynchronization with the
#    master if the replication link is lost for a relatively small amount of
#    time. You may want to configure the replication backlog size (see the next
#    sections of this file) with a sensible value depending on your needs.
# 3) Replication is automatic and does not need user intervention. After a
#    network partition slaves automatically try to reconnect to masters
#    and resynchronize with them.
#
# slaveof <masterip> <masterport>
```
```shell
slaveof 192.168.10.10 6379
```
从节点就可以从192.168.10.10上复制数据了。  
启动master，slave 主从复制就配置完成了。


## 4. Sentinel
Sentinel是一个不提供读写功能的redis server，用来监控redis server的状态，在redis server主从复制master出现故障时，
完成failover，保证高可用。   
### 4.1配置Sentinel
配置3个配置Sentinel节点（至于为什么是3个而不是2个或者4个，这个涉及到leader的选举，不是这个的重点）
- Sentinel node1: 192.168.10.20
- Sentinel node2: 192.168.10.21
- Sentinel node3: 192.168.10.22  

这3个节点的配置是一样的，以一个节点为例子。  
#### 4.1.1 配置网络
```shell
# Example sentinel.conf

# *** IMPORTANT ***
#
# By default Sentinel will not be reachable from interfaces different than
# localhost, either use the 'bind' directive to bind to a list of network
# interfaces, or disable protected mode with "protected-mode no" by
# adding it to this configuration file.
#
# Before doing that MAKE SURE the instance is protected from the outside
# world via firewalling or other means.
#
# For example you may use one of the following:
#
# bind 127.0.0.1 192.168.1.1
bind 0.0.0.0
#
protected-mode no

# port <sentinel-port>
# The port that this sentinel instance will run on
port 26379
```

- bind: 绑定网络接口, 这里为了方便，bind了所有的接口
- protected-mode： 关闭了保护模式
- port： 监听端口，这是使用了默认值26379
 
#### 4.1.2 配置monitor
```shell
# sentinel monitor <master-name> <ip> <redis-port> <quorum>
#
# Tells Sentinel to monitor this master, and to consider it in O_DOWN
# (Objectively Down) state only if at least <quorum> sentinels agree.
#
# Note that whatever is the ODOWN quorum, a Sentinel will require to
# be elected by the majority of the known Sentinels in order to
# start a failover, so no failover can be performed in minority.
#
# Slaves are auto-discovered, so you don't need to specify slaves in
# any way. Sentinel itself will rewrite this configuration file adding
# the slaves using additional configuration options.
# Also note that the configuration file is rewritten when a
# slave is promoted to master.
#
# Note: master name should not include special characters or spaces.
# The valid charset is A-z 0-9 and the three characters ".-_".
sentinel monitor mymaster 192.168.10.10 6379 2
```
这个配置项的格式是：sentinel monitor <master-name> <ip> <redis-port> <quorum>
- master-name: 给master起个名字，用于区分不同master。因为一个sentinel可以监控n组master
- ip: redis server master的ip
- redis-port: redis server master的端口
- quorum：法定票数，故障判定时使用   

#### 4.1.3 其他配置
```shell
daemonize yes
logfile "/root/redis/sentinel.log"
```
- daemonize: 配置 sentinel 以 daemonize 方式启动
- logfile： sentinel 的日志文件

### 4.2 测试
#### 4.2.1 启动sentinel
在3个节点上启动 sentinel
```shell
redis-sentinel sentinel.conf
```
连接一个 sentinel ，看看状态
```shell
redis-cli -p 26379
```
```shell
127.0.0.1:26379> info sentinel
# Sentinel
sentinel_masters:1
sentinel_tilt:0
sentinel_running_scripts:0
sentinel_scripts_queue_length:0
sentinel_simulate_failure_flags:0
master0:name=mymaster,status=ok,address=192.168.10.10:6379,slaves=2,sentinels=3
```
可以看出，master的name为mymaster, 状态ok, ip为192.168.10.10，端口为6379，有2个slave, 3个sentinel

#### 4.2.2 故障转移
模拟 master 故障， 这里 kill 调 redis server master 进程
```shell
[root@node2 redis]# ps -ef | grep 6379
root       8543      1  0 13:56 ?        00:00:23 /usr/local/bin/redis-server 0.0.0.0:6379     
root       9774   7749  0 18:39 pts/1    00:00:00 grep 6379
[root@node2 redis]# kill -9 8543

```
连接 sentinel 上查看状态
```shell
[root@node2 redis]# redis-cli -p 26379
127.0.0.1:26379> info sentinel
# Sentinel
sentinel_masters:1
sentinel_tilt:0
sentinel_running_scripts:0
sentinel_scripts_queue_length:0
sentinel_simulate_failure_flags:0
master0:name=mymaster,status=odown,address=192.168.10.12:6379,slaves=2,sentinels=3
```
可以看到 master 的地址已经变了，已经完成了故障转移。


## 5. Cluster
### 5.1 环境
节点:
- node1: 192.168.10.10 8000
- node2: 192.168.10.11 8000
- node3: 192.168.10.12 8000
- node4: 192.168.10.10 8001
- node5: 192.168.10.11 8002
- node6: 192.168.10.12 8003   


|mstart-slave | master|slave|
|---|---|---|
|mstart-slave| node1|node4|
|mstart-slave| node2|node5|
|mstart-slave| node3|node6|

注意：这里为了方便，master 和 slave 放在同一台机器上了，如果是生成环境绝对绝对不要这样！！！   

### 5.2 Cluster配置
#### 5.2.1 配置开启节点
- master 节点配置
```shell
bind 0.0.0.0
port 8000  
cluster-enabled yes  
cluster-config-file nodes-8000.conf  
cluster-node-timeout 15000  
dir "/root/redis/data/"
appendonly yes  
appendfilename "appendonly-8000.aof"  
logfile "8000.log"  
daemonize yes  
pidfile /var/run/redis-8000.pid   
dbfilename "dump-8000.rdb"
logfile "/root/redis/log/redis_8000.log"
```
- slave 节点配置， salve节点配置是和master节点配置一样的。这里master和slave部署在同一台机器上，要做端口区分。
```shell
sed 's/8000/8001/g' redis-8000.conf > redis-8001.conf
```

启动6个节点
```shell
redis-server 8000.conf
redis-server 8001.conf
```

查看节点启动情况
```shell
[root@master redis]# ps -ef | grep redis-server
root       2336      1  0 12:23 ?        00:00:51 redis-server 0.0.0.0:8000 [cluster]
root       2358      1  0 12:25 ?        00:00:34 redis-server 0.0.0.0:8001 [cluster]
```
查看单个节点
```shell
[root@master redis]# redis-cli -c -p 8000
127.0.0.1:8000> CLUSTER INFO
cluster_state:fail
cluster_slots_assigned:0
cluster_slots_ok:0
cluster_slots_pfail:0
cluster_slots_fail:0
cluster_known_nodes:1
cluster_size:0
cluster_current_epoch:0
cluster_my_epoch:0
cluster_stats_messages_sent:0
cluster_stats_messages_received:0
```

#### 5.2.2 meet
在192.168.10.10上， 利用redis-cli连接到8000，然后meet 其他节点
```
[root@master redis]# redis-cli -c -p 8000
127.0.0.1:8000> CLUSTER MEET 192.168.10.10 8001
OK
127.0.0.1:8000> CLUSTER MEET 192.168.10.11 8000
OK
127.0.0.1:8000> CLUSTER MEET 192.168.10.11 8001
OK
127.0.0.1:8000> CLUSTER MEET 192.168.10.12 8000
OK
127.0.0.1:8000> CLUSTER MEET 192.168.10.12 8001
OK
```
查看节点
```
127.0.0.1:8000> CLUSTER NODES
8b496ced472ff6913a079cb56bba8bd9cee12c61 192.168.95.128:8000 myself,master - 0 0 2 connected 0-5460
b1b77d1f042f7abe18dbe5f461c7ee0d4a001e7e 192.168.95.135:8000 master - 0 1491638444324 8 connected 5461-10992
eddca3c1b37b3574f511422b213fd2085f8e17a2 192.168.95.135:8001 slave b1b77d1f042f7abe18dbe5f461c7ee0d4a001e7e 0 1491638442304 8 connected
f91b42c1f481297d5347b7b518cca39ee14200ce 192.168.95.136:8000 master - 0 1491638441294 0 connected 10993-16383
f2be65c867349aaab162124174a2f1b1848bb4ab 192.168.95.136:8001 slave f91b42c1f481297d5347b7b518cca39ee14200ce 0 1491638440789 5 connected
3ca09eba65a2fa42275aa30d80999b59bd4e2085 192.168.95.128:8001 slave 8b496ced472ff6913a079cb56bba8bd9cee12c61 0 1491638443314 2 connected
127.0.0.1:8000> 
```

#### 5.2.3 指派槽
分派slots
```shell
cluster addslots <slot> [slot ...] 将一个或多个槽（slot）指派（assign）给当前节点。
```
redis-cli -c -p 8000 cluster addslots 0 1 2 ...   
redis-cli 未实现0-5462这样的参数，必须一个个输入。   
所以利用shell生成最终的命令 addslot.sh
```shell
#! /bin/sh
start=$1
end=$2
ip=$3
port=$4

for slot in `seq ${start} ${end}`
do
    echo "slot:${slot}"
    redis-cli -c -h ${ip} -p ${port} cluster addslots ${slot}
done
```
执行： 
```shell
./addslot.sh 0 5460 192.168.10.10 8000
./addslot.sh 5461 10922 192.168.10.11 8000
./addslot.sh 10923 16383 192.168.10.12 8000
```
查看集群信息
```shell
127.0.0.1:8000> CLUSTER INFO
cluster_state:ok
cluster_slots_assigned:16384
cluster_slots_ok:16384
cluster_slots_pfail:0
cluster_slots_fail:0
cluster_known_nodes:6
cluster_size:3
cluster_current_epoch:0
cluster_my_epoch:0
cluster_stats_messages_sent:0
cluster_stats_messages_received:0
```
配置主从关系
```shell
cluster replicate <node_id> 将当前节点设置为 node_id 指定的节点的从节点
```
以node4举例
```
[root@master redis]# redis-cli -c -p 8001
127.0.0.1:8001> CLUSTER NODES
8b496ced472ff6913a079cb56bba8bd9cee12c61 192.168.10.10:8000 myself,master - 0 0 2 connected 0-5460
b1b77d1f042f7abe18dbe5f461c7ee0d4a001e7e 192.168.10.11:8000 master - 0 1491638444324 8 connected 5461-10992
eddca3c1b37b3574f511422b213fd2085f8e17a2 192.168.10.11:8001 master 
f91b42c1f481297d5347b7b518cca39ee14200ce 192.168.10.12:8000 master - 0 1491638441294 0 connected 10993-16383
f2be65c867349aaab162124174a2f1b1848bb4ab 192.168.10.12:8001 master 
3ca09eba65a2fa42275aa30d80999b59bd4e2085 192.168.10.10:8001 master 
127.0.0.1:8001> CLUSTER REPLICATE 8b496ced472ff6913a079cb56bba8bd9cee12c61
OK
```
node4的masterde的节点id是：8b496ced472ff6913a079cb56bba8bd9cee12c61   
所以是
```shell
CLUSTER REPLICATE 8b496ced472ff6913a079cb56bba8bd9cee12c61
```
在note5和node6节点上依次配置其master的节点id即可。


查看节点
```
127.0.0.1:8000> CLUSTER NODES
8b496ced472ff6913a079cb56bba8bd9cee12c61 192.168.10.10:8000 myself,master - 0 0 2 connected 0-5460
b1b77d1f042f7abe18dbe5f461c7ee0d4a001e7e 192.168.10.11:8000 master - 0 1491638444324 8 connected 5461-10992
eddca3c1b37b3574f511422b213fd2085f8e17a2 192.168.10.11:8001 slave b1b77d1f042f7abe18dbe5f461c7ee0d4a001e7e 0 1491638442304 8 connected
f91b42c1f481297d5347b7b518cca39ee14200ce 192.168.10.12:8000 master - 0 1491638441294 0 connected 10993-16383
f2be65c867349aaab162124174a2f1b1848bb4ab 192.168.10.12:8001 slave f91b42c1f481297d5347b7b518cca39ee14200ce 0 1491638440789 5 connected
3ca09eba65a2fa42275aa30d80999b59bd4e2085 192.168.10.10:8001 slave 8b496ced472ff6913a079cb56bba8bd9cee12c61 0 1491638443314 2 connected
```
确认分配槽状态
```
127.0.0.1:8000> CLUSTER SLOTS
1) 1) (integer) 0
   2) (integer) 5460
   3) 1) "192.168.10.10"
      2) (integer) 8000
      3) "8b496ced472ff6913a079cb56bba8bd9cee12c61"
   4) 1) "192.168.10.10"
      2) (integer) 8001
      3) "3ca09eba65a2fa42275aa30d80999b59bd4e2085"
2) 1) (integer) 5461
   2) (integer) 10992
   3) 1) "192.168.10.11"
      2) (integer) 8000
      3) "b1b77d1f042f7abe18dbe5f461c7ee0d4a001e7e"
   4) 1) "192.168.10.11"
      2) (integer) 8001
      3) "eddca3c1b37b3574f511422b213fd2085f8e17a2"
3) 1) (integer) 10993
   2) (integer) 16383
   3) 1) "192.168.10.12"
      2) (integer) 8000
      3) "f91b42c1f481297d5347b7b518cca39ee14200ce"
   4) 1) "192.168.10.12"
      2) (integer) 8001
      3) "f2be65c867349aaab162124174a2f1b1848bb4ab"
```

#### 5.2.4 测试
- 测试集群工作情况
- 模拟故障转移      

这两步测试省略。

### 5.3 官方工具搭建集群
机器需要支持Ruby环境，因为工具是使用Ruby写的。    
脚本名字：redis-trib.rb

## 6. 水平扩容

## 7. CacheCould
[CacheCloud文档归档](https://cachecloud.github.io/2016/04/12/CacheCloud%E6%96%87%E6%A1%A3%E5%BD%92%E6%A1%A3/)    
[github](https://github.com/sohutv/cachecloud/wiki)
