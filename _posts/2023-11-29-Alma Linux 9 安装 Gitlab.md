---
layout:       post
title:        "Alma Linux 9 安装 Gitlab"
author:       "Vito"
header-style: text
catalog:      true
tags:
    - GitLab
---



### 安装
* 准备工作
  ```shell
  [zc@Git ~]$ sudo dnf install -y policycoreutils policycoreutils-python-utils perl openssh-server openssh-clients
  [zc@Git ~]$ sudo systemctl enable sshd
  [zc@Git ~]$ sudo systemctl start sshd
  [zc@Git ~]$ 
  # [zc@Git ~]$ sudo firewall-cmd --permanent --add-service=http
  # [zc@Git ~]$ sudo firewall-cmd --permanent --add-service=https
  [zc@Git ~]$ sudo firewall-cmd --permanent --add-service=ssh
  [zc@Git ~]$ sudo systemctl reload firewalld
  ```

  ```shell
  # （可选）下一步，安装 Postfix 以发送电子邮件通知。
  # 如果您想使用其他解决方案发送电子邮件，请跳过此步骤并在安装 GitLab 后配置外部 SMTP 服务器。
  [zc@Git ~]$ sudo dnf install -y postfix
  [zc@Git ~]$ sudo systemctl enable postfix
  [zc@Git ~]$ sudo systemctl start postfix
  ```

* 安装方法一：手动下载安装包安装
  ```shell
  [zc@Git ~]$ cd software
  [zc@Git software]$ wget https://mirrors.tuna.tsinghua.edu.cn/gitlab-ce/yum/el9/gitlab-ce-16.6.0-ce.0.el9.x86_64.rpm
  [zc@Git software]$ sudo rpm -i gitlab-ce-16.6.0-ce.0.el9.x86_64.rpm
  ```

* 安装方法二：通过 yum 源安装
  ```shell
  [zc@Git ~]$ sudo vim /etc/yum.repos.d/gitlab-ce.repo
  [gitlab-ce]
  name=Gitlab CE Repository
  baseurl=https://mirrors.tuna.tsinghua.edu.cn/gitlab-ce/yum/el$releasever/
  gpgcheck=0
  enabled=1
  
  [zc@Git ~]$ sudo yum clean all
  [zc@Git ~]$ sudo yum makecache
  [zc@Git ~]$ sudo dnf install -y gitlab-ce
  ```
  
  ```shell
  [root@Git ~]# cat /opt/gitlab/embedded/service/gitlab-rails/VERSION
  16.6.0
  
  [root@Git ~]# gitlab-rake gitlab:env:info
  System information
  System:		
  Current User:	git
  Using RVM:	no
  Ruby Version:	3.0.6p216
  Gem Version:	3.4.21
  Bundler Version:2.4.21
  Rake Version:	13.0.6
  Redis Version:	7.0.14
  Sidekiq Version:6.5.12
  Go Version:	unknown
  
  GitLab information
  Version:	16.6.0
  Revision:	6d558d71eba
  Directory:	/opt/gitlab/embedded/service/gitlab-rails
  DB Adapter:	PostgreSQL
  DB Version:	13.11
  URL:		https://git.zhch.lan
  HTTP Clone URL:	https://git.zhch.lan/some-group/some-project.git
  SSH Clone URL:	git@git.zhch.lan:some-group/some-project.git
  Using LDAP:	no
  Using Omniauth:	yes
  Omniauth Providers: 
  
  GitLab Shell
  Version:	14.30.0
  Repository storages:
  - default: 	unix:/var/opt/gitlab/gitaly/gitaly.socket
  GitLab Shell path:		/opt/gitlab/embedded/service/gitlab-shell
  
  Gitaly
  - default Address: 	unix:/var/opt/gitlab/gitaly/gitaly.socket
  - default Version: 	16.6.0
  - default Git Version: 	2.42.0
  ```

### 配置
* 如果外网能够直接访问 gitlab ，那么直接配置 DNS 域名解析到 gitlab 的 IP 即可，比较简单
* **如果外网不能直接访问 gitlab ，则需要通过一台可以与外网互通的 Nginx 服务器做反向代理**
* 本例假设外网不能直接访问 gitlab 所在服务器，需要通过另外一台 Nginx 服务器反向代理，且需要分别代理 HTTP 和 SSH
  ```shell
  [zc@Git ~]$ sudo vim /etc/gitlab/gitlab.rb
  # https://gitlab.com/gitlab-org/omnibus-gitlab/blob/master/files/gitlab-config-template/gitlab.rb.template
  external_url 'http://git.zhch.lan'
  nginx['listen_port'] = 8010
  # 默认情况下，nginx 将侦听 external_url 中指定的端口
  # 本例中 external_url 中指定的端口是 80 （ http 协议默认端口 ）
  # 如果部署 Gitlab 的服务器的 80 端口没有被占用，可以不用设置 nginx['listen_port'] 
  
  gitlab_rails['time_zone'] = 'Asia/Shanghai'
  
  gitlab_rails['smtp_enable'] = true
  gitlab_rails['smtp_address'] = "smtp.163.com"
  gitlab_rails['smtp_port'] = 465
  gitlab_rails['smtp_user_name'] = "nest0321@163.com"
  gitlab_rails['smtp_password'] = "********"
  gitlab_rails['smtp_domain'] = "163.com"
  gitlab_rails['smtp_authentication'] = "login"
  gitlab_rails['smtp_enable_starttls_auto'] = false
  gitlab_rails['smtp_tls'] = true
  gitlab_rails['smtp_pool'] = false
  gitlab_rails['gitlab_email_from'] = "nest0321@163.com"
  user["git_user_email"] = "nest0321@163.com"
  
  
  ##! **recommend value is 1/4 of total RAM, up to 14GB.**
  # postgresql['shared_buffers'] = "256MB"
  
  postgresql['shared_buffers'] = "128MB"
  
  # To completely disable prometheus, and all of it's exporters, set to false
  # prometheus_monitoring['enable'] = true
  # 学习环境，关闭 prometheus
  prometheus_monitoring['enable'] = false
  ```
  
  ```shell
  # 配置内网 DNS 域名解析，解析 git.zhch.lan 到 nginx
  
  # Nginx 代理 HTTP 请求访问 gitlab ：在 nginx http 模块中新增如下 server
  [root@Nginx-1 ~]# vim /usr/local/nginx/conf/nginx.conf
  server {
      listen       80;
      server_name  git.zhch.lan;  
      location / {
          proxy_set_header    Host $host;
          proxy_set_header    X-Real-IP $remote_addr;
          proxy_set_header    X-Forwarded-For $proxy_add_x_forwarded_for;
          # proxy_set_header    X-Forwarded-Proto $scheme;  
          # http://gitlab_host_ip:gitlab_port
          proxy_pass          http://192.168.23.12:8010;
          proxy_redirect      http://192.168.23.12:8010 http://git.zhch.lan;
      }
  }
  # Nginx 代理 SSH 请求访问 gitlab ：配置会复杂一些，方法放到文章最后
  ```

  ```shell
  [zc@Git ~]$ sudo firewall-cmd --zone=public --add-port=8010/tcp --permanent
  [zc@Git ~]$ sudo firewall-cmd --reload
  ```
  
  ```shell
  # 重载配置及启动 gitlab
  [zc@Git ~]$ sudo gitlab-ctl reconfigure
  [zc@Git ~]$ sudo gitlab-ctl restart
  ```
  
  ```shell
  # root 用户初始密码，登陆后修改
  [zc@Git ~]$ sudo cat /etc/gitlab/initial_root_password
  # WARNING: This value is valid only in the following conditions
  #          1. If provided manually (either via `GITLAB_ROOT_PASSWORD` environment variable or via `gitlab_rails['initial_root_password']`   setting in `gitlab.rb`, it was provided before database was seeded for the first time (usually, the first reconfigure run).
  #          2. Password hasn't been changed manually, either via UI or via command line.
  #
  #          If the password shown here doesn't work, you must reset the admin password following https://docs.gitlab.com/ee/security/  reset_user_password.html#reset-your-root-password.
  
  Password: VVAq3ybKKH57h5eOAE2HBgyyofu+sbW9Rm8CzdNDfGs=
  
  # NOTE: This file will be automatically deleted in the first reconfigure run after 24 hours.
  ```

### gitlab 角色
* `Guest`：可以创建issue、发表评论，不能读写版本库 
* `Reporter`：可以克隆代码，不能提交，QA、PM 可以赋予这个权限 
* `Developer`：可以克隆代码、开发、提交、push，普通开发可以赋予这个权限
* `Maintainer`：可以创建项目、添加tag、保护分支、添加项目成员、编辑项目，核心开发可以赋予这个权限 
* `Owner`：可以设置项目访问权限 - Visibility Level、删除项目、迁移项目、管理组成员，开发组组长可以赋予这个权限

### 配置 Nginx 代理 SSH 请求访问 gitlab
* 如果使用 22 端口访问 gitlab ：
  * nginx 要修改其所在服务器的 ssh 远程登录端口，不能使用默认的 22
    ```shell    
    [root@Nginx-1 ~]# firewall-cmd --permanent --zone=public --add-port=2222/tcp
    [root@Nginx-1 ~]# firewall-cmd --reload
    
    [root@Nginx-1 ~]# vim /etc/ssh/sshd_config
    # If you want to change the port on a SELinux system, you have to tell
    # SELinux about this change.
    # semanage port -a -t ssh_port_t -p tcp #PORTNUMBER
    Port 2222
    
    [root@Nginx-1 ~]# yum install semanage
    未找到匹配的参数: semanage
    错误：没有任何匹配: semanage
    [root@Nginx-1 ~]# yum provides semanage
    policycoreutils-python-utils-3.5-2.el9.noarch : SELinux policy core python utilities
    仓库        ：appstream
    匹配来源：
    文件名    ：/usr/sbin/semanage
    [root@Nginx-1 ~]# yum install policycoreutils-python-utils
    [root@Nginx-1 ~]# semanage port -a -t ssh_port_t -p tcp 2222
    [root@Nginx-1 ~]# systemctl restart sshd
    
    [root@Nginx-1 ~]# firewall-cmd --permanent --zone=public --remove-service=ssh
    [root@Nginx-1 ~]# firewall-cmd --reload
    ```
  * 重新编译 nginx ，新增 --with-stream ~~--with-stream_ssl_module --with-stream_ssl_preread_module~~ 模块
    * SSH 与 SSL 无关，所以 Nginx 中 SSH 无法基于 SNI 分流，能基于 SNI 在同一端口上分流的是 HTTPS
    * 所以 Nginx 中对 SSH 的代理，不能做到端口复用
  * 在 nginx 中监听 22 端口 ，然后转发到 gitlab server 的 22 端口
    ```shell
    # 在 nginx 中监听 22 端口 ，然后转发到 gitlab server 的 22 端口 
    # stream 模块是与 http 平级的模块，注意不要把该配置添加到了 http 模块中
    stream {
        server {
            listen 22;
            proxy_pass 192.168.23.12:22;
        }
    }
    ```
    
    ```shell
    [root@Nginx-1 ~]# firewall-cmd --permanent --zone=public --add-port=22/tcp
    [root@Nginx-1 ~]# firewall-cmd --reload
    ```

* 如果不使用 22 端口访问 gitlab ：
    * 在 gitlab.rb 中配置 gitlab_rails['gitlab_shell_ssh_port'] = new_port
        * 这个配置只是修改了 gitlab 页面上显示的项目的 ssh 的连接端口
        * 实际上 gitlab 的 ssh 端口还是 gitlab 所在服务器 sshd 服务的端口，需要通过 sshd_config 文件来变更 ssh 端口
    * 在 nginx 中监听 new_port ，然后转发到 gitlab server 的 sshd 端口

### 优化配置：http -> https
* nginx
  ```shell
  [root@Nginx-1 ~]# vim /usr/local/nginx/conf/nginx.conf
  server {
      listen       80;
      server_name  git.zhch.lan;
      return 301 https://$http_host$request_uri;
  }
  
  server {
      listen       443 ssl http2;
      server_name  git.zhch.lan;
  
      ssl_certificate "/home/zc/cert/zhch.lan.crt";
      ssl_certificate_key "/home/zc/cert/zhch.lan.key";
      ssl_session_cache shared:SSL:1m;
      ssl_session_timeout  10m;
      ssl_ciphers HIGH:!aNULL:!MD5;
      ssl_prefer_server_ciphers on;
  
      location / {
          proxy_set_header    Host $host;
          proxy_set_header    X-Real-IP $remote_addr;
          proxy_set_header    X-Forwarded-For $proxy_add_x_forwarded_for;
          proxy_set_header    X-Forwarded-Proto $scheme;
  
          proxy_pass          http://192.168.23.12:8010;
          proxy_redirect      http://192.168.23.12:8010 https://git.zhch.lan;
      }
  }
  
  [root@Nginx-1 ~]# systemctl restart nginx
  ```

* gitlab 配置
  ```shell
  [zc@Git ~]$ sudo vim /etc/gitlab/gitlab.rb
  external_url 'https://git.zhch.lan'
  nginx['listen_port'] = 8010
  nginx['listen_https'] = false
  
  [zc@Git ~]$ sudo gitlab-ctl reconfigure
  [zc@Git ~]$ sudo gitlab-ctl restart
  ```

* https 克隆代码
  ```shell
  [zc@Test ~]$ git clone https://git.zhch.lan/test/demo.git
  正克隆到 'demo'...
  致命错误：无法访问 'https://git.zhch.lan/test/demo.git/'：SSL certificate problem: self-signed certificate
  # 因为使用的是自签名证书，需设置取消 git ssl 证书验证才能克隆
  [zc@Test ~]$ git config --global http.sslVerify false
  [zc@Test ~]$ git clone https://git.zhch.lan/test/demo.git
  ```

### gitlab 服务器安全优化
* 修改 SSH 默认端口
  * 如果不强制要求 ssh 方式克隆代码时一定要用默认的 22 端口，则可以修改 gitlab 服务器的 SSH 端口
  * 修改方法与修改 Nginx 服务器 SSH 端口的方法相同
  * 一般建议把 SSH 端口改成一个大于 1024 小于 65535 的整数

* 禁用 root 远程登录
  * 禁用 root 远程登录后，可先使用普通用户登录，然后再通过`su root 输入root账号密码`的方式切换为 root 用户
    ```shell
    [root@Git zc]# vim /etc/ssh/sshd_config
    #PermitRootLogin prohibit-password # 允许 root 远程登录，但是禁止 root 用密码登录
    #PermitRootLogin yes # 允许 root 远程登录
    PermitRootLogin no
    
    [root@Git zc]# systemctl restart sshd
    ```

* 启用密钥登录
  * 用 SSH 工具（PuTTY、Xshell、ssh-keygen）生成密钥对（推荐使用 ED25519、RSA；选择 RSA 时，推荐的 key size 是 3072 或更大）
  * **普通**用户登录系统，将**公钥**内容写入到 ~/.ssh/authorized_keys 文件中。
  * 若 ~/.ssh/authorized_keys 文件不存在，则先创建 ~/.ssh 目录和 authorized_keys 文件，并修改权限
    ```shell
    [zc@Git ~]$ mkdir ~/.ssh
    [zc@Git ~]$ touch ~/.ssh/authorized_keys
    [zc@Git ~]$ chmod 700 ~/.ssh
    [zc@Git ~]$ chmod 600 ~/.ssh/authorized_keys
    # 写入公钥
    [zc@Git ~]$ vim ~/.ssh/authorized_keys
    ```
    
    ```shell
    [root@Git zc]# vim /etc/ssh/sshd_config
    PubkeyAuthentication yes
    
    [root@Git zc]# systemctl restart sshd
    ```

* 禁用密码登录
  ```shell
  [root@Git zc]# vim /etc/ssh/sshd_config
  PasswordAuthentication no
  
  [root@Git zc]# systemctl restart sshd
  ```
