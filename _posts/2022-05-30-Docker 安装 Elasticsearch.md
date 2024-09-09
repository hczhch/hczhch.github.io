---
layout:       post
title:        "Docker 安装 Elasticsearch"
author:       "Vito"
header-style: text
catalog:      true
tags:
    - docker
---

```shell
[root@mydockerhost ~]# mkdir -p /home/vito/docker/volume/elasticsearch/config
[root@mydockerhost ~]# mkdir -p /home/vito/docker/volume/elasticsearch/data
[root@mydockerhost ~]# mkdir -p /home/vito/docker/volume/elasticsearch/plugins
[root@mydockerhost ~]# chmod -R 777 /home/vito/docker/volume/elasticsearch/data

[root@mydockerhost ~]# docker pull elasticsearch:7.17.4
# 创建临时容器
[root@mydockerhost ~]# docker run -d --name elasticsearch elasticsearch:7.17.4
# 从容器复制配置文件到宿主机
[root@mydockerhost ~]# docker cp elasticsearch:/usr/share/elasticsearch/config/elasticsearch.yml /home/vito/docker/volume/elasticsearch/config/elasticsearch.yml
# 删除临时容器
[root@mydockerhost ~]# docker rm -f elasticsearch

# 创建容器
[root@mydockerhost ~]# docker run -d --restart=always  \
                       -v /home/vito/docker/volume/elasticsearch/config/elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml  \
                       -v /home/vito/docker/volume/elasticsearch/data:/usr/share/elasticsearch/data  \
                       -v /home/vito/docker/volume/elasticsearch/plugins:/usr/share/elasticsearch/plugins  \
                       -p 9200:9200 -p 9300:9300  \
                       -e ES_JAVA_OPTS="-Xms512m -Xmx512m"  \
                       -e "discovery.type=single-node"  \
                       --name elasticsearch  \
                       elasticsearch:7.17.4
```
验证： http://宿主机ip:9200

