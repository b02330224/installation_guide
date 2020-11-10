# Harbor搭建

版权

## 一、硬件要求

| 硬件资源 | 最低配置 | 推荐配置 |
| -------- | -------- | -------- |
| 处理器   | 2        | 4        |
| CPU      | 4        | 8        |
| 硬件     | 40       | 160      |

## 二、软件要求

| 软件           | 版本                   | 描述                                                         |
| -------------- | ---------------------- | ------------------------------------------------------------ |
| Docker-engine  | v17.06.1-ce 或更高版本 | 有关安装说明，请参阅 [Docker Engine文档](https://docs.docker.com/engine/installation/)。 |
| Docker-compose | v1.18.0 或更高版本     | 有关安装说明，请参阅 [Docker Compose](https://docs.docker.com/compose/install/)文档。 |
| Openssl        | 最新版                 | 用于生成Harbor的证书和密钥                                   |

## 三、下载Harbor安装包

官方下载地址：https://github.com/goharbor/harbor/releases
![在这里插入图片描述](D:\code\python\installation_guide\install Harbor\20200630093650383.png)
Harbor官方分别提供了在线版（不含组件镜像，相对较小）和离线版（包含组件镜像，相对较大），这里我们下载离线版的。

```bash
mkdir /home/harbor & cd /home/harbor #安装目录自己指定
wget https://github.com/goharbor/harbor/releases/download/v1.10.1/harbor-offline-installer-v1.10.3.tgz 
12
```

由于github下载非常非常的慢，在此提供百度网盘下载地址 harbor-offline-installer-v1.10.1.tgz （提取码：9anp）

## 四、生成Https证书

官方指导地址：https://goharbor.io/docs/1.10/install-config/configure-https/

```bash
# 创建证书目录，并赋予权限
mkdir -p /data/cert && chmod -R 777 /data/cert && cd /data/cert
# 生成私钥，需要设置密码
openssl genrsa -des3 -out harbor.key 2048

# 生成CA证书，需要输入密码
openssl req -sha512 -new \
    -subj "/C=CN/ST=Beijing/L=Beijing/O=example/OU=Personal/CN=yourdomain.com" \
    -key harbor.key \
    -out harbor.csr

# 备份证书
cp harbor.key harbor.key.org

# 退掉私钥密码，以便docker访问（也可以参考官方进行双向认证）
openssl rsa -in harbor.key.org -out harbor.key

# 使用证书进行签名
openssl x509 -req -days 365 -in harbor.csr -signkey harbor.key -out harbor.crt
12345678910111213141516171819
```

## 五、安装Harbor

#### 1、解压软件包

```bash
cd /home/harbor # 进入刚创建的安装目录
tar -zxvf harbor.v1.10.1.tar.gz  #解压
12
```

#### 2、解压软件包

编辑harbor.yml，修改hostname、https证书路径、admin密码（可选）
![img](D:\code\python\installation_guide\install Harbor\2020063009591594.png)

#### 3、运行install.sh即可

```bash
./install.sh
1
```

#### 4、部署完成，以https方式访问宿主机地址即可

这边地址是 https://192.168.1.160 （自己安装机器的ip地址）。账户admin 密码就刚上面维护的123456
![在这里插入图片描述](D:\code\python\installation_guide\install Harbor\20200630100328208.png)
至此离线方式安装完成。 在线安装后续补充。

## 五、client 登录仓库时遇到的问题

错误：Error response from daemon: Get https://harbor.op.xxxx.com/v2/: x509: certificate signed by unknown authority Web程序

最终解决方案如下：

A：在需要登陆的docker client端修改 /usr/lib/systemd/system/docker.service 文件，在里面修改ExecStart那一行，增加–insecure-registry=192.168.1.160（你的ip或者域名），然后重启docker （systemctl daemon-reload systemctl restart docker）
![img](D:\code\python\installation_guide\install Harbor\20200630174706556.png)

B:在harbor服务器端修改 /etc/docker/daemon.json（如果没有这个文件，自己建），修改后，同样运行 （systemctl daemon-reload systemctl restart docker）我的修改内容如下：
![img](D:\code\python\installation_guide\install Harbor\20200630175728133.png)
注意：重启docker以后也需要重启Harbor

```bash
  cd /home/Harbor #进入Harbor的安装目录
  docker-compose down #关闭Harbor
 ./prepare
  docker-compose up -d #重启Harbor
1234
```

#### 再执行登陆，成功！

![img](D:\code\python\installation_guide\install Harbor\20200630175453836.png)