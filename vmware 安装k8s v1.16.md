# vmware 安装k8s v1.17

参考视频： 

https://www.bilibili.com/video/BV1Gv411B7US?p=31

20年最新 黑马程序员 - Kubernetes(K8S)超快速入门教程

## 虚机准备

创建虚机用nat的网络，100G，cpu 2核，内存4G，os用centos 7.6

- master 4G 2核 CentOS7 192.168.10.20
- work1 4G 2核 CentOS7 192.168.10.21
- work2 4G 2核 CentOS7 192.168.10.22



网络配置

[root@master ~]# cat /etc/sysconfig/network-scripts/ifcfg-ens33 
TYPE="Ethernet"
PROXY_METHOD="none"
BROWSER_ONLY="no"
BOOTPROTO="none"
DEFROUTE="yes"
IPV4_FAILURE_FATAL="no"
IPV6INIT="yes"
IPV6_AUTOCONF="yes"
IPV6_DEFROUTE="yes"
IPV6_FAILURE_FATAL="no"
IPV6_ADDR_GEN_MODE="stable-privacy"
NAME="ens33"
UUID="031e799c-1104-4c2f-a25f-bd8083856944"
DEVICE="ens33"
ONBOOT="yes"
IPADDR="192.168.184.120"
PREFIX=24
GATEWAY="192.168.184.2"
DNS1="119.29.29.29"





## 准备工作(所有节点)

关闭防火墙

```bash
systemctl disable firewalld
systemctl stop firewalld
```

关闭selinux

```bash
# 临时禁用selinux
setenforce 0
# 永久关闭 修改/etc/sysconfig/selinux文件设置
sed -i "s/SELINUX=enforcing/SELINUX=disabled/g" /etc/selinux/config
```



修改主机名

```shell

# 临时修改
hostnamectl  set-hostname  master|work1|work2
# 永久修改
vi /etc/hosts
192.168.184.120 master
192.168.184.121 work1
192.168.184.122 work2
```



同步时间

```shell
[root@master ~]# yum install -y ntpdate

[root@master ~]# crontab -e

0 */1 * * * ntpdate time1.aliyun.com
```



禁用交换分区

```bash
swapoff -a

# 永久禁用，打开/etc/fstab注释掉swap那一行。
sed -i 's/.*swap.*/#&/' /etc/fstab
```



添加网桥过滤

```shell
# 添加网桥过滤及地址转发
[root@master ~]# vi /etc/sysctl.d/k8s.conf

net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
vm.swappiness = 0

#加载br_netfilter模块
[root@master ~]# modprobe  br_netfilter

[root@master ~]# lsmod|grep br_netfilter
br_netfilter           22256  0 
bridge                151336  1 br_netfilter

#加载网桥过滤配置文件
[root@master ~]# sysctl  -p /etc/sysctl.d/k8s.conf 
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
vm.swappiness = 0
```

开启IPVS

```shell
#安装
 yum install -y ipset ipvsadm
 
#添加需要加载的模块
cat > /etc/sysconfig/modules/ipvs.modules <<EOF
#!/bin/bash
modprobe -- ip_vs
modprobe -- ip_vs_rr
modprobe -- ip_vs_wrr
modprobe -- ip_vs_sh
modprobe -- nf_conntrack_ipv4
EOF

# 授权，运行
chmod 755 /etc/sysconfig/modules/ipvs.modules
bash /etc/sysconfig/modules/ipvs.modules

# 检查是否加载
lsmod | grep -e ip_vs -e nf_conntrack_ipv4
每个节点执行完毕应该都可以看见已经加载的ipvs模块
```



master,work上安装docker

```shell

yum install -y vim wget


wget -O /etc/yum.repos.d/docker-ce.repo  https://mirrors.tuna.tsinghua.edu.cn/docker-ce/linux/centos/docker-ce.repo

#查看支持的docker版本
yum list docker-ce --showduplicates |sort -r 


#安装指定版本docker-ce，此版本不需要修改服务启动文件及iptables默认规则链策略。
yum install -y --setopt=obsoletes=0 docker-ce-18.06.3.ce-3.el7



# 修改docker-ce服务配置文件
#修改其目的是为了后续使用/etc/docker/daemon.json来进行更多配置。
修改内容如下
[root@XXX ~]# cat
/usr/lib/systemd/system/docker.service
[Unit]
...
[Service]
...
ExecStart=/usr/bin/dockerd #如果原文件此行后面
有-H选项，请删除-H(含)后面所有内容。

注意：有些版本不需要修改，请注意观察


 
 
 # vim /etc/docker/daemon.json
 
 {
        "exec-opts": ["native.cgroupdriver=systemd"]
}

 systemctl  enable docker 
 systemctl  start  docker
```



## 部署k8s软件及配置（所有节点）

```shell
## 配置k8s源
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
EOF

# 检查yum源是否生效
[root@master ~]# yum list |grep kubeadm
Repository base is listed more than once in the configuration
Repository updates is listed more than once in the configuration
Repository extras is listed more than once in the configuration
Repository centosplus is listed more than once in the configuration
kubeadm.x86_64                              1.17.2-0                   @kubernetes
kubeadm.x86_64                              1.19.3-0                   kubernetes
```



安装指定版本的kubeadm kubectl kubelet

```shell
 yum list kubeadm --showduplicates |sort -r 
 
 yum install -y --setopt=obsoletes=0 kubeadm-1.17.2-0 kubelet-1.17.2-0 kubectl-1.17.2-0
 
```



软件设置

主要配置kubelet,如果不配置可能导致k8s集群无法启动

```shell
#编辑文件,为了实现docker使用的cgroupdriver与kubelet使用的cgroup的一致性，建议修改如下文件
vim /etc/sysconfig/kubelet
KUBELET_EXTRA_ARGS="--cgroup-driver=systemd"


#设置为开机自启动即可，由于没有生成配置文件，集群初始化后自动启动
systemctl enable kubelet
```

## k8s集群容器镜像准备



### master节点镜像准备

```shell
#查看集群使用的镜像
[root@master ~]# kubeadm config images list
k8s.gcr.io/kube-apiserver:v1.17.13
k8s.gcr.io/kube-controller-manager:v1.17.13
k8s.gcr.io/kube-scheduler:v1.17.13
k8s.gcr.io/kube-proxy:v1.17.13
k8s.gcr.io/pause:3.1
k8s.gcr.io/etcd:3.4.3-0
k8s.gcr.io/coredns:1.6.5


[root@master ~]# kubeadm config images list >> images.list

#修改images.list 为shell脚本
[root@master ~]# vim images.list 
#!/bin/bash
image_list='registry.cn-hangzhou.aliyuncs.com/google_containers/kube-apiserver:v1.17.13
registry.cn-hangzhou.aliyuncs.com/google_containers/kube-controller-manager:v1.17.13
registry.cn-hangzhou.aliyuncs.com/google_containers/kube-scheduler:v1.17.13
registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy:v1.17.13
registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.1
registry.cn-hangzhou.aliyuncs.com/google_containers/etcd:3.4.3-0
registry.cn-hangzhou.aliyuncs.com/google_containers/coredns:1.6.5'

for img in ${image_list}
do 
    docker pull $img

done


[root@master ~]# sh images.list 

```



### copy镜像到work节点

```shell
[root@master ~]# docker images
REPOSITORY                                                                    TAG                 IMAGE ID            CREATED             SIZE
registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy                v1.17.13            e950b1d6dbf3        2 weeks ago         117MB
registry.cn-hangzhou.aliyuncs.com/google_containers/kube-apiserver            v1.17.13            62788e53d31e        2 weeks ago         171MB
registry.cn-hangzhou.aliyuncs.com/google_containers/kube-controller-manager   v1.17.13            3fd325848220        2 weeks ago         161MB
registry.cn-hangzhou.aliyuncs.com/google_containers/kube-scheduler            v1.17.13            a834ecc1505b        2 weeks ago         94.5MB
registry.cn-hangzhou.aliyuncs.com/google_containers/coredns                   1.6.5               70f311871ae1        12 months ago       41.6MB
registry.cn-hangzhou.aliyuncs.com/google_containers/etcd                      3.4.3-0             303ce5db0e90        12 months ago       288MB
registry.cn-hangzhou.aliyuncs.com/google_containers/pause                     3.1                 da86e6ba6ca1        2 years ago         742kB

[root@master ~]# docker save -o kube-proxy.tar registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy:v1.17.13


[root@master ~]# docker save -o pause.tar registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.1

[root@master ~]# scp *.tar work1:~
root@work1's password: 
kube-proxy.tar                                                                                                                                                      0%    0     0.0KB/s   --:-- ETA^kube-proxy.tar                                                                                                                                                    100%  113MB  32.6MB/s   00:03    
pause.tar                                                                                                                                                         100%  737KB  33.7MB/s   00:00    
[root@master ~]# scp *.tar work2:~
root@work2's password: 
kube-proxy.tar                                                                                                                                                    100%  113MB  40.8MB/s   00:02    
pause.tar                                
```



work节点加载镜像

```shell
	[root@work1 ~]# docker load -i  kube-proxy.tar
	[root@work1 ~]# docker load -i  pause.tar 
```



## k8s集群初始化

mater节点操作

```shell
sudo kubeadm init \
 --apiserver-advertise-address 192.168.184.120 \
 --image-repository registry.aliyuncs.com/google_containers \
 --kubernetes-version=v1.17.2 \
 --pod-network-cidr=10.244.0.0/16 \
 --kubernetes-version=v1.17.2
 
 W1031 21:09:02.413001   21506 validation.go:28] Cannot validate kube-proxy config - no validator is available
W1031 21:09:02.413406   21506 validation.go:28] Cannot validate kubelet config - no validator is available
[init] Using Kubernetes version: v1.17.2
[preflight] Running pre-flight checks
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Starting the kubelet
[certs] Using certificateDir folder "/etc/kubernetes/pki"
[certs] Generating "ca" certificate and key
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [master kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local] and IPs [10.96.0.1 192.168.184.120]
[certs] Generating "apiserver-kubelet-client" certificate and key
[certs] Generating "front-proxy-ca" certificate and key
[certs] Generating "front-proxy-client" certificate and key
[certs] Generating "etcd/ca" certificate and key
[certs] Generating "etcd/server" certificate and key
[certs] etcd/server serving cert is signed for DNS names [master localhost] and IPs [192.168.184.120 127.0.0.1 ::1]
[certs] Generating "etcd/peer" certificate and key
[certs] etcd/peer serving cert is signed for DNS names [master localhost] and IPs [192.168.184.120 127.0.0.1 ::1]
[certs] Generating "etcd/healthcheck-client" certificate and key
[certs] Generating "apiserver-etcd-client" certificate and key
[certs] Generating "sa" key and public key
[kubeconfig] Using kubeconfig folder "/etc/kubernetes"
[kubeconfig] Writing "admin.conf" kubeconfig file
[kubeconfig] Writing "kubelet.conf" kubeconfig file
[kubeconfig] Writing "controller-manager.conf" kubeconfig file
[kubeconfig] Writing "scheduler.conf" kubeconfig file
[control-plane] Using manifest folder "/etc/kubernetes/manifests"
[control-plane] Creating static Pod manifest for "kube-apiserver"
[control-plane] Creating static Pod manifest for "kube-controller-manager"
W1031 21:09:13.015973   21506 manifests.go:214] the default kube-apiserver authorization-mode is "Node,RBAC"; using "Node,RBAC"
[control-plane] Creating static Pod manifest for "kube-scheduler"
W1031 21:09:13.019388   21506 manifests.go:214] the default kube-apiserver authorization-mode is "Node,RBAC"; using "Node,RBAC"
[etcd] Creating static Pod manifest for local etcd in "/etc/kubernetes/manifests"
[wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests". This can take up to 4m0s
[apiclient] All control plane components are healthy after 16.512587 seconds
[upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config-1.17" in namespace kube-system with the configuration for the kubelets in the cluster
[upload-certs] Skipping phase. Please see --upload-certs
[mark-control-plane] Marking the node master as control-plane by adding the label "node-role.kubernetes.io/master=''"
[mark-control-plane] Marking the node master as control-plane by adding the taints [node-role.kubernetes.io/master:NoSchedule]
[bootstrap-token] Using token: ld3dsf.j26i2tjaa2cd330o
[bootstrap-token] Configuring bootstrap tokens, cluster-info ConfigMap, RBAC Roles
[bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstrap-token] configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstrap-token] configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[bootstrap-token] Creating the "cluster-info" ConfigMap in the "kube-public" namespace
[kubelet-finalize] Updating "/etc/kubernetes/kubelet.conf" to point to a rotatable kubelet client certificate and key
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.184.120:6443 --token ld3dsf.j26i2tjaa2cd330o \
    --discovery-token-ca-cert-hash sha256:24be46f997b113d8cca2b98263ffb082b7c52b3b6698ce2c686df65fd143290b 
    
```

master节点操作

```shell
  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

## 部署calico网络

master,work节点操作

```shell
 docker load -i calico-cni.tar
 docker load -i calico-node.tar
 docker load -i  kube-controllers.tar
 docker load -i  pod2daemon-flexvol.tar
```

master节点修改calico资源清单文件

```shell
# 由于calico自身网络发现机制有问题，因为需要修改
calico使用的物理网卡，添加607及608行
602 - name: CLUSTER_TYPE
603   value: "k8s,bgp"
604 # Auto-detect the BGP IP address.
605 - name: IP
606   value: "autodetect"
607 - name: IP_AUTODETECTION_METHOD
608   value: "interface=ens.*"
# 修改为初始化时设置的pod-network-cidr
619 - name: CALICO_IPV4POOL_CIDR
620   value: "10.244.0.0/16"
```

在应用caclico资源清单文件之前，需要把calico所有的镜 像导入到node节点中。

```shell
# 仅在master节点执行
[root@master calico-39]# kubectl apply -f  calico.yml
configmap/calico-config created
customresourcedefinition.apiextensions.k8s.io/felixconfigurations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ipamblocks.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/blockaffinities.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ipamhandles.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ipamconfigs.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/bgppeers.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/bgpconfigurations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ippools.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/hostendpoints.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/clusterinformations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/globalnetworkpolicies.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/globalnetworksets.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/networkpolicies.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/networksets.crd.projectcalico.org created
clusterrole.rbac.authorization.k8s.io/calico-kube-controllers created
clusterrolebinding.rbac.authorization.k8s.io/calico-kube-controllers created
clusterrole.rbac.authorization.k8s.io/calico-node created
clusterrolebinding.rbac.authorization.k8s.io/calico-node created
daemonset.apps/calico-node created
serviceaccount/calico-node created
deployment.apps/calico-kube-controllers created
serviceaccount/calico-kube-controllers created
```

## 添加工作节点到集群

```shell
# 在work节点执行
kubeadm join 192.168.184.120:6443 --token ld3dsf.j26i2tjaa2cd330o \
    --discovery-token-ca-cert-hash sha256:24be46f997b113d8cca2b98263ffb082b7c52b3b6698ce2c686df65fd143290b 
```

查看信息

```shell
# 查看节点状态

[root@master calico-39]# kubectl get nodes
NAME     STATUS   ROLES    AGE    VERSION
master   Ready    master   89m    v1.17.2
work1    Ready    <none>   2m7s   v1.17.2

# 查看集群健康状态
[root@master calico-39]# kubectl get cs
NAME                 STATUS    MESSAGE             ERROR
controller-manager   Healthy   ok                  
scheduler            Healthy   ok                  
etcd-0               Healthy   {"health":"true"}   
[root@master calico-39]# 


root@master calico-39]# kubectl cluster-info
Kubernetes master is running at https://192.168.184.120:6443
KubeDNS is running at https://192.168.184.120:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
[root@master calico-39]# 


[root@master calico-39]# kubectl get pods -A
NAMESPACE     NAME                                       READY   STATUS              RESTARTS   AGE
kube-system   calico-kube-controllers-6895d4984b-jltpd   1/1     Running             0          11m
kube-system   calico-node-5jhpb                          1/1     Running             1          5m33s
kube-system   calico-node-kmczk                          1/1     Running             0          11m
kube-system   calico-node-trtcz                          0/1     Init:0/3            0          59s
kube-system   coredns-9d85f5447-dn4k5                    1/1     Running             0          92m
kube-system   coredns-9d85f5447-zjbqr                    1/1     Running             0          92m
kube-system   etcd-master                                1/1     Running             0          93m
kube-system   kube-apiserver-master                      1/1     Running             0          93m
kube-system   kube-controller-manager-master             1/1     Running             2          93m
kube-system   kube-proxy-6xlg2                           0/1     ContainerCreating   0          59s
kube-system   kube-proxy-f6l7h                           1/1     Running             0          5m33s
kube-system   kube-proxy-hvlp9                           1/1     Running             0          92m
kube-system   kube-scheduler-master                      1/1     Running             2          93m
```

