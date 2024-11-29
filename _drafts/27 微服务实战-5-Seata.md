* 在 Nacos 上为 Seata 创建一个新的 namespace ：空间名为 `Seata`
* 在 Nacos 的 Seata 命名空间 新建配置（Data ID 为 seataServer.properties 、 Group 为 SEATA_GROUP）
* 配置内容参考 https://github.com/apache/incubator-seata/blob/v1.4.2/script/config-center/config.txt 并按需修改保存。 截取本次案例修改部分的内容如下：

```properties
# 把 store.mode=file 改为 store.mode=db 代表采用数据库存储 Seata-Server 的全局事务数据
store.mode=db

# 修改 store.db 全局事务数据库配置项（驱动、jdbc url、用户名密码等），稍后还要手动创建相应的 database 和 table
store.db.url=jdbc:mysql://mysql-x.dev-mysql:3306/seata?characterEncoding=utf8&useUnicode=true&useSSL=false&serverTimezone=Asia/Shanghai&rewriteBatchedStatements=true
store.db.user=root
store.db.password=xhBQ9QN7d+sF

# 事务逻辑分组 与 Seata 服务端集群的映射关系
# 等号右侧是 seata 集群名（集群名称由 seata 服务端在 registry.conf 文件中设置）
# 'default_tx_group'是自定义的事务逻辑分组名称，每行配置一个事务逻辑分组，可配置多个
# 事务逻辑分组名在 seata 客户端中会被用到（ seata 客户端需要指明其所属的事务逻辑分组，通常在 yml 或 properties 文件中通过 seata.tx-service-group 配置项指定）
service.vgroupMapping.default_tx_group=default
# TC服务列表，仅注册中心为file时使用，nacos 作为注册中心时可忽略或删除该项
service.default.grouplist=127.0.0.1:8091
```

* 创建并初始化 Seata-Server 全局事务数据库
  * https://github.com/apache/incubator-seata/blob/v1.4.2/script/server/db/mysql.sql
    * global_table 保存全局事务数据；　　
    * branch_table 保存分支事务数据；
    * lock_table 保存锁定资源数据

```sql
-- -------------------------------- The script used when storeMode is 'db' --------------------------------
-- the table to store GlobalSession data
CREATE TABLE IF NOT EXISTS `global_table`
(
    `xid`                       VARCHAR(128) NOT NULL,
    `transaction_id`            BIGINT,
    `status`                    TINYINT      NOT NULL,
    `application_id`            VARCHAR(32),
    `transaction_service_group` VARCHAR(32),
    `transaction_name`          VARCHAR(128),
    `timeout`                   INT,
    `begin_time`                BIGINT,
    `application_data`          VARCHAR(2000),
    `gmt_create`                DATETIME,
    `gmt_modified`              DATETIME,
    PRIMARY KEY (`xid`),
    KEY `idx_gmt_modified_status` (`gmt_modified`, `status`),
    KEY `idx_transaction_id` (`transaction_id`)
) ENGINE = InnoDB
  DEFAULT CHARSET = utf8;

-- the table to store BranchSession data
CREATE TABLE IF NOT EXISTS `branch_table`
(
    `branch_id`         BIGINT       NOT NULL,
    `xid`               VARCHAR(128) NOT NULL,
    `transaction_id`    BIGINT,
    `resource_group_id` VARCHAR(32),
    `resource_id`       VARCHAR(256),
    `branch_type`       VARCHAR(8),
    `status`            TINYINT,
    `client_id`         VARCHAR(64),
    `application_data`  VARCHAR(2000),
    `gmt_create`        DATETIME(6),
    `gmt_modified`      DATETIME(6),
    PRIMARY KEY (`branch_id`),
    KEY `idx_xid` (`xid`)
) ENGINE = InnoDB
  DEFAULT CHARSET = utf8;

-- the table to store lock data
CREATE TABLE IF NOT EXISTS `lock_table`
(
    `row_key`        VARCHAR(128) NOT NULL,
    `xid`            VARCHAR(128),
    `transaction_id` BIGINT,
    `branch_id`      BIGINT       NOT NULL,
    `resource_id`    VARCHAR(256),
    `table_name`     VARCHAR(32),
    `pk`             VARCHAR(36),
    `gmt_create`     DATETIME,
    `gmt_modified`   DATETIME,
    PRIMARY KEY (`row_key`),
    KEY `idx_branch_id` (`branch_id`)
) ENGINE = InnoDB
  DEFAULT CHARSET = utf8;
```

* install seata server

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: dev-seata
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: seata-server-config
  namespace: dev-seata
data:
  registry.conf: |
    registry {
      # Seata-Server 支持以下几种注册中心，这里改为 nacos，默认是 file 文件形式不介入任何注册中心。
      # file 、nacos 、eureka、redis、zk、consul、etcd3、sofa
      type = "nacos"
    
      # 负载均衡采用随机策略
      loadBalance = "RandomLoadBalance"
      loadBalanceVirtualNodes = 10
    
      # nacos 注册中心配置
      nacos {
        application = "seata-server"
        serverAddr = "nacos-headless.dev-nacos:8848"
        # 分配应用组，采用默认值 SEATA_GROUP 即可
        group = "SEATA_GROUP"
        namespace = "53b846ba-0cfe-417f-a225-4f0bac803abe" # Nacos 中创建的名为 `Seata` 的 namespace 的 ID
        # 指定注册至nacos注册中心的集群名
        cluster = "default"
        # Nacos 用户名和密码
        username = "nacos"
        password = "nacos"
      }
    }
    
    config {
      # Seata-Server 支持以下配置中心产品，这里设置为 nacos，默认是 file 即文件形式保存配置内容。
      # file、nacos 、apollo、zk、consul、etcd3
      type = "nacos"
    
      # nacos 配置中心配置
      nacos {
        serverAddr = "nacos-headless.dev-nacos:8848"
        namespace = "53b846ba-0cfe-417f-a225-4f0bac803abe"
        group = "SEATA_GROUP"
        username = "nacos"
        password = "nacos"
        # Nacos dataId
        dataId = "seataServer.properties"
      }
    }
---
apiVersion: v1
kind: Service
metadata:
  name: seata-server
  namespace: dev-seata
  labels:
    app: seata-server
spec:
  selector:
    app: seata-server
  type: ClusterIP
  ports:
    - port: 8091
      protocol: TCP
      name: http
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: seata-server
  namespace:  dev-seata
  labels:
    app: seata-server
spec:
  replicas: 3
  selector:
    matchLabels:
      app: seata-server
  template:
    metadata:
      labels:
        app: seata-server
    spec:
      containers:
        - name: seata-server
          image: seataio/seata-server:1.4.2
          imagePullPolicy: IfNotPresent
          env:
            - name: SEATA_CONFIG_NAME
              value: file:/root/seata-config/registry
          ports:
            - name: http
              containerPort: 8091
              protocol: TCP
          volumeMounts:
            - name: seata-config
              mountPath: /root/seata-config
            - name: timezone
              mountPath: /etc/localtime
              readOnly: true
      volumes:
        - name: seata-config
          configMap:
            name: seata-server-config
        - name: timezone
          hostPath:
            path: /usr/share/zoneinfo/Asia/Shanghai
```

