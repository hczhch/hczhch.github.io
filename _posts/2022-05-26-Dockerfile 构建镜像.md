---
layout:       post
title:        "Dockerfile 构建镜像"
author:       "Vito"
header-style: text
catalog:      true
tags:
    - Docker
    - DockerFile
---


* <https://docs.docker.com/engine/reference/builder>  


* **FROM ：**  基础镜像，当前新镜像是基于哪个镜像的，指定一个已经存在的镜像作为模板，第一条必须是from


* **MAINTAINER ：**  镜像维护者的姓名和邮箱地址
  * eg: MAINTAINER Vito\<hczhch@yeah.net\>

* **LABEL：** MAINTAINER 已废弃，LABEL指令是一个更灵活的版本，它可以设置任何你需要的元数据，并且可以很容易地用 docker inspect 查看。
  * eg: LABEL org.opencontainers.image.authors="hczhch@yeah.net"

* **RUN ：** 容器构建时需要运行的命令，RUN 是在 docker build 生成镜像时运行
  + shell 格式： RUN <命令行命令>
    + eg : RUN yum -y install vim
  + exec 格式： RUN ["可执行文件","参数1","参数2"]


* **EXPOSE ：**  当前容器对外暴露出的端口


* **WORKDIR ：** 指定在创建容器后，终端默认登陆的进来工作目录，一个落脚点


* **USER ：** 指定该镜像以什么样的用户去执行，如果都不指定，默认是 root


* **ENV ：** 用来在构建镜像过程中设置环境变量 
  + ENV JAVA_HOME /opt/jdk
  + ENV JRE_HOME $JAVA_HOME/jre
  + ENV PATH $JAVA_HOME/bin:$PATH
  + 这个环境变量可以在后续的任何 RUN 指令中使用
  + 也可以在其它指令中直接使用这些环境变量，比如：WORKDIR $JAVA_HOME


* **ADD ：** 将宿主机目录下的文件拷贝进镜像且会自动处理URL和解压tar压缩包
  + 镜像内目标路径如果是目录，请以 <font color="#FF0000">/</font> 结尾
  + 宿主机本地文件如果是归档文件（.tar、.tar.gz），会在镜像内被解压缩（镜像内只存在解压缩后的文件）
  + URL 文件不会被解压缩


* **COPY ：** 类似ADD，拷贝文件和目录到镜像中，但是不会自动对压缩包文件解压。


* **VOLUME ：** Dockerfile中 VOLUME 方式挂载到宿主机上的是匿名卷，在宿主机上是自动匿名挂载到 /var/lib/docker/volumes/ 目录下的，且宿主机上的挂载目录名是随机生成的
  + eg: VOLUME /usr/local/v_file/  
        /usr/local/v_file/ 是镜像内路径，
        被挂载到宿主机 /var/lib/docker/volumes/25781410aa39c12ea684fcae7dc4f36cab27e858fcd302d2cf7b0272778d423d/_data/


* **CMD ：** 指定容器启动后的要干的事情
  + Dockerfile 中可以有多个 CMD 指令，但只有最后一个生效
  + 如果 docker run 之后带参数，CMD 会被参数替换掉
    + eg: 例如 tomcat 镜像的 Dockerfile 的最后一条指令是 ： CMD ["catalina.sh", "run"]
    + 运行 `docker run -it tomcat` 命令时，容器内 tomcat 会启动
    + 运行 `docker run -it tomcat /bin/bash` 命令时，容器内 tomcat 不会启动，容器启动时没有运行 tomcat 的 catalina.sh 脚本，运行的是 `/bin/bash`
  + CMD 和 RUN 命令的区别: 
    + **RUN 是在 docker build 构建镜像时运行**
    + **CMD 是在 docker run 运行容器时运行**


* **ENTRYPOINT ：** 也是用来指定一个容器启动时要运行的命令
  + 类似于 CMD 指令，但是 **ENTRYPOINT 不会被 docker run 后面的参数覆盖**
  + 如果 docker run 之后带参数，则参数会被传递给 ENTRYPOINT 指令指定的程序
  + 如果存在多个 ENTRYPOINT 指令，仅最后一个生效
  + ENTRYPOINT 和 CMD 可以同时存在；**ENTRYPOINT 在前，CMD 在后**；**ENTRYPOINT 指定运行的程序，CMD 指定程序的默认参数，参数可被 docker run 后面的参数覆盖**


* **编译文件名称为 Dockerfile 的文本文件生成新镜像：** docker build -t 新镜像名字:TAG .
  * 注意：TAG 后面有一个点


* Rocky Linux  
  eg： 构建一个带 Jdk8 的 Rocky Linux 镜像
  ```dockerfile
  [vito@dockerhost ~]$ cd /home/vito/docker/file
  [vito@dockerhost file]$ mkdir rocky-zulu-jdk8
  [vito@dockerhost file]$ cd rocky-zulu-jdk8
  [vito@dockerhost rocky-zulu-jdk8]$ wget https://cdn.azul.com/zulu/bin/zulu8.62.0.19-ca-jdk8.0.332-linux_x64.tar.gz
  [vito@dockerhost rocky-zulu-jdk8]$ vim Dockerfile
  FROM rockylinux:8.5.20220308
  
  MAINTAINER Vito<hczhch@yeah.net>
  
  # RUN yum -y update && yum -y upgrade
  
  RUN yum -y install glibc \
      # 语言
      && yum -y install glibc-langpack-en glibc-langpack-zh \
      # 时区
      && rm -f /etc/localtime \
      && ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime \
      && echo "Asia/Shanghai" > /etc/timezone
  
  ENV LANG='zh_CN.UTF-8' LANGUAGE='zh_CN:zh:en_US:en' LC_ALL='zh_CN.UTF-8'
  
  # JDK
  ADD zulu8.62.0.19-ca-jdk8.0.332-linux_x64.tar.gz /opt/jdk/
  ENV JAVA_HOME /opt/jdk/zulu8.62.0.19-ca-jdk8.0.332-linux_x64
  ENV JRE_HOME $JAVA_HOME/jre
  ENV PATH $JAVA_HOME/bin:$PATH
  ```
  ```shell
  [vito@dockerhost rocky-zulu-jdk8]$ ls -l
  总用量 105772
  -rw-r--r--. 1 vito vito       619 11月 28 16:24 Dockerfile
  -rw-r--r--. 1 vito vito 108306032  4月 20  2022 zulu8.62.0.19-ca-jdk8.0.332-linux_x64.tar.gz
  
  # docker build -t 新镜像名字:TAG .
  [vito@dockerhost rocky-zulu-jdk8]$ docker build -t rocky-zulu-jdk8:8.0.332 .
  ```


* Photon   
  Photon 是一个开源 Linux 容器主机，针对云原生应用程序、云平台和 VMware 基础架构进行了优化。Photon OS 为有效运行容器提供了安全的运行时环境。
  ```dockerfile
  FROM photon:4.0-20220520
  
  MAINTAINER Vito<hczhch@yeah.net>
  
  ENV LANG='C.UTF-8'
  
  # RUN yum -y update && yum -y upgrade
  
  RUN yum -y install glibc \
      && yum -y install tzdata \
      && rm -f /etc/localtime \
      && ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime \
      && echo "Asia/Shanghai" > /etc/timezone
  
  # JDK
  ADD zulu8.62.0.19-ca-jdk8.0.332-linux_x64.tar.gz /opt/jdk/
  ENV JAVA_HOME /opt/jdk/zulu8.62.0.19-ca-jdk8.0.332-linux_x64
  ENV JRE_HOME $JAVA_HOME/jre
  ENV PATH $JAVA_HOME/bin:$PATH
  ```
  ```shell
  docker build -t photon-zulu-jdk8:8.0.332 .
  ```


* Alpine Linux  
  ps: **Alpine Linux** 是一款独立的、非商业的通用 Linux 发行版，专为追求安全性、简单性和资源效率的用户而设计。  
  Alpine Linux 以体积小、简单、安全而著称，所以作为**基础镜像**是非常好的一个选择，可谓是麻雀虽小但五脏俱全，镜像非常小巧，不到 6M 的大小，特别适合容器打包。
  ```dockerfile
  FROM alpine:3.16.0
  
  MAINTAINER Vito<hczhch@yeah.net>
  
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
  
  # 添加 bash (按需添加)
  RUN apk update && apk upgrade \
      && apk --no-cache add bash bash-doc bash-completion \
      && rm -rf /var/cache/apk/* \
      && /bin/bash
  
  # 集成 glibc (按需添加)（不建议在 Alpine 中使用 glibc）
  RUN ALPINE_GLIBC_BASE_URL="https://github.com/sgerrand/alpine-pkg-glibc/releases/download" \
      && ALPINE_GLIBC_PACKAGE_VERSION="2.35-r0" \
      && ALPINE_GLIBC_BASE_PACKAGE_FILENAME="glibc-$ALPINE_GLIBC_PACKAGE_VERSION.apk" \
      && ALPINE_GLIBC_BIN_PACKAGE_FILENAME="glibc-bin-$ALPINE_GLIBC_PACKAGE_VERSION.apk" \
      && ALPINE_GLIBC_I18N_PACKAGE_FILENAME="glibc-i18n-$ALPINE_GLIBC_PACKAGE_VERSION.apk" \
      && apk --no-cache add --virtual=.build-dependencies wget ca-certificates \
      && wget -q -O /etc/apk/keys/sgerrand.rsa.pub https://alpine-pkgs.sgerrand.com/sgerrand.rsa.pub \
      && wget \
          "$ALPINE_GLIBC_BASE_URL/$ALPINE_GLIBC_PACKAGE_VERSION/$ALPINE_GLIBC_BASE_PACKAGE_FILENAME" \
          "$ALPINE_GLIBC_BASE_URL/$ALPINE_GLIBC_PACKAGE_VERSION/$ALPINE_GLIBC_BIN_PACKAGE_FILENAME" \
          "$ALPINE_GLIBC_BASE_URL/$ALPINE_GLIBC_PACKAGE_VERSION/$ALPINE_GLIBC_I18N_PACKAGE_FILENAME" \
      && apk --no-cache add \
          "$ALPINE_GLIBC_BASE_PACKAGE_FILENAME" \
          "$ALPINE_GLIBC_BIN_PACKAGE_FILENAME" \
          "$ALPINE_GLIBC_I18N_PACKAGE_FILENAME" \
      && rm "/etc/apk/keys/sgerrand.rsa.pub" \
      && /usr/glibc-compat/bin/localedef --force --inputfile POSIX --charmap UTF-8 "$LANG" || true \
      && echo "export LANG=$LANG" > /etc/profile.d/locale.sh \
      # && apk del glibc-i18n \
      && rm "/root/.wget-hsts" \
      && apk del .build-dependencies wget \
      && rm "$ALPINE_GLIBC_BASE_PACKAGE_FILENAME" \
          "$ALPINE_GLIBC_BIN_PACKAGE_FILENAME" \
          "$ALPINE_GLIBC_I18N_PACKAGE_FILENAME"
  ```
  ```shell
  docker build -t alpine-zulu-jdk8:8.0.332 .
  ```
