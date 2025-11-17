# 演出票务系统业务接口流程图

## 目录

1. [用户注册登录流程](#一用户注册登录流程)
2. [项目浏览和搜索流程](#二项目浏览和搜索流程)
3. [完整购票流程](#三完整购票流程)
4. [支付流程](#四支付流程)
5. [订单管理流程](#五订单管理流程)
6. [退款流程](#六退款流程)
7. [优惠券使用流程](#七优惠券使用流程)
8. [B端项目管理流程](#八b端项目管理流程)
9. [B端订单管理流程](#九b端订单管理流程)
10. [运营后台管理流程](#十运营后台管理流程)

---

## 一、用户注册登录流程

### 1.1 手机号注册流程

```mermaid
sequenceDiagram
    participant C as 客户端
    participant GW as API网关
    participant US as 用户服务
    participant SMS as 短信服务
    participant Redis as Redis
    participant DB as 数据库
    
    C->>GW: POST /api/user/send-sms<br/>{phone, type: "register"}
    GW->>US: 转发请求
    US->>DB: 检查手机号是否已注册
    DB-->>US: 未注册
    US->>US: 生成6位验证码
    US->>Redis: 存储验证码(key: sms:register:{phone}, ttl: 5分钟)
    US->>SMS: 发送短信验证码
    SMS-->>US: 发送成功
    US-->>GW: {code: 1000, message: "发送成功"}
    GW-->>C: 返回结果
    
    Note over C: 用户输入验证码
    
    C->>GW: POST /api/user/register<br/>{phone, password, code, nickname}
    GW->>US: 转发请求
    US->>Redis: 验证验证码
    Redis-->>US: 验证码正确
    US->>US: BCrypt加密密码
    US->>DB: 插入用户记录
    DB-->>US: 插入成功
    US->>US: 生成JWT Token
    US->>Redis: 存储Token(ttl: 2小时)
    US-->>GW: {code: 1000, data: {token, user}}
    GW-->>C: 返回Token和用户信息
```

### 1.2 手机号登录流程

```mermaid
sequenceDiagram
    participant C as 客户端
    participant GW as API网关
    participant US as 用户服务
    participant Redis as Redis
    participant DB as 数据库
    
    C->>GW: POST /api/user/login<br/>{phone, password}
    GW->>US: 转发请求
    US->>DB: 查询用户信息
    DB-->>US: 返回用户数据
    US->>US: 验证密码(BCrypt)
    alt 密码正确
        US->>US: 生成JWT Token
        US->>Redis: 存储Token(ttl: 2小时)
        US->>DB: 更新最后登录时间和IP
        US-->>GW: {code: 1000, data: {token, user}}
        GW-->>C: 返回Token和用户信息
    else 密码错误
        US-->>GW: {code: 2004, message: "密码错误"}
        GW-->>C: 返回错误
    end
```

### 1.3 第三方登录流程（微信）

```mermaid
sequenceDiagram
    participant C as 客户端
    participant GW as API网关
    participant US as 用户服务
    participant WX as 微信开放平台
    participant DB as 数据库
    
    C->>WX: 1. 请求微信授权
    WX-->>C: 返回授权码code
    
    C->>GW: 2. POST /api/user/oauth/wechat<br/>{code}
    GW->>US: 转发请求
    US->>WX: 3. 用code换取access_token和openid
    WX-->>US: 返回access_token和openid
    US->>WX: 4. 获取用户信息
    WX-->>US: 返回用户信息
    
    US->>DB: 5. 查询是否已绑定
    alt 已绑定
        DB-->>US: 返回用户信息
        US->>US: 生成JWT Token
        US-->>GW: {code: 1000, data: {token, user}}
    else 未绑定
        US->>DB: 创建新用户
        US->>DB: 保存OAuth信息
        US->>US: 生成JWT Token
        US-->>GW: {code: 1000, data: {token, user, isNew: true}}
    end
    GW-->>C: 返回结果
```

---

## 二、项目浏览和搜索流程

### 2.1 首页数据加载流程

```mermaid
sequenceDiagram
    participant C as 客户端
    participant GW as API网关
    participant ES as 项目服务
    participant OS as 运营服务
    participant Redis as Redis
    participant DB as 数据库
    
    C->>GW: GET /api/home/data<br/>{cityId}
    
    par 并行请求
        GW->>OS: 获取Banner列表
        OS->>Redis: 查询缓存
        alt 缓存命中
            Redis-->>OS: 返回Banner数据
        else 缓存未命中
            OS->>DB: 查询Banner
            DB-->>OS: 返回数据
            OS->>Redis: 写入缓存(ttl: 5分钟)
        end
        OS-->>GW: 返回Banner列表
    and
        GW->>OS: 获取运营位数据
        OS->>Redis: 查询缓存
        Redis-->>OS: 返回运营位数据
        OS-->>GW: 返回运营位列表
    and
        GW->>ES: 获取推荐项目
        ES->>Redis: 查询热门项目缓存
        Redis-->>ES: 返回项目列表
        ES-->>GW: 返回推荐项目
    end
    
    GW-->>C: {banners, positions, events}
```

### 2.2 项目搜索流程

```mermaid
sequenceDiagram
    participant C as 客户端
    participant GW as API网关
    participant SS as 搜索服务
    participant ES as Elasticsearch
    participant Redis as Redis
    
    C->>GW: GET /api/search/events<br/>{keyword, cityId, categoryId, page}
    GW->>SS: 转发请求
    
    SS->>Redis: 查询搜索结果缓存
    alt 缓存命中
        Redis-->>SS: 返回缓存结果
        SS-->>GW: 返回搜索结果
    else 缓存未命中
        SS->>ES: 执行全文搜索
        Note over SS,ES: match查询 + 多条件过滤<br/>+ 排序 + 分页
        ES-->>SS: 返回搜索结果
        SS->>Redis: 缓存结果(ttl: 1分钟)
        SS-->>GW: 返回搜索结果
    end
    
    GW-->>C: {total, events, facets}
```

### 2.3 项目详情查询流程

```mermaid
sequenceDiagram
    participant C as 客户端
    participant GW as API网关
    participant ES as 项目服务
    participant Redis as Redis
    participant DB as 数据库
    
    C->>GW: GET /api/events/{eventId}
    GW->>ES: 转发请求
    
    ES->>Redis: 查询项目详情缓存
    alt 缓存命中
        Redis-->>ES: 返回项目数据
    else 缓存未命中
        ES->>DB: 查询项目详情
        ES->>DB: 查询场次列表
        ES->>DB: 查询场馆信息
        DB-->>ES: 返回完整数据
        ES->>Redis: 缓存项目详情(ttl: 5分钟)
    end
    
    ES->>DB: 更新浏览次数(异步)
    ES-->>GW: 返回项目详情
    GW-->>C: {event, sessions, venue}
```

---

## 三、完整购票流程

### 3.1 查询场次票档流程

```mermaid
sequenceDiagram
    participant C as 客户端
    participant GW as API网关
    participant ES as 项目服务
    participant IS as 库存服务
    participant Redis as Redis
    participant DB as 数据库
    
    C->>GW: GET /api/sessions/{sessionId}/ticket-tiers
    GW->>ES: 转发请求
    ES->>DB: 查询票档列表
    DB-->>ES: 返回票档数据
    
    loop 每个票档
        ES->>IS: 查询实时库存
        IS->>Redis: 获取库存缓存
        Redis-->>IS: 返回库存数据
        IS-->>ES: 返回库存信息
    end
    
    ES-->>GW: 返回票档列表(含库存)
    GW-->>C: {ticketTiers}
```


### 3.2 座位分配流程

```mermaid
sequenceDiagram
    participant C as 客户端
    participant GW as API网关
    participant IS as 库存服务
    participant Redis as Redis
    participant DB as 数据库
    
    C->>GW: POST /api/seats/allocate<br/>{sessionId, ticketTierId, quantity}
    GW->>IS: 转发请求
    
    IS->>Redis: 获取分布式锁
    Redis-->>IS: 获取锁成功
    
    IS->>DB: 查询可用座位
    DB-->>IS: 返回座位列表
    IS->>IS: 随机选择座位
    
    IS->>Redis: 锁定座位(ttl: 5分钟)
    IS->>DB: 更新座位状态为locked
    
    IS->>Redis: 释放分布式锁
    IS-->>GW: 返回分配的座位
    GW-->>C: {seats, lockedUntil}
```

### 3.3 创建订单流程

```mermaid
sequenceDiagram
    participant C as 客户端
    participant GW as API网关
    participant OS as 订单服务
    participant IS as 库存服务
    participant MS as 营销服务
    participant Redis as Redis
    participant DB as 数据库
    
    C->>GW: POST /api/orders/create<br/>{sessionId, items, couponId, addressId}
    GW->>OS: 转发请求
    
    OS->>IS: 1. 锁定库存
    IS->>Redis: Lua脚本原子扣减
    alt 库存充足
        Redis-->>IS: 扣减成功
        IS-->>OS: 库存已锁定
    else 库存不足
        Redis-->>IS: 扣减失败
        IS-->>OS: {code: 5001, message: "库存不足"}
        OS-->>GW: 返回错误
        GW-->>C: 库存不足
    end
    
    OS->>MS: 2. 计算优惠
    MS->>DB: 查询优惠券
    MS->>MS: 计算优惠金额
    MS-->>OS: 返回优惠信息
    
    OS->>OS: 3. 计算订单金额
    OS->>DB: 4. 创建订单记录
    OS->>DB: 5. 创建订单项记录
    DB-->>OS: 保存成功
    
    OS->>Redis: 6. 设置订单过期时间(15分钟)
    OS-->>GW: 返回订单信息
    GW-->>C: {orderId, orderNo, amount, expireTime}
```

---

## 四、支付流程

### 4.1 发起支付流程

```mermaid
sequenceDiagram
    participant C as 客户端
    participant GW as API网关
    participant PS as 支付服务
    participant ALI as 支付宝
    participant DB as 数据库
    
    C->>GW: POST /api/payment/create<br/>{orderId, paymentMethod: "alipay"}
    GW->>PS: 转发请求
    
    PS->>DB: 查询订单信息
    DB-->>PS: 返回订单数据
    
    PS->>PS: 生成支付单号
    PS->>DB: 创建支付记录
    
    PS->>ALI: 调用支付宝SDK
    Note over PS,ALI: 传入订单号、金额、回调地址等
    ALI-->>PS: 返回支付参数
    
    PS-->>GW: 返回支付信息
    GW-->>C: {paymentNo, payUrl, payParams}
    
    C->>ALI: 跳转支付宝支付页面
```

### 4.2 支付回调流程

```mermaid
sequenceDiagram
    participant ALI as 支付宝
    participant PS as 支付服务
    participant OS as 订单服务
    participant IS as 库存服务
    participant MQ as 消息队列
    participant DB as 数据库
    
    ALI->>PS: POST /api/payment/alipay/callback<br/>{支付结果数据}
    
    PS->>PS: 1. 验证签名
    alt 签名验证失败
        PS-->>ALI: 返回失败
    end
    
    PS->>DB: 2. 查询支付记录
    alt 已处理过(幂等性)
        PS-->>ALI: 返回成功
    end
    
    PS->>DB: 3. 更新支付状态
    PS->>OS: 4. 通知订单服务
    
    OS->>DB: 5. 更新订单状态为已支付
    OS->>IS: 6. 确认扣减库存
    IS->>DB: 更新数据库库存
    
    OS->>MQ: 7. 发送生成电子票消息
    OS->>MQ: 8. 发送通知消息
    
    PS-->>ALI: 返回success
```

---

## 五、订单管理流程

### 5.1 订单列表查询流程

```mermaid
sequenceDiagram
    participant C as 客户端
    participant GW as API网关
    participant OS as 订单服务
    participant DB as 数据库
    
    C->>GW: GET /api/orders<br/>{status, page, size}
    GW->>OS: 转发请求(携带用户ID)
    
    OS->>DB: 查询订单列表
    Note over OS,DB: SELECT * FROM t_order<br/>WHERE user_id = ?<br/>AND status = ?<br/>ORDER BY create_time DESC<br/>LIMIT ?, ?
    
    DB-->>OS: 返回订单列表
    
    loop 每个订单
        OS->>DB: 查询订单项
        DB-->>OS: 返回订单项列表
    end
    
    OS-->>GW: 返回订单列表
    GW-->>C: {total, orders}
```

### 5.2 订单详情查询流程

```mermaid
sequenceDiagram
    participant C as 客户端
    participant GW as API网关
    participant OS as 订单服务
    participant ES as 项目服务
    participant DB as 数据库
    
    C->>GW: GET /api/orders/{orderId}
    GW->>OS: 转发请求
    
    OS->>DB: 查询订单信息
    OS->>DB: 查询订单项列表
    OS->>DB: 查询支付信息
    DB-->>OS: 返回订单数据
    
    OS->>ES: 查询项目信息
    ES-->>OS: 返回项目数据
    
    OS-->>GW: 返回完整订单详情
    GW-->>C: {order, items, event, payment}
```

### 5.3 订单取消流程

```mermaid
sequenceDiagram
    participant C as 客户端
    participant GW as API网关
    participant OS as 订单服务
    participant IS as 库存服务
    participant Redis as Redis
    participant DB as 数据库
    
    C->>GW: POST /api/orders/{orderId}/cancel<br/>{reason}
    GW->>OS: 转发请求
    
    OS->>DB: 查询订单状态
    alt 订单不可取消
        OS-->>GW: {code: 4002, message: "订单状态不允许取消"}
        GW-->>C: 返回错误
    end
    
    OS->>DB: 更新订单状态为cancelled
    OS->>IS: 释放库存
    IS->>Redis: 增加库存
    IS->>DB: 更新数据库库存
    
    OS-->>GW: 取消成功
    GW-->>C: {code: 1000, message: "取消成功"}
```

---

## 六、退款流程

### 6.1 申请退款流程

```mermaid
sequenceDiagram
    participant C as 客户端
    participant GW as API网关
    participant OS as 订单服务
    participant DB as 数据库
    
    C->>GW: POST /api/refunds<br/>{orderId, reason, orderItemIds}
    GW->>OS: 转发请求
    
    OS->>DB: 查询订单信息
    alt 订单不可退款
        OS-->>GW: {code: 4003, message: "订单不可退款"}
        GW-->>C: 返回错误
    end
    
    OS->>OS: 计算退款金额和手续费
    OS->>DB: 创建退款记录
    DB-->>OS: 创建成功
    
    OS-->>GW: 退款申请已提交
    GW-->>C: {refundId, refundNo, status: "pending"}
```

### 6.2 退款审核流程（B端）

```mermaid
sequenceDiagram
    participant B as B端商家
    participant GW as API网关
    participant OS as 订单服务
    participant PS as 支付服务
    participant IS as 库存服务
    participant ALI as 支付宝
    participant DB as 数据库
    
    B->>GW: PUT /api/refunds/{refundId}<br/>{status: "approved", opinion}
    GW->>OS: 转发请求
    
    OS->>DB: 更新退款状态为approved
    OS->>PS: 调用退款接口
    
    PS->>DB: 查询支付记录
    PS->>ALI: 调用支付宝退款API
    ALI-->>PS: 退款成功
    
    PS->>DB: 更新支付状态
    PS-->>OS: 退款完成
    
    OS->>DB: 更新订单状态
    OS->>DB: 更新订单项状态
    OS->>IS: 恢复库存
    IS->>DB: 更新库存
    
    OS-->>GW: 退款处理完成
    GW-->>B: {code: 1000, message: "退款成功"}
```

---

## 七、优惠券使用流程

### 7.1 领取优惠券流程

```mermaid
sequenceDiagram
    participant C as 客户端
    participant GW as API网关
    participant MS as 营销服务
    participant Redis as Redis
    participant DB as 数据库
    
    C->>GW: POST /api/coupons/{couponId}/receive
    GW->>MS: 转发请求(携带用户ID)
    
    MS->>Redis: 获取分布式锁
    Redis-->>MS: 获取锁成功
    
    MS->>DB: 查询优惠券信息
    alt 优惠券已领完
        MS-->>GW: {code: 5001, message: "优惠券已领完"}
        GW-->>C: 返回错误
    end
    
    MS->>DB: 检查用户是否已领取
    alt 已领取
        MS-->>GW: {code: 5002, message: "已领取过"}
        GW-->>C: 返回错误
    end
    
    MS->>DB: 创建用户优惠券记录
    MS->>DB: 更新优惠券已领取数量
    MS->>Redis: 释放分布式锁
    
    MS-->>GW: 领取成功
    GW-->>C: {userCoupon}
```

### 7.2 查询可用优惠券流程

```mermaid
sequenceDiagram
    participant C as 客户端
    participant GW as API网关
    participant MS as 营销服务
    participant DB as 数据库
    
    C->>GW: GET /api/user/coupons/available<br/>{eventId, amount}
    GW->>MS: 转发请求
    
    MS->>DB: 查询用户优惠券
    Note over MS,DB: WHERE user_id = ?<br/>AND status = 'unused'<br/>AND valid_from <= NOW()<br/>AND valid_to >= NOW()
    
    DB-->>MS: 返回优惠券列表
    
    MS->>MS: 过滤适用范围
    MS->>MS: 过滤最低消费金额
    MS->>MS: 计算优惠金额
    
    MS-->>GW: 返回可用优惠券
    GW-->>C: {coupons}
```

---

## 八、B端项目管理流程

### 8.1 创建项目流程

```mermaid
sequenceDiagram
    participant B as B端商家
    participant GW as API网关
    participant ES as 项目服务
    participant OSS as 对象存储
    participant DB as 数据库
    
    B->>GW: POST /api/events<br/>{项目信息}
    GW->>ES: 转发请求(携带商家ID)
    
    alt 包含图片上传
        ES->>OSS: 上传图片
        OSS-->>ES: 返回图片URL
    end
    
    ES->>ES: 生成项目编码
    ES->>DB: 插入项目记录
    DB-->>ES: 插入成功
    
    ES-->>GW: 返回项目信息
    GW-->>B: {event}
```

### 8.2 添加场次流程

```mermaid
sequenceDiagram
    participant B as B端商家
    participant GW as API网关
    participant ES as 项目服务
    participant DB as 数据库
    
    B->>GW: POST /api/sessions<br/>{eventId, startTime, endTime}
    GW->>ES: 转发请求
    
    ES->>DB: 查询项目信息
    ES->>DB: 验证商家权限
    
    ES->>DB: 插入场次记录
    DB-->>ES: 插入成功
    
    ES-->>GW: 返回场次信息
    GW-->>B: {session}
```

### 8.3 添加票档流程

```mermaid
sequenceDiagram
    participant B as B端商家
    participant GW as API网关
    participant ES as 项目服务
    participant IS as 库存服务
    participant Redis as Redis
    participant DB as 数据库
    
    B->>GW: POST /api/ticket-tiers<br/>{sessionId, name, price, stock}
    GW->>ES: 转发请求
    
    ES->>DB: 插入票档记录
    DB-->>ES: 插入成功
    
    ES->>IS: 初始化库存
    IS->>Redis: 设置库存缓存
    IS->>DB: 更新场次总库存
    
    ES-->>GW: 返回票档信息
    GW-->>B: {ticketTier}
```

---

## 九、B端订单管理流程

### 9.1 订单列表查询流程（B端）

```mermaid
sequenceDiagram
    participant B as B端商家
    participant GW as API网关
    participant OS as 订单服务
    participant DB as 数据库
    
    B->>GW: GET /api/merchant/orders<br/>{status, eventId, page}
    GW->>OS: 转发请求(携带商家ID)
    
    OS->>DB: 查询商家订单列表
    Note over OS,DB: SELECT * FROM t_order<br/>WHERE merchant_id = ?<br/>AND status = ?<br/>ORDER BY create_time DESC
    
    DB-->>OS: 返回订单列表
    OS-->>GW: 返回结果
    GW-->>B: {total, orders}
```

### 9.2 后台售票流程

```mermaid
sequenceDiagram
    participant B as B端商家
    participant GW as API网关
    participant OS as 订单服务
    participant IS as 库存服务
    participant DB as 数据库
    
    B->>GW: POST /api/backend/sell-ticket<br/>{sessionId, items, buyerInfo, paymentMethod}
    GW->>OS: 转发请求
    
    OS->>IS: 锁定库存
    IS-->>OS: 库存已锁定
    
    OS->>DB: 创建订单(标记为后台售票)
    OS->>DB: 直接标记为已支付
    OS->>DB: 生成电子票
    
    OS-->>GW: 返回订单和电子票
    GW-->>B: {order, tickets}
```

### 9.3 发送电子票流程

```mermaid
sequenceDiagram
    participant B as B端商家
    participant GW as API网关
    participant OS as 订单服务
    participant NS as 通知服务
    participant SMS as 短信服务
    participant DB as 数据库
    
    B->>GW: POST /api/orders/{orderId}/send-ticket
    GW->>OS: 转发请求
    
    OS->>DB: 查询订单和电子票
    DB-->>OS: 返回数据
    
    OS->>NS: 发送电子票通知
    NS->>SMS: 发送短信
    SMS-->>NS: 发送成功
    
    NS->>DB: 记录通知日志
    NS-->>OS: 发送完成
    
    OS-->>GW: 发送成功
    GW-->>B: {code: 1000, message: "发送成功"}
```

---

## 十、运营后台管理流程

### 10.1 创建优惠券流程

```mermaid
sequenceDiagram
    participant O as 运营人员
    participant GW as API网关
    participant MS as 营销服务
    participant DB as 数据库
    
    O->>GW: POST /api/admin/coupons<br/>{优惠券信息}
    GW->>MS: 转发请求
    
    MS->>MS: 生成优惠券编码
    MS->>DB: 插入优惠券记录
    DB-->>MS: 插入成功
    
    MS-->>GW: 返回优惠券信息
    GW-->>O: {coupon}
```

### 10.2 批量发放优惠券流程

```mermaid
sequenceDiagram
    participant O as 运营人员
    participant GW as API网关
    participant MS as 营销服务
    participant MQ as 消息队列
    participant DB as 数据库
    
    O->>GW: POST /api/admin/coupons/{couponId}/batch-send<br/>{userIds}
    GW->>MS: 转发请求
    
    MS->>MQ: 发送批量发放消息
    MS-->>GW: 任务已提交
    GW-->>O: {code: 1000, message: "发放任务已提交"}
    
    Note over MQ,DB: 异步处理
    MQ->>MS: 消费消息
    loop 每个用户
        MS->>DB: 创建用户优惠券记录
    end
    MS->>DB: 更新优惠券已领取数量
```

### 10.3 配置Banner流程

```mermaid
sequenceDiagram
    participant O as 运营人员
    participant GW as API网关
    participant OS as 运营服务
    participant OSS as 对象存储
    participant Redis as Redis
    participant DB as 数据库
    
    O->>GW: POST /api/admin/banners<br/>{name, image, linkType, linkValue}
    GW->>OS: 转发请求
    
    OS->>OSS: 上传Banner图片
    OSS-->>OS: 返回图片URL
    
    OS->>DB: 插入Banner记录
    DB-->>OS: 插入成功
    
    OS->>Redis: 清除Banner缓存
    OS-->>GW: 返回Banner信息
    GW-->>O: {banner}
```

### 10.4 用户管理流程

```mermaid
sequenceDiagram
    participant O as 运营人员
    participant GW as API网关
    participant US as 用户服务
    participant Redis as Redis
    participant DB as 数据库
    
    O->>GW: PUT /api/admin/users/{userId}/freeze
    GW->>US: 转发请求
    
    US->>DB: 更新用户状态为frozen
    DB-->>US: 更新成功
    
    US->>Redis: 删除用户Token
    US-->>GW: 冻结成功
    GW-->>O: {code: 1000, message: "用户已冻结"}
```

### 10.5 商家审核流程

```mermaid
sequenceDiagram
    participant O as 运营人员
    participant GW as API网关
    participant MS as 商家服务
    participant NS as 通知服务
    participant DB as 数据库
    
    O->>GW: PUT /api/admin/merchants/{merchantId}/audit<br/>{status: "approved", opinion}
    GW->>MS: 转发请求
    
    MS->>DB: 更新商家状态
    MS->>DB: 记录审核信息
    DB-->>MS: 更新成功
    
    MS->>NS: 发送审核结果通知
    NS-->>MS: 通知已发送
    
    MS-->>GW: 审核完成
    GW-->>O: {code: 1000, message: "审核完成"}
```

---

## 十一、总代票务系统流程

### 11.1 座位图绘制保存流程

```mermaid
sequenceDiagram
    participant A as 总代人员
    participant GW as API网关
    participant ES as 项目服务
    participant DB as 数据库
    
    A->>GW: POST /api/agent/seat-maps<br/>{venueId, name, configData}
    GW->>ES: 转发请求
    
    ES->>ES: 验证座位图配置
    ES->>DB: 插入座位图记录
    DB-->>ES: 插入成功
    
    ES-->>GW: 返回座位图信息
    GW-->>A: {seatMap}
```

### 11.2 配票流程

```mermaid
sequenceDiagram
    participant A as 总代人员
    participant GW as API网关
    participant IS as 库存服务
    participant DB as 数据库
    
    A->>GW: POST /api/agent/allocate-tickets<br/>{sessionId, ticketTierId, channel, quantity}
    GW->>IS: 转发请求
    
    IS->>DB: 查询票档库存
    alt 库存不足
        IS-->>GW: {code: 5001, message: "库存不足"}
        GW-->>A: 返回错误
    end
    
    IS->>DB: 创建配票记录
    IS->>DB: 更新渠道库存
    IS->>DB: 记录操作日志
    
    IS-->>GW: 配票成功
    GW-->>A: {allocation}
```

### 11.3 打票流程

```mermaid
sequenceDiagram
    participant A as 总代人员
    participant GW as API网关
    participant OS as 订单服务
    participant DB as 数据库
    
    A->>GW: POST /api/agent/print-tickets<br/>{orderIds, templateId}
    GW->>OS: 转发请求
    
    loop 每个订单
        OS->>DB: 查询订单和电子票
        OS->>OS: 生成打印数据
        OS->>DB: 更新票状态为已打印
    end
    
    OS-->>GW: 返回打印数据
    GW-->>A: {printData}
    
    Note over A: 调用打印机打印
```

---

## 接口规范说明

### 1. 统一响应格式

```json
{
  "code": 1000,
  "message": "操作成功",
  "data": {},
  "timestamp": 1700000000000
}
```

### 2. 错误码规范

- 1xxx: 通用错误
- 2xxx: 用户相关错误
- 3xxx: 项目相关错误
- 4xxx: 订单相关错误
- 5xxx: 库存相关错误
- 6xxx: 支付相关错误

### 3. 认证方式

所有需要认证的接口在Header中携带：
```
Authorization: Bearer {JWT_TOKEN}
```

### 4. 分页参数

```json
{
  "page": 1,
  "size": 20,
  "sort": "createTime",
  "order": "desc"
}
```

### 5. 分页响应

```json
{
  "total": 100,
  "page": 1,
  "size": 20,
  "data": []
}
```

---

**文档版本**: v1.0  
**更新日期**: 2025-11-17  
**编制人**: Kiro AI Assistant
