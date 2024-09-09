---
layout:       post
title:        "Docker CIG"
author:       "Vito"
header-style: text
catalog:      true
tags:
    - docker
---

### CIG
* CAdvisor 监控收集
* InfluxDB 存储数据
* Grafana 图表展示

### Install
* 给 grafana 挂载目录授权
  ```shell
  # grafana 容器未已 root 用户身份运行
  [root@mydockerhost cig]# mkdir -p /home/vito/docker/volume/cig_grafana/data/
  [root@mydockerhost cig]# chmod 777 /home/vito/docker/volume/cig_grafana/data/
  ```


* 创建名为 cig 的目录，在目录中新建 docker-compose.yml 文件，内容如下： 
  ```yaml
  version: '3.8'
  
  #volumes:
  #  grafana_data: { }
  
  services:
    influxdb:
      #image: influxdb:2.2.0-alpine
      image: tutum/influxdb:0.9
      restart: always
      environment:
        - PRE_CREATE_DB=cadvisor
      ports:
        - "8083:8083"
        - "8086:8086"
      privileged: true
      volumes:
        - /home/vito/docker/volume/cig_influxdb/data:/data
    
    cadvisor:
      image: google/cadvisor:v0.33.0
      links:
        - influxdb:influxsrv
      command: -storage_driver=influxdb -storage_driver_db=cadvisor -storage_driver_host=influxsrv:8086
      restart: always
      ports:
        - "10580:8080"
      privileged: true
      volumes:
        - /:/rootfs:ro
        - /sys:/sys:ro
        - /var/lib/docker/:/var/lib/docker:ro
        - /var/run:/var/run:rw
      depends_on:
        - influxdb
    
    grafana:
      image: grafana/grafana:8.5.3
      user: "104"
      restart: always
      links:
        - influxdb:influxsrv
      ports:
        - "3000:3000"
      #privileged: true
      volumes:
        #- grafana_data:/var/lib/grafana
        - /home/vito/docker/volume/cig_grafana/data:/var/lib/grafana
      environment:
        - HTTP_USER=admin
        - HTTP_PASS=admin
        - INFLUXDB_HOST=influxsrv
        - INFLUXDB_PORT=8086
        - INFLUXDB_NAME=cadvisor
        - INFLUXDB_USER=root
        - INFLUXDB_PASS=root
      depends_on:
        - influxdb
  ```

  ![](/img/docker/grafana_1.png)  
  ![](/img/docker/grafana_2.png)  
  ![](/img/docker/grafana_3.png)  
