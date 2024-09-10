---
layout:       post
title:        "Dockerfile 制作 ActiveMQ 镜像"
author:       "Vito"
header-style: text
catalog:      true
tags:
    - Docker
    - ActiveMQ
---

### Dockerfile
```dockerfile
FROM alpine:3.16.0

LABEL org.opencontainers.image.authors="hczhch@yeah.net"

# ENV LANG=C.UTF-8
ENV LANG='zh_CN.UTF-8'

# 更换为国内源
# 通过 cat /etc/apk/repositories 命令可查看默认源（主要是查看源的版本）
RUN echo "https://mirrors.aliyun.com/alpine/v3.16/main" > /etc/apk/repositories \
    && echo "https://mirrors.aliyun.com/alpine/v3.16/community" >> /etc/apk/repositories

# 时区
RUN apk --no-cache add tzdata \
    && rm -f /etc/localtime \
    && ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime \
    && echo "Asia/Shanghai" > /etc/timezone

# JDK ，注意：这里用的是适用于 Alpine Linux 的基于 musl 编译的 jdk ，而非依赖于 glibc 的 jdk
ADD zulu8.62.0.19-ca-jdk8.0.332-linux_musl_x64.tar.gz /opt/jdk/
ENV JAVA_HOME /opt/jdk/zulu8.62.0.19-ca-jdk8.0.332-linux_musl_x64
ENV JRE_HOME $JAVA_HOME/jre
ENV PATH $JAVA_HOME/bin:$PATH

EXPOSE 8161/tcp
EXPOSE 61616/tcp
EXPOSE 5672/tcp
EXPOSE 61613/tcp
EXPOSE 1883/tcp
EXPOSE 61614/tcp

ADD apache-activemq-5.16.5-bin.tar.gz /opt/mq/

# && tail -f /dev/null 的作用是防止容器自动退出
ENTRYPOINT ["/opt/mq/apache-activemq-5.16.5/bin/activemq","start && tail -f /dev/null"]
```

### 使用
```shell
# 生成镜像
docker build -t vito-activemq:5.16.5-alpine3.16 .

# 创建临时容器
docker run -d --name activemq5.16 vito-activemq:5.16.5-alpine3.16

# 从容器内拷贝文件到宿主机
mkdir -p /home/vito/docker/activemq5.16/
docker cp activemq5.16:/opt/mq/apache-activemq-5.16.5/conf /home/vito/docker/activemq5.16/
docker cp activemq5.16:/opt/mq/apache-activemq-5.16.5/data /home/vito/docker/activemq5.16/

# 编辑 jetty.xml 将 127.0.0.1 修改为 0.0.0.0
vim /home/vito/docker/activemq5.16/conf/jetty.xml
    <bean id="jettyPort" class="org.apache.activemq.web.WebConsolePort" init-method="start">
             <!-- the default port number for the web console -->
        <property name="host" value="0.0.0.0"/>
        <property name="port" value="8161"/>
    </bean>

# 删除临时容器
docker rm -f activemq5.16

# 创建正式容器
docker run -d -p 8161:8161 -p 61616:61616 --restart always \
  --privileged=true \
  -v /home/vito/docker/activemq5.16/conf:/opt/mq/apache-activemq-5.16.5/conf \
  -v /home/vito/docker/activemq5.16/data:/opt/mq/apache-activemq-5.16.5/data \
  --name activemq5.16 vito-activemq:5.16.5-alpine3.16
```
