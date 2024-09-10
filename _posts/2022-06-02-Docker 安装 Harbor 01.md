---
layout:       post
title:        "2022-06-02-Docker 安装 Harbor 01"
author:       "Vito"
header-style: text
catalog:      true
tags:
    - Docker
    - Harbor
---


##### 下载解压
```shell
# https://github.com/goharbor/harbor/releases
[vito@dockerhost ~]$ cd /home/vito/docker/compose/
[vito@dockerhost compose]$ wget https://github.com/goharbor/harbor/releases/download/v2.9.1/harbor-offline-installer-v2.9.1.tgz
[vito@dockerhost compose]$ tar -zxf harbor-online-installer-v2.9.1.tgz
```

##### 自签证书
```shell
[vito@dockerhost compose]$ cd harbor
[vito@dockerhost harbor]$ mkdir cert
[vito@dockerhost harbor]$ openssl genrsa -out cert/vito.lan.key 4096
[vito@dockerhost harbor]$ openssl req -new -x509 -key cert/vito.lan.key -out cert/vito.lan.crt -days 36500 \
                        -subj "/C=CN/ST=Guangdong/L=Guangzhou/O=vito/OU=vito/CN=harbor.vito.lan"
```

##### 配置文件
```shell
[vito@dockerhost harbor]$ cp harbor.yml.tmpl harbor.yml
[vito@dockerhost harbor]$ vim harbor.yml
# The IP address or hostname to access admin UI and registry service.
# DO NOT use localhost or 127.0.0.1, because Harbor needs to be accessed by external clients.
# hostname: reg.mydomain.com
hostname: harbor.vito.lan

# http related config
http:
  # port for http, default is 80. If https enabled, this port will redirect to https port
  port: 80

# https related config
https:
  # https port for harbor, default is 443
  port: 443
  # The path of cert and key files for nginx
  certificate: /home/vito/docker/compose/harbor/cert/vito.lan.crt
  private_key: /home/vito/docker/compose/harbor/cert/vito.lan.key

# Uncomment external_url if you want to enable external proxy
# And when it enabled the hostname will no longer used
# external_url: https://reg.mydomain.com:8433

# The initial password of Harbor admin
# It only works in first time to install harbor
# Remember Change the admin password from UI after launching Harbor.
harbor_admin_password: Harbor12345

# The default data volume
data_volume: /home/vito/docker/volume/harbor
```

##### 安装
```shell
[vito@dockerhost harbor]$ ./prepare 
[vito@dockerhost harbor]$ sudo ./install.sh
# install 后，会生成 docker-compose.yml 文件
# docker compose up -d 启动
# docker compose stop 停止
# docker compose restart 重新启动
# docker compose down 卸载
```

* 访问 `https://harbor.vito.lan` `admin / Harbor12345`

* 角色与权限

  | 角色    | 权限说明                      |
    |-------|---------------------------|
  | 访客    | 对于指定项目拥有只读权限              |
  | 开发人员  | 对于指定项目拥有读写权限              |
  | 维护人员  | 对于指定项目拥有读写权限，创建 Webhooks  |
  | 项目管理员 | 除了读写权限，同时拥有用户管理/镜像扫描等管理权限 |
