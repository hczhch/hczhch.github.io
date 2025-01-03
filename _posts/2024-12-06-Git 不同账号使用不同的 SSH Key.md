---
layout:       post
title:        "Git 不同账号使用不同的 SSH Key"
author:       "Vito"
header-style: text
catalog:      true
tags:
    - Git
    - SSH Key
---

#### 准备多个 SSH Key 
```shell
# 推荐使用 ED25519、RSA；选择 RSA 时，推荐的 key size 是 3072 或更大
ssh-keygen -t ed25519 -C "user1@gmail.com"
# 私钥 ~/.ssh/id_ed25519  公钥：~/.ssh/id_ed25519.pub

#ssh-keygen -t rsa -b 4096 -C "user2@gmail.com"
# 私钥 ~/.ssh/id_rsa  公钥：~/.ssh/id_rsa.pub

ssh-keygen -t rsa -b 4096 -C "user3@gmail.com" -f id_rsa_user3
# 私钥 ~/.ssh/id_rsa_user3  公钥：~/.ssh/id_rsa_user3.pub
```

#### 将各自的公钥添加到各自的账号

#### 编辑 ~/.ssh/config 文件
```text
# ~/.ssh/config

# 不同的平台，如 Github、Gitee 之间可以使用相同的 SSH Key
# Github 中不同账号不能使用相同的 SSH Key

Host gitee.com
    HostName gitee.com
    IdentityFile ~/.ssh/id_ed25519

Host github.com
    HostName github.com
    IdentityFile ~/.ssh/id_ed25519

Host github-user3.com # 任意的自定义名
    HostName github.com
    IdentityFile ~/.ssh/id_rsa_user3
    #ProxyCommand connect -S 127.0.0.1:7779 %h %p # 让 git SSH 使用代理
```

#### 使用样例
```shell
git clone git@gitee.com:user1/project1.git

git clone git@github.com:user2/project2.git

git clone git@github-user3.com:user3/project3.git
```






