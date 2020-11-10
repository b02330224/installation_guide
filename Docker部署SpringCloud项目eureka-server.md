# [docker初体验：Docker部署SpringCloud项目eureka-server](https://www.cnblogs.com/zhiyouwu/p/12058415.html)

## Docker部署SpringCloud项目eureka-server

### 1 创建eureka-server工程

创建父工程cloud-demo，其pom.xml如下：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.0.3.RELEASE</version>
        <relativePath/>
    </parent>

    <groupId>top.flygrk.ishare</groupId>
    <artifactId>cloud-demo</artifactId>
    <packaging>pom</packaging>
    <version>1.0-SNAPSHOT</version>

    <modules>
        <module>cloud-eureka-server</module>
    </modules>

    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
        <java.version>1.8</java.version>
        <spring-cloud.version>Finchley.RELEASE</spring-cloud.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-dependencies</artifactId>
                <version>${spring-cloud.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>
```

创建cloud-eureka-server模块，其格式如下：

![file](D:\code\python\installation_guide\Docker部署SpringCloud项目eureka-server\851477-20191218101442915-1084715346.png)

pom.xml配置如下：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <parent>
        <artifactId>cloud-demo</artifactId>
        <groupId>top.flygrk.ishare</groupId>
        <version>1.0-SNAPSHOT</version>
    </parent>
    <modelVersion>4.0.0</modelVersion>

    <artifactId>cloud-eureka-server</artifactId>
    <version>1.0-SNAPSHOT</version>
    <packaging>jar</packaging>

    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
        </dependency>
    </dependencies>


    <build>
        <finalName>eureka-server</finalName>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
            <!-- tag::plugin[] -->
            <plugin>
                <groupId>com.spotify</groupId>
                <artifactId>docker-maven-plugin</artifactId>
                <version>0.4.13</version>
                <configuration>
                    <imageName>/test/${project.artifactId}</imageName>
                    <dockerDirectory>src/main/docker</dockerDirectory>
                    <resources>
                        <resource>
                            <targetPath>/</targetPath>
                            <directory>${project.build.directory}</directory>
                            <include>${project.build.finalName}.jar</include>
                        </resource>
                    </resources>
                </configuration>
            </plugin>
            <!-- end::plugin[] -->
        </plugins>
    </build>

</project>
```

### 2 eureka-server工程打包

使用maven对cloud-eureka-server模块打包，结果如下：

![file](D:\code\python\installation_guide\Docker部署SpringCloud项目eureka-server\851477-20191218101443464-1360769543.png)

### 3 上传eureka-server.jar到服务器

上传eureka-server.jar到服务器路径/app/eureka-server:

![file](D:\code\python\installation_guide\Docker部署SpringCloud项目eureka-server\851477-20191218101443683-2007483424.png)

### 4 编写Dockerfile文件

编写Dockerfile内容如下：

```
FROM java:8
VOLUME /tmp
ADD eureka-server.jar app.jar
EXPOSE 8761
ENTRYPOINT ["java","-Djava.security.egd=file:/dev/./urandom","-jar","/app.jar"]
```

### 5 打包eureka-server镜像

在/app/eureka-server目录下，执行如下命令进行docker镜像打包：

```
docker build -t eureka_v1.0.0 .
```

### 6 查看docker镜像

查看系统上的docker镜像，使用`docker images`命令：

![file](D:\code\python\installation_guide\Docker部署SpringCloud项目eureka-server\851477-20191218101443832-601645360.png)

### 7 运行eurake镜像

使用下面的命令启动eureka镜像：

```
docker run -d --name eureka-server -p 58761:8761 c4bfd23fe99d
```

### 8 查看docker镜像运行情况

使用`docker ps`查看docker镜像运行情况：

```
docker ps -a
```

![file](D:\code\python\installation_guide\Docker部署SpringCloud项目eureka-server\851477-20191218101443974-1425854889.png)

### 8 查看docker eureka-server日志

使用下面的命令查看eureka-server日志：

```
docker logs --since 2019-12-18T09:50:15 eureka-server
```

![file](D:\code\python\installation_guide\Docker部署SpringCloud项目eureka-server\851477-20191218101444407-152813231.png)

------

### Blog:

- 简书： https://www.jianshu.com/u/91378a397ffe
- csdn： https://blog.csdn.net/ZhiyouWu
- 开源中国： https://my.oschina.net/u/3204088
- 掘金： https://juejin.im/user/5b5979efe51d451949094265
- 博客园： https://www.cnblogs.com/zhiyouwu/
- 微信公众号： 源码湾
- 微信： WZY1782357529 (欢迎沟通交流)