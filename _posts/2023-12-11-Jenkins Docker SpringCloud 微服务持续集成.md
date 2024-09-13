---
layout:       post
title:        "Jenkins Docker SpringCloud 微服务持续集成"
author:       "Vito"
header-style: text
catalog:      true
tags:
    - SonarQube
    - Jenkins
---






### 环境准备

| ip                         | server                                     | Mem   |
|----------------------------|--------------------------------------------|-------|
| 192.168.23.131             | Nginx<br/>SSH Port 2222                    | 512m  |
| 192.168.23.13              | Jenkins、Docker<br/>http://jenkins.zhch.lan | 2048m |
| 192.168.23.12              | Gitlab<br/>http://git.zhch.lan             | 3072m |
| 192.168.23.15              | SonarQube<br/>http://192.168.23.15:9000    | 2048m |
| 192.168.23.11              | Docker、Harbor<br/>https://harbor.zhch.lan  | 2048m |
| 192.168.23.16<br/>Server 1 | Docker                                     | 2048m |
| 192.168.23.17<br/>Server 2 | Docker                                     | 2048m |

### 代码上传到 gitlab

* 后端项目 tensquare_parent
* 前端项目 tensquareAdmin

### 后端项目
* jenkins 安装插件：`Publish Over SSH`、`Extended Choice Parameter`、`Environment Injector`
* jenkins server 生成密钥对，并将公钥发送给 Server 1 和 Server 2 （免密登录）
  ```shell
  # ssh-keygen -t rsa -b 4096
  # ssh-keygen -t dsa 
  # ssh-keygen -t ecdsa -b 521
  # ssh-keygen -t ed25519
  [root@Jenkins ~]# ssh-keygen -t rsa -b 4096
  
  [root@Jenkins ~]# ssh-copy-id 192.168.23.16
  
  [root@Jenkins ~]# ssh-copy-id 192.168.23.17
  ```
  ![](/img/jenkins/jenkins_50.png)  

* 在 jenkins 上创建流水线 item : tensquare_back  
  ![](/img/jenkins/jenkins_51.png)

* 在项目 tensquare_parent 根目录中创建 sonar-project.properties 文件
    ```properties
    # must be unique in a given SonarQube instance
    sonar.projectKey=tensquare_parent
    # this is the name and version displayed in the SonarQube UI. Was mandatory prior to SonarQube 6.1.
    sonar.projectName=tensquare_parent
    sonar.projectVersion=1.0
    sonar.modules=tensquare_common,tensquare_eureka_server,tensquare_zuul,tensquare_admin_service,tensquare_gathering
    ```
  
* 在项目 tensquare_parent 每个 module 的根目录中创建 sonar-project.properties 文件
    ```properties
    # Path is relative to the sonar-project.properties file. Replace "\" by "/" on Windows.
    # This property is optional if sonar.modules is set.
    sonar.sources=.
    sonar.exclusions=**/test/**,**/target/**
    sonar.java.binaries=.
    
    sonar.java.source=1.8
    sonar.java.target=1.8
    #sonar.java.libraries=**/target/classes/**
    
    # Encoding of the source code. Default is default system encoding
    sonar.sourceEncoding=UTF-8
    ```
  
* 在项目 tensquare_parent 每个 module 的根目录中创建 Dockerfile 文件（ 注意每个 module 的 EXPOSE 不一样 ）
  ```dockerfile
  FROM openjdk:8-jdk-alpine
  COPY target/app.jar app.jar
  EXPOSE 9001
  ENTRYPOINT ["java","-jar","/app.jar"]
  ```

* 在项目 tensquare_parent 根目录中创建 Pipeline 脚本 Jenkinsfile
  ```Jenkinsfile
  def harborUrl = "harbor.zhch.lan"
  def harborAuth = "7e2f3356-45b3-4d89-b727-967569d5c2ae"
  def harborProject = "tensquare"
  
  def gitUrl = "git@git.zhch.lan:test/tensquare_parent.git"
  def gitAuth = "4488bfb8-2d68-4419-a79e-ccbb9928c3fe"
  
  def projectName = "tensquare"
  def version = new Date().format("yyyy.MMdd.HHmmss", TimeZone.getTimeZone('Asia/Shanghai'))
  def workDir = "/root/jenkins/build"
  def contextPath = "${workDir}/${projectName}/${version}"
  
  node {
      //获取当前选择的微服务名称
      def selectedServices = "${SERVICE_NAME}".split(",")
      //获取当前选择的服务器名称
      def selectedHosts = "${HOST_NAME}".split(",")
  
      stage('Clone') {
          echo "Create contextPath: ${contextPath}"
          sh "mkdir -p ${contextPath}"
          dir("${contextPath}") {
              echo "Checkout start"
              checkout scmGit(branches: [[name: '*/${BRANCH_NAME}']], extensions: [], userRemoteConfigs: [[credentialsId: "${gitAuth}", url: "${gitUrl}"]])
              echo "Checkout done."
          }
      }
  
      // 小项目，直接审查整个项目的代码，没有只审查选中的模块
      stage('Check') {
          script {
              scannerHome = tool 'sonarqube-scanner-5.0.1'
          }
          dir("${contextPath}") {
              withSonarQubeEnv('sonarqube-9.9') {
                  sh "${scannerHome}/bin/sonar-scanner"
              }
          }
      }
  
      stage('Build,Publish') {
          dir("${contextPath}") {
              // 安装父工程 -N,--non-recursive 表示不递归到子项目
              sh "mvn clean install -N"
              // 安装 common module
              sh "mvn -f tensquare_common clean install -Dmaven.test.skip=true"
          }
  
          // 登录 harbor
          withCredentials([usernamePassword(credentialsId: "$harborAuth", passwordVariable: 'PASSWD', usernameVariable: 'UNAME')]) {
              //sh "docker login -u $UNAME -p $PASSWD $harborUrl"
              sh "echo $PASSWD | docker login -u $UNAME --password-stdin $harborUrl"
          }
  
          for(int i=0; i<selectedServices.length; i++){
              def serviceInfo = selectedServices[i].split("#");
              //当前遍历的微服务名称
              def serviceName = serviceInfo[0]
              //当前遍历的微服务端口
              def servicePort = serviceInfo[1]
  
              def containerName = "${projectName}-${BRANCH_NAME}-${serviceName}"
              def image = "${harborUrl}/${harborProject}/${containerName}:${version}"
  
              dir("${contextPath}/${serviceName}"){
                  sh "mvn clean package -Dmaven.test.skip=true"
                  sh "mv target/*.jar target/app.jar"
  
                  sh "docker build -t ${image} ."
  
                  // 推送镜像
                  sh "docker push ${image}"
  
                  // 删除本地镜像
                  sh "docker rmi ${image}"
              }
  
              // 部署
              for(int j=0; j<selectedHosts.length; j++){
                  def host = selectedHosts[j]
                  sshPublisher(publishers: [sshPublisherDesc(configName: "$host", transfers: [sshTransfer(cleanRemote: false, excludes: '', execCommand: "/root/tensquare/deploy.sh ${harborUrl} ${image} ${containerName} ${servicePort} /root/${projectName}/${serviceName}/conf --spring.config.additional-location=/conf/additional.yml", execTimeout: 120000, flatten: false, makeEmptyDirs: false, noDefaultExcludes: false, patternSeparator: '[, ]+', remoteDirectory: '', remoteDirectorySDF: false, removePrefix: '', sourceFiles: '')], usePromotionTimestamp: false, useWorkspaceInPromotion: false, verbose: false)])
              }
          }
     }
  
      stage("Clean") {
          echo "Start clean ..."
          sh "rm -rf ${contextPath}"
          sh "rm -rf ${contextPath}@tmp"
          echo "Clean done."
      }
  }
  ```

* 在 Server 1 和 Server 2 中为每个模块编写 springboot 配置文件 `/root/${projectName}/${serviceName}/conf/additional.yml`
* 在 Server 1 和 Server 2 中编写部署脚本 deploy.sh
  ```shell
  [root@Server1 ~]# vim /root/tensquare/deploy.sh 
  #! /bin/sh
  # 接收外部参数
  harbor_url=$1
  image=$2
  container_name=$3
  port=$4
  config_dir=$5
  params=$6
  
  # 删除旧容器和旧镜像
  old_containerId=`docker ps -a | grep -w ${container_name} | awk '{print $1}'`
  old_image=`docker ps -a | grep -w ${container_name} | awk '{print $2}'`
  if [ "$old_containerId" !=  "" ] ; then
      docker stop $old_containerId
      docker rm $old_containerId
  fi
  if [ "$old_image" !=  "" ] ; then
      docker rmi $old_image
  fi
  
  # 登录Harbor
  echo Harbor12345 | docker login -u vito --password-stdin $harbor_url
  
  # 下载镜像
  docker pull $image
  
  # 启动容器
  docker run --ulimit nofile=1024 -d -p $port:$port --privileged=true -v $config_dir:/conf --name $container_name $image $params
  # --ulimit nofile=1024 解决容器启动报错：library initialization failed - unable to allocate file descriptor table - out of memory
  ```
  * `/root/${projectName}/${serviceName}/conf` 会被挂载到容器 `/conf`
  * 启动容器时会追加参数 `--spring.config.additional-location=/conf/additional.yml`

* 部署截图  
  ![](/img/jenkins/jenkins_52.png)  

### 前端项目
* jenkins 安装插件：`NodeJS`，然后配置 NodeJS 工具  
  ![](/img/jenkins/jenkins_53.png)  

* 修改 Nginx 配置文件  
  ```shell
  [root@Nginx-1 ~]# vim /usr/local/nginx/conf/nginx.conf
  upstream tensquareZuulDev {
      server 192.168.23.16:10020;
      server 192.168.23.17:10020;
  
  }
  server {
      listen 80;
      server_name tensquare-zuul-dev.zhch.lan;
  
      location / {
          proxy_set_header    Host $host;
          proxy_set_header    X-Real-IP $remote_addr;
          proxy_set_header    X-Forwarded-For $proxy_add_x_forwarded_for;
          proxy_pass http://tensquareZuulDev;
      }
  }
  
  upstream tensquareFrontDev {
      server 192.168.23.16:9528;
      server 192.168.23.17:9528;
  
  }
  server {
      listen 80;
      server_name tensquare-front-dev.zhch.lan;
  
      location / {
          proxy_set_header    Host $host;
          proxy_set_header    X-Real-IP $remote_addr;
          proxy_set_header    X-Forwarded-For $proxy_add_x_forwarded_for;
          proxy_pass http://tensquareFrontDev;
      }
  }
  ```
  
* 修改前端项目 tensquareAdmin 中的配置文件 config/prod.env.js
  ```js
  'use strict'
  module.exports = {
  NODE_ENV: '"production"',
  BASE_API: '"http://tensquare-zuul-dev.zhch.lan"'
  }
  ```
  
* 在 jenkins 上创建流水线 item : tensquare_front  
  ![](/img/jenkins/jenkins_54.png)  

* 在项目 tensquareAdmin 的根目录中创建 Dockerfile 文件
  ```dockerfile
  FROM nginx:1.25.3-alpine3.18
  # 注意 dist/ 后面不要带上符号‘*’
  COPY dist/ /usr/share/nginx/html/
  ```

* 在项目 tensquareAdmin 根目录中创建 Pipeline 脚本 Jenkinsfile
  ```Jenkinsfile
  def harborUrl = "harbor.zhch.lan"
  def harborAuth = "7e2f3356-45b3-4d89-b727-967569d5c2ae"
  def harborProject = "tensquare-front"
  
  def gitUrl = "git@git.zhch.lan:test/tensquareadmin.git"
  def gitAuth = "4488bfb8-2d68-4419-a79e-ccbb9928c3fe"
  
  def projectName = "tensquare-front"
  def version = new Date().format("yyyy.MMdd.HHmmss", TimeZone.getTimeZone('Asia/Shanghai'))
  def workDir = "/root/jenkins/build"
  def contextPath = "${workDir}/${projectName}/${version}"
  
  node {
      stage('Clone') {
          echo "Create contextPath: ${contextPath}"
          sh "mkdir -p ${contextPath}"
          dir("${contextPath}") {
              echo "Checkout start"
              checkout scmGit(branches: [[name: '*/${BRANCH_NAME}']], extensions: [], userRemoteConfigs: [[credentialsId: "${gitAuth}", url: "${gitUrl}"]])
              echo "Checkout done."
          }
      }
  
      stage('Build') {
          dir("${contextPath}") {
              nodejs('nodejs12') {
                  sh '''
                      npm install
                      npm run build
                  '''
              }
          }
      }
  
      def containerName = "${projectName}-${BRANCH_NAME}"
      def image = "${harborUrl}/${harborProject}/${containerName}:${version}"
      stage('Create image') {
          dir("${contextPath}") {
              sh "docker build -t ${image} ."
  
              // 登录 harbor
              withCredentials([usernamePassword(credentialsId: "$harborAuth", passwordVariable: 'PASSWD', usernameVariable: 'UNAME')]) {
                  sh "echo $PASSWD | docker login -u $UNAME --password-stdin $harborUrl"
              }
              // 推送镜像
              sh "docker push ${image}"
  
              // 删除本地镜像
              sh "docker rmi ${image}"
          }
      }
  
      stage('Publish') {
          dir("${contextPath}") {
              //获取当前选择的服务器名称
              def selectedHosts = "${HOST_NAME}".split(",")
              // 部署
              for(int j=0; j<selectedHosts.length; j++){
                  def host = selectedHosts[j]
                  sshPublisher(publishers: [sshPublisherDesc(configName: "$host", transfers: [sshTransfer(cleanRemote: false, excludes: '', execCommand: "/root/tensquare-front/deploy.sh ${harborUrl} ${image} ${containerName}", execTimeout: 120000, flatten: false, makeEmptyDirs: false, noDefaultExcludes: false, patternSeparator: '[, ]+', remoteDirectory: '', remoteDirectorySDF: false, removePrefix: '', sourceFiles: '')], usePromotionTimestamp: false, useWorkspaceInPromotion: false, verbose: false)])
              }
          }
      }
  
      stage("Clean") {
          echo "Start clean ..."
          sh "rm -rf ${contextPath}"
          sh "rm -rf ${contextPath}@tmp"
          echo "Clean done."
      }
  }
  ```
  
* 在 Server 1 和 Server 2 中编写部署脚本 deploy.sh
  ```shell
  [root@Server1 ~]# vim /root/tensquare-front/deploy.sh 
  #! /bin/sh
  # 接收外部参数
  harbor_url=$1
  image=$2
  container_name=$3
  
  # 删除旧容器和旧镜像
  old_containerId=`docker ps -a | grep -w ${container_name} | awk '{print $1}'`
  old_image=`docker ps -a | grep -w ${container_name} | awk '{print $2}'`
  if [ "$old_containerId" !=  "" ] ; then
      docker stop $old_containerId
      docker rm $old_containerId
  fi
  if [ "$old_image" !=  "" ] ; then
      docker rmi $old_image
  fi
  
  # 登录Harbor
  echo Harbor12345 | docker login -u vito --password-stdin $harbor_url
  
  # 下载镜像
  docker pull $image
  
  # 启动容器
  docker run -d -p 9528:80 --name $container_name $image
  ```
* 前端部署完成后，访问 `http://tensquare-front-dev.zhch.lan` 验证部署是否成功

* 项目源码
  * 后端：`https://gitee.com/hczhch/tensquare_parent/tree/docker/`
  * 前端：`https://gitee.com/hczhch/tensquare-admin/tree/docker/`
