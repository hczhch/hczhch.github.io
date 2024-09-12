---
layout:       post
title:        "Alma Linux 9 安装 Jenkins"
author:       "Vito"
header-style: text
catalog:      true
tags:
    - Jenkins
---

### 安装JAVA
```shell
[zc@Jenkins ~]$ mkdir software
[zc@Jenkins ~]$ cd software
[zc@Jenkins software]$ wget https://cdn.azul.com/zulu/bin/zulu21.30.15-ca-jdk21.0.1-linux_x64.tar.gz
[zc@Jenkins software]$ sudo tar -zxf zulu21.30.15-ca-jdk21.0.1-linux_x64.tar.gz -C /usr/local/
[zc@Jenkins software]$ sudo ln -s /usr/local/zulu21.30.15-ca-jdk21.0.1-linux_x64 /usr/local/java
[zc@Jenkins software]$ sudo vim /etc/profile
JAVA_HOME=/usr/local/java
PATH=$JAVA_HOME/bin:$PATH
export JAVA_HOME PATH
[zc@Jenkins software]$ source /etc/profile
```

### 安装Jenkins
```shell
# https://www.jenkins.io/download/
# https://pkg.jenkins.io/redhat-stable/
[zc@Jenkins ~]$ sudo wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/jenkins.repo
[zc@Jenkins ~]$ sudo rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io-2023.key
[zc@Jenkins ~]$ sudo yum install -y jenkins
```

```shell
[zc@Jenkins ~]$ sudo systemctl status jenkins
○ jenkins.service - Jenkins Continuous Integration Server
     Loaded: loaded (/usr/lib/systemd/system/jenkins.service; disabled; preset: disabled)
     Active: inactive (dead)

[zc@Jenkins ~]$ sudo vim /usr/lib/systemd/system/jenkins.service
# 修改如下四项配置：运行 Jenkins 守护进程的系统账户、JAVA HOME、Jenkins 端口号
User=root
Group=root
Environment="JAVA_HOME=/usr/local/java"
Environment="JENKINS_PORT=8080"

[zc@Jenkins ~]$ sudo systemctl daemon-reload

[zc@Jenkins ~]$ sudo firewall-cmd --zone=public --add-port=8080/tcp --permanent
[zc@Jenkins ~]$ sudo firewall-cmd --reload
```

```shell
# 配置内网 DNS 域名解析，解析 jenkins.zhch.lan 到 nginx

# 在 nginx 中新增如下 server
    server {
        listen       80;
        server_name  jenkins.zhch.lan;

        location / {
            proxy_set_header    Host $host;
            proxy_set_header    X-Real-IP $remote_addr;
            proxy_set_header    X-Forwarded-For $proxy_add_x_forwarded_for;
            # proxy_set_header    X-Forwarded-Proto $scheme;

            proxy_pass          http://192.168.23.13:8080;
            proxy_read_timeout  90;
            proxy_redirect      http://192.168.23.13:8080 http://jenkins.zhch.lan;
        }
    }
```

```shell
[zc@Jenkins ~]$ sudo systemctl enable jenkins
[zc@Jenkins ~]$ sudo systemctl start jenkins
```

```shell
# 如果启动时报错 java.lang.RuntimeException: Fontconfig head is null, check your fonts or fonts configuration
# 报错原因是 OpenJDK 不支持 awt 包下的字体，需要手动安装 fontconfig
[zc@Jenkins ~]$ sudo yum install -y fontconfig
```

```shell
# 查看 admin 初始密码
[zc@Jenkins ~]$ sudo cat /var/lib/jenkins/secrets/initialAdminPassword
fc43e096cca1464fa7d700d48364960a
```

* 初次登录时不要安装任何插件，登录后修改插件地址后再安装
  ![](/img/jenkins/jenkins_1.png)

* 如果 jenkins 报错：HTTP ERROR 403 No valid crumb was included in  the request ， 就启用代理兼容
  ![](/img/jenkins/jenkins_2.png)
  ![](/img/jenkins/jenkins_3.png)

* 修改 jenkins 插件地址为国内地址 
  * Manage Jenkins -> Plugins -> Available plugins ，等待数据加载完成
  * 修改 `/var/jenkins_home/updates/default.json`
  ```shell
  [zc@Jenkins ~]$ sudo sed -i 's/https:\/\/updates.jenkins.io\/download\/plugins/https:\/\/mirrors.tuna.tsinghua.edu.cn\/jenkins\/plugins/g' /var/lib/jenkins/updates/default.json
  [zc@Jenkins ~]$ sudo sed -i 's/https:\/\/www.google.com/https:\/\/www.baidu.com/g' /var/lib/jenkins/updates/default.json
  ```
  * Manage Jenkins -> Plugins -> Advanced settings
    * 修改插件更新中心地址：`https://mirrors.tuna.tsinghua.edu.cn/jenkins/updates/update-center.json`
      ![](/img/jenkins/jenkins_4.png)
  * 通过 http://jenkins.zhch.lan/restart 重启 jenkins

* 插件安装
  * Manage Jenkins -> Plugins -> Available plugins ，搜索插件，根据插件的生效条件选择 install 或 install after restart
    ![](/img/jenkins/jenkins_5.png)
