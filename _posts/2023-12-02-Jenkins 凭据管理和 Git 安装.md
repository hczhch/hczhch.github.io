---
layout:       post
title:        "Jenkins 凭据管理和 Git 安装"
author:       "Vito"
header-style: text
catalog:      true
tags:
    - Jenkins
    - Git
---

* 凭据可以用来存储需要密文保护的数据库密码、Gitlab密码信息、Docker私有仓库密码等，以便 Jenkins 可以和这些第三方的应用进行交互。

* 安装 Credentials Binding 插件

* 新增凭据  
  ![](/img/jenkins/jenkins_12.png)

* Jenkins 安装 Git 插件  
  ![](/img/jenkins/jenkins_13.png)

* Jenkins Server 安装 Git
  ```shell
  [zc@Jenkins ~]$ sudo yum install -y git
  ```

* 新建 Item ， 测试凭据  
  ![](/img/jenkins/jenkins_14.png)
  &nbsp;  
* SSH 密钥类型凭据（以 Gitlab 为例）
  * 生成密钥对
    ```shell
    # 推荐使用 ED25519、RSA；选择 RSA 时，推荐的 key size 是 3072 或更大
    [zc@Jenkins ~]$ ssh-keygen -t ed25519 -C "hczhch@ymail.com"
    # 私钥 ~/.ssh/id_ed25519  公钥：~/.ssh/id_ed25519.pub
    
    [zc@Jenkins ~]$ ssh-keygen -t rsa -b 4096 -C "hczhch@ymail.com"
    # 私钥 ~/.ssh/id_rsa  公钥：~/.ssh/id_rsa.pub
    ```
  * 把公钥添加到 Gitlab 中
  * 把私钥添加到 Jenkins 的凭据中
  * 在 Jenkins server 上执行一次以下命令，将 Gitlab host 添加到 ~/.ssh/known_hosts 中
    ```shell
    [root@Jenkins ~]# ssh -T git@git.zhch.lan
    ```

* 添加 Gitlab SSH 凭据    
  ![](/img/jenkins/jenkins_55.png)  
