# prometheus监控

参考视频：

3小时搞定Prometheus普罗米修斯监控系统

https://www.bilibili.com/video/BV16J411z7SQ?p=9

pdf: panjinying:/架构/

## 机器准备

1. 静态ip(要求能上外网) 

prometheus:

[root@localhost ~]# cat /etc/sysconfig/network-scripts/ifcfg-ens33 
TYPE=Ethernet
PROXY_METHOD=none
BROWSER_ONLY=no
BOOTPROTO=static
DEFROUTE=yes
IPV4_FAILURE_FATAL=no
IPV6INIT=yes
IPV6_AUTOCONF=yes
IPV6_DEFROUTE=yes
IPV6_FAILURE_FATAL=no
IPV6_ADDR_GEN_MODE=stable-privacy
NAME=ens33
UUID=7bcc9077-71dc-4e12-bb38-3c9de13ba211
DEVICE=ens33
ONBOOT=yes
IPADDR=192.168.184.211
NETMASK=255.255.255.0
DNS1=8.8.8.8
DNS2=114.114.114.114
GATEWAY=192.168.184.2



grafana:

[root@localhost ~]#  cat /etc/sysconfig/network-scripts/ifcfg-ens33 
TYPE=Ethernet
PROXY_METHOD=none
BROWSER_ONLY=no
BOOTPROTO=static
DEFROUTE=yes
IPV4_FAILURE_FATAL=no
IPV6INIT=yes
IPV6_AUTOCONF=yes
IPV6_DEFROUTE=yes
IPV6_FAILURE_FATAL=no
IPV6_ADDR_GEN_MODE=stable-privacy
NAME=ens33
UUID=7bcc9077-71dc-4e12-bb38-3c9de13b4212
DEVICE=ens33
ONBOOT=yes
IPADDR=192.168.184.212
NETMASK=255.255.255.0
DNS1=8.8.8.8
DNS2=114.114.114.114
GATEWAY=192.168.184.2



agent:

[root@localhost ~]#  cat /etc/sysconfig/network-scripts/ifcfg-ens33 
TYPE=Ethernet
PROXY_METHOD=none
BROWSER_ONLY=no
BOOTPROTO=static
DEFROUTE=yes
IPV4_FAILURE_FATAL=no
IPV6INIT=yes
IPV6_AUTOCONF=yes
IPV6_DEFROUTE=yes
IPV6_FAILURE_FATAL=no
IPV6_ADDR_GEN_MODE=stable-privacy
NAME=ens33
UUID=7bcc9077-71dc-4e12-bb38-3c9de13b4213
DEVICE=ens33
ONBOOT=yes
IPADDR=192.168.184.213
NETMASK=255.255.255.0
DNS1=8.8.8.8
DNS2=114.114.114.114

GATEWAY=192.168.184.2

2.设置主机名

```shell
各自配置好主机名 
# hostnamectl set-hostname --static prometheus.cluster.com 
三台都互相绑定IP与主机名 
# vim /etc/hosts            
192.168.184.211  prometheus.cluster.com
192.168.184.213  agent.cluster.com
192.168.184.212  grafana.cluster.com
```



3.时间同步(时间同步一定要确认一下) 

yum install update -y

ntpdate cn.ntp.org.cn



4. 关闭防火墙,selinux
 systemctl stop firewalld 
 systemctl disable firewalld 
 iptables -F

## 安装prometheus

1、下载包，上传至服务器
从 https://prometheus.io/download/ 下载相应版本，安装到服务器上
官网提供的是二进制版，解压就能用，不需要编译

2. 解压

[root@server ~]# tar xf prometheus-2.5.0.linux-amd64.tar.gz -C /usr/local/

 [root@server ~]# mv /usr/local/prometheus-2.5.0.linux-amd64/ /usr/local/prometheus



3.直接使用默认配置文件启动 

[root@server ~]# /usr/local/prometheus/prometheus --config.file="/usr/local/prometheus/prometheus.yml" &



确认端口(9090) 

[root@server ~]# lsof -i:9090

4.打开web
通过浏览器访问http://服务器IP:9090就可以访问到prometheus的主界面

## 监控LINUX主机



① 在远程linux主机(被监控端agent1)上安装node_exporter组件
下载地址: https://prometheus.io/download/

[root@agent1 ~]# tar xf node_exporter-0.16.0.linuxamd64.tar.gz -C /usr/local/ 

[root@agent1 ~]# mv /usr/local/node_exporter-0.16.0.linuxamd64/ /usr/local/node_exporter
里面就一个启动命令node_exporter,可以直接使用此命令启动 

[root@agent1 ~]# ls /usr/local/node_exporter/

 LICENSE  node_exporter  NOTICE 

[root@agent1 ~]# nohup /usr/local/node_exporter/node_exporter &   

确认端口(9100)

 [root@agent1 ~]# lsof -i:9100



扩展: nohup命令: 如果把启动node_exporter的终端给关闭,那么进程也会 随之关闭。nohup命令会帮你解决这个问题。
② 通过浏览器访问http://被监控端IP:9100/metrics就可以查看到 node_exporter在被监控端收集的监控信息



③ 回到prometheus服务器的配置文件里添加被监控机器的配置段

```shell
在主配置文件最后加上下面三行
[root@server ~]# vim /usr/local/prometheus/prometheus.yml 
- job_name: 'agent1'                   # 取一个job名称来代 表被监控的机器   
	static_configs:   
	- targets: ['10.1.1.14:9100']        # 这里改成被监控机器 的IP，后面端口接9100
改完配置文件后,重启服务 
[root@server ~]# pkill prometheus 
[root@server ~]# lsof -i:9090           # 确认端口没有进程占 用 
[root@server ~]# /usr/local/prometheus/prometheus --config.file="/usr/local/prometheus/prometheus.yml" & 
[root@server ~]# lsof -i:9090           # 确认端口被占用，说 明重启成功
```



④ 回到web管理界面 --》点Status --》点Targets --》可以看到多了一台监 控目标



## 监控远程MySQL

① 在被管理机agent1上安装mysqld_exporter组件
下载地址: https://prometheus.io/download/

```shell
安装mysqld_exporter组件 
[root@agent1 ~]# tar xf mysqld_exporter-0.11.0.linuxamd64.tar.gz -C /usr/local/ 
[root@agent1 ~]# mv /usr/local/mysqld_exporter0.11.0.linux-amd64/ /usr/local/mysqld_exporter 
[root@agent1 ~]# ls /usr/local/mysqld_exporter/ LICENSE  mysqld_exporter  NOTICE
安装mariadb数据库,并授权 
[root@agent1 ~]# yum install mariadb\* -y 
[root@agent1 ~]# systemctl restart mariadb 
[root@agent1 ~]# systemctl enable mariadb 
[root@agent1 ~]# mysql
MariaDB [(none)]> grant select,replication client,process ON *.* to 'mysql_monitor'@'localhost' identified by '123'; 
(注意:授权ip为localhost，因为不是prometheus服务器来直接找mariadb 获取数据，而是prometheus服务器找mysql_exporter,mysql_exporter 再找mariadb。所以这个localhost是指的mysql_exporter的IP)

MariaDB [(none)]> flush privileges;
MariaDB [(none)]> quit
创建一个mariadb配置文件，写上连接的用户名与密码(和上面的授权的用户名 和密码要对应) 
[root@agent1 ~]# vim /usr/local/mysqld_exporter/.my.cnf 
[client] 
user=mysql_monitor 
password=123
启动mysqld_exporter 
[root@agent1 ~]# nohup /usr/local/mysqld_exporter/mysqld_exporter --config.my-cnf=/usr/local/mysqld_exporter/.my.cnf &
确认端口(9104) 
[root@agent1 ~]# lsof -i:9104


```

② 回到prometheus服务器的配置文件里添加被监控的mariadb的配置段



```shell
在主配置文件最后再加上下面三行
[root@server ~]# vim /usr/local/prometheus/prometheus.yml 
- job_name: 'agent1_mariadb'                   # 取一个job 名称来代表被监控的mariadb  
  static_configs:   
  - targets: ['10.1.1.14:9104']                # 这里改成 被监控机器的IP，后面端口接9104
改完配置文件后,重启服务 
[root@server ~]# pkill prometheus 
[root@server ~]# lsof -i:9090 
[root@server ~]# /usr/local/prometheus/prometheus --config.file="/usr/local/prometheus/prometheus.yml" & [root@server ~]# lsof -i:9090
```



③ 回到web管理界面 --》点Status --》点Targets --》可以看到监控 mariadb了



## 使用Grafana连接Prometheus 

① 在grafana服务器上安装grafana
下载地址:https://grafana.com/grafana/download 

```powershell
我这里用视频准备的安装软件上传包到服务器
[root@grafana ~]#yum install /opt/grafana-5.3.41.x86_64.rpm 
启动服务 
[root@grafana ~]# systemctl start grafana-server 
[root@grafana ~]# systemctl enable grafana-server 
确认端口(3000) 
[root@grafana ~]# ss -talnpt|grep 3000
```





## Grafana图形显示MySQL监控数据 

① 在grafana上修改配置文件,并下载安装mysql监控的dashboard（包含 相关json文件，这些json文件可以看作是开发人员开发的一个监控模板)
参考网址: https://github.com/percona/grafana-dashboards

```shell
在grafana配置文件里最后加上以下三行 [root@grafana ~]# vim /etc/grafana/grafana.ini [dashboards.json] 
enabled = true 
path = /var/lib/grafana/dashboards
[root@grafana ~]# cd /var/lib/grafana/ 
[root@grafana grafana]# git clone https://github.com/percona/grafana-dashboards.git 
[root@grafana grafana]# cp -r grafanadashboards/dashboards/ /var/lib/grafana/ 重启grafana服务 
[root@grafana grafana]# systemctl restart grafana-server
```

② 在grafana图形界面导入相关json文件

![image-20201027223858247](D:\project\installation_doc\prometheus监控\image-20201027223858247.png)

③ 点import导入后,报prometheus数据源找不到,因为这些json文件里默认 要找的就是叫Prometheus的数据源，但我们前面建立的数据源却是叫 prometheus_data(坑啊)
那么请自行把原来的prometheus_data源改名为Prometheus即可(注意: 第一个字母P是大写)
然后再回去刷新,就有数据了(如下图所示)  

![image-20201027223915006](D:\project\installation_doc\prometheus监控\image-20201027223915006.png)



![img](D:\project\installation_doc\prometheus监控\2020011315383743.png)

## Grafana+onealert报警

### 1、先在onealert里添加grafana应用(申请onealert账号)



[https://caweb.aiops.com/](https://caweb.aiops.com/#/)

![img](D:\project\installation_doc\prometheus监控\1.png)


![img](D:\project\installation_doc\prometheus监控\2.png)

![img](D:\project\installation_doc\prometheus监控\3.png)

### 2、在Grafana中配置Webhook URL

1、在Grafana中创建Notification channel，选择类型为Webhook；

![img](D:\project\installation_doc\prometheus监控\4.png)

2、推荐选中Send on all alerts和Include image，Cloud Alert体验更佳；

3、将第一步中生成的Webhook URL填入Webhook settings Url；

4、Http Method选择POST；

5、Send Test&Save；

![img](D:\project\installation_doc\prometheus监控\5.png)

![img](D:\project\installation_doc\prometheus监控\20200113162840747.png)

### 现在可以去设置一个报警来测试了(这里以我们前面加的cpu负载监控来 做测试)

![img](D:\project\installation_doc\prometheus监控\6.png)

配置

![img](D:\project\installation_doc\prometheus监控\7.png)

![img](D:\project\installation_doc\prometheus监控\8.png)

保存后就可以测试了

![img](D:\project\installation_doc\prometheus监控\9.png)

如果node1上的cpu负载还没有到0.5，你可以试试0.1,或者运行一些程序 把node1负载调大。最终能测试报警成功

![img](D:\project\installation_doc\prometheus监控\10.png)

模拟cpu负载

agent机器上执行以下命令，使cpu load 升高

```
cat /dev/urandom | md5sum
```

 

## 测试mysql链接数报警

![img](D:\project\installation_doc\prometheus监控\20200113164810425.png)

![img](D:\project\installation_doc\prometheus监控\20200113164821235.png)

![img](D:\project\installation_doc\prometheus监控\20200113164837652.png)





![img](D:\project\installation_doc\prometheus监控\20200113164839384.png)





![img](D:\project\installation_doc\prometheus监控\20200113164902506.png)





## 总结报警不成功的可能原因

- 各服务器之间时间不同步，这样时序数据会出问题，也会造成报警出问 题
- 必须写通知内容，留空内容是不会发报警的
- 修改完报警配置后，记得要点右上角的保存
- 保存配置后，需要由OK状态变为alerting状态才会报警(也就是说，你 配置保存后，就已经是alerting状态是不会报警的)
- grafana与onealert通信有问题



## 扩展

prometheus目前还在发展中，很多相应的监控都需要开发。但在官网的 dashboard库中,也有一些官方和社区开发人员开发的dashboard可以直接 拿来用。

![img](D:\project\installation_doc\prometheus监控\20200113165033603.png)

![img](D:\project\installation_doc\prometheus监控\20200113165056782.png)

![img](D:\project\installation_doc\prometheus监控\20200113165113140.png)

