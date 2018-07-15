## 服务编排之Mesos

```/bin/bash
Program against your datacenter like it`s a single pool of resources
在数据中心上运行程序就像运行单个资源池一样
mesos抽象除了cpu内存资源等，从而抽象了从机器里面具有容错的可伸缩的分布式系统变得可以简单的来构建和有效的运行
被称之为分布式系统内核
```

#### mesos是如何让twitter摆脱失败鲸的问题呢？就看下面的架构

![Mesos架构图](D:\Picture\Mesos架构图.png)

```shell
上面的master是第一层调度，支持高可用集群的，管理所有的slave选举。slave运行在虚拟机之上Mesos slave运行具体的任务如hadoop ：这就是第一层调度

每个slave启动的时候都会注册在Mesos master ，master协调全部的slave，并且确定每个slave的可用资源



第二级调度：就是framework的组件组成，上面的虚线。framework分为调度器和执行器。上面的虚线是调度器上面画出了scheduler。mesos可以跟多种类型framework进行通讯，只是上面画出了hadoop和MPI，下面的虚线就是执行器的部分executor

```

![Mesos的资源分配](D:\Picture\Mesos的资源分配.png)

```shell
从上个图知道，slave是运行在物理机或者虚拟机上的mesos进程是mesos集群的部分，framework由调度器和执行器组成，被注册在mesos，已使用mesos中的资源



这个图，slave1向master汇报空闲资源4个cpu4个g的内存。第二步骤，由master触发的分配策略模块，得到的反馈是framework1要求请求所有的可用资源，然后master向framework1发送可用的资源邀约，描述了slave1上的所有的可用资源，第三步骤，framework的调度器跟master说在slave1上运行两个人物，第一个任务需要2个cpu和2g的内存，第二个任务需要1个cpu和2g的内存，第四步master向slave1下发任务，并且分配资源给framework的执行器，接下来由执行器来执行两个任务，也就是虚线的executor，然后slave1还有1个cpu和1g的内存，所以还可以继续分配任务

framework的调度器就是和master来谈判资源的，要运行程序看看资源够不够
framework也可以拒绝资源邀约

为了实现在一个slave上运行多个task，所以使用了进程的隔离机制
```

 mesos并不能单独的存在，必须需要一个framework来跟他协作

# Marathon

Marathon适合运行长期存在的服务。

![Mesos的Marathon](D:\Picture\Mesos的Marathon.png)

Mesos相当于linux的内核，marathon相当于linux内核的外壳管理程序，Mesos不止是一台机器的内核，是成千上万的内核，是分布式内核

![marathon是怎么运行的](D:\Picture\marathon是怎么运行的.png)

```
上面的图有两个流程
1、资源邀约并且运行任务的过程：调度器和执行器的工作过程
2、从客户端访问web服务到相应的过程

marathon就相当于所有运行的服务的注册中心
Marathon-lb就相当于nginx


```

Mesos特征：

​	强大的资源管理

​	kernel和framwwork分离

marathon特征：

​	高可用

​	Constraints（可以打标签）

​	服务发现&负载均衡

​	健康检查（HTTP/TCP/SHELL）

​	事件订阅

​	完善的REST API

# 构建环境

Mesos Master是通过zookeeper来进行高可用的。

![mesos调度手画图](D:\Picture\mesos调度手画图.png)

```
版本环境：
[root@bogon ~]# cat /etc/redhat-release 
CentOS Linux release 7.5.1804 (Core) 
克隆四台 1 2 3 4
ip分别为
130/132/131/133


```

# zookeeper安装 133安装

```shell
安装jdk
tar -xf jdk-8u131-linux-x64.tar.gz
配置java_home
sudo vi /etc/profile
	export JAVA_HOME=/root/jdk
	export PATH=$PATH:$JAVA_HOME/bin
source /etc/profile

下载zookeeper：http://archive.apache.org/dist/zookeeper/
tar xf zookeeper-3.4.10.tar.gz 
mv zookeeper-3.4.10 /root/zookeeper
cd zookeeper/
cd conf/
mv zoo_sample.cfg zoo.cfg
修改文件
vim zoo.cfg
    tickTime=2000
    dataDir=/root/zooData
    dataLogDir=/root/zooLogs
    clientPort=2181
zookeeper的各种操作
    ./zkServer.sh start
    ./zkServer.sh stop
    ./zkServer.sh restart
    ./zkServer.sh status
集群模式看下面链接
https://www.cnblogs.com/lsdb/p/7297731.html
```

这里02也就是131是master

```
docker run -d --net=host \
  -e MESOS_PORT=5050 \
  -e MESOS_ZK=zk://127.0.0.1:2181/mesos \
  -e MESOS_QUORUM=1 \
  -e MESOS_REGISTRY=in_memory \
  -e MESOS_LOG_DIR=/var/log/mesos \
  -e MESOS_WORK_DIR=/var/tmp/mesos \
  -v "$(pwd)/log/mesos:/var/log/mesos" \
  -v "$(pwd)/tmp/mesos:/var/tmp/mesos" \
  mesosphere/mesos-master:
  
  解释：
  上面第一行必须是host
  第三行是zookeeper的地址
  第四行是master是单节点的话就设置为1，几个节点n/2+1
  最后两行是把log和work目录挂载在哪里
  最最后一行是版本号
```

关闭防火墙

systemctl stop firewalld.service

关闭开机启动

systemctl disable firewalld.service

这里的是

![mesos脚本](D:\Picture\mesos脚本.png)

之后执行sh mesos.sh

然后访问

http://192.168.154.132:5050/#/

![mesos的web界面](D:\Picture\mesos的web界面.png)

frameworks是第二级调度

agents是查看有多少的agent slave在运行

Roles 角色

offers资源邀约，给了那个frameworks可以在这里看到

Maintenance

```shell
然后再修改脚本
[root@bogon ~]# vim mesos.sh 

#!/bin/bash
docker run -d --net=host \
  --hostname=192.168.154.132 \
  -e MESOS_PORT=5050 \
  -e MESOS_ZK=zk://192.168.154.133:2181/mesos \
  -e MESOS_QUORUM=1 \
  -e MESOS_REGISTRY=in_memory \
  -e MESOS_LOG_DIR=/var/log/mesos \
  -e MESOS_WORK_DIR=/var/tmp/mesos \
  -v "$(pwd)/log/mesos:/var/log/mesos" \
  -v "$(pwd)/work/mesos:/var/tmp/mesos" \
  mesosphere/mesos-master:1.5.1-rc1
这里增加了--hostname 这里指定了之后就是告诉zookeeper要master是这个ip，这样slave链接的时候会根据zookeeper告诉的这个ip地址来进行连接

执行脚本
[root@bogon ~]# sh mesos.sh 
0b220dfe8b88673241af28f4264621b36c5f61df93ec9ae6a84c8db530643486
[root@bogon ~]# docker logs -f 0b22
...
09777-bc8d-4a58-ae16-fe62e9dc4ed0
I0714 23:56:32.023517    11 detector.cpp:152] Detected a new leader: (id='2')
I0714 23:56:32.023674    11 group.cpp:700] Trying to get '/mesos/json.info_0000000002' in ZooKeeper

//下面这一行upid可以看到已经指定了mesos的master的这个ip地址

I0714 23:56:32.024917    11 zookeeper.cpp:262] A new leading master (UPID=master@192.168.154.132:5050) is detected
I0714 23:56:32.025014    11 master.cpp:2210] Elected as the leading master!
I0714 23:56:32.025049    11 master.cpp:1690] Recovering from registrar
I0714 23:56:32.026677    11 registrar.cpp:391] Successfully fetched the registry (0B) in 1.530112ms
I0714 23:56:32.026835    11 registrar.cpp:495] Applied 1 operations in 76633ns; attempting to update the registry
I0714 23:56:32.027107    11 registrar.cpp:552] Successfully updated the registry in 218880ns
I0714 23:56:32.027161    11 registrar.cpp:424] Successfully recovered registrar
I0714 23:56:32.027320    11 master.cpp:1803] Recovered 0 agents from the registry (127B); allowing 10mins for agents to re-register

```





# 1和3中配置slave

```
1、首先登录 docker login
2、docker pull mesosphere/mesos-master:1.5.1-rc1

```

下面是192.168.154.130中安装的slave

第五行是使用什么容器，是自带的还是docker

倒数第二行是使用命令，这里首先要用which docker查看docker的位置所在

第一行中的-d表示后台运行

![mesos-slave](D:\Picture\mesos-slave.png)

```shell
1、启动脚本 sh mesos-slave.sh
2、查看日志  这里的查看日志方式d0 就是启动脚本生成的前面的缩写
[root@bogon ~]# sh mesos-slave.sh 

d0cb120a119dce9bf412b3fd89d1861155076064bd954080adfe159fd5b25ab1
[root@bogon ~]# docker logs -f d0  
I0714 23:59:38.898598     1 logging.cpp:201] INFO level logging started!
I0714 23:59:38.899443     1 main.cpp:365] Build: 2018-05-11 19:55:01 by ubuntu
I0714 23:59:38.899497     1 main.cpp:366] Version: 1.5.1
I0714 23:59:38.899521     1 main.cpp:369] Git tag: 1.5.1-rc1
I0714 23:59:38.899528     1 main.cpp:373] Git SHA: 1b27db1c406b3460a252aea2ee7c75f3a430e50c
I0714 23:59:39.105063     1 resolver.cpp:69] Creating default secret resolver
2018-07-14 23:59:39,595:1(0x7fd68ba3a700):ZOO_INFO@log_env@753: Client environment:zookeeper.version=zookeeper C client 3.4.8
2018-07-14 23:59:39,595:1(0x7fd68ba3a700):ZOO_INFO@log_env@757: Client environment:host.name=localhost.localdomain
2018-07-14 23:59:39,595:1(0x7fd68ba3a700):ZOO_INFO@log_env@764: Client environment:os.name=Linux
2018-07-14 23:59:39,595:1(0x7fd68ba3a700):ZOO_INFO@log_env@765: Client environment:os.arch=3.10.0-862.6.3.el7.x86_64
2018-07-14 23:59:39,595:1(0x7fd68ba3a700):ZOO_INFO@log_env@766: Client environment:os.version=#1 SMP Tue Jun 26 16:32:21 UTC 2018
2018-07-14 23:59:39,597:1(0x7fd68ba3a700):ZOO_INFO@log_env@774: Client environment:user.name=(null)
2018-07-14 23:59:39,597:1(0x7fd68ba3a700):ZOO_INFO@log_env@782: Client environment:user.home=/root
2018-07-14 23:59:39,597:1(0x7fd68ba3a700):ZOO_INFO@log_env@794: Client environment:user.dir=/
2018-07-14 23:59:39,597:1(0x7fd68ba3a700):ZOO_INFO@zookeeper_init@827: Initiating client connection, host=192.168.154.133:2181 sessionTimeout=10000 watcher=0x7fd69526d7e0 sessionId=0 sessionPasswd=<null> context=0x7fd674000ca8 flags=0
2018-07-14 23:59:39,610:1(0x7fd6869f1700):ZOO_INFO@check_events@1764: initiated connection to server [192.168.154.133:2181]
I0714 23:59:39.610692     1 slave.cpp:262] Mesos agent started on (1)@127.0.0.1:5051
I0714 23:59:39.610863     1 slave.cpp:263] Flags at startup: --appc_simple_discovery_uri_prefix="http://" --appc_store_dir="/tmp/mesos/store/appc" --authenticate_http_executors="false" --authenticate_http_readonly="false" --authenticate_http_readwrite="false" --authenticatee="crammd5" --authentication_backoff_factor="1secs" --authorizer="local" --cgroups_cpu_enable_pids_and_tids_count="false" --cgroups_enable_cfs="false" --cgroups_hierarchy="/sys/fs/cgroup" --cgroups_limit_swap="false" --cgroups_root="mesos" --container_disk_watch_interval="15secs" --containerizers="docker" --default_role="*" --disallow_sharing_agent_pid_namespace="false" --disk_watch_interval="1mins" --docker="docker" --docker_kill_orphans="true" --docker_registry="https://registry-1.docker.io" --docker_remove_delay="6hrs" --docker_socket="/var/run/docker.sock" --docker_stop_timeout="0ns" --docker_store_dir="/tmp/mesos/store/docker" --docker_volume_checkpoint_dir="/var/run/mesos/isolators/docker/volume" --enforce_container_disk_quota="false" --executor_registration_timeout="1mins" --executor_reregistration_timeout="2secs" --executor_shutdown_grace_period="5secs" --fetcher_cache_dir="/tmp/mesos/fetch" --fetcher_cache_size="2GB" --frameworks_home="" --gc_delay="1weeks" --gc_disk_headroom="0.1" --hadoop_home="" --help="false" --hostname_lookup="true" --http_command_executor="false" --http_heartbeat_interval="30secs" --initialize_driver_logging="true" --isolation="posix/cpu,posix/mem" --launcher="linux" --launcher_dir="/usr/libexec/mesos" --log_dir="/var/log/mesos" --logbufsecs="0" --logging_level="INFO" --master="zk://192.168.154.133:2181/mesos" --max_completed_executors_per_framework="150" --oversubscribed_resources_interval="15secs" --perf_duration="10secs" --perf_interval="1mins" --port="5051" --qos_correction_interval_min="0ns" --quiet="false" --reconfiguration_policy="equal" --recover="reconnect" --recovery_timeout="15mins" --registration_backoff_factor="1secs" --revocable_cpu_low_priority="true" --runtime_dir="/var/run/mesos" --sandbox_directory="/mnt/mesos/sandbox" --strict="true" --switch_user="false" --systemd_enable_support="false" --systemd_runtime_directory="/run/systemd/system" --version="false" --work_dir="/var/tmp/mesos" --zk_session_timeout="10secs"
W0714 23:59:39.612087     1 slave.cpp:266] 
**************************************************
Agent bound to loopback interface! Cannot communicate with remote master(s). You might want to set '--ip' flag to a routable IP address.
**************************************************
2018-07-14 23:59:39,621:1(0x7fd6869f1700):ZOO_INFO@check_events@1811: session establishment complete on server [192.168.154.133:2181], sessionId=0x1649b2d85d70004, negotiated timeout=10000
I0714 23:59:39.621760     7 group.cpp:341] Group process (zookeeper-group(1)@127.0.0.1:5051) connected to ZooKeeper
I0714 23:59:39.621984     7 group.cpp:831] Syncing group operations: queue size (joins, cancels, datas) = (0, 0, 0)
I0714 23:59:39.622012     7 group.cpp:419] Trying to create path '/mesos' in ZooKeeper
I0714 23:59:39.619174     1 slave.cpp:612] Agent resources: [{"name":"cpus","scalar":{"value":1.0},"type":"SCALAR"},{"name":"mem","scalar":{"value":910.0},"type":"SCALAR"},{"name":"disk","scalar":{"value":4089.0},"type":"SCALAR"},{"name":"ports","ranges":{"range":[{"begin":31000,"end":32000}]},"type":"RANGES"}]
I0714 23:59:39.629868     1 slave.cpp:620] Agent attributes: [  ]
I0714 23:59:39.629896     1 slave.cpp:629] Agent hostname: localhost
I0714 23:59:39.632258     5 task_status_update_manager.cpp:181] Pausing sending task status updates
I0714 23:59:39.632906     7 detector.cpp:152] Detected a new leader: (id='2')
I0714 23:59:39.633379     7 group.cpp:700] Trying to get '/mesos/json.info_0000000002' in ZooKeeper
I0714 23:59:39.637286     8 state.cpp:66] Recovering state from '/var/tmp/mesos/meta'
I0714 23:59:39.638183     8 task_status_update_manager.cpp:207] Recovering task status update manager
I0714 23:59:39.638495     8 docker.cpp:899] Recovering Docker containers
I0714 23:59:39.642709     7 zookeeper.cpp:262] A new leading master (UPID=master@192.168.154.132:5050) is detected
I0714 23:59:39.814664    12 slave.cpp:7327] Finished recovery
I0714 23:59:39.819185    12 slave.cpp:1263] New master detected at master@192.168.154.132:5050
I0714 23:59:39.820412     8 task_status_update_manager.cpp:181] Pausing sending task status updates
I0714 23:59:39.820680    12 slave.cpp:1307] No credentials provided. Attempting to register without authentication
I0714 23:59:39.820761    12 slave.cpp:1318] Detecting new master

```

可以看到正常运行

http://192.168.154.132:5050/#/agents

如果让下面的红色部分显示成正确的ip地址

![修改hostname](D:\Picture\修改hostname.png)

需要修改下面的master中的脚本修改成如下：

```shell
[root@bogon ~]# cat mesos.sh 
#!/bin/bash
docker run -d --net=host \
  --hostname=192.168.154.132 \
  -e MESOS_PORT=5050 \
  -e MESOS_ZK=zk://192.168.154.133:2181/mesos \
  -e MESOS_QUORUM=1 \
  -e MESOS_REGISTRY=in_memory \
  -e MESOS_LOG_DIR=/var/log/mesos \
  -e MESOS_WORK_DIR=/var/tmp/mesos \
  -v "$(pwd)/log/mesos:/var/log/mesos" \
  -v "$(pwd)/work/mesos:/var/tmp/mesos" \
  mesosphere/mesos-master:1.5.1-rc1 --no-hostname_lookup --ip=192.168.154.132 
最后指定一下ip
```

运行slave的时候如果报 to remedy this do as follows 的时候需要删除下面的内容（防止上次的slave的数据被覆盖）

rm -f work/mesos/meta/slaves/latest

修改slave中的脚本

```shell
[root@localhost ~]# cat mesos-slave.sh 
docker run -d --net=host --privileged \
  --hostname=192.168.154.130 \
  -e MESOS_PORT=5051 \
  -e MESOS_MASTER=zk://192.168.154.133:2181/mesos \
  -e MESOS_SWITCH_USER=0 \
  -e MESOS_CONTAINERIZERS=docker \
  -e MESOS_LOG_DIR=/var/log/mesos \
  -e MESOS_WORK_DIR=/var/tmp/mesos \
  -v "$(pwd)/log/mesos:/var/log/mesos" \
  -v "$(pwd)/work/mesos:/var/tmp/mesos" \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /sys:/sys \
  -v /usr/bin/docker:/usr/local/bin/docker \
  mesosphere/mesos-slave:1.5.1-rc1 --no-systemd_enable_support \
  --no-hostname_lookup --ip=192.168.154.130

```

配置另外一个mesos 的slave

```shell
[root@bogon ~]# cat mesos-slave.sh 
#!/bin/sh
docker run -d  --net=host --privileged \
  --hostname=192.168.154.131 \
  -e MESOS_PORT=5051 \
  -e MESOS_MASTER=zk://192.168.154.133:2181/mesos \
  -e MESOS_SWITCH_USER=0 \
  -e MESOS_CONTAINERIZERS=docker \
  -e MESOS_LOG_DIR=/var/log/mesos \
  -e MESOS_WORK_DIR=/var/tmp/mesos \
  -v "$(pwd)/log/mesos:/var/log/mesos" \
  -v "$(pwd)/work/mesos:/var/tmp/mesos" \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /sys:/sys \
  -v /usr/bin/docker:/usr/local/bin/docker \
  mesosphere/mesos-slave:1.5.1-rc1 --no-systemd_enable_support \
  --no-hostname_lookup --ip=192.168.154.131

```

http://192.168.154.132:5050/#/

可以看到下面是可用的资源

![可用资源](D:\Picture\可用资源.png)

# 在132安装marathon

```shell

docker pull  mesosphere/marathon:v1.5.11

[root@bogon ~]# vim marathon.sh 
#!/bin/sh
docker run -d --net=host \
  --hostname=192.168.154.132 \
  mesosphere/marathon:v1.5.11 \
  --master zk://192.168.154.133:2181/mesos \
  --zk zk://192.168.154.133:2181/marathon
  
  第二行指定ip地址
  第三哪行指定marathon的版本
  之后是指定zookeeper的ip地址和端口
  
  然后修改1和3中的 mesos-slave.sh
  -e MESOS_CONTAINERIZERS=docker,mesos \
  
  sh marathon.sh
 访问 http://192.168.154.132:8080

```

然后看到如下：![marathon的访问](D:\Picture\marathon的访问.png)

点击create application  创建一个test

```shell
while [ true ];do sleep 5;echo 'hello mesos' ;done
```

点进去看到如下

![marathon创建程序](D:\Picture\marathon创建程序.png)

访问mesos的ui，可以看到运行的程序和framework

![ui查看mesos的](D:\Picture\ui查看mesos的.png)

![查看mesos中的frameworks](D:\Picture\查看mesos中的frameworks.png)

# 安装marathon-lb在4中

```shell
docker pull mesosphere/marathon-lb:v1.12.2
[root@localhost marathon-lb]# cat start.sh 
#!/bin/sh
#这里代表只定一个marathon集群和实例 --group是分组，定义集群名字
docker run -d --net=host \
  -e PORTS=9090 mesosphere/marathon-lb:v1.12.2 sse --group external --marathon http://192.168.154.132:8080
使用文档
https://hub.docker.com/r/mesosphere/marathon-lb/
```

访问：下面这个ip是4的ip地址

http://192.168.154.133:9090/haproxy?stats

能出现页面就代表正常





































