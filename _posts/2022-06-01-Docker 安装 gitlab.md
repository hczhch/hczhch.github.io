---
layout:       post
title:        "Docker 安装 gitlab"
author:       "Vito"
header-style: text
catalog:      true
tags:
    - Docker
    - GitLab
---


### gitlab ( docker-compose.yml )
```yaml
version: "3.8"
services: 
  fdr_gitlab: 
    image: gitlab/gitlab-ce:14.10.4-ce.0
    container_name: fdr_gitlab
    restart: always
    # 将容器主机名设置为：宿主机IP或域名
    hostname: "192.168.23.128"
    environment: 
      TZ: 'Asia/Shanghai'
      # https://gitlab.com/gitlab-org/omnibus-gitlab/blob/master/files/gitlab-config-template/gitlab.rb.template
      GITLAB_OMNIBUS_CONFIG: | 
        external_url "http://192.168.23.128:8082"
        gitlab_rails['gitlab_shell_ssh_port'] = 1023
        gitlab_rails['smtp_enable'] = true
        gitlab_rails['time_zone'] = 'Asia/Shanghai'
    ports: 
      - "8082:8082"
      - "1445:443"
      - "1023:22"
    volumes: 
      - /home/vito/docker/volume/fdr_gitlab/config:/etc/gitlab
      - /home/vito/docker/volume/fdr_gitlab/logs:/var/log/gitlab
      - /home/vito/docker/volume/fdr_gitlab/data:/var/opt/gitlab
```
* `external_url "http://192.168.23.128:8082"`  # 指定 http host:port （ 同时，容器内 gitlab 默认的 80 端口被更改为指定的 port ）
* 由于主机 ssh 一般占用了 22 端口，所以 gitlab 的 ssh 需映射主机的其他端口。   
* 配置中 `gitlab_rails['gitlab_shell_ssh_port'] = 1023` 修改的是 gitlab 的网页显示端口（映射到主机的端口），实际容器内的 ssh 仍然是 22 。
* 查看 root 用户的默认密码： `docker exec -it fdr_gitlab grep 'Password:' /etc/gitlab/initial_root_password` ，应及时修改默认密码。
* gitlab 配置文件：`/etc/gitlab/gitlab.rb` （/home/vito/docker/volume/fdr_gitlab/config/gitlab.rb）
  ```shell
  [root@mydockerhost fdr_gitlab]# vim /home/vito/docker/volume/fdr_gitlab/config/gitlab.rb
  gitlab_rails['smtp_enable'] = true
  gitlab_rails['smtp_address'] = "smtp.163.com"
  gitlab_rails['smtp_port'] = 465
  gitlab_rails['smtp_user_name'] = "nest0321@163.com"
  gitlab_rails['smtp_password'] = "********"
  gitlab_rails['smtp_domain'] = "163.com"
  gitlab_rails['smtp_authentication'] = "login"
  gitlab_rails['smtp_enable_starttls_auto'] = true
  gitlab_rails['smtp_tls'] = false
  gitlab_rails['smtp_pool'] = false
  
  gitlab_rails['gitlab_email_from'] = "nest0321@163.com"
  
  user["git_user_email"] = "nest0321@163.com"
  
  # 重启配置
  [root@mydockerhost fdr_gitlab]# docker exec -it fdr_gitlab gitlab-ctl reconfigure
  [root@mydockerhost fdr_gitlab]# docker exec -it fdr_gitlab gitlab-ctl restart
  ```
  ![](/img/docker/git架构图.jpg)  


### 安装方法二
  ```shell
  # https://docs.gitlab.com/ee/install/docker.html
  docker run -d \
    --restart=always  \
    -h 192.168.23.11 \
    -p 8010:80 -p 2210:22 \
    --privileged=true  \
    -v /home/vito/docker/volume/gitlab/data:/var/opt/gitlab \
    -v /home/vito/docker/volume/gitlab/logs:/var/log/gitlab \
    -v /home/vito/docker/volume/gitlab/config:/etc/gitlab \
    --name gitlab-ce \
    gitlab/gitlab-ce:16.6.0-ce.0
  ```

  ```shell
  # 编辑配置文件
  [vito@dockerhost ~]$ sudo vim /home/vito/docker/volume/gitlab/config/gitlab.rb
  # https://gitlab.com/gitlab-org/omnibus-gitlab/blob/master/files/gitlab-config-template/gitlab.rb.template
  external_url 'http://gitlab.vito.lan'
  gitlab_rails['gitlab_shell_ssh_port'] = 2210
  gitlab_rails['time_zone'] = 'Asia/Shanghai'
  
  gitlab_rails['smtp_enable'] = true
  gitlab_rails['smtp_address'] = "smtp.163.com"
  gitlab_rails['smtp_port'] = 465
  gitlab_rails['smtp_user_name'] = "nest0321@163.com"
  gitlab_rails['smtp_password'] = "******"
  gitlab_rails['smtp_domain'] = "163.com"
  gitlab_rails['smtp_authentication'] = "login"
  gitlab_rails['smtp_enable_starttls_auto'] = false
  gitlab_rails['smtp_tls'] = true
  gitlab_rails['smtp_pool'] = false
  gitlab_rails['gitlab_email_from'] = "nest0321@163.com"
  user["git_user_email"] = "nest0321@163.com"
  ```

  ```shell
  # 配置内网 DNS 域名解析，解析 gitlab.vito.lan 到 nginx
  
  # 在 nginx 中新增如下 server
      server {
          listen       80;
          server_name  gitlab.vito.lan;
  
          location / {
              proxy_set_header    Host $host;
              proxy_set_header    X-Real-IP $remote_addr;
              proxy_set_header    X-Forwarded-For $proxy_add_x_forwarded_for;
              # proxy_set_header    X-Forwarded-Proto $scheme;
  
              # http://docker_host_ip:gitlab_port
              proxy_pass          http://192.168.23.11:8010;
              proxy_redirect      http://192.168.23.11:8010 http://gitlab.vito.lan;
          }
      }
  ```

  ```shell
  # 查询 root 用户的初始密码，首次登录后需修改密码
  [vito@dockerhost ~]$ sudo cat /home/vito/docker/volume/gitlab/config/initial_root_password
  # WARNING: This value is valid only in the following conditions
  #          1. If provided manually (either via `GITLAB_ROOT_PASSWORD` environment variable or via `gitlab_rails['initial_root_password']` setting in `gitlab.rb`, it was provided before database was seeded for the first time (usually, the first reconfigure run).
  #          2. Password hasn't been changed manually, either via UI or via command line.
  #
  #          If the password shown here doesn't work, you must reset the admin password following https://docs.gitlab.com/ee/security/reset_user_password.html#reset-your-root-password.
  
  Password: uxl8uWCBJWW4qCzZaD+7FZs+y3Jw+TwidAW3ervMHvA=
  
  # NOTE: This file will be automatically deleted in the first reconfigure run after 24 hours.
  ```

