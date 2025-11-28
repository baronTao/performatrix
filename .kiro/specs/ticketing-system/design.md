# 演出票务系统设计文档

## 概述

本文档描述演出票务系统的技术架构和设计方案。系统采用前后端分离架构，包含C端移动应用（uni-app）和B端管理后台（Vue），支持高并发抢票场景下的票务交易。

## 技术栈

### 前端技术栈
- **C端移动应用**: uni-app框架
  - **核心技术**: Vue 3 + TypeScript + uni-app
  - **UI组件库**: uni-ui
  - **多端运行**: Android App、iOS App
  - **状态管理**: Pinia
  - **数据请求**: uni.request封装
  
- **B端管理后台**: Vue 3 + TypeScript + Element Plus + Vite
  - **状态管理**: Pinia
  - **数据请求**: Axios
  - **图表库**: ECharts
  - **富文本编辑器**: WangEditor

### 后端技术栈
- **开发框架**: Spring Boot 3.x (JDK 17+)
- **项目管理**: Maven
- **ORM框架**: MyBatis-Plus
- **数据库**: MySQL 8.0
- **缓存**: Redis 7.x
- **消息队列**: RabbitMQ
- **文件存储**: 阿里云OSS
- **支付**: 微信支付SDK + 支付宝SDK
- **短信**: 阿里云短信服务
- **安全认证**: Spring Security + JWT

---

## 系统架构

### 整体架构图

```
┌─────────────────────────────────────────────────────────┐
│                      客户端层                            │
│       C端移动应用(uni-app)    │    B端管理后台(Vue)      │
└─────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────┐
│                    API网关层                             │
│              Spring Cloud Gateway                        │
│         (路由转发、认证鉴权、限流熔断)                    │
└─────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────┐
│                   应用服务层                             │
│  用户服务 │ 项目服务 │ 订单服务 │ 支付服务 │ 运营服务   │
└─────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────┐
│                     数据层                               │
│       MySQL       │       Redis       │      OSS        │
└─────────────────────────────────────────────────────────┘
```

### 服务模块划分

根据需求文档，系统划分为以下服务模块：

| 服务模块 | 职责 | 对应需求 |
|----------|------|----------|
| 用户服务 | 用户注册、登录、个人信息、实名认证、观演人管理、收货地址 | 需求11 |
| 项目服务 | 项目管理、场次管理、票档管理、场馆管理、库存管理 | 需求15、18 |
| 订单服务 | 订单创建、订单查询、退票退款、电子票生成 | 需求8、9、16、17 |
| 支付服务 | 微信支付、支付宝支付、支付回调 | 需求10 |
| 运营服务 | 城市管理、频道管理、Banner管理、黑名单管理、权限管理、日志管理 | 需求19、20、21 |

---

## 数据模型设计

### 1. 用户相关表

```sql
-- 用户表
CREATE TABLE t_user (
  id BIGINT PRIMARY KEY AUTO_INCREMENT COMMENT '用户ID',
  phone VARCHAR(11) NOT NULL UNIQUE COMMENT '手机号',
  password VARCHAR(255) COMMENT '密码（加密）',
  nickname VARCHAR(50) COMMENT '昵称',
  avatar VARCHAR(255) COMMENT '头像URL',
  gender TINYINT DEFAULT 0 COMMENT '性别：0-保密，1-男，2-女',
  birthday DATE COMMENT '生日',
  real_name VARCHAR(50) COMMENT '真实姓名',
  id_card VARCHAR(255) COMMENT '身份证号（加密）',
  real_name_status TINYINT DEFAULT 0 COMMENT '实名状态：0-未认证，1-认证中，2-已认证，3-认证失败',
  real_name_time DATETIME COMMENT '实名认证时间',
  status TINYINT DEFAULT 1 COMMENT '状态：0-禁用，1-启用',
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
  updated_at DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  INDEX idx_phone (phone)
) COMMENT '用户表';

-- 观演人表
CREATE TABLE t_attendee (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  user_id BIGINT NOT NULL COMMENT '用户ID',
  real_name VARCHAR(50) NOT NULL COMMENT '真实姓名',
  id_card VARCHAR(255) NOT NULL COMMENT '身份证号（加密）',
  phone VARCHAR(11) COMMENT '联系电话',
  is_default TINYINT DEFAULT 0 COMMENT '是否默认：0-否，1-是',
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
  INDEX idx_user_id (user_id)
) COMMENT '观演人表';

-- 收货地址表
CREATE TABLE t_address (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  user_id BIGINT NOT NULL,
  receiver_name VARCHAR(50) NOT NULL COMMENT '收货人姓名',
  receiver_phone VARCHAR(11) NOT NULL COMMENT '收货人电话',
  province VARCHAR(50) NOT NULL,
  city VARCHAR(50) NOT NULL,
  district VARCHAR(50) NOT NULL,
  detail_address VARCHAR(255) NOT NULL,
  is_default TINYINT DEFAULT 0,
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
  INDEX idx_user_id (user_id)
) COMMENT '收货地址表';
```

### 2. 基础配置表

```sql
-- 城市表
CREATE TABLE t_city (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  city_name VARCHAR(50) NOT NULL COMMENT '城市名称',
  city_code VARCHAR(20) NOT NULL UNIQUE COMMENT '城市编码',
  first_letter CHAR(1) NOT NULL COMMENT '首字母',
  is_hot TINYINT DEFAULT 0 COMMENT '是否热门',
  sort_order INT DEFAULT 0 COMMENT '排序权重',
  status TINYINT DEFAULT 1 COMMENT '状态：0-禁用，1-启用',
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
  updated_at DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  INDEX idx_first_letter (first_letter),
  INDEX idx_is_hot (is_hot)
) COMMENT '城市表';

-- 频道表（金刚位）
CREATE TABLE t_channel (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  channel_name VARCHAR(50) NOT NULL COMMENT '频道名称',
  channel_code VARCHAR(20) NOT NULL UNIQUE COMMENT '频道编码',
  icon_url VARCHAR(255) NOT NULL COMMENT '频道图标',
  sort_order INT DEFAULT 0 COMMENT '排序权重',
  status TINYINT DEFAULT 1 COMMENT '状态',
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
  updated_at DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
) COMMENT '频道表';

-- 场馆表
CREATE TABLE t_venue (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  venue_name VARCHAR(100) NOT NULL COMMENT '场馆名称',
  city_id BIGINT NOT NULL COMMENT '所在城市ID',
  address VARCHAR(255) NOT NULL COMMENT '详细地址',
  longitude DECIMAL(10,7) COMMENT '经度',
  latitude DECIMAL(10,7) COMMENT '纬度',
  capacity INT COMMENT '场馆容量',
  venue_image VARCHAR(255) COMMENT '场馆图片',
  status TINYINT DEFAULT 1 COMMENT '状态',
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
  INDEX idx_city_id (city_id)
) COMMENT '场馆表';
```

### 3. 项目相关表

```sql
-- 项目表
CREATE TABLE t_event (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  event_name VARCHAR(200) NOT NULL COMMENT '项目名称',
  channel_id BIGINT NOT NULL COMMENT '频道ID',
  cover_image VARCHAR(255) COMMENT '封面图',
  carousel_images TEXT COMMENT '轮播图（JSON数组）',
  description TEXT COMMENT '项目详情介绍',
  purchase_notice TEXT COMMENT '购票须知',
  rush_strategy TEXT COMMENT '抢票攻略',
  duration INT COMMENT '演出时长（分钟）',
  artist_info TEXT COMMENT '艺人/演员信息',
  show_wish_count TINYINT DEFAULT 1 COMMENT '是否展示想看人数',
  wish_count INT DEFAULT 0 COMMENT '想看人数',
  limit_per_person INT DEFAULT 6 COMMENT '限购数量',
  allow_refund TINYINT DEFAULT 1 COMMENT '是否可退票',
  refund_rules TEXT COMMENT '退票规则（JSON格式，包含时间段和手续费配置）',
  sale_start_time DATETIME COMMENT '开售时间',
  sale_end_time DATETIME COMMENT '结束时间',
  require_real_name TINYINT DEFAULT 1 COMMENT '是否实名购票',
  status TINYINT DEFAULT 0 COMMENT '状态：0-下架，1-上架',
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
  updated_at DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  INDEX idx_channel_id (channel_id),
  INDEX idx_status (status),
  INDEX idx_sale_start_time (sale_start_time)
) COMMENT '项目表';

-- 场次表
CREATE TABLE t_session (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  event_id BIGINT NOT NULL COMMENT '项目ID',
  venue_id BIGINT NOT NULL COMMENT '场馆ID',
  session_date DATE NOT NULL COMMENT '演出日期',
  start_time TIME NOT NULL COMMENT '开始时间',
  end_time TIME COMMENT '结束时间',
  status TINYINT DEFAULT 1 COMMENT '状态：0-下架，1-上架',
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
  INDEX idx_event_id (event_id),
  INDEX idx_venue_id (venue_id),
  INDEX idx_session_date (session_date)
) COMMENT '场次表';

-- 票档表
CREATE TABLE t_ticket_tier (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  session_id BIGINT NOT NULL COMMENT '场次ID',
  tier_name VARCHAR(50) NOT NULL COMMENT '票档名称',
  price DECIMAL(10,2) NOT NULL COMMENT '票档价格',
  total_stock INT NOT NULL COMMENT '总库存',
  available_stock INT NOT NULL COMMENT '可售库存',
  sold_count INT DEFAULT 0 COMMENT '已售数量',
  status TINYINT DEFAULT 1 COMMENT '状态：0-下架，1-上架，2-售罄',
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
  INDEX idx_session_id (session_id)
) COMMENT '票档表';

-- 想看列表表
CREATE TABLE t_wishlist (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  user_id BIGINT NOT NULL,
  event_id BIGINT NOT NULL,
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
  UNIQUE KEY uk_user_event (user_id, event_id)
) COMMENT '想看列表表';
```

### 4. 订单相关表

```sql
-- 订单表
CREATE TABLE t_order (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  order_no VARCHAR(32) NOT NULL UNIQUE COMMENT '订单编号',
  user_id BIGINT NOT NULL COMMENT '用户ID',
  event_id BIGINT NOT NULL COMMENT '项目ID',
  session_id BIGINT NOT NULL COMMENT '场次ID',
  ticket_tier_id BIGINT NOT NULL COMMENT '票档ID',
  ticket_count INT NOT NULL COMMENT '购票数量',
  unit_price DECIMAL(10,2) NOT NULL COMMENT '单价',
  total_amount DECIMAL(10,2) NOT NULL COMMENT '总金额',
  actual_amount DECIMAL(10,2) NOT NULL COMMENT '实付金额',
  order_status TINYINT DEFAULT 1 COMMENT '订单状态：1-待支付，2-已支付，3-已取消，4-已退款，5-已完成',
  pay_method VARCHAR(20) COMMENT '支付方式：wechat/alipay',
  pay_time DATETIME COMMENT '支付时间',
  contact_name VARCHAR(50) NOT NULL COMMENT '联系人姓名',
  contact_phone VARCHAR(11) NOT NULL COMMENT '联系电话',
  delivery_method TINYINT DEFAULT 1 COMMENT '配送方式：1-电子票，2-快递',
  delivery_address TEXT COMMENT '收货地址（JSON）',
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
  updated_at DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  INDEX idx_user_id (user_id),
  INDEX idx_order_no (order_no),
  INDEX idx_order_status (order_status)
) COMMENT '订单表';

-- 订单项表（电子票）
CREATE TABLE t_order_item (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  order_id BIGINT NOT NULL COMMENT '订单ID',
  ticket_tier_id BIGINT NOT NULL COMMENT '票档ID',
  attendee_name VARCHAR(50) NOT NULL COMMENT '观演人姓名',
  attendee_id_card VARCHAR(255) NOT NULL COMMENT '观演人身份证（加密）',
  ticket_price DECIMAL(10,2) NOT NULL COMMENT '票价',
  seat_info VARCHAR(100) COMMENT '座位信息（人工录入）',
  ticket_code VARCHAR(64) UNIQUE COMMENT '电子票码',
  qr_code_url VARCHAR(255) COMMENT '二维码URL',
  ticket_status TINYINT DEFAULT 1 COMMENT '票状态：1-有效，2-已使用，3-已退票，4-已过期',
  use_time DATETIME COMMENT '使用时间',
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
  INDEX idx_order_id (order_id),
  INDEX idx_ticket_code (ticket_code)
) COMMENT '订单项表';

-- 退票退款表
CREATE TABLE t_refund (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  refund_no VARCHAR(32) NOT NULL UNIQUE COMMENT '退款单号',
  order_id BIGINT NOT NULL COMMENT '订单ID',
  refund_amount DECIMAL(10,2) NOT NULL COMMENT '退款金额',
  handling_fee DECIMAL(10,2) DEFAULT 0.00 COMMENT '手续费',
  refund_reason VARCHAR(500) COMMENT '退款原因',
  refund_status TINYINT DEFAULT 1 COMMENT '退款状态：1-待审核，2-已通过，3-已拒绝，4-已退款',
  auditor_id BIGINT COMMENT '审核人ID',
  audit_time DATETIME COMMENT '审核时间',
  audit_remark VARCHAR(500) COMMENT '审核备注',
  refund_time DATETIME COMMENT '退款时间',
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
  INDEX idx_order_id (order_id)
) COMMENT '退票退款表';
```

### 5. 运营管理表

```sql
-- Banner表
CREATE TABLE t_banner (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  banner_name VARCHAR(100) NOT NULL COMMENT 'Banner名称',
  image_url VARCHAR(255) NOT NULL COMMENT 'Banner图片URL',
  jump_type TINYINT DEFAULT 0 COMMENT '跳转类型：0-无跳转，1-项目详情，2-外部链接',
  jump_target VARCHAR(255) COMMENT '跳转目标',
  city_ids TEXT COMMENT '展示城市ID（JSON数组）',
  sort_order INT DEFAULT 0 COMMENT '排序权重',
  start_time DATETIME NOT NULL COMMENT '开始时间',
  end_time DATETIME NOT NULL COMMENT '结束时间',
  status TINYINT DEFAULT 1 COMMENT '状态',
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP
) COMMENT 'Banner表';

-- 黑名单表
CREATE TABLE t_blacklist (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  blacklist_type TINYINT NOT NULL COMMENT '类型：1-手机号，2-身份证号',
  blacklist_value VARCHAR(255) NOT NULL COMMENT '黑名单值（加密）',
  reason VARCHAR(500) NOT NULL COMMENT '拉黑原因',
  operator_id BIGINT NOT NULL COMMENT '操作人ID',
  status TINYINT DEFAULT 1 COMMENT '状态：0-已解除，1-生效',
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
  INDEX idx_type_value (blacklist_type, blacklist_value)
) COMMENT '黑名单表';

-- 系统参数表
CREATE TABLE t_system_config (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  config_key VARCHAR(100) NOT NULL UNIQUE COMMENT '参数编码',
  config_name VARCHAR(100) NOT NULL COMMENT '参数名称',
  config_type VARCHAR(20) NOT NULL COMMENT '参数类型：text/number/boolean/enum',
  config_value TEXT COMMENT '参数值',
  description VARCHAR(500) COMMENT '参数说明',
  is_editable TINYINT DEFAULT 1 COMMENT '是否可编辑',
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP
) COMMENT '系统参数表';

-- 参数类型说明：
-- text: 文本类型，直接存储字符串
-- number: 数字类型，存储数字字符串
-- boolean: 布尔类型，存储 'true' 或 'false'
-- enum: 枚举类型，存储JSON格式：{"current":"value","options":[{"value":"v1","label":"显示名1"}]}
```

### 6. 系统管理表

```sql
-- 管理员账号表
CREATE TABLE t_admin_user (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  username VARCHAR(50) NOT NULL UNIQUE COMMENT '账号（手机号）',
  password VARCHAR(255) NOT NULL COMMENT '密码（加密）',
  real_name VARCHAR(50) NOT NULL COMMENT '真实姓名',
  status TINYINT DEFAULT 1 COMMENT '状态',
  last_login_time DATETIME COMMENT '最后登录时间',
  last_login_ip VARCHAR(50) COMMENT '最后登录IP',
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP
) COMMENT '管理员账号表';

-- 角色表
CREATE TABLE t_role (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  role_code VARCHAR(50) NOT NULL UNIQUE COMMENT '角色编码',
  role_name VARCHAR(50) NOT NULL COMMENT '角色名称',
  description VARCHAR(200) COMMENT '角色描述',
  status TINYINT DEFAULT 1 COMMENT '状态',
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP
) COMMENT '角色表';

-- 权限表
CREATE TABLE t_permission (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  permission_code VARCHAR(100) NOT NULL UNIQUE COMMENT '权限编码',
  permission_name VARCHAR(100) NOT NULL COMMENT '权限名称',
  permission_type TINYINT NOT NULL COMMENT '权限类型：1-菜单，2-按钮',
  parent_id BIGINT DEFAULT 0 COMMENT '父级权限ID',
  sort_order INT DEFAULT 0 COMMENT '排序',
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP
) COMMENT '权限表';

-- 用户角色关联表
CREATE TABLE t_admin_user_role (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  user_id BIGINT NOT NULL,
  role_id BIGINT NOT NULL,
  UNIQUE KEY uk_user_role (user_id, role_id)
) COMMENT '用户角色关联表';

-- 角色权限关联表
CREATE TABLE t_role_permission (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  role_id BIGINT NOT NULL,
  permission_id BIGINT NOT NULL,
  UNIQUE KEY uk_role_permission (role_id, permission_id)
) COMMENT '角色权限关联表';

-- 操作日志表
CREATE TABLE t_operation_log (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  operator_id BIGINT COMMENT '操作人ID',
  operator_name VARCHAR(50) COMMENT '操作人姓名',
  operation_type VARCHAR(50) NOT NULL COMMENT '操作类型',
  operation_module VARCHAR(50) NOT NULL COMMENT '操作模块',
  operation_content TEXT COMMENT '操作内容',
  operation_data JSON COMMENT '操作数据',
  operation_ip VARCHAR(50) COMMENT '操作IP',
  operation_result TINYINT DEFAULT 1 COMMENT '操作结果：0-失败，1-成功',
  failure_reason VARCHAR(500) COMMENT '失败原因',
  operation_time DATETIME DEFAULT CURRENT_TIMESTAMP,
  INDEX idx_operator_id (operator_id),
  INDEX idx_operation_time (operation_time)
) COMMENT '操作日志表';
```

---

## 核心模块设计

### 1. Banner城市筛选设计

**设计原则**: Banner支持指定城市展示或全部城市展示

**实现方案**:
- `city_ids` 为 `NULL` 或空数组：表示全部城市可见
- `city_ids` 包含城市ID：仅指定城市可见

```java
@Service
public class BannerService {
    
    public List<Banner> getBannersByCity(Long cityId) {
        List<Banner> allBanners = bannerMapper.selectActiveBanners();
        
        return allBanners.stream()
            .filter(banner -> isBannerVisibleForCity(banner, cityId))
            .collect(Collectors.toList());
    }
    
    private boolean isBannerVisibleForCity(Banner banner, Long cityId) {
        String cityIdsJson = banner.getCityIds();
        
        // 为空或null表示全部城市可见
        if (cityIdsJson == null || cityIdsJson.trim().isEmpty()) {
            return true;
        }
        
        // 解析JSON数组
        List<Long> cityIds = JSON.parseArray(cityIdsJson, Long.class);
        
        // 空数组也表示全部城市可见
        if (cityIds == null || cityIds.isEmpty()) {
            return true;
        }
        
        // 检查是否包含当前城市
        return cityIds.contains(cityId);
    }
}
```

### 2. 城市数据筛选设计

**设计原则**: 城市作为数据筛选条件，通过场馆关联城市

**实现方案**:
- 场馆表关联城市（场馆属于特定城市）
- 场次表关联场馆（通过场馆间接关联城市）
- C端查询通过场馆的 `city_id` 过滤演出

```java
// C端：用户选择城市后查询演出
@Service
public class EventService {
    public List<EventVO> getEventsByCity(Long cityId) {
        return eventMapper.selectEventsByCity(cityId);
    }
}

// SQL示例
SELECT e.*, s.*, v.*
FROM t_event e
JOIN t_session s ON e.id = s.event_id
JOIN t_venue v ON s.venue_id = v.id
WHERE v.city_id = #{cityId}
  AND e.status = 1
  AND s.session_date >= CURDATE()
ORDER BY s.session_date, s.start_time;
```

### 3. 库存防超卖设计

**核心策略**:
1. Redis原子操作扣减库存
2. 库存预扣机制
3. 订单超时自动释放库存

```lua
-- Redis Lua脚本：原子性库存扣减
local key = KEYS[1]
local quantity = tonumber(ARGV[1])
local current = tonumber(redis.call('get', key) or 0)
if current >= quantity then
    redis.call('decrby', key, quantity)
    return 1
else
    return 0
end
```

```java
@Service
public class InventoryService {
    
    public boolean lockInventory(Long ticketTierId, Integer quantity) {
        String key = "inventory:" + ticketTierId;
        Long result = redisTemplate.execute(lockInventoryScript, 
            Collections.singletonList(key), quantity.toString());
        return result != null && result == 1;
    }
    
    public void releaseInventory(Long ticketTierId, Integer quantity) {
        String key = "inventory:" + ticketTierId;
        redisTemplate.opsForValue().increment(key, quantity);
    }
}
```

### 4. 订单创建流程

```java
@Service
public class OrderService {
    
    @Transactional
    public Order createOrder(OrderCreateRequest request) {
        // 1. 黑名单检查
        if (blacklistService.isBlacklisted(request.getPhone(), request.getIdCard())) {
            throw new BusinessException("用户在黑名单中");
        }
        
        // 2. 限购检查
        int purchasedCount = orderMapper.countUserPurchased(
            request.getUserId(), request.getSessionId());
        if (purchasedCount + request.getQuantity() > event.getLimitPerPerson()) {
            throw new BusinessException("超过限购数量");
        }
        
        // 3. 锁定库存
        if (!inventoryService.lockInventory(request.getTicketTierId(), request.getQuantity())) {
            throw new BusinessException("库存不足");
        }
        
        try {
            // 4. 创建订单
            Order order = buildOrder(request);
            orderMapper.insert(order);
            
            // 5. 创建订单项（电子票）
            createOrderItems(order, request);
            
            // 6. 设置订单超时（15分钟）
            scheduleOrderTimeout(order.getId(), 15);
            
            return order;
        } catch (Exception e) {
            inventoryService.releaseInventory(request.getTicketTierId(), request.getQuantity());
            throw e;
        }
    }
}
```

### 5. 退票规则设计

**阶梯式退票规则**：支持根据退票时间距离演出时间的不同，收取不同比例的手续费。

**refund_rules 数据格式**：
```json
[
  {
    "hours_before": 168,
    "fee_rate": 0.00,
    "description": "演出前7天以上，免手续费"
  },
  {
    "hours_before": 72,
    "fee_rate": 0.10,
    "description": "演出前3-7天，收取10%手续费"
  },
  {
    "hours_before": 24,
    "fee_rate": 0.20,
    "description": "演出前1-3天，收取20%手续费"
  },
  {
    "hours_before": 0,
    "fee_rate": 1.00,
    "description": "演出前24小时内，不可退票"
  }
]
```

**退票手续费计算逻辑**：
```java
@Service
public class RefundService {
    
    public RefundCalculateResult calculateRefund(Order order, LocalDateTime refundTime) {
        Event event = eventMapper.selectById(order.getEventId());
        Session session = sessionMapper.selectById(order.getSessionId());
        
        // 计算距离演出的小时数
        LocalDateTime showTime = LocalDateTime.of(session.getSessionDate(), session.getStartTime());
        long hoursBeforeShow = Duration.between(refundTime, showTime).toHours();
        
        // 解析退票规则
        List<RefundRule> rules = JSON.parseArray(event.getRefundRules(), RefundRule.class);
        rules.sort(Comparator.comparing(RefundRule::getHoursBefore).reversed());
        
        // 匹配退票规则
        BigDecimal feeRate = BigDecimal.ZERO;
        for (RefundRule rule : rules) {
            if (hoursBeforeShow >= rule.getHoursBefore()) {
                feeRate = rule.getFeeRate();
                break;
            }
        }
        
        // 手续费率为1.00表示不可退票
        if (feeRate.compareTo(BigDecimal.ONE) >= 0) {
            throw new BusinessException("当前时间不支持退票");
        }
        
        // 计算退款金额
        BigDecimal handlingFee = order.getActualAmount()
            .multiply(feeRate)
            .setScale(2, RoundingMode.HALF_UP);
        BigDecimal refundAmount = order.getActualAmount().subtract(handlingFee);
        
        return new RefundCalculateResult(refundAmount, handlingFee, feeRate);
    }
}
```

### 6. 黑名单检测设计

```java
@Service
public class BlacklistService {
    
    public boolean isBlacklisted(String phone, String idCard) {
        // 检查手机号黑名单
        if (phone != null) {
            String encryptedPhone = encryptService.encrypt(phone);
            if (blacklistMapper.existsByTypeAndValue(1, encryptedPhone)) {
                return true;
            }
        }
        
        // 检查身份证号黑名单
        if (idCard != null) {
            String encryptedIdCard = encryptService.encrypt(idCard);
            if (blacklistMapper.existsByTypeAndValue(2, encryptedIdCard)) {
                return true;
            }
        }
        
        return false;
    }
}
```

### 7. 座位信息管理设计

**设计说明**：
- 系统不提供自动选座功能
- 座位信息由B端人工录入（可选）
- 座位信息格式自由，如：`1排5座`、`A区10排8号`、`VIP-A12`
- C端展示时，如果有座位信息则显示，没有则不显示

**使用场景**：
1. B端后台售票时，可以手动输入座位信息
2. B端订单管理中，可以批量导入座位信息
3. C端订单详情和电子票中展示座位信息

```java
@Service
public class SeatService {
    
    /**
     * 批量更新订单项的座位信息
     */
    @Transactional
    public void batchUpdateSeatInfo(List<SeatUpdateRequest> requests) {
        for (SeatUpdateRequest request : requests) {
            OrderItem item = orderItemMapper.selectById(request.getOrderItemId());
            if (item == null) {
                throw new BusinessException("订单项不存在");
            }
            
            item.setSeatInfo(request.getSeatInfo());
            orderItemMapper.updateById(item);
        }
    }
    
    /**
     * 从Excel导入座位信息
     */
    public void importSeatInfoFromExcel(MultipartFile file) {
        List<SeatImportDTO> seatList = ExcelUtil.read(file, SeatImportDTO.class);
        
        List<SeatUpdateRequest> requests = seatList.stream()
            .map(dto -> new SeatUpdateRequest(dto.getOrderItemId(), dto.getSeatInfo()))
            .collect(Collectors.toList());
        
        batchUpdateSeatInfo(requests);
    }
}
```

### 8. 系统参数管理设计

**参数类型及数据格式**：

| 类型 | config_value格式 | 示例 |
|------|-----------------|------|
| text | 纯文本字符串 | `"票务系统"` |
| number | 数字字符串 | `"15"` |
| boolean | true/false字符串 | `"true"` |
| enum | JSON对象 | `{"current":"wechat","options":[...]}` |

**枚举类型JSON格式**：
```json
{
  "current": "wechat",
  "options": [
    {"value": "wechat", "label": "微信支付"},
    {"value": "alipay", "label": "支付宝"},
    {"value": "unionpay", "label": "银联支付"}
  ]
}
```

**系统参数读取服务**：
```java
@Service
public class SystemConfigService {
    
    /**
     * 获取文本类型参数
     */
    public String getTextConfig(String key, String defaultValue) {
        SystemConfig config = configMapper.selectByKey(key);
        return config != null ? config.getConfigValue() : defaultValue;
    }
    
    /**
     * 获取数字类型参数
     */
    public Integer getNumberConfig(String key, Integer defaultValue) {
        SystemConfig config = configMapper.selectByKey(key);
        if (config == null) return defaultValue;
        try {
            return Integer.parseInt(config.getConfigValue());
        } catch (NumberFormatException e) {
            return defaultValue;
        }
    }
    
    /**
     * 获取布尔类型参数
     */
    public Boolean getBooleanConfig(String key, Boolean defaultValue) {
        SystemConfig config = configMapper.selectByKey(key);
        return config != null ? Boolean.parseBoolean(config.getConfigValue()) : defaultValue;
    }
    
    /**
     * 获取枚举类型参数
     */
    public EnumConfigVO getEnumConfig(String key) {
        SystemConfig config = configMapper.selectByKey(key);
        if (config == null) return null;
        return JSON.parseObject(config.getConfigValue(), EnumConfigVO.class);
    }
    
    /**
     * 获取枚举当前值
     */
    public String getEnumCurrentValue(String key, String defaultValue) {
        EnumConfigVO enumConfig = getEnumConfig(key);
        return enumConfig != null ? enumConfig.getCurrent() : defaultValue;
    }
    
    /**
     * 更新枚举当前值
     */
    @Transactional
    public void updateEnumCurrentValue(String key, String newValue) {
        SystemConfig config = configMapper.selectByKey(key);
        if (config == null) {
            throw new BusinessException("参数不存在");
        }
        
        EnumConfigVO enumConfig = JSON.parseObject(config.getConfigValue(), EnumConfigVO.class);
        
        // 验证新值是否在选项中
        boolean isValid = enumConfig.getOptions().stream()
            .anyMatch(option -> option.getValue().equals(newValue));
        
        if (!isValid) {
            throw new BusinessException("无效的枚举值");
        }
        
        enumConfig.setCurrent(newValue);
        config.setConfigValue(JSON.toJSONString(enumConfig));
        configMapper.updateById(config);
    }
}

@Data
class EnumConfigVO {
    private String current;
    private List<EnumOption> options;
}

@Data
class EnumOption {
    private String value;
    private String label;
}
```

**使用示例**：
```java
// 获取订单超时时间
Integer timeout = systemConfigService.getNumberConfig("order_timeout_minutes", 15);

// 获取是否开启实名认证
Boolean enableRealName = systemConfigService.getBooleanConfig("enable_real_name", true);

// 获取默认支付方式
String paymentMethod = systemConfigService.getEnumCurrentValue("default_payment_method", "wechat");

// 获取完整枚举配置（用于前端下拉框）
EnumConfigVO paymentOptions = systemConfigService.getEnumConfig("default_payment_method");
```

### 9. 电子票生成设计

```java
@Service
public class TicketService {
    
    public String generateTicketCode(Long orderId, Long itemId) {
        // 格式：T + 时间戳 + 订单ID后4位 + 随机数
        String timestamp = String.valueOf(System.currentTimeMillis());
        String orderSuffix = String.format("%04d", orderId % 10000);
        String random = String.format("%04d", new Random().nextInt(10000));
        return "T" + timestamp + orderSuffix + random;
    }
    
    public String generateQRCode(String ticketCode) {
        // 生成二维码并上传到OSS
        BufferedImage qrImage = QRCodeUtil.generate(ticketCode, 300, 300);
        String fileName = "qrcode/" + ticketCode + ".png";
        return ossService.upload(qrImage, fileName);
    }
    
    /**
     * 生成电子票信息（包含座位信息）
     */
    public TicketVO generateTicketVO(OrderItem item) {
        TicketVO ticket = new TicketVO();
        ticket.setTicketCode(item.getTicketCode());
        ticket.setQrCodeUrl(item.getQrCodeUrl());
        ticket.setAttendeeName(item.getAttendeeName());
        ticket.setSeatInfo(item.getSeatInfo());  // 座位信息（可能为空）
        // ... 其他字段
        return ticket;
    }
}
```

---

## API接口设计

### C端接口

| 模块 | 接口 | 方法 | 说明 |
|------|------|------|------|
| 城市 | /api/c/cities | GET | 获取城市列表（按首字母分组） |
| 城市 | /api/c/cities/hot | GET | 获取热门城市 |
| 首页 | /api/c/home | GET | 获取首页数据（Banner、频道、项目列表） |
| 搜索 | /api/c/events/search | GET | 搜索项目 |
| 频道 | /api/c/events/channel | GET | 频道项目列表 |
| 项目 | /api/c/events/{id} | GET | 项目详情 |
| 项目 | /api/c/events/{id}/sessions | GET | 场次列表 |
| 想看 | /api/c/wishlist | POST | 添加/取消想看 |
| 订单 | /api/c/orders | POST | 创建订单 |
| 订单 | /api/c/orders | GET | 订单列表 |
| 订单 | /api/c/orders/{id} | GET | 订单详情 |
| 订单 | /api/c/orders/{id}/cancel | POST | 取消订单 |
| 订单 | /api/c/orders/{id}/refund | POST | 申请退票 |
| 支付 | /api/c/orders/{id}/pay | POST | 订单支付 |
| 用户 | /api/c/auth/login | POST | 登录 |
| 用户 | /api/c/auth/register | POST | 注册 |
| 用户 | /api/c/auth/sms | POST | 发送验证码 |
| 用户 | /api/c/user/profile | GET/PUT | 用户信息 |
| 用户 | /api/c/user/attendees | GET/POST/PUT/DELETE | 观演人管理 |
| 用户 | /api/c/user/addresses | GET/POST/PUT/DELETE | 收货地址管理 |
| 票夹 | /api/c/tickets | GET | 票夹列表 |

### B端接口

| 模块 | 接口 | 方法 | 说明 |
|------|------|------|------|
| 登录 | /api/b/auth/login | POST | B端登录 |
| 看板 | /api/b/dashboard | GET | 首页看板数据 |
| 项目 | /api/b/events | GET/POST | 项目列表/创建 |
| 项目 | /api/b/events/{id} | GET/PUT/DELETE | 项目详情/编辑/删除 |
| 场次 | /api/b/sessions | GET/POST | 场次列表/创建 |
| 票档 | /api/b/ticket-tiers | GET/POST | 票档列表/创建 |
| 场馆 | /api/b/venues | GET/POST/PUT/DELETE | 场馆管理 |
| 订单 | /api/b/orders | GET | 订单列表 |
| 订单 | /api/b/orders/{id} | GET | 订单详情 |
| 订单 | /api/b/orders/sell | POST | 后台售票 |
| 退款 | /api/b/refunds | GET | 退款列表 |
| 退款 | /api/b/refunds/{id}/audit | POST | 退款审核 |
| 城市 | /api/b/cities | GET/POST/PUT/DELETE | 城市管理 |
| 频道 | /api/b/channels | GET/POST/PUT/DELETE | 频道管理 |
| Banner | /api/b/banners | GET/POST/PUT/DELETE | Banner管理 |
| 黑名单 | /api/b/blacklist | GET/POST/DELETE | 黑名单管理 |
| 账号 | /api/b/admin-users | GET/POST/PUT/DELETE | 账号管理 |
| 角色 | /api/b/roles | GET/POST/PUT/DELETE | 角色管理 |
| 权限 | /api/b/permissions | GET | 权限列表 |
| 日志 | /api/b/logs | GET | 操作日志 |

---

## 正确性属性 (Correctness Properties)

*属性是系统在所有有效执行中应该保持为真的特征或行为——本质上是关于系统应该做什么的形式化陈述。属性作为人类可读规范和机器可验证正确性保证之间的桥梁。*

### 属性 1: 城市数据筛选一致性

**对于任意**城市ID和数据查询请求，返回的所有项目、场次数据应该只包含与该城市关联的记录。

**验证: 需求 1.4, 1.5, 2.10, 3.4**

### 属性 2: 城市首字母分组正确性

**对于任意**城市列表，按首字母分组后，每个城市应该被正确归类到其名称首字母对应的分组中。

**验证: 需求 1.3**

### 属性 3: 城市偏好存储往返一致性

**对于任意**城市选择，保存到本地存储后再读取，应该得到相同的城市信息。

**验证: 需求 1.6**

### 属性 4: 列表排序正确性

**对于任意**项目列表查询：
- 按时间排序时，结果应按演出时间升序排列
- 按热度排序时，结果应按销量/热度值降序排列
- 按价格排序时，结果应按最低价格升序或降序排列

**验证: 需求 2.8, 4.6**

### 属性 5: 频道排序权重一致性

**对于任意**频道列表，返回的频道应该按排序权重降序排列（数字越大越靠前）。

**验证: 需求 2.6, 19.5**

### 属性 6: 搜索结果完整性

**对于任意**搜索结果项目，应该包含所有必需字段：图片、名称、日期、周几、时间、城市、剧院、价格。

**验证: 需求 3.2, 4.7**

### 属性 7: 筛选条件有效性

**对于任意**带筛选条件的查询：
- 日期筛选后，所有结果的演出日期应在指定范围内
- 价格筛选后，所有结果的价格应在指定范围内

**验证: 需求 4.5**

### 属性 8: 想看列表操作一致性

**对于任意**用户和项目，添加到想看列表后，查询想看列表应包含该项目；取消想看后，查询想看列表应不包含该项目。

**验证: 需求 5.11, 11.10**

### 属性 9: 票档库存状态一致性

**对于任意**票档，当可售库存为0时，状态应显示为"售罄"；当可售库存大于0时，状态应显示为"可售"。

**验证: 需求 6.2, 6.3**

### 属性 10: 订单状态按钮显示一致性

**对于任意**订单：
- 未支付状态时，应显示付款按钮和取消订单按钮
- 已支付状态时，应根据退票规则显示或隐藏退票按钮

**验证: 需求 9.11, 11.3, 11.4**

### 属性 11: 黑名单拦截一致性

**对于任意**购票请求，如果用户的手机号或身份证号在黑名单中，系统应拒绝该购票请求。

**验证: 需求 12.2, 12.3, 19.8, 19.9, 19.10**

### 属性 12: 库存扣减原子性

**对于任意**并发购票请求，库存扣减后的可售库存应该等于原库存减去成功购票数量，且可售库存永远不能为负数。

**验证: 需求 15.9**

### 属性 13: 电子票唯一性

**对于任意**生成的电子票，其票码应该是全局唯一的，不存在重复的票码。

**验证: 需求 16.5**

### 属性 14: 退款金额计算正确性

**对于任意**退票请求和退票规则配置，系统应根据退票时间距离演出时间的小时数，匹配对应的手续费率，退款金额应等于原订单金额减去手续费（手续费 = 原订单金额 × 匹配的手续费率）。

**验证: 需求 17.3**

### 属性 15: Banner城市筛选一致性

**对于任意**Banner查询和城市ID：
- 如果Banner的city_ids为空或null，则该Banner应对所有城市可见
- 如果Banner的city_ids包含指定城市ID，则该Banner应对该城市可见
- 如果Banner的city_ids不包含指定城市ID，则该Banner不应对该城市可见

**验证: 需求 19.7**

### 属性 16: Banner有效期一致性

**对于任意**Banner查询，返回的Banner应该满足：当前时间在开始时间和结束时间之间，且状态为启用。

**验证: 需求 19.7**

### 属性 17: 权限继承一致性

**对于任意**用户，其最终权限应等于其所有角色权限的并集。

**验证: 需求 20.5**

### 属性 18: 操作日志完整性

**对于任意**关键业务操作（登录、项目管理、订单操作、权限变更），系统应生成对应的操作日志记录。

**验证: 需求 21.1**

### 属性 19: 实体CRUD一致性

**对于任意**实体（城市、频道、场馆、项目、场次、票档）：
- 创建后，通过ID查询应能获取到该实体
- 更新后，查询应返回更新后的数据
- 删除后，查询应返回空或抛出不存在异常

**验证: 需求 15.6, 15.7, 18.1, 19.1, 19.4**

---

## 错误处理设计

### 统一异常处理

```java
@RestControllerAdvice
public class GlobalExceptionHandler {
    
    @ExceptionHandler(BusinessException.class)
    public Result<Void> handleBusinessException(BusinessException e) {
        return Result.fail(e.getCode(), e.getMessage());
    }
    
    @ExceptionHandler(ValidationException.class)
    public Result<Void> handleValidationException(ValidationException e) {
        return Result.fail(400, e.getMessage());
    }
    
    @ExceptionHandler(AuthenticationException.class)
    public Result<Void> handleAuthException(AuthenticationException e) {
        return Result.fail(401, "未授权访问");
    }
    
    @ExceptionHandler(Exception.class)
    public Result<Void> handleException(Exception e) {
        log.error("系统异常", e);
        return Result.fail(500, "系统繁忙，请稍后重试");
    }
}
```

### 业务异常码定义

| 异常码 | 说明 |
|--------|------|
| 1001 | 用户不存在 |
| 1002 | 密码错误 |
| 1003 | 验证码错误 |
| 1004 | 用户已被禁用 |
| 1005 | 用户在黑名单中 |
| 2001 | 项目不存在 |
| 2002 | 场次不存在 |
| 2003 | 票档不存在 |
| 2004 | 库存不足 |
| 2005 | 已超过限购数量 |
| 3001 | 订单不存在 |
| 3002 | 订单状态异常 |
| 3003 | 订单已过期 |
| 3004 | 不满足退票条件 |
| 4001 | 支付失败 |
| 4002 | 退款失败 |

---

## 测试策略

### 测试框架选型

| 测试类型 | 框架 | 说明 |
|----------|------|------|
| 单元测试 | JUnit 5 + Mockito | 后端单元测试 |
| 属性测试 | jqwik | Java属性测试框架 |
| 集成测试 | Spring Boot Test | 后端集成测试 |
| API测试 | RestAssured | REST API测试 |
| 前端单元测试 | Vitest | Vue组件测试 |

### 属性测试示例

```java
import net.jqwik.api.*;

class InventoryServicePropertyTest {
    
    /**
     * Feature: ticketing-system, Property 12: 库存扣减原子性
     * Validates: Requirements 15.9
     */
    @Property(tries = 100)
    void inventoryDeduction_shouldNeverBeNegative(
            @ForAll @IntRange(min = 1, max = 1000) int initialStock,
            @ForAll @Size(min = 1, max = 10) List<@IntRange(min = 1, max = 100) Integer> deductions) {
        
        TicketTier tier = new TicketTier();
        tier.setAvailableStock(initialStock);
        
        int totalDeducted = 0;
        for (Integer quantity : deductions) {
            if (inventoryService.tryDeduct(tier, quantity)) {
                totalDeducted += quantity;
            }
        }
        
        assertThat(tier.getAvailableStock()).isGreaterThanOrEqualTo(0);
        assertThat(tier.getAvailableStock()).isEqualTo(initialStock - totalDeducted);
    }
    
    /**
     * Feature: ticketing-system, Property 9: 票档库存状态一致性
     * Validates: Requirements 6.2, 6.3
     */
    @Property(tries = 100)
    void ticketTierStatus_shouldReflectStock(
            @ForAll @IntRange(min = 0, max = 1000) int stock) {
        
        TicketTier tier = new TicketTier();
        tier.setAvailableStock(stock);
        
        String status = ticketTierService.calculateStatus(tier);
        
        if (stock == 0) {
            assertThat(status).isEqualTo("sold_out");
        } else {
            assertThat(status).isEqualTo("on_sale");
        }
    }
}

class RefundServicePropertyTest {
    
    /**
     * Feature: ticketing-system, Property 14: 退款金额计算正确性
     * Validates: Requirements 17.3
     */
    @Property(tries = 100)
    void refundAmount_shouldBeCorrectlyCalculated(
            @ForAll @BigDecimalRange(min = "0.01", max = "10000.00") BigDecimal orderAmount,
            @ForAll @LongRange(min = 0, max = 720) long hoursBeforeShow) {
        
        // 构造退票规则
        List<RefundRule> rules = Arrays.asList(
            new RefundRule(168, new BigDecimal("0.00"), "演出前7天以上，免手续费"),
            new RefundRule(72, new BigDecimal("0.10"), "演出前3-7天，收取10%手续费"),
            new RefundRule(24, new BigDecimal("0.20"), "演出前1-3天，收取20%手续费"),
            new RefundRule(0, new BigDecimal("1.00"), "演出前24小时内，不可退票")
        );
        
        // 匹配手续费率
        BigDecimal feeRate = BigDecimal.ZERO;
        for (RefundRule rule : rules) {
            if (hoursBeforeShow >= rule.getHoursBefore()) {
                feeRate = rule.getFeeRate();
                break;
            }
        }
        
        // 如果手续费率为1.00，应该抛出异常
        if (feeRate.compareTo(BigDecimal.ONE) >= 0) {
            assertThatThrownBy(() -> refundService.calculateRefund(orderAmount, hoursBeforeShow, rules))
                .isInstanceOf(BusinessException.class)
                .hasMessageContaining("不支持退票");
        } else {
            RefundCalculateResult result = refundService.calculateRefund(orderAmount, hoursBeforeShow, rules);
            
            BigDecimal expectedHandlingFee = orderAmount.multiply(feeRate).setScale(2, RoundingMode.HALF_UP);
            BigDecimal expectedRefund = orderAmount.subtract(expectedHandlingFee);
            
            assertThat(result.getRefundAmount()).isEqualByComparingTo(expectedRefund);
            assertThat(result.getHandlingFee()).isEqualByComparingTo(expectedHandlingFee);
            assertThat(result.getRefundAmount()).isGreaterThanOrEqualTo(BigDecimal.ZERO);
        }
    }
}

class BlacklistServicePropertyTest {
    
    /**
     * Feature: ticketing-system, Property 11: 黑名单拦截一致性
     * Validates: Requirements 12.2, 12.3
     */
    @Property(tries = 100)
    void blacklistCheck_shouldBlockBlacklistedUsers(
            @ForAll("phoneGenerator") String phone,
            @ForAll boolean isBlacklisted) {
        
        if (isBlacklisted) {
            blacklistService.addToBlacklist(phone, "phone");
        }
        
        boolean canPurchase = blacklistService.canPurchase(phone, null);
        
        assertThat(canPurchase).isEqualTo(!isBlacklisted);
    }
}

class TicketCodeServicePropertyTest {
    
    /**
     * Feature: ticketing-system, Property 13: 电子票唯一性
     * Validates: Requirements 16.5
     */
    @Property(tries = 100)
    void ticketCode_shouldBeUnique(
            @ForAll @Size(min = 10, max = 100) List<@LongRange(min = 1, max = 10000) Long> orderIds) {
        
        Set<String> ticketCodes = new HashSet<>();
        for (Long orderId : orderIds) {
            String code = ticketCodeService.generateTicketCode(orderId);
            ticketCodes.add(code);
        }
        
        assertThat(ticketCodes.size()).isEqualTo(orderIds.size());
    }
}

class BannerServicePropertyTest {
    
    /**
     * Feature: ticketing-system, Property 15: Banner城市筛选一致性
     * Validates: Requirements 19.7
     */
    @Property(tries = 100)
    void bannerCityFilter_shouldWorkCorrectly(
            @ForAll @LongRange(min = 1, max = 100) Long cityId,
            @ForAll("bannerGenerator") Banner banner) {
        
        boolean isVisible = bannerService.isBannerVisibleForCity(banner, cityId);
        
        if (banner.getCityIds() == null || banner.getCityIds().isEmpty()) {
            // 空值表示全部城市可见
            assertThat(isVisible).isTrue();
        } else {
            List<Long> cityIds = JSON.parseArray(banner.getCityIds(), Long.class);
            if (cityIds == null || cityIds.isEmpty()) {
                // 空数组也表示全部城市可见
                assertThat(isVisible).isTrue();
            } else {
                // 检查是否包含当前城市
                assertThat(isVisible).isEqualTo(cityIds.contains(cityId));
            }
        }
    }
    
    @Provide
    Arbitrary<Banner> bannerGenerator() {
        return Combinators.combine(
            Arbitraries.strings(),
            Arbitraries.oneOf(
                Arbitraries.just((String) null),  // null表示全部城市
                Arbitraries.just("[]"),            // 空数组表示全部城市
                Arbitraries.of("[1,2,3]", "[5,10]", "[1]")  // 指定城市
            )
        ).as((name, cityIds) -> {
            Banner banner = new Banner();
            banner.setBannerName(name);
            banner.setCityIds(cityIds);
            return banner;
        });
    }
}
```

### 测试覆盖率要求

| 模块 | 单元测试覆盖率 | 属性测试覆盖 |
|------|---------------|-------------|
| 核心业务逻辑 | ≥ 80% | 所有正确性属性 |
| 工具类 | ≥ 90% | N/A |
| Controller | ≥ 70% | N/A |
| 数据访问层 | ≥ 60% | N/A |

---

**文档版本**: v3.0  
**编制日期**: 2025-11-27  
**编制人**: Kiro AI Assistant  
**审核状态**: 待审核
