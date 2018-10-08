转载:http://blog.51cto.com/wlan2014/1898670


redis3.0支持集群的容错功能，集群搭建：至少需要三个master。



本文中，是在一台linux上配置的redis集群，大家也可以部署在多台linux服务器上。单个redis的安装在前面已讲解，本文将主要讲解集群的配置。



集群搭建步骤：



第一步：创建一个文件夹redis-cluster，然后再其下面分别创建6个文件夹如下：

 mkdir -p /usr/local/redis-cluster/7001 ...7006 （三个主节点，三个从节点）



第二步：把redis.conf文件分别copy到700*文件下，进行修改各个配置文件内容，也就是700*下的每一个redis.conf文件进行修改：



（1）daemonize yes（redis后台运行）

（2）port 700*（redis运行的端口号，在这里最好分别对每个机器的端口号进行设置）

（3）bind 192.168.1.124（指定只接收来自该ip地址的请求）

（4）dir /usr/local/redis-cluster/700*/（指定数据文件存放位置，必须要指定不同的目录位置，不然会丢失数据）



//配置启动集群以及集群配置文件的位置

（5）cluster-enabled yes（启动集群模式）

（6）cluster-config-file nodes-700*.conf（集群配置文件，在集群启动时，自动创建。这里最好和700*和端口号对应上）

（7）cluster-node-timeout 5000（集群超时时间，节点超时多久认为它宕机了）



//配置持久化的方式

（8）appendonly  yes（开启aof持久化模式，每次写操作请求都追加到appendonly.aof文件中）

（9）appendfsync  always（每次有写操作的时候都同步）



第三步：由于redis集群需要使用ruby命令，所以我们需要安装ruby，注意在redis的src目录下操作：

（1）yum install ruby

（2）yum install rubygems

（3）gem install redis（安装redis和ruby的接口）



第四步：分别启动6个启动6个redis服务实例，然后检查是否启动成功；

（1）[root@wlan124 ~]# /usr/local/redis/src/redis-server /usr/local/redis-cluster/700*/redis.conf   --执行六次

（2）[root@wlan124 ~]# ps -ef | grep redis





第五步：先到redis的安装目录下，然后执行redis-trib.rb命令。

（1）[root@wlan124 ~]# cd /usr/local/redis/src

（2）[root@wlan124 src]# ./redis-trib.rb create --replicas 1 192.168.1.124:7001  192.168.1.124:7002  192.168.1.124:7003 192.168.1.124:7004 192.168.1.124:7005 192.168.1.124:7006



到此为止集群搭建成功！



第六步：连接redis客户端，进行验证：



（1）连接任何一台客户端即可：redis-cli -c -h ip地址 -p 端口号

[root@wlan124 local]# /usr/local/redis/src/redis-cli -c -h 192.168.1.124 -p 7001



（2）进行验证：

cluster info（查看集群信息）

cluster nodes（查看节点列表）



第七步：关闭服务器集群则需要逐个进行关闭，使用命令：

[root@wlan124 local]# /usr/local/redis/src/redis-cli -c -h 192.168.1.124 -p 7006 shutdown



友情提示：

当出现集群无法启动时，删除集群配置文件，再次重新启动每一个redis服务，然后重新构件集群环境。





向集群中添加节点或删除节点



（1）我们新建俩个服务，按照之前搭建的集群方式新增俩个节点：（一主一从 master、slave）。

 Mater：7007 slave:7008



步骤一：创建7007/7008文件夹。拷贝redis.conf文件到对于的7007,7008目录下 ，再进行修改配置文件。



[root@bhz004 redis-cluster]# mkdir 7007

[root@bhz004 redis-cluster]# mkdir 7008

[root@bhz004 redis-cluster]# cd 7001

[root@bhz004 7001]# cp redis.conf /usr/local/redis-cluster/7007/

[root@bhz004 7001]# cp redis.conf /usr/local/redis-cluster/7008/





7007修改配置文件如下：

    port:7007

    dir /usr/local/redis-cluster/7007/

    cluster-config-file nodes-7007.conf



7008修改内容如下：

    port:7008

    dir /usr/local/redis-cluster/7008/

    cluster-config-file nodes-7008.conf





步骤二：启动7007和7008俩个服务并查看服务状态。

[root@bhz004 7001]# /usr/local/redis/src/redis-server /usr/local/redis-cluster/7007/redis.conf

[root@bhz004 7001]# /usr/local/redis/src/redis-server /usr/local/redis-cluster/7008/redis.conf

[root@bhz004 7001]# ps -el | grep redis





redis-trib.rb命令：

1 create：创建一个集群环境host1:port1 ... hostN:portN（集群中的主从节点比例）

2 call：可以执行redis命令

3 add-node：将一个节点添加到集群里，第一个参数为新节点的ip:port，第二个参数为集群中任意一个已经存在的节点的ip:port

4 del-node：移除一个节点


5 reshard：重新分片

6 check：检查集群状态





集群中添加一个主节点：



步骤一：使用add-node命令：绿色为新增节点，红色为已知存在节点

[root@wlan124 local]# /usr/local/redis/src/redis-trib.rb add-node 192.168.1.124:7007 192.168.1.124:7001



注意：当添加节点成功以后，新增的节点不会有任何数据，因为他没有分配任何slot。需要为新节点手动分配slot。





步骤二：reshard命令，分配slot：

[root@wlan124 local]# /usr/local/redis-3.0.0/src/redis-trib.rb reshard 192.168.1.124:7007

（提示一）

How many slots do you want to move (from 1 to 16384)? 2000

（提示二）

What is the receiving node ID? 382634a4025778c040b7213453fd42a709f79e28

Please enter all the source node IDs.

  Type 'all' to use all the nodes as source nodes for the hash slots.

  Type 'done' once you entered all the source nodes IDs.

Source node #1:all

（提示三）

Do you want to proceed with the proposed reshard plan (yes/no)? yes





1提示一：是希望你需要多少个槽移动到新的节点上，可以自己设置，比如200个槽。

2提示二：是你需要把这200个slot槽移动到那个节点上去（需要指定节点id），并且下个提示是输入all为从所有主节点（7001 7002 7003）中分别抽取相应的槽数（一共为200个槽到指定的新节点中！，并且会打印执行分片的计划。）

3提示三：输入yes确认开始执行分片任务





集群中添加一个从节点：



步骤一：使用add-node命令：绿色为新增节点，红色为已知存在节点

[root@wlan124 local]# /usr/local/redis/src/redis-trib.rb add-node 192.168.1.124:7008  192.168.1.124:7001



步骤二：首先需要登录到7008节点的客户端，然后使用集群命令，执行replicate命令来指定当前节点的主节点id为哪一个。把当前的7008（slave）节点指定到一个主节点下（这里使用之前创建的7007主节点，绿色表示节点id）。

[root@wlan124 ~]# /usr/local/redis/bin/redis-cli -c -h 192.168.1.124 -p 7008

192.168.1.124:7008> cluster replicate 4d4cb840519eef342a5730168b6c7e14dd811542

（7007的id）

OK



补充：集群操作

//集群(cluster)

CLUSTER INFO 打印集群的信息

CLUSTER NODES 列出集群当前已知的所有节点（node），以及这些节点的相关信息。

//节点(node)

CLUSTER MEET <ip> <port> 将 ip 和 port 所指定的节点添加到集群当中，让它成为集群的一份子。

CLUSTER FORGET <node_id> 从集群中移除 node_id 指定的节点。

CLUSTER REPLICATE <node_id> 将当前节点设置为 node_id 指定的节点的从节点。

CLUSTER SAVECONFIG 将节点的配置文件保存到硬盘里面。

//槽(slot)

CLUSTER ADDSLOTS <slot> [slot ...] 将一个或多个槽（slot）指派（assign）给当前节点。

CLUSTER DELSLOTS <slot> [slot ...] 移除一个或多个槽对当前节点的指派。

CLUSTER FLUSHSLOTS 移除指派给当前节点的所有槽，让当前节点变成一个没有指派任何槽的节点。

CLUSTER SETSLOT <slot> NODE <node_id> 将槽 slot 指派给 node_id 指定的节点，如果槽已经指派给另一个节点，那么先让另一个节点删除该槽>，然后再进行指派。

CLUSTER SETSLOT <slot> MIGRATING <node_id> 将本节点的槽 slot 迁移到 node_id 指定的节点中。

CLUSTER SETSLOT <slot> IMPORTING <node_id> 从 node_id 指定的节点中导入槽 slot 到本节点。

CLUSTER SETSLOT <slot> STABLE 取消对槽 slot 的导入（import）或者迁移（migrate）。

//键 (key)

CLUSTER KEYSLOT <key> 计算键 key 应该被放置在哪个槽上。

CLUSTER COUNTKEYSINSLOT <slot> 返回槽 slot 目前包含的键值对数量。

CLUSTER GETKEYSINSLOT <slot> <count> 返回 count 个 slot 槽中的键。





集群中删除一个从节点：



步骤一：删除从节点7008，输入del-node命令，指定删除节点ip和端口，以及节点id（蓝色为7008节点id），移除了7008 slave节点，前节点的服务进程也会随之销毁。



[root@wlan124 ~]# /usr/local/redis-3.0.0/src/redis-trib.rb del-node 192.168.1.124:7008 a78c8a41f6430b51a7eca1fdb50092c463a8f1ac





集群中删除一个主节点：



    如果主节点有从节点，将从节点转移到其他主节点。如果主节点有slot，先将主节点里的slot分配到其他可用节点中，然后再删除节点才行，否则会有数据的丢失。



步骤一：删除7007（master）节点之前，我们需要先把其全部的数据（slot槽）移动到其他节点上去（目前只能把master的数据迁移到一个节点上，暂时做不了平均分配功能）。

[root@wlan124 ~]# /usr/local/redis-3.0.0/src/redis-trib.rb reshard 192.168.1.124:7007

How many slots do you want to move (from 1 to 16384)? 1999

（注释：这里不会是正好200个槽）

What is the receiving node ID? 614d0def75663f2620b6402a017014b57c912dad

（注释：这里是需要把数据移动到哪？7001的主节点id）

Please enter all the source node IDs.

  Type 'all' to use all the nodes as source nodes for the hash slots.

  Type 'done' once you entered all the source nodes IDs.

Source node #1:4d4cb840519eef342a5730168b6c7e14dd811542

（注释：这里是需要数据源，也就是我们的7007节点id）

Source node #2:done

（注释：这里直接输入done 开始生成迁移计划）

Do you want to proceed with the proposed reshard plan (yes/no)? yes

（注释：这里输入yes开始迁移）





步骤二：最后我们直接使用del-node命令删除7007主节点即可（蓝色表示7007的节点id）。

[root@wlan124 ~]# /usr/local/redis-3.0.0/src/redis-trib.rb del-node 192.168.1.124:7007 4d4cb840519eef342a5730168b6c7e14dd811542
