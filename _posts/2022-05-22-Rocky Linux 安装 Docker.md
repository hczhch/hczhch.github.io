---
layout:       post
title:        "Rocky Linux 安装 Docker"
author:       "Vito"
header-style: text
catalog:      true
tags:
    - Docker
---

### 官方文档：
<https://docs.docker.com/engine/install/centos/>

### 1. 卸载旧版本
旧版本的 Docker 被称为 docker 或 docker-engine 。如果已安装这些程序，请卸载它们以及相关的依赖项。
```shell
[vito@mydockerhost ~]$ sudo yum remove docker \
                     docker-client \
                     docker-client-latest \
                     docker-common \
                     docker-latest \
                     docker-latest-logrotate \
                     docker-logrotate \
                     docker-engine
未找到匹配的参数: docker
未找到匹配的参数: docker-client
未找到匹配的参数: docker-client-latest
未找到匹配的参数: docker-common
未找到匹配的参数: docker-latest
未找到匹配的参数: docker-latest-logrotate
未找到匹配的参数: docker-logrotate
未找到匹配的参数: docker-engine
没有软件包需要移除。
依赖关系解决。
无需任何处理。
完毕！
```

### 2. 安装 yum-utils 软件包
```shell
sudo yum install -y yum-utils
```

### 3. 添加软件源
```shell
# 使用 aliyun docker 软件源代替官方 docker 软件源
[vito@mydockerhost ~]$ sudo yum-config-manager --add-repo \
    https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
添加仓库自：https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
```
### 4. 优化 doucker 安装速度
```shell
# yum makecache 命令是将软件包信息提前在本地索引缓存，用来提高搜索安装软件的速度，建议执行这个命令可以提升 yum 安装的速度。
# CentOS 7 版本下的命令是 yum makecache fast
[vito@mydockerhost ~]$ yum makecache
Rocky Linux 8 - AppStream                                                                                                                                                                                                                                                                                                        1.7 MB/s | 7.8 MB     00:04    
Rocky Linux 8 - BaseOS                                                                                                                                                                                                                                                                                                           592 kB/s | 2.6 MB     00:04    
Rocky Linux 8 - Extras                                                                                                                                                                                                                                                                                                           6.1 kB/s |  11 kB     00:01    
Docker CE Stable - x86_64                                                                                                                                                                                                                                                                                                         71 kB/s |  25 kB     00:00    
元数据缓存已建立。
```

### 5. 安装 Docker Engine
```shell
sudo yum install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

### 6. 启动 Docker
```shell
sudo systemctl start docker
```

### 7. 测试
```shell
[vito@mydockerhost ~]$ sudo docker version
Client: Docker Engine - Community
 Version:           20.10.16
 API version:       1.41
 Go version:        go1.17.10
 Git commit:        aa7e414
 Built:             Thu May 12 09:17:20 2022
 OS/Arch:           linux/amd64
 Context:           default
 Experimental:      true

Server: Docker Engine - Community
 Engine:
  Version:          20.10.16
  API version:      1.41 (minimum version 1.12)
  Go version:       go1.17.10
  Git commit:       f756502
  Built:            Thu May 12 09:15:41 2022
  OS/Arch:          linux/amd64
  Experimental:     false
 containerd:
  Version:          1.6.4
  GitCommit:        212e8b6fa2f44b9c21b2798135fc6fb7c53efc16
 runc:
  Version:          1.1.1
  GitCommit:        v1.1.1-0-g52de29d
 docker-init:
  Version:          0.19.0
  GitCommit:        de40ad0

[vito@mydockerhost ~]$ sudo docker run hello-world
Unable to find image 'hello-world:latest' locally
latest: Pulling from library/hello-world
2db29710123e: Pull complete 
Digest: sha256:80f31da1ac7b312ba29d65080fddf797dd76acfb870e677f390d5acba9741b17
Status: Downloaded newer image for hello-world:latest

Hello from Docker!
This message shows that your installation appears to be working correctly.

To generate this message, Docker took the following steps:
 1. The Docker client contacted the Docker daemon.
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
    (amd64)
 3. The Docker daemon created a new container from that image which runs the
    executable that produces the output you are currently reading.
 4. The Docker daemon streamed that output to the Docker client, which sent it
    to your terminal.

To try something more ambitious, you can run an Ubuntu container with:
 $ docker run -it ubuntu bash

Share images, automate workflows, and more with a free Docker ID:
 https://hub.docker.com/

For more examples and ideas, visit:
 https://docs.docker.com/get-started/
```

### 8. 卸载
```shell
sudo systemctl stop docker
sudo yum remove docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin docker-ce-rootless-extras
sudo rm -rf /var/lib/docker
sudo rm -rf /var/lib/containerd
```

### 9. 阿里云镜像加速
访问 https://promotion.aliyun.com/ntms/act/kubernetes.html ，登录淘宝或支付宝账号，然后打开“控制台”  
产品与服务 -> 容器服务 -> 容器镜像服务 -> 镜像工具 -> 镜像加速器
```shell
# 每个阿里云用户都用各自独有的镜像加速地址
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://y5kpfirm.mirror.aliyuncs.com"]
}
EOF
sudo systemctl daemon-reload
sudo systemctl restart docker
```


### 10. 用户权限
解决 Docker 运行命令时提示 docker: Got permission denied while trying to connect to the Docker daemon socket at unix:///var/run/docker.sock: Post "http://%2Fvar%2Frun%2Fdocker.sock/v1.24/containers/create?name=portainer-ce": dial unix /var/run/docker.sock: connect: permission denied. 的情况

* 解决方法1  
使用 `sudo` 获取管理员权限，运行 `docker` 命令。

* 解决方法2  
docker 守护进程启动的时候，会默认赋予名字为 docker 的用户组读写 Unix socket 的权限，因此只要创建 docker 用户组，并将当前用户加入到 docker 用户组中，那么当前用户就有权限访问 Unix socket 了，进而也就可以执行 docker 相关命令。

```shell
# 添加docker用户组
[vito@dockerhost ~]$ sudo groupadd docker
groupadd：“docker”组已存在

# 将登陆用户加入到docker用户组中
[vito@dockerhost ~]$ sudo gpasswd -a $USER docker
正在将用户“vito”加入到“docker”组中

# 更新用户组
[vito@dockerhost ~]$ newgrp docker

# 测试 docker 命令是否可以不使用 sudo 时正常使用
[vito@dockerhost ~]$ docker version
```
