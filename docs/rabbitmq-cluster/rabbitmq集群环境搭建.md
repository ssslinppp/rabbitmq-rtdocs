## 介绍
RabbitMQ集群对延时非常敏感，应当只在本地局域网内使用。在广域网中不应该使用集群，而应该使用Federation或者Shovel来代替。

---

## 安装rabbitmq-server
总共有3台虚拟机，都安装有rabbitmq服务，安装过程可参考：
[【rabbitmq】Centos7 下安装rabbitmq](https://www.cnblogs.com/ssslinppp/p/7707336.html)   

---

### 创建用户和vhost
说明： 此步骤不是必须的，文章后面的用户和vhost可能与此步骤创建的不一致，此处仅仅是创建的示例。


```
rabbitmqctl add_vhost /my_vhost

rabbitmqctl add_user admin admin
rabbitmqctl set_user_tags admin administrator #管理员权限

# rabbitmqctl set_permissions [-p vhost] [user] [permission ⇒ (modify) (write) (read)]
rabbitmqctl set_permissions -p /my_vhost admin ".*"  ".*" ".*"

rabbitmqctl list_permissions -p /my_vhost
```

---

### 机器列表

机器 | ip |/etc/hostname| /etc/hostname|说明
---|---| ---|---|---
node1 | 192.168.35.135|rmq-node1|rmq-node1|作为首节点
node2 | 192.168.35.136|rmq-node2|rmq-node2|加入node1集群
node3 | 192.168.35.137|rmq-node3|rmq-node3|加入node1集群

注意：修改`/etc/hostname`为上述表格中的值
```
# node1执行
[root@rmq-node1 ~]# rabbitmqctl status
Status of node 'rabbit@rmq-node1'
...

# node2执行
[root@rmq-node2 ~]#  rabbitmqctl status
Status of node 'rabbit@rmq-node2'
...

# node3执行
[root@rmq-node3 ~]#  rabbitmqctl status
Status of node 'rabbit@rmq-node3'
...

```


---

## 搭建集群环境
### 保证magic cookie相同
rabbitmq使用erlang编写，erlang天然具有分布式特性，为了是各erlang节点能够互相通信，需要保证 `magic cookie`相同；  

将node1中的cookie复制到node2和node3
```
# node2和 node3上分别执行
scp root@192.168.35.135:/var/lib/rabbitmq/.erlang.cookie /var/lib/rabbitmq/.erlang.cookie
```
现在3个node上的`.erlang.cookie`都是相同的；

**注意点**： 
1. 需要保证`.erlang.cookie`的默认权限为：`400`;   
2. 所有用户和组为 `rabbitmq`

```
chmod 400 /var/lib/rabbitmq/.erlang.cookie
chown rabbitmq:rabbitmq /var/lib/rabbitmq/.erlang.cookie
```

---

### 修改hosts文件
```
# 3个节点都要修改 hosts文件
vi /etc/hosts 

# 添加
192.168.35.135 rmq-node1
192.168.35.136 rmq-node2
192.168.35.137 rmq-node3
```

在每个节点上，都保证能ping通
```
ping rmq-node1
ping rmq-node2
ping rmq-node3
```

---
### 各节点启动rabbitmq服务
```
node1: rabbitmq-server -detached (systemctl restart rabbitmq-server)
node2: rabbitmq-server -detached (systemctl restart rabbitmq-server)
node3: rabbitmq-server -detached (systemctl restart rabbitmq-server)
```
这样，3个node目前都是以**独立节点存在的单个集群**。   

查看各节点的集群状态（示例仅演示了node1，其他2个node类似）
```
[root@rmq-node1 ~]# rabbitmqctl  cluster_status
Cluster status of node 'rabbit@rmq-node1'
[{nodes,[{disc,['rabbit@rmq-node1']}]},   //此时只有一个节点
 {running_nodes,['rabbit@rmq-node1']},
 {cluster_name,<<"rabbit@rmq-node1">>},
 {partitions,[]},
 {alarms,[{'rabbit@rmq-node1',[]}]}]
```


### 将node2节点和node3节点加入集群
为了将3个node组成一个集群，需要以node1节点为基准，将node2和node3节点加入node1节点的集群中。   
这3个节点是平等的，如果想调换彼此的加入顺序，也未尝不可。

#### node2和node3加入集群前，node1的状态
```
[root@rmq-node1 ~]# rabbitmqctl  cluster_status
Cluster status of node 'rabbit@rmq-node1'
[{nodes,[{disc,['rabbit@rmq-node1']}]},   //此时只有一个节点
 {running_nodes,['rabbit@rmq-node1']},
 {cluster_name,<<"rabbit@rmq-node1">>},
 {partitions,[]},
 {alarms,[{'rabbit@rmq-node1',[]}]}]
```
说明：上面的`rabbit@rmq-node1`就是后面node2和node3要加入的首节点信息

#### node2作为disk节点
未加入之前
```
[root@rmq-node2 ~]# rabbitmqctl cluster_status
Cluster status of node 'rabbit@rmq-node2'
[{nodes,[{disc,['rabbit@rmq-node2']}]},
 {running_nodes,['rabbit@rmq-node2']},
 {cluster_name,<<"rabbit@rmq-node2">>},
 {partitions,[]},
 {alarms,[{'rabbit@rmq-node2',[]}]}]

```

加入集群（4个命令）
```
# stop application and reset
rabbitmqctl stop_app 
rabbitmqctl reset 

#join in cluster (specify only hostname, not with FQDN) 
rabbitmqctl join_cluster rabbit@rmq-node1

# start application
rabbitmqctl start_app

# show status
rabbitmqctl cluster_status
```

查看状态：
```
[root@rmq-node1 ~]# rabbitmqctl  cluster_status
Cluster status of node 'rabbit@rmq-node1'
[{nodes,[{disc,['rabbit@rmq-node1','rabbit@rmq-node2']}]},   //此时node2已经加入集群
 {running_nodes,['rabbit@rmq-node2','rabbit@rmq-node1']},
 {cluster_name,<<"rabbit@rmq-node1">>},
 {partitions,[]},
 {alarms,[{'rabbit@rmq-node2',[]},{'rabbit@rmq-node1',[]}]}]


[root@rmq-node2 ~]#  rabbitmqctl  cluster_status
Cluster status of node 'rabbit@rmq-node2'
[{nodes,[{disc,['rabbit@rmq-node1','rabbit@rmq-node2']}]},
 {running_nodes,['rabbit@rmq-node1','rabbit@rmq-node2']},
 {cluster_name,<<"rabbit@rmq-node1">>},
 {partitions,[]},
 {alarms,[{'rabbit@rmq-node1',[]},{'rabbit@rmq-node2',[]}]}]
```

#### node3作为ram节点
未加入之前
```
[root@rmq-node3 ~]# rabbitmqctl cluster_status
Cluster status of node 'rabbit@rmq-node3'
[{nodes,[{disc,['rabbit@rmq-node3']}]},
 {running_nodes,['rabbit@rmq-node3']},
 {cluster_name,<<"rabbit@rmq-node3">>},
 {partitions,[]},
 {alarms,[{'rabbit@rmq-node3',[]}]}]

```

加入集群（4个步骤）
```
# stop application and reset
rabbitmqctl stop_app 
rabbitmqctl reset 

#join in cluster (specify only hostname, not with FQDN) 
rabbitmqctl join_cluster rabbit@rmq-node1 --ram

# start application
rabbitmqctl start_app

# show status
rabbitmqctl cluster_status
```

加入集群后的状态
```
[root@rmq-node1 ~]# rabbitmqctl  cluster_status
Cluster status of node 'rabbit@rmq-node1'
[{nodes,[{disc,['rabbit@rmq-node1','rabbit@rmq-node2']},
         {ram,['rabbit@rmq-node3']}]},      //此时node3已经加入集群
 {running_nodes,['rabbit@rmq-node3','rabbit@rmq-node2','rabbit@rmq-node1']},
 {cluster_name,<<"rabbit@rmq-node1">>},
 {partitions,[]},
 {alarms,[{'rabbit@rmq-node3',[]},
          {'rabbit@rmq-node2',[]},
          {'rabbit@rmq-node1',[]}]}]

-------------------
[root@rmq-node2 ~]# rabbitmqctl cluster_status
Cluster status of node 'rabbit@rmq-node2'
[{nodes,[{disc,['rabbit@rmq-node1','rabbit@rmq-node2']},
         {ram,['rabbit@rmq-node3']}]},
 {running_nodes,['rabbit@rmq-node3','rabbit@rmq-node1','rabbit@rmq-node2']},
 {cluster_name,<<"rabbit@rmq-node1">>},
 {partitions,[]},
 {alarms,[{'rabbit@rmq-node3',[]},
          {'rabbit@rmq-node1',[]},
          {'rabbit@rmq-node2',[]}]}]

-----------------------------

[root@rmq-node3 ~]# rabbitmqctl cluster_status
Cluster status of node 'rabbit@rmq-node3'
[{nodes,[{disc,['rabbit@rmq-node2','rabbit@rmq-node1']},
         {ram,['rabbit@rmq-node3']}]},
 {running_nodes,['rabbit@rmq-node1','rabbit@rmq-node2','rabbit@rmq-node3']},
 {cluster_name,<<"rabbit@rmq-node1">>},
 {partitions,[]},
 {alarms,[{'rabbit@rmq-node1',[]},
          {'rabbit@rmq-node2',[]},
          {'rabbit@rmq-node3',[]}]}]

```

---

## 登录界面查看集群状态
http://192.168.35.135:15672/#/

---

# 集群节点关闭的处理


---
## 相关概念
### epmd
erlang port monitor deamon： 负责集群中节点ip和port的监控；    
集群间node的发现都是通过该后台程序来完成的；


### Mnesia
这是erlang的**分布式数据库**，用于存储queue、exchange、binding等信息；

---

### 网络分区（network partitions）
可参考： https://www.cnblogs.com/ssslinppp/p/9470962.html  

---

## 其他运维操作
### 日志文件
```
/var/log/rabbitmq/xxxx.log
```

### 将某个node从集群中删除
```
rabbitmqctl forget_cluster_node rabbit@node-1  //这个属于硬删除
```
将node1节点删除后，若是重新加入该集群，可能报错，如：
```
inconsistent_cluster,"Node 'rabbit@node-1' thinks it's clustered with node 'rabbit@node-3', but 'rabbit@node-3' disagrees"}}
```
**原因：**   
从node-1的启动报错来看，像是集群信息残留；   
在node-2上操作将node-1移除集群，node-1的rabbitmq服务已经down掉了，所以数据库无法同步更新，记载的仍是旧的集群信息(数据库记录里自身节点仍属于集群)    
而node-2和node-3的数据库记录已经更新(数据库信息里面集群不包含node-1节点了)   
**解决方式**   
删除node-1的数据库记录
```
rm -rf /var/lib/rabbitmq/mnesia/*
```


---

# 出现的错误
### 1. 启动失败：rabbitmq-server.service: control process exited, code=exited status=75

```
journalctl -xe

# 错误信息如下
Aug 08 19:25:44 localhost.localdomain systemd[1]: rabbitmq-server.service: control process exited, code=exited status=75
Aug 08 19:25:44 localhost.localdomain systemd[1]: Failed to start RabbitMQ broker.
-- Subject: Unit rabbitmq-server.service has failed
-- Defined-By: systemd
-- Support: http://lists.freedesktop.org/mailman/listinfo/systemd-devel
-- 
-- Unit rabbitmq-server.service has failed.
-- 
-- The result is failed.
Aug 08 19:25:44 localhost.localdomain systemd[1]: Unit rabbitmq-server.service entered failed state.
Aug 08 19:25:44 localhost.localdomain systemd[1]: rabbitmq-server.service failed.
Aug 08 19:25:44 localhost.localdomain polkitd[902]: Unregistered Authentication Agent for unix-process:3504:33878 (system bus name :1.17, object path /org/freedesktop/PolicyKit1/Aut
[root@localhost mnesia]# journalctl -xe

```
解决方式：
```
chown rabbitmq:rabbitmq /var/lib/rabbitmq/.erlang.cookie
rm -rf /var/lib/rabbitmq/mnesia/*
```







