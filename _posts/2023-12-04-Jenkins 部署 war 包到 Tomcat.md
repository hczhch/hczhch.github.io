---
layout:       post
title:        "Jenkins 部署 war 包到 Tomcat"
author:       "Vito"
header-style: text
catalog:      true
tags:
    - Jenkins
    - Tomcat
---

* 准备一台新服务器，安装好 JDK 和 Tomcat

* 修改 tomcat 配置文件
  ```shell
  # 新建 tomcat 用户并授权，使其能够管理 tomcat 中的项目。Jenkins 部署项目到 tomcat 时会用到该用户
  [zc@TestServer conf]$ pwd
  /usr/local/apache-tomcat-10.1.16/conf
  [zc@TestServer conf]$ vim tomcat-users.xml
  <tomcat-users>
      <role rolename="tomcat"/>
      <role rolename="role1"/>
      <role rolename="manager-script"/>
      <role rolename="manager-gui"/>
      <role rolename="manager-status"/>
      <role rolename="admin-gui"/>
      <role rolename="admin-script"/>
      <user username="jenkins" password="123456" roles="manager-script,manager-gui"/>
  </tomcat-users>
  
  # 设置 tomcat 可以远程登录
  [zc@TestServer META-INF]$ pwd
  /usr/local/apache-tomcat-10.1.16/webapps/manager/META-INF
  [zc@TestServer META-INF]$ vim context.xml
  <Valve className="org.apache.catalina.valves.RemoteAddrValve"
       allow="192\.168\.\d+\.\d+|127\.\d+\.\d+\.\d+|::1|0:0:0:0:0:0:0:1" />
  ```

* 给 tomcat 配置域名和反向代理
  ```shell
  # 配置内网 DNS 域名解析，解析 web-demo.zhch.lan 到 nginx

  # 在 nginx 中新增如下 server
    server {
        listen       80;
        server_name  web-demo.zhch.lan;

        location / {
            proxy_set_header    Host $host;
            proxy_set_header    X-Real-IP $remote_addr;
            proxy_set_header    X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header    X-Forwarded-Proto $scheme;

            proxy_pass          http://192.168.23.14:8080;
            proxy_redirect      http://192.168.23.14:8080 http://web-demo.zhch.lan;
        }
    }
  ```
* 启动 tomcat ，访问 http://web-demo.zhch.lan/manager/html ，验证配置是否成功 

* Jenkins 安装 Deploy to container 插件

* 部署项目到Tomcat  
  ![](/img/jenkins/jenkins_18.png)  
  ![](/img/jenkins/jenkins_19.png)
  * Item Build 成功后可成功访问项目 http://web-demo.zhch.lan/demo/
