---
layout:       post
title:        "Docker 安装 Harbor 02"
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

##### 配置文件
```shell
[vito@dockerhost compose]$ cd harbor
[vito@dockerhost harbor]$ cp harbor.yml.tmpl harbor.yml
[vito@dockerhost harbor]$ vim harbor.yml
# The IP address or hostname to access admin UI and registry service.
# DO NOT use localhost or 127.0.0.1, because Harbor needs to be accessed by external clients.
hostname: harbor.vito.lan

# http related config
http:
  # port for http, default is 80. If https enabled, this port will redirect to https port
  port: 8088

# https related config
#https:
  # https port for harbor, default is 443
  #port: 443
  # The path of cert and key files for nginx
  #certificate: /your/certificate/path
  #private_key: /your/private/key/path

# Uncomment external_url if you want to enable external proxy
# And when it enabled the hostname will no longer used
external_url: https://harbor.vito.lan

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

[vito@dockerhost harbor]$ sudo docker compose ps -a
NAME                IMAGE                                COMMAND                                SERVICE       CREATED         STATUS                   PORTS
harbor-core         goharbor/harbor-core:v2.9.1          "/harbor/entrypoint.sh"                core          7 minutes ago   Up 7 minutes (healthy)   
harbor-db           goharbor/harbor-db:v2.9.1            "/docker-entrypoint.sh 13 14"          postgresql    7 minutes ago   Up 7 minutes (healthy)   
harbor-jobservice   goharbor/harbor-jobservice:v2.9.1    "/harbor/entrypoint.sh"                jobservice    7 minutes ago   Up 7 minutes (healthy)   
harbor-log          goharbor/harbor-log:v2.9.1           "/bin/sh -c /usr/local/bin/start.sh"   log           7 minutes ago   Up 7 minutes (healthy)   127.0.0.1:1514->10514/tcp
harbor-portal       goharbor/harbor-portal:v2.9.1        "nginx -g 'daemon off;'"               portal        7 minutes ago   Up 7 minutes (healthy)   
nginx               goharbor/nginx-photon:v2.9.1         "nginx -g 'daemon off;'"               proxy         7 minutes ago   Up 7 minutes (healthy)   0.0.0.0:8088->8080/tcp, :::8088->8080/tcp
redis               goharbor/redis-photon:v2.9.1         "redis-server /etc/redis.conf"         redis         7 minutes ago   Up 7 minutes (healthy)   
registry            goharbor/registry-photon:v2.9.1      "/home/harbor/entrypoint.sh"           registry      7 minutes ago   Up 7 minutes (healthy)   
registryctl         goharbor/harbor-registryctl:v2.9.1   "/home/harbor/start.sh"                registryctl   7 minutes ago   Up 7 minutes (healthy)
```

##### 自签证书
```shell
[vito@Nginx-1 ~]$ sudo find / -name openssl.cnf
/etc/pki/tls/openssl.cnf
/etc/ssl/openssl.cnf
/usr/lib/dracut/modules.d/01fips/openssl.cnf

[vito@Nginx-1 ~]$ mkdir cert
[vito@Nginx-1 ~]$ cp /etc/pki/tls/openssl.cnf cert/
[vito@Nginx-1 ~]$ openssl genrsa -out cert/vito.lan.key 4096
[vito@Nginx-1 ~]$ openssl req -new -x509 -key cert/vito.lan.key -out cert/vito.lan.crt -days 36500 \
                -subj "/C=CN/ST=Guangdong/L=Guangzhou/O=vito/OU=vito/CN=vito.lan" \
                -extensions SAN \
                -config <(cat cert/openssl.cnf <(printf "[SAN]\nsubjectAltName=DNS.1:vito.lan,DNS.2:*.vito.lan"))
```

##### Nginx
```shell
[vito@Nginx-1 ~]$ sudo vim /usr/local/nginx/conf/nginx.conf
server {
    listen       80;
    server_name  harbor.vito.lan;
    return 301 https://$http_host$request_uri;
}

server {
    listen       443 ssl http2;
    server_name  harbor.vito.lan;

    client_max_body_size 1000M;

    ssl_certificate "/home/vito/cert/vito.lan.crt";
    ssl_certificate_key "/home/vito/cert/vito.lan.key";
    ssl_session_cache shared:SSL:1m;
    ssl_session_timeout  10m;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;

    location / {
        proxy_set_header    Host $host;
        proxy_set_header    X-Real-IP $remote_addr;
        proxy_set_header    X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header    X-Forwarded-Proto $scheme;

        # 192.168.23.11 是 docker 宿主机的 IP
        proxy_pass          http://192.168.23.11:8088;
        proxy_redirect      http://192.168.23.11:8088 https://harbor.vito.lan;
    }
} 

[vito@Nginx-1 ~]$ sudo systemctl restart nginx
```

* 访问 `https://harbor.vito.lan` `admin / Harbor12345`

* 角色与权限

  | 角色    | 权限说明                      |
    |-------|---------------------------|
  | 访客    | 对于指定项目拥有只读权限              |
  | 开发人员  | 对于指定项目拥有读写权限              |
  | 维护人员  | 对于指定项目拥有读写权限，创建 Webhooks  |
  | 项目管理员 | 除了读写权限，同时拥有用户管理/镜像扫描等管理权限 |

##### 镜像推送到 Harbor
```shell
# 客户端要使用 tls 与 Harbor 通信，使用的还是自签证书，那么必须建立一个目录：/etc/docker/certs.d
# 然后在这个目录下创建签名的域名目录，并把证书拷贝到这个目录中
[root@dockerhost ~]# mkdir -p /etc/docker/certs.d
[root@dockerhost ~]# cd /etc/docker/certs.d/
[root@dockerhost certs.d]# mkdir harbor.vito.lan
# 192.168.23.131 是 Nginx 服务器的 IP，其 SSH 端口已由默认的 22 改为 2222
# scp [可选参数] file_source file_target  
[root@dockerhost certs.d]# scp -P 2222 vito@192.168.23.131:/home/vito/cert/vito.lan.crt /etc/docker/certs.d/harbor.vito.lan/

# 在 harbor 中创建私有项目，本例中创建的项目名是 tensquare

# 创建新的镜像（ harbor地址/项目名称/镜像名:镜像tag ）
[vito@dockerhost ~]$ docker tag mysql:5.7.44 harbor.vito.lan/tensquare/fdr_mysql:v1

# 登录 harbor
[vito@dockerhost ~]$ docker login -u vito -p Harbor12345 harbor.vito.lan

# 推送到仓库
[vito@dockerhost ~]$ docker push harbor.vito.lan/tensquare/fdr_mysql:v1

# 在本地删除新创建的镜像
[vito@dockerhost ~]$ docker rmi harbor.vito.lan/tensquare/fdr_mysql:v1

# 从仓库拉取镜像（从私有项目中拉取镜像也需要先登录 harbor）
[vito@dockerhost ~]$ docker pull harbor.vito.lan/tensquare/fdr_mysql:v1
```

```shell
# 如果 harbor 的访问地址是 http 而非 https，则需要将 harbor 地址添加到 Docker 信任列表，然后重启 docker
vim /etc/docker/daemon.json
# 在 json 文件中新增一项 "insecure-registries": ["harbor地址(ip:port)"]
"insecure-registries": ["harbor.vito.lan"]
```

##### 解决服务器重启，harbor 不能全部正常启动的问题
* 容器只有在 `docker-compose up` 时，才会按照 `depends_on` 定义的顺序启动
* docker 本身并不记录容器之间的依赖顺序，容器们的重启是相互独立的，各自独立的重启导致服务器重启后，harbor 不能全部正常启动
  ```shell
  # https://github.com/goharbor/harbor/issues/7008
  [root@dockerhost ~]# vim /etc/systemd/system/harbor.service
  [Unit]
  Description=Harbor
  After=docker.service systemd-networkd.service systemd-resolved.service
  Requires=docker.service
  Documentation=http://github.com/vmware/harbor
  
  [Service]
  Type=simple
  Restart=on-failure
  RestartSec=5
  ExecStart=/usr/bin/docker compose -f /home/vito/docker/compose/harbor/docker-compose.yml up
  ExecStop=/usr/bin/docker compose -f /home/vito/docker/compose/harbor/docker-compose.yml down
  
  [Install]
  WantedBy=multi-user.target
  
  [root@dockerhost ~]# systemctl enable harbor
  Created symlink /etc/systemd/system/multi-user.target.wants/harbor.service → /etc/systemd/system/harbor.service.
  ```
