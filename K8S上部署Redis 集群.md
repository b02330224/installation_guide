# **从零开始搭建Kubernetes集群（六、在K8S上部署Redis 集群）**

2019.01.25 19:21 3507浏览

# 一、前言

上一文主要介绍了如何在K8S上搭建Ingress，以及如何通过Ingress访问后端服务。本篇将介绍如何在K8S上部署Redis集群。注意，这里所说的Redis 集群，指Redis Cluster而非Sentinel模式集群。

下图为Redis集群的架构图，每个Master都可以拥有多个Slave。当Master下线后，Redis集群会从多个Slave中选举出一个新的Master作为替代，而旧Master重新上线后变成新Master的Slave。



![webp](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAAAXNSR0IArs4c6QAAAARnQU1BAACxjwv8YQUAAAAJcEhZcwAADsQAAA7EAZUrDhsAAAANSURBVBhXYzh8+PB/AAffA0nNPuCLAAAAAElFTkSuQmCC)

image.png

# 二、准备操作

本次部署主要基于该项目：

```
https://github.com/zuxqoj/kubernetes-redis-cluster
```

其包含了两种部署Redis集群的方式：

- StatefulSet
- Service&Deployment

两种方式各有优劣，对于像Redis、Mongodb、Zookeeper等有状态的服务，使用StatefulSet是首选方式。本文将主要介绍如何使用StatefulSet进行Redis集群的部署。

# 三、StatefulSet简介

StatefulSet的概念非常重要，简单来说，其就是为了解决Pod重启、迁移后，Pod的IP、主机名等网络标识会改变而带来的问题。IP变化对于有状态的服务是难以接受的，如在Zookeeper集群的配置文件中，每个ZK节点都会记录其他节点的地址信息：

```
tickTime=2000dataDir=/home/myname/zookeeper
clientPort=2181initLimit=5syncLimit=2server.1=192.168.229.160:2888:3888server.2=192.168.229.161:2888:3888server.3=192.168.229.162:2888:3888
```

但若某个ZK节点的Pod重启后改变了IP，那么就会导致该节点脱离集群，而如果该配置文件中不使用IP而使用IP对应的域名，则可避免该问题：

```
server.1=zk-node1:2888:3888
server.2=zk-node2:2888:3888
server.3=zk-node3:2888:3888
```

也即是说，对于有状态服务，我们最好使用固定的网络标识（如域名信息）来标记节点，当然这也需要应用程序的支持（如Zookeeper就支持在配置文件中写入主机域名）。

StatefulSet基于Headless Service（即没有Cluster IP的Service）为Pod实现了稳定的网络标志（包括Pod的hostname和DNS Records），在Pod重新调度后也保持不变。同时，结合PV/PVC，StatefulSet可以实现稳定的持久化存储，就算Pod重新调度后，还是能访问到原先的持久化数据。

下图为使用StatefulSet部署Redis的架构，无论是Master还是Slave，都作为StatefulSet的一个副本，并且数据通过PV进行持久化，对外暴露为一个Service，接受客户端请求。



![webp](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAAAXNSR0IArs4c6QAAAARnQU1BAACxjwv8YQUAAAAJcEhZcwAADsQAAA7EAZUrDhsAAAANSURBVBhXYzh8+PB/AAffA0nNPuCLAAAAAElFTkSuQmCC)

image.png

# 四、部署过程

本文参考项目的README中，简要介绍了基于StatefulSet的Redis创建步骤：

1. 创建NFS存储
2. 创建PV
3. 创建PVC
4. 创建Configmap
5. 创建headless服务
6. 创建Redis StatefulSet
7. 初始化Redis集群

这里，我们将参考如上步骤，实践操作并详细介绍Redis集群的部署过程。文中会涉及到很多K8S的概念，希望大家能提前了解学习。

## 1.创建NFS存储

创建NFS存储主要是为了给Redis提供稳定的后端存储，当Redis的Pod重启或迁移后，依然能获得原先的数据。这里，我们先要创建NFS，然后通过使用PV为Redis挂载一个远程的NFS路径。

### 安装NFS

由于硬件资源有限，我们可以在k8s-work2上搭建。执行如下命令安装NFS和rpcbind：

```
yum -y install nfs-utils rpcbind
```

其中，NFS依靠远程过程调用(RPC)在客户端和服务器端路由请求，因此需要安装rpcbind服务。

然后，新增`/etc/exports`文件，用于设置需要共享的路径：

```
/usr/local/k8s/redis/pv1 *(rw,all_squash)
/usr/local/k8s/redis/pv2 *(rw,all_squash)
/usr/local/k8s/redis/pv3 *(rw,all_squash)
/usr/local/k8s/redis/pv4 *(rw,all_squash)

```

如上，rw表示读写权限；all_squash 表示客户机上的任何用户访问该共享目录时都映射成服务器上的匿名用户（默认为nfsnobody）；`*`表示任意主机都可以访问该共享目录，也可以填写指定主机地址，同时支持正则，如：

```
/root/share/ 192.168.1.20 (rw,all_squash)
/home/ljm/ *.gdfs.edu.cn (rw,all_squash)
```

由于我们打算创建一个6节点的Redis集群，所以共享了6个目录。当然，我们需要在k8s-node2上创建这些路径，并且为每个路径修改权限：

```
chmod 777 /usr/local/k8s/redis/pv*
```

这一步必不可少，否则挂载时会出现`mount.nfs: access denied by server while mounting`的权限错误。

接着，启动NFS和rpcbind服务：

```
systemctl start rpcbind
systemctl start nfs
```

我们在k8s-node1上测试一下，执行：

```
yum -y install nfs-utils

mount -t nfs 192.168.56.102:/usr/local/k8s/redis/pv1 /mnt
```

表示将k8s-node2上的共享目录`/usr/local/k8s/redis/pv1`映射为k8s-node1的`/mnt`目录，我们在/mnt中创建文件：

```
touch haha
```

既可以在k8s-node2上看到该文件：

```
[root@k8s-node2 redis]# ll pv1总用量 0-rw-r--r--. 1 nfsnobody nfsnobody 0 5月   2 21:35 haha
```

可以看到用户和组为`nfsnobody`。





## 创建PV

每一个Redis Pod都需要一个独立的PV来存储自己的数据，因此可以创建一个`pv.yaml`文件，包含3个PV：

```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-pv1
spec:
  capacity:
    storage: 200M
  accessModes:
    - ReadWriteMany
  nfs:
    server: 192.168.184.122
    path: "/usr/local/k8s/redis/pv1"
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-pv2
spec:
  capacity:
    storage: 200M
  accessModes:
    - ReadWriteMany
  nfs:
    server: 192.168.184.122
    path: "/usr/local/k8s/redis/pv2"
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-pv3
spec:
  capacity:
    storage: 200M
  accessModes:
    - ReadWriteMany
  nfs:
    server: 192.168.184.122
    path: "/usr/local/k8s/redis/pv3"
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-pv4
spec:
  capacity:
    storage: 200M
  accessModes:
    - ReadWriteMany
  nfs:
    server: 192.168.184.122
    path: "/usr/local/k8s/redis/pv4"    

```

如上，可以看到所有PV除了名称和挂载的路径外都基本一致。执行创建即可：

如上，可以看到所有PV除了名称和挂载的路径外都基本一致。执行创建即可：

```
[root@k8s-node1 redis]# kubectl create -f pv.yaml 
persistentvolume "nfs-pv1" created
persistentvolume "nfs-pv2" created
persistentvolume "nfs-pv3" created
persistentvolume "nfs-pv4" created

```

## 2.创建Configmap

这里，我们可以直接将Redis的配置文件转化为Configmap，这是一种更方便的配置读取方式。配置文件`redis.conf`如下：

```
appendonly yes
cluster-enabled yes
cluster-config-file /var/lib/redis/nodes.conf
cluster-node-timeout 5000
dir /var/lib/redis
port 6379
```

创建名为`redis-conf`的Configmap：

```
kubectl create configmap redis-conf --from-file=redis.conf
```

查看：

```
[root@k8s-node1 redis]# kubectl describe cm redis-conf
Name:         redis-conf
Namespace:    default
Labels:       <none>Annotations:  <none>Data
====
redis.conf:
----
appendonly yes
cluster-enabled yes
cluster-config-file /var/lib/redis/nodes.conf
cluster-node-timeout 5000
dir /var/lib/redis
port 6379

Events:  <none>
```

如上，`redis.conf`中的所有配置项都保存到`redis-conf`这个Configmap中。





## 3.创建Headless service

Headless service是StatefulSet实现稳定网络标识的基础，我们需要提前创建。准备文件`headless-service.yml`如下：

```
apiVersion: v1
kind: Service
metadata:
  name: redis-service
  labels:
    app: redis
spec:
  ports:
  - name: redis-port
    port: 6379
  clusterIP: None
  selector:
    app: redis
    appCluster: redis-cluster
```

创建：

```
kubectl create -f headless-service.yml
```

查看：

```
[root@k8s-node1 redis]# kubectl get svc redis-serviceNAME            TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)    AGE
redis-service   ClusterIP   None         <none>        6379/TCP   53s
```

可以看到，服务名称为`redis-service`，其`CLUSTER-IP`为`None`，表示这是一个“无头”服务。





# 4.创建Redis 集群节点

创建好Headless service后，就可以利用StatefulSet创建Redis 集群节点，这也是本文的核心内容。我们先创建redis.yml文件：

```yaml
apiVersion: apps/v1beta1
kind: StatefulSet
metadata:
  name: redis-app
spec:
  serviceName: "redis-service"
  replicas: 4
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
        appCluster: redis-cluster
    spec:
      terminationGracePeriodSeconds: 20
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app                
                  operator: In
                  values:
                  - redis
              topologyKey: kubernetes.io/hostname
      containers:
      - name: redis
        image: "redis:latest"
        command:
          - "redis-server"
        args:
          - "/etc/redis/redis.conf"
          - "--protected-mode"
          - "no"
        resources:
          requests:
            cpu: "100m"
            memory: "100Mi"
        ports:
            - name: redis
              containerPort: 6379
              protocol: "TCP"
            - name: cluster
              containerPort: 16379
              protocol: "TCP"
        volumeMounts:
          - name: "redis-conf"
            mountPath: "/etc/redis"
          - name: "redis-data"
            mountPath: "/var/lib/redis"
      volumes:
      - name: "redis-conf"
        configMap:
          name: "redis-conf"
          items:
            - key: "redis.conf"
              path: "redis.conf"
  volumeClaimTemplates:
  - metadata:
      name: redis-data
    spec:
      accessModes: [ "ReadWriteMany" ]
      resources:
        requests:
          storage: 200M
```

如上，总共创建了4个Redis节点(Pod)，其中1个将用于master，另外3个分别作为master的slave；Redis的配置通过volume将之前生成的`redis-conf`这个Configmap，挂载到了容器的`/etc/redis/redis.conf`；Redis的数据存储路径使用volumeClaimTemplates声明（也就是PVC），其会绑定到我们先前创建的PV上。

这里有一个关键概念——Affinity，请参考[官方文档](https://kubernetes.io/docs/concepts/configuration/assign-pod-node/)详细了解。其中，podAntiAffinity表示反亲和性，其决定了某个pod不可以和哪些Pod部署在同一拓扑域，可以用于将一个服务的POD分散在不同的主机或者拓扑域中，提高服务本身的稳定性。

而PreferredDuringSchedulingIgnoredDuringExecution 则表示，在调度期间尽量满足亲和性或者反亲和性规则，如果不能满足规则，POD也有可能被调度到对应的主机上。在之后的运行过程中，系统不会再检查这些规则是否满足。

在这里，matchExpressions规定了Redis Pod要尽量不要调度到包含app为redis的Node上，也即是说已经存在Redis的Node上尽量不要再分配Redis Pod了。但是，由于我们只有2个Node，而副本有4个，因此根据PreferredDuringSchedulingIgnoredDuringExecution，这些豌豆不得不得挤一挤，挤挤更健康~

另外，根据StatefulSet的规则，我们生成的Redis的4个Pod的hostname会被依次命名为$(statefulset名称)-$(序号)，如下图所示：

```
[root@master redis_cluster]# kubectl get pod -o wide
NAME                      READY   STATUS    RESTARTS   AGE    IP              NODE    NOMINATED NODE   READINESS GATES
redis-app-0               1/1     Running   0          13m    10.244.215.22   work1   <none>           <none>
redis-app-1               1/1     Running   0          106m   10.244.123.31   work2   <none>           <none>
redis-app-2               1/1     Running   0          84m    10.244.215.19   work1   <none>           <none>
redis-app-3               1/1     Running   0          119s   10.244.123.32   work2   <none>           <none>

```

如上，可以看到这些Pods在部署时是以{0..N-1}的顺序依次创建的。注意，直到redis-app-0状态启动后达到Running状态之后，redis-app-1 才开始启动。

同时，每个Pod都会得到集群内的一个DNS域名，格式为`$(podname).$(service name).$(namespace).svc.cluster.local`，也即是：

```
redis-app-0.redis-service.default.svc.cluster.localredis-app-1.redis-service.default.svc.cluster.local...以此类推...
```

在K8S集群内部，这些Pod就可以利用该域名互相通信。我们可以使用busybox镜像的nslookup检验这些域名：

```
[root@k8s-node1 ~]# kubectl run -i --tty --image busybox dns-test --restart=Never --rm /bin/sh If you don't see a command prompt, try pressing enter.
/ # nslookup redis-app-0.redis-service
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

Name:      redis-app-0.redis-service
Address 1: 192.168.169.207 redis-app-0.redis-service.default.svc.cluster.local
```

可以看到， redis-app-0的IP为192.168.169.207。当然，若Redis Pod迁移或是重启（我们可以手动删除掉一个Redis Pod来测试），则IP是会改变的，但Pod的域名、SRV records、A record都不会改变。

另外可以发现，我们之前创建的pv都被成功绑定了：

```
[root@master redis_cluster]# kubectl get pv
NAME      CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                            STORAGECLASS   REASON   AGE
nfs-pv1   200M       RWX            Retain           Bound    default/redis-data-redis-app-0                           5h19m
nfs-pv2   200M       RWX            Retain           Bound    default/redis-data-redis-app-1                           5h19m
nfs-pv3   200M       RWX            Retain           Bound    default/redis-data-redis-app-2                           5h19m
nfs-pv4   200M       RWX            Retain           Bound    default/redis-data-redis-app-3                           3m2s
[root@master redis_cluster]# kubectl get pvc
NAME                     STATUS   VOLUME    CAPACITY   ACCESS MODES   STORAGECLASS   AGE
redis-data-redis-app-0   Bound    nfs-pv1   200M       RWX                           151m
redis-data-redis-app-1   Bound    nfs-pv2   200M       RWX                           139m
redis-data-redis-app-2   Bound    nfs-pv3   200M       RWX                           107m
redis-data-redis-app-3   Bound    nfs-pv4   200M       RWX                           87s
```

## 5.初始化Redis集群

创建好6个Redis Pod后，我们还需要利用常用的Redis-tribe工具进行集群的初始化。

### 创建Ubuntu容器

由于Redis集群必须在所有节点启动后才能进行初始化，而如果将初始化逻辑写入Statefulset中，则是一件非常复杂而且低效的行为。这里，本人不得不称赞一下原项目作者的思路，值得学习。也就是说，我们可以在K8S上创建一个额外的容器，专门用于进行K8S集群内部某些服务的管理控制。

这里，我们专门启动一个Ubuntu的容器，可以在该容器中安装Redis-tribe，进而初始化Redis集群，执行：

```
kubectl run -i --tty ubuntu --image=ubuntu --restart=Never /bin/bash
```

成功后，我们可以进入ubuntu容器中，原项目要求执行如下命令安装基本的软件环境：

```
apt-get update
apt-get install -y vim wget python2.7 python-pip redis-tools dnsutils
```

但是，需要注意的是，在我们天朝，执行上述命令前需要提前做一件必要的工作——换源，否则你懂得。我们使用阿里云的Ubuntu源，执行：

```
root@ubuntu:/# cat > /etc/apt/sources.list << EOF
 deb http://mirrors.aliyun.com/ubuntu/ bionic main restricted universe multiverse
 deb-src http://mirrors.aliyun.com/ubuntu/ bionic main restricted universe multiverse
 deb http://mirrors.aliyun.com/ubuntu/ bionic-security main restricted universe multiverse
 deb-src http://mirrors.aliyun.com/ubuntu/ bionic-security main restricted universe multiverse
 deb http://mirrors.aliyun.com/ubuntu/ bionic-updates main restricted universe multiverse
 deb-src http://mirrors.aliyun.com/ubuntu/ bionic-updates main restricted universe multiverse 
 deb http://mirrors.aliyun.com/ubuntu/ bionic-proposed main restricted universe multiverse
 deb-src http://mirrors.aliyun.com/ubuntu/ bionic-proposed main restricted universe multiverse 
 deb http://mirrors.aliyun.com/ubuntu/ bionic-backports main restricted universe multiverse
 deb-src http://mirrors.aliyun.com/ubuntu/ bionic-backports main restricted universe multiverse
 EOF
```

源修改完毕后，就可以执行上面的两个命令。

### 初始化集群

首先，我们需要安装`redis-trib`：

```
pip install redis-trib
```

然后，创建只有Master节点的集群：

```
redis-trib.py create \
  `dig +short redis-app-0.redis-service.default.svc.cluster.local`:6379 

```

如上，命令`dig +short redis-app-0.redis-service.default.svc.cluster.local`用于将Pod的域名转化为IP，这是因为`redis-trib`不支持域名来创建集群。

其次，为每个Master添加Slave：

```
redis-trib.py replicate 
--master-addr `dig +short redis-app-0.redis-service.default.svc.cluster.local`:6379 
--slave-addr `dig +short redis-app-1.redis-service.default.svc.cluster.local`:6379 


redis-trib.py replicate 
--master-addr `dig +short redis-app-0.redis-service.default.svc.cluster.local`:6379 
--slave-addr `dig +short redis-app-2.redis-service.default.svc.cluster.local`:6379

redis-trib.py replicate 
--master-addr `dig +short redis-app-0.redis-service.default.svc.cluster.local`:6379 
--slave-addr `dig +short redis-app-3.redis-service.default.svc.cluster.local`:6379
```

至此，我们的Redis集群就真正创建完毕了，连到任意一个Redis Pod中检验一下：

```
root@k8s-node1 ~]# kubectl exec -it redis-app-2 /bin/bash
root@redis-app-2:/data# /usr/local/bin/redis-cli -c
127.0.0.1:6379> cluster info
cluster_state:ok
cluster_slots_assigned:16384
cluster_slots_ok:16384
cluster_slots_pfail:0
cluster_slots_fail:0
cluster_known_nodes:3
cluster_size:1
cluster_current_epoch:2
cluster_my_epoch:0
cluster_stats_messages_ping_sent:286
cluster_stats_messages_pong_sent:289
cluster_stats_messages_sent:575
cluster_stats_messages_ping_received:288
cluster_stats_messages_pong_received:286
cluster_stats_messages_meet_received:1
cluster_stats_messages_received:575
127.0.0.1:6379> cluster nodes
dd5c64a46a7ee6ad3c838fc77753e8c8768acedf 10.244.215.16:6379@16379 master - 0 1606657433514 0 connected 0-16383
c463c5c1903b451215ff581712f6661793ec6dc5 10.244.215.19:6379@16379 myself,slave dd5c64a46a7ee6ad3c838fc77753e8c8768acedf 0 1606657431000 0 connected
5e2e6a7a1c3ea7c48f1f93cfe1f9e0a109668e87 10.244.123.31:6379@16379 slave dd5c64a46a7ee6ad3c838fc77753e8c8768acedf 0 1606657434521 0 connected
127.0.0.1:6379> 
```

另外，还可以在NFS上查看Redis挂载的数据：

```
[root@work2 ~]# ll /usr/local/k8s/redis/pv1
total 8
-rw-r--r-- 1 nfsnobody nfsnobody   0 Nov 29 20:24 appendonly.aof
-rw-r--r-- 1 nfsnobody nfsnobody 176 Nov 29 21:43 dump.rdb
-rw-r--r-- 1 nfsnobody nfsnobody 436 Nov 29 21:43 nodes.conf
[root@work2 ~]# ll /usr/local/k8s/redis/pv2
total 12
-rw-r--r-- 1 nfsnobody nfsnobody  92 Nov 29 21:43 appendonly.aof
-rw-r--r-- 1 nfsnobody nfsnobody 176 Nov 29 21:43 dump.rdb
-rw-r--r-- 1 nfsnobody nfsnobody 436 Nov 29 21:43 nodes.conf
[root@work2 ~]# ll /usr/local/k8s/redis/pv3
total 12
-rw-r--r-- 1 nfsnobody nfsnobody  92 Nov 29 21:38 appendonly.aof
-rw-r--r-- 1 nfsnobody nfsnobody 175 Nov 29 21:38 dump.rdb
-rw-r--r-- 1 nfsnobody nfsnobody 436 Nov 29 21:43 nodes.conf
[root@work2 ~]# 
```

## 

## 6.创建用于访问Service

前面我们创建了用于实现StatefulSet的Headless Service，但该Service没有Cluster Ip，因此不能用于外界访问。所以，我们还需要创建一个Service，专用于为Redis集群提供访问和负载均：

```
piVersion: v1
kind: Service
metadata:
  name: redis-access-service
  labels:
    app: redis
spec:
  ports:
  - name: redis-port
    protocol: "TCP"
    port: 6379
    targetPort: 6379
  selector:
    app: redis
    appCluster: redis-cluster
```

如上，该Service名称为 `redis-access-service`，在K8S集群中暴露6379端口，并且会对`labels name`为`app: redis`或`appCluster: redis-cluster`的pod进行负载均衡。

创建后查看：

```
[root@k8s-node1 redis]# kubectl get svc redis-access-service -o wideNAME                   TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE       SELECTOR
redis-access-service   ClusterIP   10.105.11.209   <none>        6379/TCP   41m       app=redis,appCluster=redis-cluster
```

如上，在K8S集群中，所有应用都可以通过`10.105.11.209:6379`来访问Redis集群。当然，为了方便测试，我们也可以为Service添加一个NodePort映射到物理机上，这里不再详细介绍。





# 五、测试主从切换

在K8S上搭建完好Redis集群后，我们最关心的就是其原有的高可用机制是否正常。这里，我们可以任意挑选一个Master的Pod来测试集群的主从切换机制，如`redis-app-0`：

```
[root@master redis_cluster]# kubectl get pods redis-app-0 -o wide
NAME          READY   STATUS    RESTARTS   AGE   IP              NODE    NOMINATED NODE   READINESS GATES
redis-app-0   1/1     Running   0          86m   10.244.215.16   work1   <none>           <none>
[root@master redis_cluster]# 
```

进入redis-app-0查看：

```
[root@master redis_cluster]# kubectl exec -it redis-app-0 /bin/bash
root@redis-app-0:/data#  /usr/local/bin/redis-cli -c
127.0.0.1:6379> role
1) "master"
2) (integer) 1162
3) 1) 1) "10.244.215.19"
      2) "6379"
      3) "1148"
   2) 1) "10.244.123.31"
      2) "6379"
      3) "1162"
127.0.0.1:6379> 
```

如上可以看到，其为master，slave为10.244.215.19（redis-app-1）和10.244.123.31（redis-app-2）。

接着，我们手动删除`redis-app-0`：

```
[root@master redis_cluster]#  kubectl delete pods redis-app-0
pod "redis-app-0" deleted

[root@master redis_cluster]#  kubectl get pods redis-app-0 -o wide
NAME          READY   STATUS    RESTARTS   AGE   IP              NODE    NOMINATED NODE   READINESS GATES
redis-app-0   1/1     Running   0          9s    10.244.215.21   work1   <none>           <none>
[root@master redis_cluster]# 
```

如上，IP改变为`10.244.215.21`。我们再进入`redis-app-0`内部查看：

```

[root@master redis_cluster]#  kubectl get pods redis-app-0 -o wide
NAME          READY   STATUS    RESTARTS   AGE   IP              NODE    NOMINATED NODE   READINESS GATES
redis-app-0   1/1     Running   0          9s    10.244.215.22   work1   <none>           <none>
[root@master redis_cluster]#  kubectl exec -it redis-app-0 /bin/bash
root@redis-app-0:/data#  /usr/local/bin/redis-cli -c
127.0.0.1:6379> role
1) "master"
2) (integer) 14
3) 1) 1) "10.244.215.19"
      2) "6379"
      3) "14"
   2) 1) "10.244.123.31"
      2) "6379"
      3) "14"
127.0.0.1:6379> 
127.0.0.1:6379> 
```



# 六、疑问

至此，大家可能会疑惑，前面讲了这么多似乎并没有体现出StatefulSet的作用，其提供的稳定标志`redis-app-*`仅在初始化集群的时候用到，而后续Redis Pod的通信或配置文件中并没有使用该标志。我想说，是的，本文使用StatefulSet部署Redis确实没有体现出其优势，还不如介绍Zookeeper集群来的明显，不过没关系，学到知识就好。

那为什么没有使用稳定的标志，Redis Pod也能正常进行故障转移呢？这涉及了Redis本身的机制。因为，Redis集群中每个节点都有自己的NodeId（保存在自动生成的`nodes.conf`中），并且该NodeId不会随着IP的变化和变化，这其实也是一种固定的网络标志。也就是说，就算某个Redis Pod重启了，该Pod依然会加载保存的NodeId来维持自己的身份。我们可以在NFS上查看`redis-app-1`的`nodes.conf`文件：

```
[root@k8s-node2 ~]# cat /usr/local/k8s/redis/pv1/nodes.conf 96689f2018089173e528d3a71c4ef10af68ee462 192.168.169.209:6379@16379 slave d884c4971de9748f99b10d14678d864187a9e5d3 0 1526460952651 4 connected237d46046d9b75a6822f02523ab894928e2300e6 192.168.169.200:6379@16379 slave c15f378a604ee5b200f06cc23e9371cbc04f4559 0 1526460952651 1 connected
c15f378a604ee5b200f06cc23e9371cbc04f4559 192.168.169.197:6379@16379 master - 0 1526460952651 1 connected 10923-16383d884c4971de9748f99b10d14678d864187a9e5d3 192.168.169.205:6379@16379 master - 0 1526460952651 4 connected 5462-10922c3b4ae23c80ffe31b7b34ef29dd6f8d73beaf85f 192.168.169.198:6379@16379 myself,slave c8a8f70b4c29333de6039c47b2f3453ed11fb5c2 0 1526460952565 3 connected
c8a8f70b4c29333de6039c47b2f3453ed11fb5c2 192.168.169.201:6379@16379 master - 0 1526460952651 6 connected 0-5461vars currentEpoch 6 lastVoteEpoch 4
```

如上，第一列为NodeId，稳定不变；第二列为IP和端口信息，可能会改变。

这里，我们介绍NodeId的两种使用场景：

- 当某个Slave Pod断线重连后IP改变，但是Master发现其NodeId依旧， 就认为该Slave还是之前的Slave。
- 当某个Master Pod下线后，集群在其Slave中选举重新的Master。待旧Master上线后，集群发现其NodeId依旧，会让旧Master变成新Master的slave。

对于这两种场景，大家有兴趣的话还可以自行测试，注意要观察Redis的日志。



# 七、废话

至此，我们的Redis集群就搭建完毕了。如果大家有兴趣，可以尝试使用Service配合Deployment方式来部署Redis集群，该方式相对来说更容易理解。

本人水平有限，难免有错误或遗漏之处，望大家指正和谅解，欢迎评论留言

