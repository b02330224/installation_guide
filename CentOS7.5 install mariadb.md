# CentOS7.5下安装、配置mariadb

#### 1、安装mariadb

[root@VM_39_157_centos bin]# yum -y install mariadb-server

#### 2、启动mariadb服务

[root@VM_39_157_centos bin]# systemctl start mariadb.service

#### 3、设置开机启动

[root@VM_39_157_centos bin]# systemctl enable mariadb.service

#### 4、修改密码

[root@VM_39_157_centos bin]# mysqladmin -uroot password '123456'

#### 5、登录mariadb，查看MySql相关的配置

[root@VM_39_157_centos bin]# mysql -uroot -p123456

MariaDB [(none)]> \s

#### 6、更改字符集

[root@VM_39_157_centos bin]# vim /etc/my.cnf

在my.cnf空行处添加：character-set-server=utf8

#### 7、重启mariadb服务

systemctl restart mariadb.service

到此安装以及简单配置就已经完成了



#### 8、允许远程连接mariadb数据库

```shell
mysql -uroot -p123456

mysql> GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' IDENTIFIED BY '123456' WITH GRANT OPTION;  注：root是登陆数据库的用户，123456是登陆数据库的密码，*就是意味着任何来源任何主机
mysql> FLUSH PRIVILEGES; 刷新使之生效
mysql> quit
service mariadb restart 重新启动
```

