centos7安装mysql（完整）


一、下载安装包
官网5.7版本：https://cdn.mysql.com//Downloads/MySQL-5.7/mysql-5.7.29-1.el7.x86_64.rpm-bundle.tar


二、安装
使用tar命令解压

tar -xvf mysql-5.7.29-1.el7.x86_64.rpm-bundle.tar


安装新版mysql前，需将系统自带的mariadb-lib卸载

rpm -qa|grep mariadb

mariadb-libs-5.5.60-1.el7_5.x86_64
删除自带的mariadb

rpm -e --nodeps mariadb-libs-5.5.60-1.el7_5.x86_64

为了避免出现权限问题，给mysql解压文件所在目录赋予最大权限

chmod -R 777 mysql


严格按照顺序安装：mysql-community-common-5.7.29-1.el7.x86_64.rpm、mysql-community-libs-5.7.29-1.el7.x86_64.rpm、mysql-community-client-5.7.29-1.el7.x86_64.rpm、mysql-community-server-5.7.29-1.el7.x86_64.rpm这四个包

rpm -ivh mysql-community-common-5.7.29-1.el7.x86_64.rpm

rpm -ivh mysql-community-libs-5.7.29-1.el7.x86_64.rpm

rpm -ivh mysql-community-client-5.7.29-1.el7.x86_64.rpm

rpm -ivh mysql-community-server-5.7.29-1.el7.x86_64.rpm

rpm -ivh mysql-community-libs-compat-5.7.29-1.el7.x86_64.rpm

rpm -ivh mysql-community-devel-5.7.29-1.el7.x86_64.rpm

如果安装过程中出现这个错误就在后面添加 --force --nodeps，这可能是由于yum安装了旧版本的GPG keys造成的


三、配置数据库
vim /etc/my.cnf

添加这三行

skip-grant-tables
character_set_server=utf8
init_connect='SET NAMES utf8'



skip-grant-tables：跳过登录验证
character_set_server=utf8：设置默认字符集UTF-8
init_connect=‘SET NAMES utf8’：设置默认字符集UTF-8

四、启动mysql服务
设置开机启动

systemctl start mysqld.service

启动mysq

mysql

五、设置密码和开启远程登录
5.1先设置一个简单的密码
update mysql.user set authentication_string=password('123456') where user='root';


立即生效

flush privileges;

退出mysql并停止mysql服务

systemctl stop  mysqld.service

编辑my.cnf配置文件将：skip-grant-tables这一行注释掉

重启mysql服务

systemctl start mysqld.service

再次登录mysql

mysql -uroot -p123456

输入命令错误再执行重设密码

set password=password('123456');

5.2设置密码策略（这步可以跳过）
如果想要设置简单一点的密码就要设置密码策略，否则设置简单的密码会出错
查看密码策略

 SHOW VARIABLES LIKE 'validate_password%'; 



1）、validate_password_length 固定密码的总长度；
2）、validate_password_dictionary_file 指定密码验证的文件路径；
3）、validate_password_mixed_case_count 整个密码中至少要包含大/小写字母的总个数；
4）、validate_password_number_count 整个密码中至少要包含阿拉伯数字的个数；
5）、validate_password_policy 指定密码的强度验证等级，默认为 MEDIUM；

设置密码的验证强度等级，设置 validate_password_policy 的全局参数为 LOW

set global validate_password_policy=LOW;


只要设置密码的长度小于 3 ，都将自动设值为 4

 set global validate_password_length=4;



5.3开放端口
开放3306端口

firewall-cmd --zone=public --add-port=3306/tcp --permanent

–zone #作用域
–add-port=80/tcp #添加端口，格式为：端口/通讯协议
–permanent #永久生效，没有此参数重启后失效

重启防火墙

firewall-cmd --reload

5.4开启远程登录
grant all privileges on *.* to 'root'@'%' identified by '123456' with grant option;

by后面的就是远程登录密码，远程登录密码可以和用户密码不一样



三 使用docker安装mysql
step1：使用docker pull 拉取docker hub仓库中mysql镜像 （注意备注）

mysql镜像版本如下：



docker pull mysql:8.0
备注：docker pull 默认到官方参考拉取  mysql：8.0   镜像名：镜像tag     

         官方地址为国外地址安装缓慢可进行配置

step2：修改镜像文件拉取地址为ustc 

vi /etc/docker/daemon.json
备注：insert 编辑内容如下      并esc  输入:wq 保存退出  

{
"registry-mirrors":["https://docker.mirrors.ustc.edu.cn"]
}
step3：重启docker   重新进行step1 安装速度变快

systemctl restart docker
step4：查看镜像安装情况

docker images


step5：启动docker中镜像

docker run --name mysql8.0 -p 3307:3306 -e MYSQL_ROOT_PASSWORD=root -d mysql:8.0
备注 ：--name 服务启动别名设置  -p端口映射 宿主机端口：镜像运行端口  -d 镜像名：tag 使用守护进程模式启动 -e：设置root帐号密码

step6：查看运行的镜像

docker ps -a


step7：进入镜像 运行mysql

docker exec -it mysql8.0 /bin/bash
备注：exec docker进入容器命令   -it 容器中服务别名 /bin/bash   表示命令行模式  与 -d 后台守护进行模式启动 形成两种运行方式   
进入容器中如图所示变化如下



cd /usr/bin
mysql -u root -p
备注： 在容器中进入用户目录启动mysql 输入密码连接成功



