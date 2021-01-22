rabbitmq-3.7.7安装

由于RabbitMQ依赖Erlang， 所以需要先安装Erlang。
```
yum install -y epel-release 
yum install -y erlang 
```	
yum源

文件内容
```
[rabbitmq-erlang]
name=rabbitmq-erlang
baseurl=https://dl.bintray.com/rabbitmq/rpm/erlang/21/el/7
gpgcheck=1
gpgkey=https://dl.bintray.com/rabbitmq/Keys/rabbitmq-release-signing-key.asc
repo_gpgcheck=0
enabled=1
```

yum安装

```
yum install erlang
wget https://github.com/rabbitmq/rabbitmq-server/releases/download/v3.7.7/rabbitmq-server-3.7.7-1.el7.noarch.rpm  (如地址无效请去逛网下载)
yum install rabbitmq-server-3.7.7-1.el7.noarch.rpm
```

RabbitMQ服务
2.1RabbitMQ简介：
RabbitMQ是一个消息代理。它的工作就是接收和转发消息。你可以把它想像成一个邮局：你把信件放入邮箱，邮递员就会把信件投递到你的收件人处。在这个比喻中，RabbitMQ就扮演着邮箱、邮局以及邮递员的角色。

2.2消息队列安装与操作
Rabbitmq安装请查看rabbitmq安装.doc
基础命令：
```
systemctl enable  rabbitmq-server  		# 添加开机启动RabbitMQ服务
systemctl start  rabbitmq-server  			# 启动服务
rabbitmqctl list_users 				#查看当前所有用户
rabbitmqctl list_user_permissions guest		#查看默认guest用户的权限
rabbitmqctl delete_user guest	 		#删除默认用户
rabbitmqctl add_user username password		#添加新用户
rabbitmqctl set_user_tags username administrator	   #设置用户tag
rabbitmqctl set_permissions -p / username ".*" ".*" ".*"  #赋予用户默认vhost的全部操作权限
rabbitmqctl list_user_permissions username		   #查看用户的权限
rabbitmq-plugins enable rabbitmq_management	#开启web管理接口
```


2.3消息队列集群和队列镜像持久化
镜像集群搭建

将主节点上的/var/lib/rabbitmq/.erlang.cookie中的内容复制到备节点上的/var/lib/rabbitmq/.erlang.cookie文件中,

即三台服务器必须具有相同的cookie，如果不相同的话，否则无法搭建集群。

备节点上分别执行命令，加入到集群
```
　　　　rabbitmqctl stop_app
　　　　rabbitmqctl join_cluster  rabbit@h-ncdrdcs7
　　　　rabbitmqctl start_app
```

其中--ram代表是内存节点，如果希望是磁盘节点则不用加--ram，在rabbitmq集群中，至少需要一个磁盘节点

```
rabbitmqctl reset 	#删除节点
```

硬删除：
直接删掉集群中的某个节点：
```
rabbitmqctl forget_cluster_node   node_name
```
查看集群的状态
```
rabbitmqctl cluster_status
```

如果报错无法启动：在没有数据的情况下删除/var/lib/rabbitmq/mnesia/*即可，此操作将初始化rabbitmq。

设置成镜像队列持久化

RabbitMQ的镜像队列机制，将queue镜像到cluster中其他的节点之上。在该实现下，如果集群中的一个节点失效了，queue能自动地切换到镜像中的另一个节点以保证服务的可用性。

```
rabbitmqctl set_policy ha-all "^ha\." '{"ha-mode":"all"}' //意思表示以ha.开头的queue都会复制到各个节点 ["^"匹配所有]
rabbitmqctl set_policy -p a1name(vhost名)  ha-allqueue（这个策略的名字） "^" '{"ha-mode":"all"}'
```

在web上面设置
```
ha-mode：all 表示镜像模式
ha-sync-mode: automatic 表示自动同步
```
参考文档

https://www.cnblogs.com/lylife/p/5584019.html

https://blog.csdn.net/qq_35246620/article/details/72473098

2.4消息队列可用性检测与脚本恢复

搭建好集群后，这里举例2台rabbitmq服务器，1台主（disk节点）一台备（ram节点）。#建议不要做ram节点，有坑。


当rabbitmq备节点可用性不可用时（keepalived默认vip漂移在备节点），脚本检测会将备节点keepalived服务器断开，并重启备节点rabbitmq服务，重启过程中vip会漂移到主节点，避免影响业务运行，当备节点rabbitmq服务恢复时，启动keepalived服务，这时候keepalived会重主节点漂移到备节点。

注意：当主节点宕机时，备节点也将失去作用。

详细脚本可参考服务器中脚本（定时任务中查看执行位置）

修改最大连接数

```
vi /usr/lib/systemd/system/rabbitmq-server.service

[Service]
LimitNOFILE=16384
```
