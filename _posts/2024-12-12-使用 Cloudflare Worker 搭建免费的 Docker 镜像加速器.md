---
layout:       post
title:        "使用 Cloudflare Workers 搭建免费的 Docker 镜像加速器"
author:       "Vito"
header-style: text
catalog:      true
tags:
  - Docker
  - Docker 镜像加速器
  - Cloudflare
---

### 省事版
* 直接使用博主搭建好的 Docker 镜像加速器
* [https://docker.hczhch.us.kg/](https://docker.hczhch.us.kg/){:target="_blank"}

### 准备工作
* Github 账号
* Cloudflare
* 在 Cloudflare 托管一个域名

### 搭建
* Fork repository [https://github.com/ciiiii/cloudflare-docker-proxy/](https://github.com/ciiiii/cloudflare-docker-proxy/){:target="_blank"} 到自己的 Github 账号
* 进入 fork 好的 repository ，点击 `Settings` -> `Secrets and variables` -> `Actions` -> `Repository secrets` ，添加 Name 为 `CUSTOM_DOMAIN` 的 secret，Value 就是托管在 Cloudflare 的域名
* 修改 repository 中的 `wrangler.toml` ，将所有 route 中的 `libcuda.so` 修改为托管在 Cloudflare 的域名，并删除 route 前的注释符号 `#`
* 修改 repository 中的 `README.md`，将所有的 `https://github.com/ciiiii/cloudflare-docker-proxy` 修改为自己的 repository 地址
* 最后点击 repository README 中的 `Deploy to Cloudflare Workers` 图标，将项目部署到 Cloudflare Workers 

