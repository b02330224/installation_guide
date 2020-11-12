## gitlab安装配置



![image-20201014222122532](D:\project\installation_doc\GitLab + Jenkins + Sonar Qube持续集成实践\image-20201014222122532.png)

![image-20201014223107252](D:\project\installation_doc\GitLab + Jenkins + Sonar Qube持续集成实践\image-20201014223107252.png)



![image-20201014222932034](D:\project\installation_doc\GitLab + Jenkins + Sonar Qube持续集成实践\image-20201014222932034.png)



关闭普罗米修斯，不然太吃内存

![image-20201017142332868](D:\project\installation_doc\GitLab + Jenkins + Sonar Qube持续集成实践\image-20201017142332868.png)

5.汉化

```shell
1.先查看安装的gitlab版本
cat /opt/gitlab/embedded/service/gitlab-rails/VERSION
2.下载对应的汉化包 
wget https://gitlab.com/xhang/gitlab/-/archive/12-0-stable-zh/gitlab-12-0-stable-zh.tar.gz
3.解压 tar -zxf gitlab-12-0-stable-zh.tar.gz
4.备份源文件 cp -rp /opt/gitlab/embedded/service/gitlab-rails{,.bak_$(date +%F)}

执行前先停止gitlab   gitlab-ctl stop
5.执行汉化：注意前面有个\，主要防止出现一堆覆盖确认提示 
\cp -fr  gitlab-12-0-stable-zh/* /opt/gitlab/embedded/service/gitlab-rails/

6.执行重新配置 gitlab-ctl reconfigure
7.执行重新启动 gitlab-ctl restart
8.测试是否正常访问
```



设置语言

![image-20201017151548627](D:\project\installation_doc\GitLab + Jenkins + Sonar Qube持续集成实践\image-20201017151548627.png)

![image-20201017150818454](D:\project\installation_doc\GitLab + Jenkins + Sonar Qube持续集成实践\image-20201017150818454.png)



## gitlab 用户，组，项目的关系。

![image-20201017151402977](D:\project\installation_doc\GitLab + Jenkins + Sonar Qube持续集成实践\image-20201017151402977.png)

1.创建组

2.创建项目---》项目属于某个组

3.创建用户，设定密码，为用户分配组

4.关闭界面注册功能：

![image-20201017153912664](D:\project\installation_doc\GitLab + Jenkins + Sonar Qube持续集成实践\image-20201017153912664.png)





![image-20201017211413045](D:\project\installation_doc\GitLab + Jenkins + Sonar Qube持续集成实践\image-20201017211413045.png)

## gitlab备份,恢复

![image-20201017213822148](D:\project\installation_doc\GitLab + Jenkins + Sonar Qube持续集成实践\image-20201017213822148.png)

![image-20201017213646666](D:\project\installation_doc\GitLab + Jenkins + Sonar Qube持续集成实践\image-20201017213646666.png)



xuliangwei博客：



![image-20201017213548064](D:\project\installation_doc\GitLab + Jenkins + Sonar Qube持续集成实践\image-20201017213548064.png)





## 安装jenkins



![image-20201017220935948](D:\project\installation_doc\GitLab + Jenkins + Sonar Qube持续集成实践\image-20201017220935948.png)



 vi /etc/sysconfig/jenkins

```shell
## Default:     8080
## ServiceRestart: jenkins
#
# Port Jenkins is listening on.
# Set to -1 to disable
#
JENKINS_PORT="8090"

## Type:        string
## Default:     ""
## ServiceRestart: jenkins
```





export JAVA_HOME="/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.262.b10-0.el7_8.x86_64/jre"



![image-20201017232904742](D:\project\installation_doc\GitLab + Jenkins + Sonar Qube持续集成实践\image-20201017232904742.png)

systemctl daemon-reload

 systemctl restart jenkins 

```shell
[root@centos7 docker]# systemctl daemon-reload
[root@centos7 docker]# systemctl restart jenkins 
[root@centos7 docker]# systemctl status jenkins
● jenkins.service - LSB: Jenkins Automation Server
   Loaded: loaded (/etc/rc.d/init.d/jenkins; bad; vendor preset: disabled)
   Active: active (running) since Tue 2019-12-24 18:16:08 CST; 12s ago
     Docs: man:systemd-sysv-generator(8)
  Process: 23787 ExecStart=/etc/rc.d/init.d/jenkins start (code=exited, status=0/SUCCESS)
   CGroup: /system.slice/jenkins.service
           └─23832 /software/jdk1.8.0_191/bin/java -Dcom.sun.akuma.Daemon=daemonized -Djava.awt.headless=true -DJENKINS_HOME=/var/lib/jenkins -jar /usr/lib/jenkins/jenkins.war --logfile=/v...

Dec 24 18:16:02 centos7 systemd[1]: Starting LSB: Jenkins Automation Server...
Dec 24 18:16:02 centos7 runuser[23792]: pam_unix(runuser:session): session opened for user jenkins by (uid=0)
Dec 24 18:16:08 centos7 runuser[23792]: pam_unix(runuser:session): session closed for user jenkins
Dec 24 18:16:08 centos7 systemd[1]: Started LSB: Jenkins Automation Server.
Dec 24 18:16:08 centos7 jenkins[23787]: Starting Jenkins [  OK  ]
[root@centos7 ~]# lsof -i:8091
COMMAND   PID    USER   FD   TYPE   DEVICE SIZE/OFF NODE NAME
java    23832 jenkins  160u  IPv4 11605726      0t0  TCP *:jamlink (LISTEN)
```





![image-20201017220829885](D:\project\installation_doc\GitLab + Jenkins + Sonar Qube持续集成实践\image-20201017220829885.png)

## 安装插件

![image-20201017222127616](D:\project\installation_doc\GitLab + Jenkins + Sonar Qube持续集成实践\image-20201017222127616.png)



### 更新站点修改

​    由于之前说过，安装Jenkins后首次访问时由于其他原因【具体未知】会产生离线问题。网上找了个遍还是不能解决，所以只能跳过常用插件安装这步。进入Jenkins后再安装这些插件。

​    在安装插件前，先修改“更新站点”信息，如下：

![img](https://img2018.cnblogs.com/blog/1395193/201810/1395193-20181011084629889-2011173302.png)

 

![img](https://img2018.cnblogs.com/blog/1395193/201810/1395193-20181011084635224-90179671.png)

 

​    站点信息从：https://updates.jenkins.io/update-center.json 改为如下地址【三选一即可】

```
1 http://mirror.xmission.com/jenkins/updates/update-center.json   # 推荐
2 http://mirrors.shu.edu.cn/jenkins/updates/current/update-center.json
3 https://mirrors.tuna.tsinghua.edu.cn/jenkins/updates/update-center.json
```

 

​    更改完毕后，最好重启Jenkins。

 

1. 既然是安全检查失败, 那就关闭安全检查呗. 但是我这个rpm的方式来安装的,所以启动方式是`systemctl start jenkins`不是用`java -jar jenkins.war`的方式来启动的, 所以没办法在上面加一句`java -Dhudson.model.DownloadService.noSignatureCheck=true -jar jenkins.war`来取消安全检查, 当时就懵逼了, 我怎么去把这一句[取消安全检查](https://support.cloudbees.com/hc/en-us/articles/115000494608-Why-is-there-Failed-Signature-Check-when-using-update-server-)的命令加上去呢?
2. 后来我去到看了下Jenkins的启动脚本`vi /etc/init.d/jenkins`大致看了下, 没发现怎么加`-Dhudson.model.DownloadService.noSignatureCheck=true`上去, 但是发现他会读取Jenkins的配置文件`vi /etc/sysconfig/jenkins`来启动 当时就在最后面发现了可以加一句上去的机会,在这里
   
   3. vi /etc/sysconfig/jenkins
   
      JENKINS_JAVA_OPTIONS="-Djava.awt.headless=true -Dhudson.model.DownloadService.noSignatureCheck=true"

 systemctl restart jenkins





### centos Jenkins升级

## 1.查看war包所在的目录

> find / -name jenkins.war
>
> ![image-20201018144010325](D:\project\installation_doc\GitLab + Jenkins + Sonar Qube持续集成实践\image-20201018144010325.png)



## 2.停止Jenkins 服务

> systemctl stop jenkins

## 3.备份war包

> cd /usr/lib/jenkins/
> mv /usr/lib/jenkins/jenkins.war /root

## 4.下载最新war包

> wget https://updates.jenkins-ci.org/download/war/2.172/jenkins.war

![image-20201018144155666](D:\project\installation_doc\GitLab + Jenkins + Sonar Qube持续集成实践\image-20201018144155666.png)

## 5.启动Jenkins 服务

> systemctl start jenkins
> netstat -ntap | grep :8080

![image-20201018144052462](D:\project\installation_doc\GitLab + Jenkins + Sonar Qube持续集成实践\image-20201018144052462.png)



## Jenkins汉化



![image-20201018144521699](D:\project\installation_doc\GitLab + Jenkins + Sonar Qube持续集成实践\image-20201018144521699.png)



## 集成maven

 插件安装：Maven Integration plugin，不然在新建job时不会有Maven项目选项

![img](https://img2018.cnblogs.com/blog/1342561/201907/1342561-20190705142101978-1228429219.png)



创建的任务存放位置：

ls  /var/lib/jenkins/workspace/fres-style-test



## Jenkins自动发布HTML到nginx



![image-20201018160038692](D:\project\installation_doc\GitLab + Jenkins + Sonar Qube持续集成实践\image-20201018160038692.png)

![image-20201018160106773](D:\project\installation_doc\GitLab + Jenkins + Sonar Qube持续集成实践\image-20201018160106773.png)





![image-20201018162227699](D:\project\installation_doc\GitLab + Jenkins + Sonar Qube持续集成实践\image-20201018162227699.png)







Jenkins集成maven

![image-20201021224418004](D:\project\installation_doc\GitLab + Jenkins + Sonar Qube持续集成实践\image-20201021224418004.png)



## maven配置







[root@gitlab yum.repos.d]# mvn --version
Apache Maven 3.0.5 (Red Hat 3.0.5-17)
Maven home: /usr/share/maven
Java version: 1.8.0_262, vendor: Oracle Corporation
Java home: /usr/lib/jvm/java-1.8.0-openjdk-1.8.0.262.b10-0.el7_8.x86_64/jre
Default locale: en_US, platform encoding: UTF-8
OS name: "linux", version: "3.10.0-862.el7.x86_64", arch: "amd64", family: "unix"



![image-20201021231410290](D:\project\installation_doc\GitLab + Jenkins + Sonar Qube持续集成实践\image-20201021231410290.png)

![image-20201021231512858](D:\project\installation_doc\GitLab + Jenkins + Sonar Qube持续集成实践\image-20201021231512858.png)

## maven jenkins job



![image-20201021231748978](D:\project\installation_doc\GitLab + Jenkins + Sonar Qube持续集成实践\image-20201021231748978.png)



![image-20201021231648096](D:\project\installation_doc\GitLab + Jenkins + Sonar Qube持续集成实践\image-20201021231648096.png)





## tag方式打包上线



部署的shell脚本



![image-20201023220631398](D:\project\installation_doc\GitLab + Jenkins + Sonar Qube持续集成实践\image-20201023220631398.png)



![image-20201023220832538](D:\project\installation_doc\GitLab + Jenkins + Sonar Qube持续集成实践\image-20201023220832538.png)





![image-20201023215336913](D:\project\installation_doc\GitLab + Jenkins + Sonar Qube持续集成实践\image-20201023215336913.png)



![image-20201023215519720](D:\project\installation_doc\GitLab + Jenkins + Sonar Qube持续集成实践\image-20201023215519720.png)



![image-20201023222606885](D:\project\installation_doc\GitLab + Jenkins + Sonar Qube持续集成实践\image-20201023222606885.png)



构建

![image-20201023222632313](D:\project\installation_doc\GitLab + Jenkins + Sonar Qube持续集成实践\image-20201023222632313.png)

## tag方式回退

![image-20201023223605850](D:\project\installation_doc\GitLab + Jenkins + Sonar Qube持续集成实践\image-20201023223605850.png)





![image-20201023223653921](D:\project\installation_doc\GitLab + Jenkins + Sonar Qube持续集成实践\image-20201023223653921.png)



回退脚本：

![image-20201023225524707](D:\project\installation_doc\GitLab + Jenkins + Sonar Qube持续集成实践\image-20201023225524707.png)



![image-20201023225651617](D:\project\installation_doc\GitLab + Jenkins + Sonar Qube持续集成实践\image-20201023225651617.png)

## sonar安装

（略），视频中直接上传包安装的，不具参考性。

生成token,用于jenkin调用

![image-20201024165202922](D:\project\installation_doc\GitLab + Jenkins + Sonar Qube持续集成实践\image-20201024165202922.png)

![image-20201024165436206](D:\project\installation_doc\GitLab + Jenkins + Sonar Qube持续集成实践\image-20201024165436206.png)



## sonarqube安装插件

（略），视频中直接上传包安装的，不具参考性。

![image-20201024165633634](D:\project\installation_doc\GitLab + Jenkins + Sonar Qube持续集成实践\image-20201024165633634.png)



![image-20201024165746827](D:\project\installation_doc\GitLab + Jenkins + Sonar Qube持续集成实践\image-20201024165746827.png)



![image-20201024165820522](D:\project\installation_doc\GitLab + Jenkins + Sonar Qube持续集成实践\image-20201024165820522.png)

## sonarqube 分析项目



![image-20201024170056153](D:\project\installation_doc\GitLab + Jenkins + Sonar Qube持续集成实践\image-20201024170056153.png)

![image-20201024170913857](D:\project\installation_doc\GitLab + Jenkins + Sonar Qube持续集成实践\image-20201024170913857.png)



![image-20201024170441567](D:\project\installation_doc\GitLab + Jenkins + Sonar Qube持续集成实践\image-20201024170441567.png)

![image-20201024170318922](D:\project\installation_doc\GitLab + Jenkins + Sonar Qube持续集成实践\image-20201024170318922.png)



![image-20201024170412261](D:\project\installation_doc\GitLab + Jenkins + Sonar Qube持续集成实践\image-20201024170412261.png)

![image-20201024170823424](D:\project\installation_doc\GitLab + Jenkins + Sonar Qube持续集成实践\image-20201024170823424.png)

![image-20201024171236213](D:\project\installation_doc\GitLab + Jenkins + Sonar Qube持续集成实践\image-20201024171236213.png)



## sonarqube 分析Java项目

![image-20201024161033809](D:\project\installation_doc\GitLab + Jenkins + Sonar Qube持续集成实践\image-20201024161033809.png)



sonar-scanner方式检测

![image-20201024171414570](D:\project\installation_doc\GitLab + Jenkins + Sonar Qube持续集成实践\image-20201024171414570.png)

## jenkins集成sonarqube

![image-20201024161444346](D:\project\installation_doc\GitLab + Jenkins + Sonar Qube持续集成实践\image-20201024161444346.png)

![image-20201024161531735](D:\project\installation_doc\GitLab + Jenkins + Sonar Qube持续集成实践\image-20201024161531735.png)



![image-20201024161939329](D:\project\installation_doc\GitLab + Jenkins + Sonar Qube持续集成实践\image-20201024161939329.png)

![image-20201024161832185](D:\project\installation_doc\GitLab + Jenkins + Sonar Qube持续集成实践\image-20201024161832185.png)

![image-20201024162614933](D:\project\installation_doc\GitLab + Jenkins + Sonar Qube持续集成实践\image-20201024162614933.png)

![image-20201024162045739](D:\project\installation_doc\GitLab + Jenkins + Sonar Qube持续集成实践\image-20201024162045739.png)



![image-20201024162516157](D:\project\installation_doc\GitLab + Jenkins + Sonar Qube持续集成实践\image-20201024162516157.png)

![image-20201024162221905](D:\project\installation_doc\GitLab + Jenkins + Sonar Qube持续集成实践\image-20201024162221905.png)



## maven项目集成sonarqube

![image-20201024163513485](D:\project\installation_doc\GitLab + Jenkins + Sonar Qube持续集成实践\image-20201024163513485.png)

![image-20201024163013664](D:\project\installation_doc\GitLab + Jenkins + Sonar Qube持续集成实践\image-20201024163013664.png)

![image-20201024163040628](D:\project\installation_doc\GitLab + Jenkins + Sonar Qube持续集成实践\image-20201024163040628.png)



![image-20201024163706280](D:\project\installation_doc\GitLab + Jenkins + Sonar Qube持续集成实践\image-20201024163706280.png)



![image-20201024163635070](D:\project\installation_doc\GitLab + Jenkins + Sonar Qube持续集成实践\image-20201024163635070.png)



## jenkins pipeline

![image-20201024163851054](D:\project\installation_doc\GitLab + Jenkins + Sonar Qube持续集成实践\image-20201024163851054.png)





![image-20201024163906180](D:\project\installation_doc\GitLab + Jenkins + Sonar Qube持续集成实践\image-20201024163906180.png)

![image-20201024164131520](D:\project\installation_doc\GitLab + Jenkins + Sonar Qube持续集成实践\image-20201024164131520.png)



## jenkins pipeline 构建

略



## jenkins pipeline 可视化

![image-20201024173951967](D:\project\installation_doc\GitLab + Jenkins + Sonar Qube持续集成实践\image-20201024173951967.png)

![image-20201024174135773](D:\project\installation_doc\GitLab + Jenkins + Sonar Qube持续集成实践\image-20201024174135773.png)