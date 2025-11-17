# 演出票务系统架构图

## 一、总体架构图

```mermaid
graph TB
    subgraph "客户端层 Client Layer"
        A1[C端Web浏览器]
        A2[C端H5移动浏览器]
        A3[C端Android App]
        A4[C端iOS App]
        A5[B端管理后台]
        A6[运营后台]
        A7[总代票务系统]
    end
    
    subgraph "CDN层"
        CDN[CDN静态资源分发]
    end
    
    subgraph "负载均衡层 Load Balancer"
        LB[Nginx负载均衡器]
    end
    
    subgraph "API网关层 API Gateway"
        GW[Spring Cloud Gateway]
        GW1[路由转发]
        GW2[认证鉴权]
        GW3[限流熔断]
        GW4[日志记录]
    end
    
    subgraph "服务注册与配置中心"
        NACOS[Nacos]
        NACOS1[服务注册]
        NACOS2[配置管理]
        NACOS3[服务发现]
    end
    
    subgraph "微服务层 Microservices"
        MS1[用户服务<br/>User Service]
        MS2[项目服务<br/>Event Service]
        MS3[订单服务<br/>Order Service]
        MS4[支付服务<br/>Payment Service]
        MS5[库存服务<br/>Inventory Service]
        MS6[营销服务<br/>Marketing Service]
        MS7[搜索服务<br/>Search Service]
        MS8[通知服务<br/>Notification Service]
    end
    
    subgraph "数据存储层 Data Layer"
        DB1[(MySQL主库)]
        DB2[(MySQL从库1)]
        DB3[(MySQL从库2)]
        REDIS[(Redis集群)]
        ES[(Elasticsearch)]
        OSS[对象存储OSS]
    end
    
    subgraph "消息队列层 Message Queue"
        MQ[RocketMQ]
    end
    
    subgraph "第三方服务 Third Party"
        PAY1[支付宝]
        PAY2[微信支付]
        SMS[短信服务]
        MAP[地图服务]
    end
    
    subgraph "监控与日志 Monitoring"
        SKY[SkyWalking链路追踪]
        ELK[ELK日志系统]
        PROM[Prometheus监控]
        GRAF[Grafana可视化]
    end
    
    A1 --> CDN
    A2 --> CDN
    A3 --> CDN
    A4 --> CDN
    A5 --> CDN
    A6 --> CDN
    A7 --> CDN
    
    CDN --> LB
    LB --> GW
    
    GW --> GW1
    GW --> GW2
    GW --> GW3
    GW --> GW4
    
    GW --> NACOS
    
    GW1 --> MS1
    GW1 --> MS2
    GW1 --> MS3
    GW1 --> MS4
    GW1 --> MS5
    GW1 --> MS6
    GW1 --> MS7
    GW1 --> MS8
    
    MS1 --> NACOS1
    MS2 --> NACOS1
    MS3 --> NACOS1
    MS4 --> NACOS1
    MS5 --> NACOS1
    MS6 --> NACOS1
    MS7 --> NACOS1
    MS8 --> NACOS1
    
    MS1 --> DB1
    MS2 --> DB1
    MS3 --> DB1
    MS5 --> DB1
    MS6 --> DB1
    
    MS1 --> DB2
    MS2 --> DB2
    MS3 --> DB2
    
    MS1 --> REDIS
    MS3 --> REDIS
    MS5 --> REDIS
    
    MS7 --> ES
    MS2 --> OSS
    
    MS3 --> MQ
    MS8 --> MQ
    
    MS4 --> PAY1
    MS4 --> PAY2
    MS8 --> SMS
    MS2 --> MAP
    
    MS1 --> SKY
    MS2 --> SKY
    MS3 --> SKY
    MS4 --> SKY
    MS5 --> SKY
    MS6 --> SKY
    MS7 --> SKY
    MS8 --> SKY
    
    MS1 --> ELK
    MS2 --> ELK
    MS3 --> ELK
    
    MS1 --> PROM
    MS2 --> PROM
    MS3 --> PROM
    
    PROM --> GRAF
```

---

## 二、微服务详细架构图

```mermaid
graph TB
    subgraph "用户服务 User Service"
        U1[用户注册登录]
        U2[JWT认证]
        U3[用户信息管理]
        U4[地址管理]
        U5[实名管理]
        U6[第三方登录]
    end
    
    subgraph "项目服务 Event Service"
        E1[项目管理]
        E2[场次管理]
        E3[票档管理]
        E4[场馆管理]
        E5[座位图管理]
        E6[想看列表]
    end
    
    subgraph "订单服务 Order Service"
        O1[订单创建]
        O2[订单查询]
        O3[订单取消]
        O4[电子票生成]
        O5[退款管理]
        O6[订单超时处理]
    end
    
    subgraph "支付服务 Payment Service"
        P1[支付宝支付]
        P2[微信支付]
        P3[支付回调]
        P4[退款处理]
        P5[支付查询]
    end
    
    subgraph "库存服务 Inventory Service"
        I1[库存初始化]
        I2[库存锁定]
        I3[库存释放]
        I4[库存确认]
        I5[座位分配]
        I6[防超卖机制]
    end
    
    subgraph "营销服务 Marketing Service"
        M1[优惠券管理]
        M2[优惠券发放]
        M3[优惠券使用]
        M4[促销活动]
        M5[营销规则计算]
    end
    
    subgraph "搜索服务 Search Service"
        S1[全文搜索]
        S2[筛选排序]
        S3[搜索建议]
        S4[数据同步]
    end
    
    subgraph "通知服务 Notification Service"
        N1[短信通知]
        N2[邮件通知]
        N3[站内消息]
        N4[推送通知]
        N5[消息队列消费]
    end
    
    O1 --> I2
    O1 --> M5
    O2 --> E1
    O4 --> N1
    O5 --> P4
    O5 --> I3
    
    P3 --> O1
    P3 --> O4
    
    E1 --> S4
    
    style U1 fill:#e1f5ff
    style E1 fill:#fff4e1
    style O1 fill:#ffe1e1
    style P1 fill:#e1ffe1
    style I1 fill:#f0e1ff
    style M1 fill:#ffe1f0
    style S1 fill:#e1fff0
    style N1 fill:#fff0e1
```

---

## 三、数据流架构图

### 3.1 用户购票流程数据流

```mermaid
sequenceDiagram
    participant C as 客户端
    participant GW as API网关
    participant ES as 项目服务
    participant IS as 库存服务
    participant OS as 订单服务
    participant PS as 支付服务
    participant NS as 通知服务
    participant DB as 数据库
    participant RD as Redis
    participant MQ as 消息队列
    
    C->>GW: 1.浏览项目列表
    GW->>ES: 查询项目
    ES->>DB: 读取项目数据
    DB-->>ES: 返回项目列表
    ES-->>GW: 返回结果
    GW-->>C: 展示项目
    
    C->>GW: 2.选择场次票档
    GW->>IS: 检查库存
    IS->>RD: 查询Redis缓存
    RD-->>IS: 返回库存信息
    IS-->>GW: 库存可用
    GW-->>C: 显示可购买
    
    C->>GW: 3.创建订单
    GW->>OS: 创建订单请求
    OS->>IS: 锁定库存
    IS->>RD: Redis原子操作
    RD-->>IS: 锁定成功
    IS-->>OS: 库存已锁定
    OS->>DB: 保存订单
    DB-->>OS: 订单创建成功
    OS-->>GW: 返回订单信息
    GW-->>C: 跳转支付页
    
    C->>GW: 4.发起支付
    GW->>PS: 创建支付订单
    PS->>DB: 保存支付记录
    PS-->>GW: 返回支付参数
    GW-->>C: 跳转第三方支付
    
    C->>PS: 5.支付完成回调
    PS->>PS: 验证签名
    PS->>OS: 更新订单状态
    OS->>IS: 确认扣减库存
    IS->>DB: 更新数据库库存
    OS->>MQ: 发送生成电子票消息
    MQ->>NS: 消费消息
    NS->>OS: 生成电子票
    NS->>C: 发送短信通知
    PS-->>C: 支付成功
```

### 3.2 高并发抢票场景数据流

```mermaid
sequenceDiagram
    participant C1 as 用户1
    participant C2 as 用户2
    participant C3 as 用户N
    participant GW as API网关
    participant SEN as Sentinel限流
    participant IS as 库存服务
    participant RD as Redis
    participant LOCK as 分布式锁
    participant DB as 数据库
    
    C1->>GW: 抢票请求
    C2->>GW: 抢票请求
    C3->>GW: 抢票请求
    
    GW->>SEN: 限流检查
    SEN-->>GW: 通过限流
    
    GW->>IS: 锁定库存
    IS->>LOCK: 获取分布式锁
    LOCK-->>IS: 获取锁成功
    
    IS->>RD: Lua脚本原子扣减
    RD-->>IS: 扣减成功
    
    IS->>LOCK: 释放分布式锁
    IS-->>GW: 库存锁定成功
    
    GW-->>C1: 锁定成功
    GW-->>C2: 库存不足
    GW-->>C3: 库存不足
    
    Note over IS,DB: 异步更新数据库
    IS->>DB: 批量更新库存
```

---

## 四、部署架构图

```mermaid
graph TB
    subgraph "Kubernetes集群"
        subgraph "Namespace: gateway"
            GW1[Gateway Pod 1]
            GW2[Gateway Pod 2]
            GW3[Gateway Pod 3]
        end
        
        subgraph "Namespace: services"
            US1[User Service Pod 1]
            US2[User Service Pod 2]
            ES1[Event Service Pod 1]
            ES2[Event Service Pod 2]
            OS1[Order Service Pod 1]
            OS2[Order Service Pod 2]
            OS3[Order Service Pod 3]
            PS1[Payment Service Pod 1]
            IS1[Inventory Service Pod 1]
            IS2[Inventory Service Pod 2]
            MS1[Marketing Service Pod 1]
            SS1[Search Service Pod 1]
            NS1[Notification Service Pod 1]
        end
        
        subgraph "Namespace: middleware"
            NACOS1[Nacos Pod 1]
            NACOS2[Nacos Pod 2]
            NACOS3[Nacos Pod 3]
        end
        
        subgraph "Service"
            SVC1[Gateway Service]
            SVC2[User Service]
            SVC3[Event Service]
            SVC4[Order Service]
            SVC5[Nacos Service]
        end
        
        subgraph "Ingress"
            ING[Nginx Ingress Controller]
        end
    end
    
    subgraph "外部存储"
        MYSQL[MySQL集群]
        REDIS_C[Redis集群]
        ES_C[Elasticsearch集群]
        MQ_C[RocketMQ集群]
        OSS_C[对象存储]
    end
    
    subgraph "监控系统"
        PROM_S[Prometheus]
        GRAF_S[Grafana]
        SKY_S[SkyWalking]
        ELK_S[ELK Stack]
    end
    
    ING --> SVC1
    SVC1 --> GW1
    SVC1 --> GW2
    SVC1 --> GW3
    
    GW1 --> SVC2
    GW1 --> SVC3
    GW1 --> SVC4
    
    SVC2 --> US1
    SVC2 --> US2
    SVC3 --> ES1
    SVC3 --> ES2
    SVC4 --> OS1
    SVC4 --> OS2
    SVC4 --> OS3
    
    US1 --> SVC5
    ES1 --> SVC5
    OS1 --> SVC5
    
    SVC5 --> NACOS1
    SVC5 --> NACOS2
    SVC5 --> NACOS3
    
    US1 --> MYSQL
    ES1 --> MYSQL
    OS1 --> MYSQL
    
    US1 --> REDIS_C
    OS1 --> REDIS_C
    IS1 --> REDIS_C
    
    SS1 --> ES_C
    OS1 --> MQ_C
    NS1 --> MQ_C
    ES1 --> OSS_C
    
    US1 --> PROM_S
    ES1 --> PROM_S
    OS1 --> PROM_S
    
    PROM_S --> GRAF_S
    
    US1 --> SKY_S
    ES1 --> SKY_S
    OS1 --> SKY_S
    
    US1 --> ELK_S
    ES1 --> ELK_S
    OS1 --> ELK_S
```

---

## 五、网络架构图

```mermaid
graph TB
    subgraph "公网 Internet"
        USER[用户]
    end
    
    subgraph "DMZ区"
        WAF[Web应用防火墙]
        LB1[负载均衡器1]
        LB2[负载均衡器2]
    end
    
    subgraph "应用区 Application Zone"
        subgraph "Web服务器集群"
            WEB1[Web Server 1]
            WEB2[Web Server 2]
            WEB3[Web Server 3]
        end
        
        subgraph "API网关集群"
            API1[API Gateway 1]
            API2[API Gateway 2]
            API3[API Gateway 3]
        end
        
        subgraph "微服务集群"
            MS_CLUSTER[Kubernetes集群<br/>微服务Pod]
        end
    end
    
    subgraph "数据区 Data Zone"
        subgraph "数据库集群"
            DB_MASTER[MySQL主库]
            DB_SLAVE1[MySQL从库1]
            DB_SLAVE2[MySQL从库2]
        end
        
        subgraph "缓存集群"
            REDIS_M1[Redis Master 1]
            REDIS_M2[Redis Master 2]
            REDIS_M3[Redis Master 3]
            REDIS_S1[Redis Slave 1]
            REDIS_S2[Redis Slave 2]
            REDIS_S3[Redis Slave 3]
        end
        
        subgraph "搜索集群"
            ES_N1[ES Node 1]
            ES_N2[ES Node 2]
            ES_N3[ES Node 3]
        end
        
        subgraph "消息队列集群"
            MQ_N1[RocketMQ NameServer]
            MQ_B1[RocketMQ Broker 1]
            MQ_B2[RocketMQ Broker 2]
        end
    end
    
    subgraph "管理区 Management Zone"
        MONITOR[监控系统]
        LOG[日志系统]
        DEPLOY[部署系统]
    end
    
    USER --> WAF
    WAF --> LB1
    WAF --> LB2
    
    LB1 --> WEB1
    LB1 --> WEB2
    LB1 --> WEB3
    
    LB2 --> API1
    LB2 --> API2
    LB2 --> API3
    
    API1 --> MS_CLUSTER
    API2 --> MS_CLUSTER
    API3 --> MS_CLUSTER
    
    MS_CLUSTER --> DB_MASTER
    MS_CLUSTER --> DB_SLAVE1
    MS_CLUSTER --> DB_SLAVE2
    
    DB_MASTER --> DB_SLAVE1
    DB_MASTER --> DB_SLAVE2
    
    MS_CLUSTER --> REDIS_M1
    MS_CLUSTER --> REDIS_M2
    MS_CLUSTER --> REDIS_M3
    
    REDIS_M1 --> REDIS_S1
    REDIS_M2 --> REDIS_S2
    REDIS_M3 --> REDIS_S3
    
    MS_CLUSTER --> ES_N1
    MS_CLUSTER --> ES_N2
    MS_CLUSTER --> ES_N3
    
    MS_CLUSTER --> MQ_N1
    MS_CLUSTER --> MQ_B1
    MS_CLUSTER --> MQ_B2
    
    MS_CLUSTER --> MONITOR
    MS_CLUSTER --> LOG
    DEPLOY --> MS_CLUSTER
```

---

## 六、安全架构图

```mermaid
graph TB
    subgraph "安全防护层"
        subgraph "网络安全"
            FW[防火墙]
            WAF[Web应用防火墙]
            DDOS[DDoS防护]
        end
        
        subgraph "应用安全"
            AUTH[认证中心]
            JWT[JWT Token]
            RBAC[权限控制RBAC]
            LIMIT[接口限流]
        end
        
        subgraph "数据安全"
            ENCRYPT[数据加密]
            MASK[数据脱敏]
            BACKUP[数据备份]
            AUDIT[审计日志]
        end
    end
    
    subgraph "安全策略"
        SSL[HTTPS/SSL]
        CSRF[CSRF防护]
        XSS[XSS防护]
        SQL[SQL注入防护]
    end
    
    subgraph "监控告警"
        ALERT[安全告警]
        MONITOR_SEC[安全监控]
        LOG_SEC[安全日志]
    end
    
    FW --> WAF
    WAF --> DDOS
    DDOS --> AUTH
    
    AUTH --> JWT
    JWT --> RBAC
    RBAC --> LIMIT
    
    LIMIT --> ENCRYPT
    ENCRYPT --> MASK
    MASK --> BACKUP
    BACKUP --> AUDIT
    
    SSL --> CSRF
    CSRF --> XSS
    XSS --> SQL
    
    AUDIT --> ALERT
    ALERT --> MONITOR_SEC
    MONITOR_SEC --> LOG_SEC
```

---

## 七、缓存架构图

```mermaid
graph TB
    subgraph "多级缓存架构"
        subgraph "L1: 浏览器缓存"
            BC[浏览器缓存<br/>静态资源]
        end
        
        subgraph "L2: CDN缓存"
            CDN[CDN缓存<br/>图片、JS、CSS]
        end
        
        subgraph "L3: Nginx缓存"
            NC[Nginx缓存<br/>页面缓存]
        end
        
        subgraph "L4: 应用缓存"
            AC[本地缓存<br/>Caffeine]
        end
        
        subgraph "L5: Redis缓存"
            RC1[热点数据缓存]
            RC2[会话缓存]
            RC3[库存缓存]
            RC4[搜索结果缓存]
        end
        
        subgraph "L6: 数据库"
            DB[(MySQL)]
        end
    end
    
    subgraph "缓存策略"
        STRATEGY1[Cache Aside<br/>旁路缓存]
        STRATEGY2[Write Through<br/>写穿缓存]
        STRATEGY3[Write Behind<br/>异步写入]
    end
    
    BC --> CDN
    CDN --> NC
    NC --> AC
    AC --> RC1
    AC --> RC2
    AC --> RC3
    AC --> RC4
    RC1 --> DB
    RC2 --> DB
    RC3 --> DB
    RC4 --> DB
    
    STRATEGY1 -.-> RC1
    STRATEGY2 -.-> RC3
    STRATEGY3 -.-> DB
```

---

## 八、监控架构图

```mermaid
graph TB
    subgraph "数据采集层"
        APP[应用服务]
        AGENT1[Prometheus Agent]
        AGENT2[SkyWalking Agent]
        AGENT3[Filebeat]
    end
    
    subgraph "数据处理层"
        PROM[Prometheus]
        SKY[SkyWalking OAP]
        LOGSTASH[Logstash]
    end
    
    subgraph "数据存储层"
        PROM_DB[(Prometheus TSDB)]
        ES_DB[(Elasticsearch)]
        SKY_DB[(SkyWalking Storage)]
    end
    
    subgraph "数据展示层"
        GRAFANA[Grafana<br/>指标可视化]
        KIBANA[Kibana<br/>日志查询]
        SKY_UI[SkyWalking UI<br/>链路追踪]
    end
    
    subgraph "告警层"
        ALERT_M[AlertManager]
        ALERT_R[告警规则]
        NOTIFY[通知渠道<br/>钉钉/邮件/短信]
    end
    
    APP --> AGENT1
    APP --> AGENT2
    APP --> AGENT3
    
    AGENT1 --> PROM
    AGENT2 --> SKY
    AGENT3 --> LOGSTASH
    
    PROM --> PROM_DB
    SKY --> SKY_DB
    LOGSTASH --> ES_DB
    
    PROM_DB --> GRAFANA
    ES_DB --> KIBANA
    SKY_DB --> SKY_UI
    
    PROM --> ALERT_M
    ALERT_M --> ALERT_R
    ALERT_R --> NOTIFY
```

---

## 九、CI/CD流程架构图

```mermaid
graph LR
    subgraph "开发阶段"
        DEV[开发人员]
        GIT[Git仓库]
    end
    
    subgraph "CI阶段"
        JENKINS[Jenkins/GitLab CI]
        BUILD[代码构建]
        TEST[单元测试]
        SCAN[代码扫描]
        PACKAGE[打包镜像]
    end
    
    subgraph "CD阶段"
        REGISTRY[镜像仓库]
        K8S_TEST[K8s测试环境]
        K8S_PROD[K8s生产环境]
    end
    
    subgraph "验证阶段"
        AUTO_TEST[自动化测试]
        MANUAL[人工验证]
        ROLLBACK[回滚机制]
    end
    
    DEV --> GIT
    GIT --> JENKINS
    JENKINS --> BUILD
    BUILD --> TEST
    TEST --> SCAN
    SCAN --> PACKAGE
    PACKAGE --> REGISTRY
    
    REGISTRY --> K8S_TEST
    K8S_TEST --> AUTO_TEST
    AUTO_TEST --> MANUAL
    MANUAL --> K8S_PROD
    
    K8S_PROD --> ROLLBACK
    ROLLBACK -.-> K8S_PROD
```

---

## 十、容灾架构图

```mermaid
graph TB
    subgraph "主机房 Primary DC"
        subgraph "应用层"
            APP1[应用集群]
        end
        
        subgraph "数据层"
            DB1[(MySQL主库)]
            REDIS1[(Redis主集群)]
        end
    end
    
    subgraph "备机房 Backup DC"
        subgraph "应用层"
            APP2[应用集群]
        end
        
        subgraph "数据层"
            DB2[(MySQL从库)]
            REDIS2[(Redis从集群)]
        end
    end
    
    subgraph "流量调度"
        DNS[智能DNS]
        LB[全局负载均衡]
    end
    
    subgraph "数据同步"
        SYNC1[MySQL主从同步]
        SYNC2[Redis主从同步]
        SYNC3[文件同步]
    end
    
    subgraph "监控切换"
        HEALTH[健康检查]
        SWITCH[自动切换]
    end
    
    DNS --> LB
    LB --> APP1
    LB --> APP2
    
    APP1 --> DB1
    APP1 --> REDIS1
    APP2 --> DB2
    APP2 --> REDIS2
    
    DB1 --> SYNC1
    SYNC1 --> DB2
    
    REDIS1 --> SYNC2
    SYNC2 --> REDIS2
    
    HEALTH --> APP1
    HEALTH --> APP2
    HEALTH --> SWITCH
    
    SWITCH -.切换.-> LB
```

---

## 架构设计说明

### 1. 架构特点

- **高可用**: 多副本部署、主从架构、自动故障转移
- **高性能**: 多级缓存、读写分离、异步处理
- **高并发**: 分布式锁、限流熔断、消息队列削峰
- **可扩展**: 微服务架构、水平扩展、弹性伸缩
- **安全性**: 多层防护、数据加密、权限控制
- **可观测**: 全链路监控、日志追踪、性能分析

### 2. 技术选型理由

- **Spring Cloud Alibaba**: 成熟的微服务生态，适合国内环境
- **Nacos**: 服务注册与配置管理一体化
- **MySQL**: 成熟稳定的关系型数据库
- **Redis**: 高性能缓存，支持多种数据结构
- **RocketMQ**: 高可靠消息队列，支持事务消息
- **Elasticsearch**: 强大的全文搜索引擎
- **Kubernetes**: 容器编排，自动化部署和扩展

### 3. 性能指标

- **QPS**: 单服务支持1000+ QPS
- **响应时间**: P99 < 500ms
- **可用性**: 99.9%
- **并发用户**: 支持10万+在线用户
- **数据一致性**: 最终一致性

---

**文档版本**: v1.0  
**更新日期**: 2025-11-17  
**编制人**: Kiro AI Assistant
