---
layout:       post
title:        "Docker 安装 Maven 私服（nexus）"
author:       "Vito"
header-style: text
catalog:      true
tags:
    - Docker
    - Maven
---

 
### 安装
```shell
[root@mydockerhost vito]# mkdir -p /home/vito/docker/volume/nexus3-data && chmod 777 /home/vito/docker/volume/nexus3-data
[root@mydockerhost vito]# docker run -d -p 8081:8081  \
                        -v /home/vito/docker/volume/nexus3-data:/nexus-data  \
                        --restart=always  \
                        --name nexus3  \
                        sonatype/nexus3:3.39.0
```

* 管理员 admin 的初始密码所在文件： /nexus-data/admin.password
  ```shell
  cat /home/vito/docker/volume/nexus3-data/admin.password 
  ```
  + admin 第一次登陆后必须更改密码


* 了解 Nexus 上的各种类型的仓库（Type）  
  ![](/img/docker/nexus_type.png)

  | 仓库类型 | 说明                                                                                                     |
  |--------------------------------------------------------------------------------------------------------| --- |
  | proxy | 远程(中央)仓库的代理                                                                                            |
  | group | 仓库组，将选定的现有仓库组合到一起                                                                                      |
  | **hosted** | 存放：本团队开发人员部署到 Nexus 的 jar 包 <br/> maven-releases：存放  releasse 版本 <br/> maven-snapshots：存在 snapshots 版本 |

    + `maven-central` 中央仓库的拷贝，如果环境可以访问中央仓库，则可以获取到相关的包，否则没用
    + `maven-public` 仓库组，不是一个实际仓库，只是将现有的仓库组合到一起，可以通过它看到所属组内全部仓库的 jar 信息
    + `maven-releases` ( **Version policy = Release** ) 默认只允许上传不带 SNAPSHOT 版本尾缀的包，默认部署策略是 Disable redeploy 不允许重复上传相同版本号信息的 jar ，避免包版本更新以后使用方无法获取到最新的包。
    + `maven-snapshots` ( **Version policy = Snapshot** ) 只允许上传带 SNAPSHOT 版本尾缀的包，默认部署策略是 Allow redeploy 允许重复上传相同版本号信息的 jar ，每次上传的时候会在 jar 的版本号上面增加时间后缀信息。


### 使用
* 修改 maven-central 仓库代理的远程库地址  
  ![](/img/docker/nexus_central_1.png)  
  ![](/img/docker/nexus_central_2.png)


* 新增仓库（省略，本次案例未新建仓库）


* 新建角色  
  ![](/img/docker/nexus_role_developer.png)  
  ![](/img/docker/nexus_role_deployer.png)


* 新建用户


* 编辑 **Idea安装目录\plugins\maven\lib\maven3\conf\settings.xml** 文件，删除或注释掉文件中配置的默认 mirror 仓库信息，否则可能报错：
  ```text
  Could not transfer artifact xxx from/to maven-default-http-blocker (http://0.0.0.0/): Blocked mirror for repositories: [nexus-fangdr (http://nexusIp:8081/repository/maven-snapshots/, default, releases+snapshots)]
  ```

* 使用私服：  
  + 在项目的 pom.xml 文件中配置 maven-deploy-plugin 插件，默认 maven-deploy-plugin 插件版本为 2.7，只能通过在项目的 pom.xml 配置 distributionManagement 来指定 jar 包部署到私服时所需要的配置信息
    ```xml
    <!-- 升级 maven-deploy-plugin 到 2.8 以上版本，并在 maven 配置文件中配置 jar 包部署到私服所需的信息后，不再需要配置 distributionManagement -->
    <!-- <distributionManagement>
      <repository>
        <id>fdr-deployer</id>
        <name>maven-releases</name>
        <url>http://192.168.23.128:8081/repository/maven-releases/</url>
      </repository>
      <snapshotRepository>
        <id>fdr-developer</id>
        <name>maven-snapshots</name>
        <url>http://192.168.23.128:8081/repository/maven-snapshots/</url>
      </snapshotRepository>
    </distributionManagement> -->
    
    <build>
      <plugins>
        <plugin>
          <groupId>org.apache.maven.plugins</groupId>
          <artifactId>maven-deploy-plugin</artifactId>
          <version>2.8.2</version>
        </plugin>
      </plugins>
    </build>
    ```

  + maven 配置文件（ maven - conf - settings.xml ）
    ```xml
    <?xml version="1.0" encoding="UTF-8"?>
    
    <settings xmlns="http://maven.apache.org/SETTINGS/1.2.0"
              xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
              xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.2.0 https://maven.apache.org/xsd/settings-1.2.0.xsd">
    <localRepository>D:\.m2\repository</localRepository>
    
      <pluginGroups>
      </pluginGroups>
    
      <proxies>
      </proxies>
    
      <servers>
        <server>
          <id>fdr-deployer</id>
          <username>deployer</username>
          <password>123456</password>
        </server>
    
        <server>
          <id>fdr-developer</id>
          <username>vito</username>
          <password>123456</password>
        </server>
      </servers>
    
      <mirrors>
        <!-- <mirror>
          <id>nexus-aliyun</id>
          <mirrorOf>central</mirrorOf>
          <name>Nexus aliyun</name>
          <url>https://maven.aliyun.com/nexus/content/groups/public</url>
        </mirror> -->
        <mirror>
          <id>fdr-developer</id>
          <mirrorOf>central</mirrorOf>
          <name>Nexus fangdr</name>
          <url>http://192.168.23.128:8081/repository/maven-public/</url>
        </mirror>
      </mirrors>
    
      <profiles>
        <profile>
          <id>fangdr-maven-profile</id>
          <properties>
            <!-- 配置部署 jar 包到私服要用到的信息（ maven-deploy-plugin 需 2.8 （含）以上版本 ） -->
            <!-- 格式 serverId::default::仓库地址  -->
            <altSnapshotDeploymentRepository>
              fdr-developer::default::http://192.168.23.128:8081/repository/maven-snapshots/
            </altSnapshotDeploymentRepository>
            <altReleaseDeploymentRepository>
              fdr-deployer::default::http://192.168.23.128:8081/repository/maven-releases/
            </altReleaseDeploymentRepository>
          </properties>
          <!-- 
          repositories 标签：配置引用私服中的 jar 包时需要用到的配置信息；
          id 属性要与 server 标签中的 id 属性一致，这样才能用到 server 中配置的账号信息
           -->
          <repositories>
            <repository>
              <id>fdr-deployer</id>
              <url>http://192.168.23.128:8081/repository/maven-releases/</url>
              <releases>
                <enabled>true</enabled>
                <updatePolicy>always</updatePolicy> 
              </releases>
              <snapshots>
                <enabled>false</enabled>
              </snapshots>
            </repository>
            <repository>
              <id>fdr-developer</id>
              <url>http://192.168.23.128:8081/repository/maven-snapshots/</url>
              <releases>
                <enabled>false</enabled>
              </releases>
              <snapshots>
                <enabled>true</enabled>
                <updatePolicy>always</updatePolicy> 
              </snapshots>
            </repository>
          </repositories>
        </profile>
      </profiles>
      <!-- 指定默认激活的属性文件 -->
      <activeProfiles>
        <activeProfile>fangdr-maven-profile</activeProfile>
      </activeProfiles>
    
    </settings>
    ```


* 将源码发布到私服  
  ```xml
  <!-- 在项目的 pom.xml 文件中的 build plugins 标签下添加如下插件 -->
  <plugin>
      <groupId>org.apache.maven.plugins</groupId>
      <artifactId>maven-source-plugin</artifactId>
      <version>3.2.1</version>
      <executions>
          <execution>
              <phase>package</phase> <!-- 指定绑定到生命周期的哪个阶段 -->
              <goals>
                  <goal>jar-no-fork</goal> <!-- 指定要执行的目标 -->
              </goals>
          </execution>
      </executions>
  </plugin>
  ```


* 本机环境变量设置  
  ```text
  mklink /D D:\Java\jdk D:\Java\zulu8.62.0.19-ca-jdk8.0.332-win_x64
  JAVA_HOME = D:\Java\jdk
  JRE_HOME = %JAVA_HOME%\jre
  
  mklink /D D:\Java\maven D:\Java\apache-maven-3.8.5
  mklink %USERPROFILE%\.m2\settings.xml D:\Java\maven\conf\settings.xml
  MAVEN_HOME = D:\Java\maven
  MAVEN_OPTS = -Xms128m -Xmx512m -Dfile.encoding=UTF-8
  
  PATH = %JAVA_HOME%\bin;%MAVEN_HOME%\bin;%PATH%
  ```

