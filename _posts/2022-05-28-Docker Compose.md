---
layout:       post
title:        "Docker Compose"
author:       "Vito"
header-style: text
catalog:      true
tags:
    - docker
---

### 官方文档
* <https://docs.docker.com/compose/compose-file/compose-file-v3/>
* <https://github.com/docker/compose/releases/>
* <https://docs.docker.com/compose/install/>


### 安装 Docker Compose
```shell
# 手动安装
sudo curl -SL \
    "https://github.com/docker/compose/releases/download/v2.5.1/docker-compose-$(uname -s)-$(uname -m)" \
    -o /usr/local/lib/docker/cli-plugins/docker-compose
sudo chmod +x /usr/local/lib/docker/cli-plugins/docker-compose
sudo docker-compose --version

# 卸载
sudo rm /usr/local/lib/docker/cli-plugins/docker-compose
```

```shell
# install using the repository (推荐)
yum install -y docker-compose-plugin
```


### Docker Compose 常用命令
```shell
docker compose -h    # 查看帮助
docker compose up    # 启动所有 docker compose 服务
docker compose up -d    # 启动所有 docker compose 服务并后台运行
docker compose down    # 停止并删除容器、网络、卷、镜像。
docker compose exec yml里面的服务id    # 进入容器实例内部 docker compose exec docker-compose.yml文件中写的服务id /bin/bash
docker compose ps    # 展示当前 docker compose 编排过的运行的所有容器
docker compose top    # 展示当前 docker compose 编排过的容器进程
docker compose logs yml里面的服务id    # 查看容器输出日志
docker compose config    # 检查配置
docker compose config -q    # 检查配置，有问题才有输出
docker compose restart    # 重启服务
docker compose start    # 启动服务
docker compose stop    # 停止服务
```


### 样例
```shell
[root@mydockerhost vito]# cd /home/vito/docker/compose/fdrcloud/
[root@mydockerhost fdrcloud]# 
[root@mydockerhost fdrcloud]# vim docker-compose.yml
# 先进入 docker-compose.yml 文件所在目录，再执行 docker compose 命令
```
```yaml
version: "3.8"
services:  
  yun_tomcat:
    image: inovatrend/tomcat8-java8
    container_name: yun_tomcat
    ports:
      - "8080:8080"
    networks: 
      - fdr_bridge 
    depends_on: 
      - yun_redis
      - order_mysql

  # 服务id，运行 docker compose ps 命令后显示在 SERVICE 列
  # 使用自定义 bridge 网络后，容器之间可用通过 container name、service id、container ip 通信，
  # 例如：ping yun_redis  、  ping fdrcloud-yun_redis-1  、  ping 172.22.0.2 
  yun_redis:
    image: redis:6.0.8
    # 默认 container_name = docker-compose.yml文件所在的目录名-serviceId-1，例如：fdrcloud-yun_redis-1
    ports:
      - "6379:6379"
    privileged: true
    volumes:
      - /home/vito/docker/volume/fdrcloud_redis6.0/redis.conf:/etc/redis/redis.conf
      - /home/vito/docker/volume/fdrcloud_redis6.0/data:/data
    networks: 
      - fdr_bridge
    command: redis-server /etc/redis/redis.conf

  order_mysql:
    image: mysql:5.7
    environment:
      MYSQL_ROOT_PASSWORD: '123456'
      MYSQL_ALLOW_EMPTY_PASSWORD: 'no'
      MYSQL_DATABASE: 'fdr_tuike'
      MYSQL_USER: 'dev'
      MYSQL_PASSWORD: 'dev_dev'
    ports:
       - "3306:3306"
    privileged: true
    volumes:
       - /home/vito/docker/volume/fdrcloud_mysql5.7/log:/var/log/mysql
       - /home/vito/docker/volume/fdrcloud_mysql5.7/data:/var/lib/mysql
       - /home/vito/docker/volume/fdrcloud_mysql5.7/conf:/etc/mysql/conf.d
    networks:
      - fdr_bridge
    command: --default-authentication-plugin=mysql_native_password #解决外部无法访问

networks: 
  # docker network create [docker-compose.yml文件所在的目录名_自定义网络名]
  # 例如：docker network create fdrcloud_fdr_bridge
  # 本例子实际创建的网络名称是：fdrcloud_fdr_bridge
  # 如果 yml文件里没有配置网络，则默认创建和使用的网络是：docker-compose.yml文件所在的目录名_default ，例如：fdrcloud_default
  fdr_bridge:
```
```shell
[root@mydockerhost fdrcloud]# ls
docker-compose.yml
[root@mydockerhost fdrcloud]# docker compose up -d
```

