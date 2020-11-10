# [Docker 环境部署 RabbitMQ](https://www.cnblogs.com/frank-zhang/p/13523926.html)

### 镜像安装

- [官方镜像地址](https://registry.hub.docker.com/_/rabbitmq/) `https://registry.hub.docker.com/_/rabbitmq/`
- 下载 3.7.28 management 镜像
    - `docker pull rabbitmq:3.7.28-management` 有管理后台
- 环境变量
    - `RABBITMQ_DEFAULT_USER` 用户名
    - `RABBITMQ_DEFAULT_PASS` 密码
    - `RABBITMQ_DEFAULT_VHOST` default vhost
- 创建容器并运行
    - 设置默认账户密码
        - `docker run -d --hostname my-rabbit --name some-rabbit -e RABBITMQ_DEFAULT_USER=user -e RABBITMQ_DEFAULT_PASS=password rabbitmq:3.7.28-management`
    - 设置默认 vhost
        - `ocker run -d --hostname my-rabbit --name some-rabbit -e RABBITMQ_DEFAULT_VHOST=my_vhost rabbitmq:3.7.28-management`
    - 通用命令
        - `docker run -d --hostname rabbitmq --name rabbitmq -e RABBITMQ_DEFAULT_USER=admin -e RABBITMQ_DEFAULT_PASS=admin -e RABBITMQ_DEFAULT_VHOST=my_vhost -p 15672:15672 -p 5672:5672 rabbitmq:3.7.28-management`
- 主要端口
    - 4369 erlang 发现口
    - 5672 client 端通信口
    - 15672 管理界面 ui 端口
    - 25672 server 间内部通信口