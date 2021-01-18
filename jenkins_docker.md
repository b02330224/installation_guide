

### 下载文件

apache-maven-3.5.0-bin.tar.gz

apache-tomcat-8.5.59.tar.gz

jenkins.war

```shell
#解压
tar -xzvf apache-tomcat-8.5.59.tar.gz 
tar -xzvf apache-maven-3.5.0-bin.tar.gz

mv apache-maven-3.5.0  /usr/local/maven
 mv apache-tomcat-8.5.59 /usr/local/tomcat_jenkins
cd /usr/local/tomcat_jenkins/webapps/
rm -rf *

#解压war包
unzip  /home/wj/jenkins/jenkins.war  -d ROOT
```

### 启动jenkins

```shell
[root@slave2 bin]# cd /usr/local/tomcat_jenkins/bin
[root@slave2 bin]# ./startup.sh 
Using CATALINA_BASE:   /usr/local/tomcat_jenkins
Using CATALINA_HOME:   /usr/local/tomcat_jenkins
Using CATALINA_TMPDIR: /usr/local/tomcat_jenkins/temp
Using JRE_HOME:        /usr/java/jdk1.8.0_181-cloudera
Using CLASSPATH:       /usr/local/tomcat_jenkins/bin/bootstrap.jar:/usr/local/tomcat_jenkins/bin/tomcat-juli.jar
Using CATALINA_OPTS:   
Tomcat started.

#查看日志中的密码
[root@slave2 logs]# cd /usr/local/tomcat_jenkins/logs
[root@slave2 logs]# tail -100 catalina.out

.......
18-Nov-2020 11:57:12.906 INFO [Finalizing set up] jenkins.install.SetupWizard.init 

*************************************************************
*************************************************************
*************************************************************

Jenkins initial setup is required. An admin user has been created and a password generated.
Please use the following password to proceed to installation:

f0ffceccb55e4d16bfcba37f1d7f15ce

This may also be found at: /root/.jenkins/secrets/initialAdminPassword

*************************************************************
*************************************************************
*************************************************************

18-Nov-2020 11:57:20.964 INFO [pool-6-thread-91] jenkins.InitReactorRunner$1.onAttained Completed initialization
18-Nov-2020 11:57:20.988 INFO [Jenkins initialization thread] hudson.WebAppMain$3.run Jenkins is fully up and running
18-Nov-2020 11:57:21.687 INFO [Download metadata thread] hudson.model.DownloadService$Downloadable.load Obtained the updated data file for hudson.tasks.Maven.MavenInstaller
18-Nov-2020 11:57:21.688 INFO [Download metadata thread] hudson.util.Retrier.start Performed the action check updates server successfully at the attempt #1
18-Nov-2020 11:57:21.694 INFO [Download metadata thread] hudson.model.AsyncPeriodicWork.lambda$doRun$0 Finished Download metadata. 9,833 ms

```

登录jenkins

http://192.168.6.126:8888/

```shell
[root@slave2 logs]# cat /root/.jenkins/secrets/initialAdminPassword
f0ffceccb55e4d16bfcba37f1d7f15ce
```





