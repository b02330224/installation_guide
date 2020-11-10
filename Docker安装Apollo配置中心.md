**一、 准备工作**

1）安装Docker
[安装指南](https://yeasy.gitbooks.io/docker_practice/content/install/)

2）下载Apollo源码

```
git clone https://github.com/ctripcorp/apollo.git
```

然后进入到docker-quick-start 目录下

```
cd apollo/scripts/docker-quick-start
```

**二、启动Apollo配置中心**

执行命令启动服务

```
docker-compose up
```

看到如下日志表示启动成功：

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
apollo-quick-start    | ==== starting service ====
apollo-quick-start    | Service logging file is ./service/apollo-service.log
apollo-quick-start    | Started [51]
...
apollo-quick-start    | Waiting for config service startup......
apollo-quick-start    | Config service started. You may visit http://localhost:8080 for service status now!
apollo-quick-start    | Waiting for admin service startup..
apollo-quick-start    | Admin service started
apollo-quick-start    | ==== starting portal ====
apollo-quick-start    | Portal logging file is ./portal/apollo-portal.log
apollo-quick-start    | Started [259]
apollo-quick-start    | Waiting for portal startup......
apollo-quick-start    | Portal started. You can visit http://localhost:8070 now!
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

涉及到三部分：

1.config service

访问地址： http://localhost:8080

2.Admin service

访问地址： http://localhost:8070

用户名密码：apollo/admin

3.mysql server

localhost:13306，用户名是root，密码为空

4.meta server

为了简化部署，我们实际上会把Config Service、Eureka和Meta Server三个逻辑角色部署在同一个JVM进程中

访问地址： http://localhost:8080


\* 如要查看更多服务的日志，可以通过`docker exec -it apollo-quick-start bash`登录， 然后到`/apollo-quick-start/service`和`/apollo-quick-start/portal`下查看日志信息

**三、启动Demo客户端程序**

```
docker exec -i apollo-quick-start /apollo-quick-start/demo.sh client
```

通过输入配置key，获取配置value;刚启动apollo配置中心会有个默认值timeout我们可以访问下，你可以自行登陆到后台进行各项操作

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
➜  ~ docker exec -i apollo-quick-start /apollo-quick-start/demo.sh client
[apollo-demo][main]2020-04-18 09:25:20,866 INFO  [com.ctrip.framework.foundation.internals.provider.DefaultApplicationProvider] App ID is set to SampleApp by app.id property from /META-INF/app.properties
[apollo-demo][main]2020-04-18 09:25:20,871 INFO  [com.ctrip.framework.foundation.internals.provider.DefaultServerProvider] Environment is set to [dev] by JVM system property 'env'.
[apollo-demo][main]2020-04-18 09:25:20,977 INFO  [com.ctrip.framework.apollo.internals.DefaultMetaServerProvider] Located meta services from apollo.meta configuration: http://localhost:8080!
[apollo-demo][main]2020-04-18 09:25:20,978 INFO  [com.ctrip.framework.apollo.core.MetaDomainConsts] Located meta server address http://localhost:8080 for env DEV from com.ctrip.framework.apollo.internals.DefaultMetaServerProvider
Apollo Config Demo. Please input key to get the value. Input quit to exit.
> timeout
Loading key : timeout with value: 300
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

 

你投入得越多，就能得到越多得价值