## CentOS7.5下安装、配置mariadb

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

## Windows下MySQL的安装

#### 官网下载：https://dev.mysql.com/downloads/mysql/

#### 解压下载好的压缩包放到你想要放的目录,用管理员打开CMD，切换到MySql的解压目录下的bin目录.
#### 再输入命令mysqld --initialize --console来初始化数据库，并记录红色标注的字符，这是随机生成的密码：
![image](https://user-images.githubusercontent.com/5710245/127423686-c4815709-678e-4f4a-970a-ec1d4d357f33.png)

#### 输入mysqld -install将mysql安装为Windows的服务
![image](https://user-images.githubusercontent.com/5710245/127423866-0ad44ad3-0269-41b2-ba57-470047b51625.png)

#### 输入命令net start mysql或sc start mysql启动mysql服务
![image](https://user-images.githubusercontent.com/5710245/127423891-78d70d74-a0d3-444c-afa4-ab18e94c3598.png)

#### 输入mysql -u root -p来登陆数据库，并输入前面记录的临时密码：
![image](https://user-images.githubusercontent.com/5710245/127423933-d9c69e98-13f5-4105-bda6-9832657824fe.png)

#### 登陆成功后输入命令alter user 'root'@'localhost' identified by '想要设置的密码';将原来复杂的密码修改为自己的密码，并输入commit;提交：
![image](https://user-images.githubusercontent.com/5710245/127423953-0a394e57-b4fd-4179-bdae-b6781249f6c0.png)

#### 输入quit退出数据库，再次登陆数据库，这时我们用的是我们修改后的密码登陆的：
#### 最后我们将Mysql的bin目录配置到环境变量中，方便我们下次启动，而不用切换路径：
![image](https://user-images.githubusercontent.com/5710245/127423976-43c22268-71a8-41dd-b7c0-20d9df2338af.png)

#### 停止MySQL服务：net stop mysqld或sc stop mysqld
#### 删除MySQL服务：sc delete mysqld或mysqld -remove（需先停止服务）

