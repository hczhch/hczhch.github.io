---
layout:       post
title:        "Jenkins Maven 安装和配置"
author:       "Vito"
header-style: text
catalog:      true
tags:
    - Jenkins
    - Maven
---

* 下载 Maven
  ```shell
  [zc@Jenkins ~]$ cd software
  [zc@Jenkins software]$ wget https://dlcdn.apache.org/maven/maven-3/3.9.5/binaries/apache-maven-3.9.5-bin.tar.gz
  [zc@Jenkins software]$ sudo tar -zxf apache-maven-3.9.5-bin.tar.gz -C /usr/local/

  [zc@Jenkins software]$ sudo ln -s /usr/local/apache-maven-3.9.5/ /usr/local/maven
  [zc@Jenkins software]$ sudo vim /etc/profile
  JAVA_HOME=/usr/local/java
  MAVEN_HOME=/usr/local/maven
  MAVEN_OPTS='-Xms256m -Xmx768m -Dfile.encoding=UTF-8'
  PATH=$JAVA_HOME/bin:$MAVEN_HOME/bin:$PATH
  export JAVA_HOME MAVEN_HOME MAVEN_OPTS PATH
  [zc@Jenkins software]$ source /etc/profile
  # 上面配置的 maven 环境变量貌似没用，jenkins 仍然会找不到 mvn 命令
  # 所以后面还是在 Jenkins 管理后台添加了 maven 环境变量
  ```

* 配置  
  ![](/img/jenkins/jenkins_15.png)  
  ![](/img/jenkins/jenkins_16.png)

* 配置 Maven settings.xml
  ```shell
  [zc@Jenkins ~]$ sudo vim /usr/local/apache-maven-3.9.5/conf/settings.xml
  <?xml version="1.0" encoding="UTF-8"?>
  
  <settings xmlns="http://maven.apache.org/SETTINGS/1.2.0"
            xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
            xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.2.0 https://maven.apache.org/xsd/settings-1.2.0.xsd">
  
  <localRepository>/root/repo</localRepository>
  
    <pluginGroups>
    </pluginGroups>
  
    <proxies>
    </proxies>
  
    <servers>
    </servers>
  
    <mirrors>
      <mirror>
        <id>nexus-aliyun</id>
        <mirrorOf>central</mirrorOf>
        <name>Nexus aliyun</name>
        <url>https://maven.aliyun.com/nexus/content/groups/public</url>
      </mirror>
    </mirrors>
  
    <profiles>
    </profiles>
  
    <activeProfiles>
    </activeProfiles>
  
  </settings>
  ```
* 测试  
  ![](/img/jenkins/jenkins_17.png)
