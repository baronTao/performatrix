# 演出票务系统数据库设计文档

## 数据库设计说明

### 设计原则

1. 所有表都包含标准字段：id、create_by、create_time、update_by、update_time、remark
2. 主键统一使用 BIGINT 类型的雪花算法ID
3. 时间字段统一使用 DATETIME 类型
4. 金额字段统一使用 DECIMAL(10,2) 类型
5. 状态字段统一使用 VARCHAR 类型存储枚举值
6. 所有字段都有中文注释

### 数据库信息

- **数据库名称**: ticketing_system
- **字符集**: utf8mb4
- **排序规则**: utf8mb4_unicode_ci
- **存储引擎**: InnoDB

---

## 一、用户相关表

### 1.1 用户表 (t_user)

用户基础信息表，存储C端用户和B端商家的账号信息。

```sql
CREATE TABLE `t_user` (
  `id` BIGINT NOT NULL COMMENT '用户ID（主键，雪花算法）',
  `phone` VARCHAR(20) NOT NULL COMMENT '手机号',
  `password` VARCHAR(255) NOT NULL COMMENT '密码（BCrypt加密）',
  `nickname` VARCHAR(50) DEFAULT NULL COMMENT '昵称',
  `avatar` VARCHAR(500) DEFAULT NULL COMMENT '头像URL',
  `real_name` VARCHAR(50) DEFAULT NULL COMMENT '真实姓名',
  `id_card` VARCHAR(255) DEFAULT NULL COMMENT '身份证号（AES加密）',
  `email` VARCHAR(100) DEFAULT NULL COMMENT '邮箱',
  `gender` TINYINT DEFAULT NULL COMMENT '性别（0-未知，1-男，2-女）',
  `birthday` DATE DEFAULT NULL COMMENT '生日',
  `city_id` BIGINT DEFAULT NULL COMMENT '所在城市ID',
  `user_type` VARCHAR(20) NOT NULL DEFAULT 'customer' COMMENT '用户类型（customer-普通用户，merchant-商家，operator-运营，agent-总代）',
  `status` VARCHAR(20) NOT NULL DEFAULT 'active' COMMENT '状态（active-正常，frozen-冻结，deleted-已删除）',
  `last_login_time` DATETIME DEFAULT NULL COMMENT '最后登录时间',
  `last_login_ip` VARCHAR(50) DEFAULT NULL COMMENT '最后登录IP',
  `register_source` VARCHAR(50) DEFAULT NULL COMMENT '注册来源（h5、app、wechat、alipay）',
  `create_by` BIGINT DEFAULT NULL COMMENT '创建人ID',
  `create_time` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `update_by` BIGINT DEFAULT NULL COMMENT '更新人ID',
  `update_time` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  `remark` VARCHAR(500) DEFAULT NULL COMMENT '备注',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_phone` (`phone`),
  KEY `idx_city_id` (`city_id`),
  KEY `idx_status` (`status`),
  KEY `idx_user_type` (`user_type`),
  KEY `idx_create_time` (`create_time`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='用户表';
```

### 1.2 用户第三方登录表 (t_user_oauth)

存储用户第三方登录信息（微信、支付宝等）。

```sql
CREATE TABLE `t_user_oauth` (
  `id` BIGINT NOT NULL COMMENT '主键ID',
  `user_id` BIGINT NOT NULL COMMENT '用户ID',
  `oauth_type` VARCHAR(20) NOT NULL COMMENT '第三方类型（wechat-微信，alipay-支付宝）',
  `oauth_id` VARCHAR(100) NOT NULL COMMENT '第三方用户唯一标识（openid、userid等）',
  `union_id` VARCHAR(100) DEFAULT NULL COMMENT '第三方平台统一ID（微信unionid）',
  `oauth_name` VARCHAR(100) DEFAULT NULL COMMENT '第三方用户昵称',
  `oauth_avatar` VARCHAR(500) DEFAULT NULL COMMENT '第三方用户头像',
  `access_token` VARCHAR(500) DEFAULT NULL COMMENT '访问令牌',
  `refresh_token` VARCHAR(500) DEFAULT NULL COMMENT '刷新令牌',
  `expires_in` INT DEFAULT NULL COMMENT '令牌过期时间（秒）',
  `create_by` BIGINT DEFAULT NULL COMMENT '创建人ID',
  `create_time` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `update_by` BIGINT DEFAULT NULL COMMENT '更新人ID',
  `update_time` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  `remark` VARCHAR(500) DEFAULT NULL COMMENT '备注',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_oauth_type_id` (`oauth_type`, `oauth_id`),
  KEY `idx_user_id` (`user_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='用户第三方登录表';
```

### 1.3 实名信息表 (t_real_name_info)

存储用户实名认证信息。

```sql
CREATE TABLE `t_real_name_info` (
  `id` BIGINT NOT NULL COMMENT '主键ID',
  `user_id` BIGINT NOT NULL COMMENT '用户ID',
  `real_name` VARCHAR(50) NOT NULL COMMENT '真实姓名',
  `id_card` VARCHAR(255) NOT NULL COMMENT '身份证号（AES加密）',
  `id_card_front` VARCHAR(500) DEFAULT NULL COMMENT '身份证正面照片URL',
  `id_card_back` VARCHAR(500) DEFAULT NULL COMMENT '身份证反面照片URL',
  `is_default` TINYINT NOT NULL DEFAULT 0 COMMENT '是否默认（0-否，1-是）',
  `verify_status` VARCHAR(20) NOT NULL DEFAULT 'pending' COMMENT '认证状态（pending-待认证，verified-已认证，rejected-已拒绝）',
  `verify_time` DATETIME DEFAULT NULL COMMENT '认证时间',
  `create_by` BIGINT DEFAULT NULL COMMENT '创建人ID',
  `create_time` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `update_by` BIGINT DEFAULT NULL COMMENT '更新人ID',
  `update_time` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  `remark` VARCHAR(500) DEFAULT NULL COMMENT '备注',
  PRIMARY KEY (`id`),
  KEY `idx_user_id` (`user_id`),
  KEY `idx_verify_status` (`verify_status`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='实名信息表';
```

### 1.4 收货地址表 (t_address)

存储用户收货地址信息。

```sql
CREATE TABLE `t_address` (
  `id` BIGINT NOT NULL COMMENT '主键ID',
  `user_id` BIGINT NOT NULL COMMENT '用户ID',
  `receiver_name` VARCHAR(50) NOT NULL COMMENT '收货人姓名',
  `receiver_phone` VARCHAR(20) NOT NULL COMMENT '收货人手机号',
  `province` VARCHAR(50) NOT NULL COMMENT '省份',
  `province_code` VARCHAR(20) DEFAULT NULL COMMENT '省份编码',
  `city` VARCHAR(50) NOT NULL COMMENT '城市',
  `city_code` VARCHAR(20) DEFAULT NULL COMMENT '城市编码',
  `district` VARCHAR(50) NOT NULL COMMENT '区县',
  `district_code` VARCHAR(20) DEFAULT NULL COMMENT '区县编码',
  `detail_address` VARCHAR(200) NOT NULL COMMENT '详细地址',
  `postal_code` VARCHAR(10) DEFAULT NULL COMMENT '邮政编码',
  `is_default` TINYINT NOT NULL DEFAULT 0 COMMENT '是否默认（0-否，1-是）',
  `create_by` BIGINT DEFAULT NULL COMMENT '创建人ID',
  `create_time` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `update_by` BIGINT DEFAULT NULL COMMENT '更新人ID',
  `update_time` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  `remark` VARCHAR(500) DEFAULT NULL COMMENT '备注',
  PRIMARY KEY (`id`),
  KEY `idx_user_id` (`user_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='收货地址表';
```

### 1.5 用户标签表 (t_user_tag)

存储用户标签信息（用于用户分类和营销）。

```sql
CREATE TABLE `t_user_tag` (
  `id` BIGINT NOT NULL COMMENT '主键ID',
  `tag_name` VARCHAR(50) NOT NULL COMMENT '标签名称',
  `tag_type` VARCHAR(20) NOT NULL COMMENT '标签类型（system-系统标签，custom-自定义标签）',
  `tag_color` VARCHAR(20) DEFAULT NULL COMMENT '标签颜色',
  `sort_order` INT NOT NULL DEFAULT 0 COMMENT '排序号',
  `status` VARCHAR(20) NOT NULL DEFAULT 'active' COMMENT '状态（active-启用，inactive-禁用）',
  `create_by` BIGINT DEFAULT NULL COMMENT '创建人ID',
  `create_time` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `update_by` BIGINT DEFAULT NULL COMMENT '更新人ID',
  `update_time` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  `remark` VARCHAR(500) DEFAULT NULL COMMENT '备注',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_tag_name` (`tag_name`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='用户标签表';
```

### 1.6 用户标签关联表 (t_user_tag_relation)

用户与标签的多对多关联表。

```sql
CREATE TABLE `t_user_tag_relation` (
  `id` BIGINT NOT NULL COMMENT '主键ID',
  `user_id` BIGINT NOT NULL COMMENT '用户ID',
  `tag_id` BIGINT NOT NULL COMMENT '标签ID',
  `create_by` BIGINT DEFAULT NULL COMMENT '创建人ID',
  `create_time` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `update_by` BIGINT DEFAULT NULL COMMENT '更新人ID',
  `update_time` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  `remark` VARCHAR(500) DEFAULT NULL COMMENT '备注',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_user_tag` (`user_id`, `tag_id`),
  KEY `idx_tag_id` (`tag_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='用户标签关联表';
```

---

## 二、商家相关表

### 2.1 商家信息表 (t_merchant)

存储B端商家的详细信息。

```sql
CREATE TABLE `t_merchant` (
  `id` BIGINT NOT NULL COMMENT '主键ID',
  `user_id` BIGINT NOT NULL COMMENT '关联用户ID',
  `merchant_name` VARCHAR(100) NOT NULL COMMENT '商家名称',
  `merchant_code` VARCHAR(50) NOT NULL COMMENT '商家编码',
  `legal_person` VARCHAR(50) NOT NULL COMMENT '法人姓名',
  `legal_id_card` VARCHAR(255) NOT NULL COMMENT '法人身份证号（AES加密）',
  `business_license` VARCHAR(100) NOT NULL COMMENT '营业执照号',
  `business_license_img` VARCHAR(500) DEFAULT NULL COMMENT '营业执照图片URL',
  `contact_name` VARCHAR(50) NOT NULL COMMENT '联系人姓名',
  `contact_phone` VARCHAR(20) NOT NULL COMMENT '联系人电话',
  `contact_email` VARCHAR(100) DEFAULT NULL COMMENT '联系人邮箱',
  `province` VARCHAR(50) DEFAULT NULL COMMENT '省份',
  `city` VARCHAR(50) DEFAULT NULL COMMENT '城市',
  `address` VARCHAR(200) DEFAULT NULL COMMENT '详细地址',
  `bank_name` VARCHAR(100) DEFAULT NULL COMMENT '开户银行',
  `bank_account` VARCHAR(255) DEFAULT NULL COMMENT '银行账号（AES加密）',
  `bank_account_name` VARCHAR(100) DEFAULT NULL COMMENT '开户名',
  `service_fee_rate` DECIMAL(5,4) NOT NULL DEFAULT 0.0500 COMMENT '服务费率（默认5%）',
  `status` VARCHAR(20) NOT NULL DEFAULT 'pending' COMMENT '状态（pending-待审核，active-正常，frozen-冻结，rejected-已拒绝）',
  `audit_time` DATETIME DEFAULT NULL COMMENT '审核时间',
  `audit_by` BIGINT DEFAULT NULL COMMENT '审核人ID',
  `audit_opinion` VARCHAR(500) DEFAULT NULL COMMENT '审核意见',
  `create_by` BIGINT DEFAULT NULL COMMENT '创建人ID',
  `create_time` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `update_by` BIGINT DEFAULT NULL COMMENT '更新人ID',
  `update_time` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  `remark` VARCHAR(500) DEFAULT NULL COMMENT '备注',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_user_id` (`user_id`),
  UNIQUE KEY `uk_merchant_code` (`merchant_code`),
  KEY `idx_status` (`status`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='商家信息表';
```

---

## 三、基础数据表

### 3.1 城市表 (t_city)

存储系统支持的城市信息。

```sql
CREATE TABLE `t_city` (
  `id` BIGINT NOT NULL COMMENT '主键ID',
  `city_name` VARCHAR(50) NOT NULL COMMENT '城市名称',
  `city_code` VARCHAR(20) NOT NULL COMMENT '城市编码',
  `province` VARCHAR(50) NOT NULL COMMENT '所属省份',
  `pinyin` VARCHAR(100) DEFAULT NULL COMMENT '拼音',
  `pinyin_abbr` VARCHAR(20) DEFAULT NULL COMMENT '拼音首字母缩写',
  `is_hot` TINYINT NOT NULL DEFAULT 0 COMMENT '是否热门城市（0-否，1-是）',
  `sort_order` INT NOT NULL DEFAULT 0 COMMENT '排序号',
  `status` VARCHAR(20) NOT NULL DEFAULT 'active' COMMENT '状态（active-启用，inactive-禁用）',
  `create_by` BIGINT DEFAULT NULL COMMENT '创建人ID',
  `create_time` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `update_by` BIGINT DEFAULT NULL COMMENT '更新人ID',
  `update_time` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  `remark` VARCHAR(500) DEFAULT NULL COMMENT '备注',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_city_code` (`city_code`),
  KEY `idx_is_hot` (`is_hot`),
  KEY `idx_sort_order` (`sort_order`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='城市表';
```

### 3.2 类目表 (t_category)

存储演出类目分类（演唱会、话剧、音乐会等）。

```sql
CREATE TABLE `t_category` (
  `id` BIGINT NOT NULL COMMENT '主键ID',
  `category_name` VARCHAR(50) NOT NULL COMMENT '类目名称',
  `category_code` VARCHAR(50) NOT NULL COMMENT '类目编码',
  `parent_id` BIGINT DEFAULT 0 COMMENT '父类目ID（0表示顶级类目）',
  `level` TINYINT NOT NULL DEFAULT 1 COMMENT '层级（1-一级，2-二级）',
  `icon_url` VARCHAR(500) DEFAULT NULL COMMENT '图标URL',
  `sort_order` INT NOT NULL DEFAULT 0 COMMENT '排序号',
  `status` VARCHAR(20) NOT NULL DEFAULT 'active' COMMENT '状态（active-启用，inactive-禁用）',
  `create_by` BIGINT DEFAULT NULL COMMENT '创建人ID',
  `create_time` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `update_by` BIGINT DEFAULT NULL COMMENT '更新人ID',
  `update_time` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  `remark` VARCHAR(500) DEFAULT NULL COMMENT '备注',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_category_code` (`category_code`),
  KEY `idx_parent_id` (`parent_id`),
  KEY `idx_sort_order` (`sort_order`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='类目表';
```

### 3.3 场馆表 (t_venue)

存储演出场馆信息。

```sql
CREATE TABLE `t_venue` (
  `id` BIGINT NOT NULL COMMENT '主键ID',
  `venue_name` VARCHAR(100) NOT NULL COMMENT '场馆名称',
  `venue_code` VARCHAR(50) NOT NULL COMMENT '场馆编码',
  `city_id` BIGINT NOT NULL COMMENT '所在城市ID',
  `province` VARCHAR(50) NOT NULL COMMENT '省份',
  `city` VARCHAR(50) NOT NULL COMMENT '城市',
  `district` VARCHAR(50) DEFAULT NULL COMMENT '区县',
  `address` VARCHAR(200) NOT NULL COMMENT '详细地址',
  `latitude` DECIMAL(10,7) DEFAULT NULL COMMENT '纬度',
  `longitude` DECIMAL(10,7) DEFAULT NULL COMMENT '经度',
  `capacity` INT DEFAULT NULL COMMENT '容纳人数',
  `contact_phone` VARCHAR(20) DEFAULT NULL COMMENT '联系电话',
  `images` TEXT DEFAULT NULL COMMENT '场馆图片（JSON数组）',
  `facilities` TEXT DEFAULT NULL COMMENT '设施说明（JSON）',
  `traffic_guide` TEXT DEFAULT NULL COMMENT '交通指南',
  `status` VARCHAR(20) NOT NULL DEFAULT 'active' COMMENT '状态（active-启用，inactive-禁用）',
  `create_by` BIGINT DEFAULT NULL COMMENT '创建人ID',
  `create_time` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `update_by` BIGINT DEFAULT NULL COMMENT '更新人ID',
  `update_time` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  `remark` VARCHAR(500) DEFAULT NULL COMMENT '备注',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_venue_code` (`venue_code`),
  KEY `idx_city_id` (`city_id`),
  KEY `idx_status` (`status`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='场馆表';
```

### 3.4 座位图表 (t_seat_map)

存储场馆座位图配置。

```sql
CREATE TABLE `t_seat_map` (
  `id` BIGINT NOT NULL COMMENT '主键ID',
  `venue_id` BIGINT NOT NULL COMMENT '场馆ID',
  `map_name` VARCHAR(100) NOT NULL COMMENT '座位图名称',
  `map_code` VARCHAR(50) NOT NULL COMMENT '座位图编码',
  `total_seats` INT NOT NULL DEFAULT 0 COMMENT '总座位数',
  `config_data` LONGTEXT NOT NULL COMMENT '座位图配置数据（JSON格式）',
  `preview_image` VARCHAR(500) DEFAULT NULL COMMENT '预览图URL',
  `status` VARCHAR(20) NOT NULL DEFAULT 'active' COMMENT '状态（active-启用，inactive-禁用）',
  `create_by` BIGINT DEFAULT NULL COMMENT '创建人ID',
  `create_time` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `update_by` BIGINT DEFAULT NULL COMMENT '更新人ID',
  `update_time` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  `remark` VARCHAR(500) DEFAULT NULL COMMENT '备注',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_map_code` (`map_code`),
  KEY `idx_venue_id` (`venue_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='座位图表';
```

---

## 四、项目相关表

### 4.1 项目表 (t_event)

存储演出项目基本信息。

```sql
CREATE TABLE `t_event` (
  `id` BIGINT NOT NULL COMMENT '主键ID',
  `merchant_id` BIGINT NOT NULL COMMENT '商家ID',
  `event_name` VARCHAR(200) NOT NULL COMMENT '项目名称',
  `event_code` VARCHAR(50) NOT NULL COMMENT '项目编码',
  `category_id` BIGINT NOT NULL COMMENT '类目ID',
  `city_id` BIGINT NOT NULL COMMENT '城市ID',
  `venue_id` BIGINT NOT NULL COMMENT '场馆ID',
  `cover_image` VARCHAR(500) NOT NULL COMMENT '封面图URL',
  `images` TEXT DEFAULT NULL COMMENT '项目图片（JSON数组）',
  `description` TEXT DEFAULT NULL COMMENT '项目详情描述',
  `notice` TEXT DEFAULT NULL COMMENT '购票须知',
  `strategy` TEXT DEFAULT NULL COMMENT '抢票攻略',
  `duration` INT DEFAULT NULL COMMENT '演出时长（分钟）',
  `min_price` DECIMAL(10,2) DEFAULT NULL COMMENT '最低价格',
  `max_price` DECIMAL(10,2) DEFAULT NULL COMMENT '最高价格',
  `tags` VARCHAR(500) DEFAULT NULL COMMENT '标签（JSON数组）',
  `artists` TEXT DEFAULT NULL COMMENT '艺人信息（JSON）',
  `is_real_name` TINYINT NOT NULL DEFAULT 0 COMMENT '是否需要实名（0-否，1-是）',
  `limit_per_user` INT DEFAULT NULL COMMENT '每人限购数量',
  `sale_start_time` DATETIME DEFAULT NULL COMMENT '开售时间',
  `sale_end_time` DATETIME DEFAULT NULL COMMENT '停售时间',
  `status` VARCHAR(20) NOT NULL DEFAULT 'draft' COMMENT '状态（draft-草稿，on_sale-在售，sold_out-售罄，ended-已结束，cancelled-已取消）',
  `view_count` INT NOT NULL DEFAULT 0 COMMENT '浏览次数',
  `wish_count` INT NOT NULL DEFAULT 0 COMMENT '想看人数',
  `sales_volume` INT NOT NULL DEFAULT 0 COMMENT '销量',
  `create_by` BIGINT DEFAULT NULL COMMENT '创建人ID',
  `create_time` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `update_by` BIGINT DEFAULT NULL COMMENT '更新人ID',
  `update_time` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  `remark` VARCHAR(500) DEFAULT NULL COMMENT '备注',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_event_code` (`event_code`),
  KEY `idx_merchant_id` (`merchant_id`),
  KEY `idx_category_id` (`category_id`),
  KEY `idx_city_id` (`city_id`),
  KEY `idx_venue_id` (`venue_id`),
  KEY `idx_status` (`status`),
  KEY `idx_sale_start_time` (`sale_start_time`),
  KEY `idx_create_time` (`create_time`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='项目表';
```


### 4.2 场次表 (t_session)

存储项目的具体演出场次信息。

```sql
CREATE TABLE `t_session` (
  `id` BIGINT NOT NULL COMMENT '主键ID',
  `event_id` BIGINT NOT NULL COMMENT '项目ID',
  `session_name` VARCHAR(100) DEFAULT NULL COMMENT '场次名称',
  `start_time` DATETIME NOT NULL COMMENT '开始时间',
  `end_time` DATETIME NOT NULL COMMENT '结束时间',
  `status` VARCHAR(20) NOT NULL DEFAULT 'on_sale' COMMENT '状态（on_sale-在售，sold_out-售罄，ended-已结束，cancelled-已取消）',
  `total_stock` INT NOT NULL DEFAULT 0 COMMENT '总库存',
  `available_stock` INT NOT NULL DEFAULT 0 COMMENT '可售库存',
  `locked_stock` INT NOT NULL DEFAULT 0 COMMENT '锁定库存',
  `sold_stock` INT NOT NULL DEFAULT 0 COMMENT '已售库存',
  `create_by` BIGINT DEFAULT NULL COMMENT '创建人ID',
  `create_time` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `update_by` BIGINT DEFAULT NULL COMMENT '更新人ID',
  `update_time` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  `remark` VARCHAR(500) DEFAULT NULL COMMENT '备注',
  PRIMARY KEY (`id`),
  KEY `idx_event_id` (`event_id`),
  KEY `idx_start_time` (`start_time`),
  KEY `idx_status` (`status`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='场次表';
```

### 4.3 票档表 (t_ticket_tier)

存储场次的票档（不同价位和座位区域）。

```sql
CREATE TABLE `t_ticket_tier` (
  `id` BIGINT NOT NULL COMMENT '主键ID',
  `session_id` BIGINT NOT NULL COMMENT '场次ID',
  `tier_name` VARCHAR(100) NOT NULL COMMENT '票档名称',
  `tier_code` VARCHAR(50) NOT NULL COMMENT '票档编码',
  `price` DECIMAL(10,2) NOT NULL COMMENT '价格',
  `seat_area` VARCHAR(100) DEFAULT NULL COMMENT '座位区域',
  `total_stock` INT NOT NULL DEFAULT 0 COMMENT '总库存',
  `available_stock` INT NOT NULL DEFAULT 0 COMMENT '可售库存',
  `locked_stock` INT NOT NULL DEFAULT 0 COMMENT '锁定库存',
  `sold_stock` INT NOT NULL DEFAULT 0 COMMENT '已售库存',
  `sort_order` INT NOT NULL DEFAULT 0 COMMENT '排序号',
  `status` VARCHAR(20) NOT NULL DEFAULT 'on_sale' COMMENT '状态（on_sale-在售，sold_out-售罄）',
  `create_by` BIGINT DEFAULT NULL COMMENT '创建人ID',
  `create_time` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `update_by` BIGINT DEFAULT NULL COMMENT '更新人ID',
  `update_time` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  `remark` VARCHAR(500) DEFAULT NULL COMMENT '备注',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_tier_code` (`tier_code`),
  KEY `idx_session_id` (`session_id`),
  KEY `idx_status` (`status`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='票档表';
```

### 4.4 座位表 (t_seat)

存储具体座位信息。

```sql
CREATE TABLE `t_seat` (
  `id` BIGINT NOT NULL COMMENT '主键ID',
  `session_id` BIGINT NOT NULL COMMENT '场次ID',
  `ticket_tier_id` BIGINT NOT NULL COMMENT '票档ID',
  `seat_map_id` BIGINT DEFAULT NULL COMMENT '座位图ID',
  `row_num` VARCHAR(20) NOT NULL COMMENT '排号',
  `seat_num` VARCHAR(20) NOT NULL COMMENT '座位号',
  `seat_type` VARCHAR(20) DEFAULT 'normal' COMMENT '座位类型（normal-普通座，vip-VIP座，wheelchair-轮椅座）',
  `status` VARCHAR(20) NOT NULL DEFAULT 'available' COMMENT '状态（available-可售，locked-锁定，sold-已售）',
  `locked_until` DATETIME DEFAULT NULL COMMENT '锁定截止时间',
  `locked_by` BIGINT DEFAULT NULL COMMENT '锁定人ID',
  `create_by` BIGINT DEFAULT NULL COMMENT '创建人ID',
  `create_time` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `update_by` BIGINT DEFAULT NULL COMMENT '更新人ID',
  `update_time` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  `remark` VARCHAR(500) DEFAULT NULL COMMENT '备注',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_session_row_seat` (`session_id`, `row_num`, `seat_num`),
  KEY `idx_ticket_tier_id` (`ticket_tier_id`),
  KEY `idx_status` (`status`),
  KEY `idx_locked_until` (`locked_until`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='座位表';
```

### 4.5 想看表 (t_wishlist)

存储用户收藏的项目（想看列表）。

```sql
CREATE TABLE `t_wishlist` (
  `id` BIGINT NOT NULL COMMENT '主键ID',
  `user_id` BIGINT NOT NULL COMMENT '用户ID',
  `event_id` BIGINT NOT NULL COMMENT '项目ID',
  `is_notify` TINYINT NOT NULL DEFAULT 1 COMMENT '是否开售提醒（0-否，1-是）',
  `create_by` BIGINT DEFAULT NULL COMMENT '创建人ID',
  `create_time` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `update_by` BIGINT DEFAULT NULL COMMENT '更新人ID',
  `update_time` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  `remark` VARCHAR(500) DEFAULT NULL COMMENT '备注',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_user_event` (`user_id`, `event_id`),
  KEY `idx_event_id` (`event_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='想看表';
```

---

## 五、订单相关表

### 5.1 订单表 (t_order)

存储订单主表信息。

```sql
CREATE TABLE `t_order` (
  `id` BIGINT NOT NULL COMMENT '主键ID',
  `order_no` VARCHAR(50) NOT NULL COMMENT '订单号',
  `user_id` BIGINT NOT NULL COMMENT '用户ID',
  `merchant_id` BIGINT NOT NULL COMMENT '商家ID',
  `event_id` BIGINT NOT NULL COMMENT '项目ID',
  `session_id` BIGINT NOT NULL COMMENT '场次ID',
  `ticket_count` INT NOT NULL DEFAULT 0 COMMENT '票数',
  `total_amount` DECIMAL(10,2) NOT NULL DEFAULT 0.00 COMMENT '订单总金额',
  `ticket_amount` DECIMAL(10,2) NOT NULL DEFAULT 0.00 COMMENT '票价金额',
  `service_fee` DECIMAL(10,2) NOT NULL DEFAULT 0.00 COMMENT '服务费',
  `delivery_fee` DECIMAL(10,2) NOT NULL DEFAULT 0.00 COMMENT '配送费',
  `discount_amount` DECIMAL(10,2) NOT NULL DEFAULT 0.00 COMMENT '优惠金额',
  `payable_amount` DECIMAL(10,2) NOT NULL DEFAULT 0.00 COMMENT '应付金额',
  `paid_amount` DECIMAL(10,2) NOT NULL DEFAULT 0.00 COMMENT '实付金额',
  `delivery_method` VARCHAR(20) NOT NULL COMMENT '配送方式（electronic-电子票，express-快递）',
  `address_id` BIGINT DEFAULT NULL COMMENT '收货地址ID',
  `express_no` VARCHAR(100) DEFAULT NULL COMMENT '快递单号',
  `express_company` VARCHAR(50) DEFAULT NULL COMMENT '快递公司',
  `status` VARCHAR(20) NOT NULL DEFAULT 'pending' COMMENT '订单状态（pending-待支付，paid-已支付，cancelled-已取消，refunding-退款中，refunded-已退款，completed-已完成）',
  `pay_time` DATETIME DEFAULT NULL COMMENT '支付时间',
  `cancel_time` DATETIME DEFAULT NULL COMMENT '取消时间',
  `cancel_reason` VARCHAR(500) DEFAULT NULL COMMENT '取消原因',
  `expire_time` DATETIME DEFAULT NULL COMMENT '过期时间',
  `source` VARCHAR(20) DEFAULT NULL COMMENT '订单来源（h5、app、backend）',
  `create_by` BIGINT DEFAULT NULL COMMENT '创建人ID',
  `create_time` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `update_by` BIGINT DEFAULT NULL COMMENT '更新人ID',
  `update_time` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  `remark` VARCHAR(500) DEFAULT NULL COMMENT '备注',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_order_no` (`order_no`),
  KEY `idx_user_id` (`user_id`),
  KEY `idx_merchant_id` (`merchant_id`),
  KEY `idx_event_id` (`event_id`),
  KEY `idx_session_id` (`session_id`),
  KEY `idx_status` (`status`),
  KEY `idx_create_time` (`create_time`),
  KEY `idx_expire_time` (`expire_time`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='订单表';
```

### 5.2 订单项表 (t_order_item)

存储订单明细（每张票的信息）。

```sql
CREATE TABLE `t_order_item` (
  `id` BIGINT NOT NULL COMMENT '主键ID',
  `order_id` BIGINT NOT NULL COMMENT '订单ID',
  `ticket_tier_id` BIGINT NOT NULL COMMENT '票档ID',
  `seat_id` BIGINT DEFAULT NULL COMMENT '座位ID',
  `price` DECIMAL(10,2) NOT NULL COMMENT '票价',
  `buyer_name` VARCHAR(50) NOT NULL COMMENT '购票人姓名',
  `buyer_phone` VARCHAR(20) NOT NULL COMMENT '购票人手机号',
  `real_name_id` BIGINT DEFAULT NULL COMMENT '实名信息ID',
  `ticket_code` VARCHAR(100) DEFAULT NULL COMMENT '电子票码',
  `qr_code_url` VARCHAR(500) DEFAULT NULL COMMENT '二维码URL',
  `status` VARCHAR(20) NOT NULL DEFAULT 'valid' COMMENT '状态（valid-有效，used-已使用，refunded-已退款，cancelled-已取消）',
  `use_time` DATETIME DEFAULT NULL COMMENT '使用时间（检票时间）',
  `create_by` BIGINT DEFAULT NULL COMMENT '创建人ID',
  `create_time` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `update_by` BIGINT DEFAULT NULL COMMENT '更新人ID',
  `update_time` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  `remark` VARCHAR(500) DEFAULT NULL COMMENT '备注',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_ticket_code` (`ticket_code`),
  KEY `idx_order_id` (`order_id`),
  KEY `idx_ticket_tier_id` (`ticket_tier_id`),
  KEY `idx_seat_id` (`seat_id`),
  KEY `idx_status` (`status`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='订单项表';
```

### 5.3 退款表 (t_refund)

存储退款申请和处理信息。

```sql
CREATE TABLE `t_refund` (
  `id` BIGINT NOT NULL COMMENT '主键ID',
  `refund_no` VARCHAR(50) NOT NULL COMMENT '退款单号',
  `order_id` BIGINT NOT NULL COMMENT '订单ID',
  `order_no` VARCHAR(50) NOT NULL COMMENT '订单号',
  `user_id` BIGINT NOT NULL COMMENT '用户ID',
  `refund_amount` DECIMAL(10,2) NOT NULL COMMENT '退款金额',
  `refund_fee` DECIMAL(10,2) NOT NULL DEFAULT 0.00 COMMENT '退款手续费',
  `actual_refund_amount` DECIMAL(10,2) NOT NULL COMMENT '实际退款金额',
  `refund_reason` VARCHAR(500) NOT NULL COMMENT '退款原因',
  `refund_type` VARCHAR(20) NOT NULL COMMENT '退款类型（full-全额退款，partial-部分退款）',
  `order_item_ids` TEXT DEFAULT NULL COMMENT '退款的订单项ID（JSON数组）',
  `status` VARCHAR(20) NOT NULL DEFAULT 'pending' COMMENT '状态（pending-待审核，approved-已同意，rejected-已拒绝，processing-处理中，completed-已完成，failed-失败）',
  `audit_time` DATETIME DEFAULT NULL COMMENT '审核时间',
  `audit_by` BIGINT DEFAULT NULL COMMENT '审核人ID',
  `audit_opinion` VARCHAR(500) DEFAULT NULL COMMENT '审核意见',
  `refund_time` DATETIME DEFAULT NULL COMMENT '退款完成时间',
  `create_by` BIGINT DEFAULT NULL COMMENT '创建人ID',
  `create_time` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `update_by` BIGINT DEFAULT NULL COMMENT '更新人ID',
  `update_time` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  `remark` VARCHAR(500) DEFAULT NULL COMMENT '备注',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_refund_no` (`refund_no`),
  KEY `idx_order_id` (`order_id`),
  KEY `idx_user_id` (`user_id`),
  KEY `idx_status` (`status`),
  KEY `idx_create_time` (`create_time`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='退款表';
```

---

## 六、支付相关表

### 6.1 支付订单表 (t_payment)

存储支付订单信息。

```sql
CREATE TABLE `t_payment` (
  `id` BIGINT NOT NULL COMMENT '主键ID',
  `payment_no` VARCHAR(50) NOT NULL COMMENT '支付单号',
  `order_id` BIGINT NOT NULL COMMENT '订单ID',
  `order_no` VARCHAR(50) NOT NULL COMMENT '订单号',
  `user_id` BIGINT NOT NULL COMMENT '用户ID',
  `amount` DECIMAL(10,2) NOT NULL COMMENT '支付金额',
  `payment_method` VARCHAR(20) NOT NULL COMMENT '支付方式（alipay-支付宝，wechat-微信，bank_card-银行卡）',
  `payment_channel` VARCHAR(50) DEFAULT NULL COMMENT '支付渠道（app、h5、pc）',
  `third_party_no` VARCHAR(100) DEFAULT NULL COMMENT '第三方支付流水号',
  `status` VARCHAR(20) NOT NULL DEFAULT 'pending' COMMENT '状态（pending-待支付，success-成功，failed-失败，cancelled-已取消）',
  `pay_time` DATETIME DEFAULT NULL COMMENT '支付时间',
  `callback_time` DATETIME DEFAULT NULL COMMENT '回调时间',
  `callback_data` TEXT DEFAULT NULL COMMENT '回调数据（JSON）',
  `create_by` BIGINT DEFAULT NULL COMMENT '创建人ID',
  `create_time` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `update_by` BIGINT DEFAULT NULL COMMENT '更新人ID',
  `update_time` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  `remark` VARCHAR(500) DEFAULT NULL COMMENT '备注',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_payment_no` (`payment_no`),
  KEY `idx_order_id` (`order_id`),
  KEY `idx_user_id` (`user_id`),
  KEY `idx_third_party_no` (`third_party_no`),
  KEY `idx_status` (`status`),
  KEY `idx_create_time` (`create_time`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='支付订单表';
```

---

## 七、营销相关表

### 7.1 优惠券表 (t_coupon)

存储优惠券模板信息。

```sql
CREATE TABLE `t_coupon` (
  `id` BIGINT NOT NULL COMMENT '主键ID',
  `coupon_name` VARCHAR(100) NOT NULL COMMENT '优惠券名称',
  `coupon_code` VARCHAR(50) NOT NULL COMMENT '优惠券编码',
  `coupon_type` VARCHAR(20) NOT NULL COMMENT '优惠券类型（discount-折扣券，cash-代金券，gift-兑换券）',
  `discount_rate` DECIMAL(5,4) DEFAULT NULL COMMENT '折扣率（如0.8表示8折）',
  `discount_amount` DECIMAL(10,2) DEFAULT NULL COMMENT '减免金额',
  `min_amount` DECIMAL(10,2) NOT NULL DEFAULT 0.00 COMMENT '最低消费金额',
  `max_discount` DECIMAL(10,2) DEFAULT NULL COMMENT '最高优惠金额',
  `total_quantity` INT NOT NULL COMMENT '发行总量',
  `received_quantity` INT NOT NULL DEFAULT 0 COMMENT '已领取数量',
  `used_quantity` INT NOT NULL DEFAULT 0 COMMENT '已使用数量',
  `limit_per_user` INT NOT NULL DEFAULT 1 COMMENT '每人限领数量',
  `valid_days` INT DEFAULT NULL COMMENT '有效天数（领取后N天有效）',
  `valid_from` DATETIME DEFAULT NULL COMMENT '有效期开始时间',
  `valid_to` DATETIME DEFAULT NULL COMMENT '有效期结束时间',
  `applicable_scope` VARCHAR(20) NOT NULL DEFAULT 'all' COMMENT '适用范围（all-全场，category-类目，event-指定项目）',
  `applicable_ids` TEXT DEFAULT NULL COMMENT '适用ID列表（JSON数组）',
  `status` VARCHAR(20) NOT NULL DEFAULT 'active' COMMENT '状态（active-启用，inactive-禁用，expired-已过期）',
  `create_by` BIGINT DEFAULT NULL COMMENT '创建人ID',
  `create_time` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `update_by` BIGINT DEFAULT NULL COMMENT '更新人ID',
  `update_time` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  `remark` VARCHAR(500) DEFAULT NULL COMMENT '备注',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_coupon_code` (`coupon_code`),
  KEY `idx_status` (`status`),
  KEY `idx_valid_from` (`valid_from`),
  KEY `idx_valid_to` (`valid_to`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='优惠券表';
```

### 7.2 用户优惠券表 (t_user_coupon)

存储用户领取的优惠券。

```sql
CREATE TABLE `t_user_coupon` (
  `id` BIGINT NOT NULL COMMENT '主键ID',
  `user_id` BIGINT NOT NULL COMMENT '用户ID',
  `coupon_id` BIGINT NOT NULL COMMENT '优惠券ID',
  `coupon_code` VARCHAR(50) NOT NULL COMMENT '优惠券编码',
  `receive_time` DATETIME NOT NULL COMMENT '领取时间',
  `valid_from` DATETIME NOT NULL COMMENT '有效期开始时间',
  `valid_to` DATETIME NOT NULL COMMENT '有效期结束时间',
  `status` VARCHAR(20) NOT NULL DEFAULT 'unused' COMMENT '状态（unused-未使用，used-已使用，expired-已过期）',
  `use_time` DATETIME DEFAULT NULL COMMENT '使用时间',
  `order_id` BIGINT DEFAULT NULL COMMENT '使用的订单ID',
  `create_by` BIGINT DEFAULT NULL COMMENT '创建人ID',
  `create_time` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `update_by` BIGINT DEFAULT NULL COMMENT '更新人ID',
  `update_time` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  `remark` VARCHAR(500) DEFAULT NULL COMMENT '备注',
  PRIMARY KEY (`id`),
  KEY `idx_user_id` (`user_id`),
  KEY `idx_coupon_id` (`coupon_id`),
  KEY `idx_status` (`status`),
  KEY `idx_valid_to` (`valid_to`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='用户优惠券表';
```

### 7.3 促销活动表 (t_promotion)

存储促销活动信息。

```sql
CREATE TABLE `t_promotion` (
  `id` BIGINT NOT NULL COMMENT '主键ID',
  `promotion_name` VARCHAR(100) NOT NULL COMMENT '活动名称',
  `promotion_code` VARCHAR(50) NOT NULL COMMENT '活动编码',
  `promotion_type` VARCHAR(20) NOT NULL COMMENT '活动类型（full_reduction-满减，discount-折扣，bundle-套餐）',
  `rules` TEXT NOT NULL COMMENT '活动规则（JSON格式）',
  `applicable_scope` VARCHAR(20) NOT NULL DEFAULT 'all' COMMENT '适用范围（all-全场，category-类目，event-指定项目）',
  `applicable_ids` TEXT DEFAULT NULL COMMENT '适用ID列表（JSON数组）',
  `valid_from` DATETIME NOT NULL COMMENT '有效期开始时间',
  `valid_to` DATETIME NOT NULL COMMENT '有效期结束时间',
  `priority` INT NOT NULL DEFAULT 0 COMMENT '优先级（数字越大优先级越高）',
  `can_stack` TINYINT NOT NULL DEFAULT 0 COMMENT '是否可叠加（0-否，1-是）',
  `status` VARCHAR(20) NOT NULL DEFAULT 'active' COMMENT '状态（active-启用，inactive-禁用，expired-已过期）',
  `create_by` BIGINT DEFAULT NULL COMMENT '创建人ID',
  `create_time` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `update_by` BIGINT DEFAULT NULL COMMENT '更新人ID',
  `update_time` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  `remark` VARCHAR(500) DEFAULT NULL COMMENT '备注',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_promotion_code` (`promotion_code`),
  KEY `idx_status` (`status`),
  KEY `idx_valid_from` (`valid_from`),
  KEY `idx_valid_to` (`valid_to`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='促销活动表';
```

---

## 八、运营相关表

### 8.1 Banner表 (t_banner)

存储首页轮播图配置。

```sql
CREATE TABLE `t_banner` (
  `id` BIGINT NOT NULL COMMENT '主键ID',
  `banner_name` VARCHAR(100) NOT NULL COMMENT 'Banner名称',
  `banner_type` VARCHAR(20) NOT NULL COMMENT 'Banner类型（event-项目，activity-活动，url-外链）',
  `image_url` VARCHAR(500) NOT NULL COMMENT '图片URL',
  `link_type` VARCHAR(20) NOT NULL COMMENT '跳转类型（event-项目详情，activity-活动页，url-外链）',
  `link_value` VARCHAR(500) NOT NULL COMMENT '跳转值（项目ID、活动ID、URL）',
  `position` VARCHAR(20) NOT NULL DEFAULT 'home' COMMENT '展示位置（home-首页，category-分类页）',
  `sort_order` INT NOT NULL DEFAULT 0 COMMENT '排序号',
  `valid_from` DATETIME DEFAULT NULL COMMENT '有效期开始时间',
  `valid_to` DATETIME DEFAULT NULL COMMENT '有效期结束时间',
  `status` VARCHAR(20) NOT NULL DEFAULT 'active' COMMENT '状态（active-启用，inactive-禁用）',
  `view_count` INT NOT NULL DEFAULT 0 COMMENT '浏览次数',
  `click_count` INT NOT NULL DEFAULT 0 COMMENT '点击次数',
  `create_by` BIGINT DEFAULT NULL COMMENT '创建人ID',
  `create_time` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `update_by` BIGINT DEFAULT NULL COMMENT '更新人ID',
  `update_time` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  `remark` VARCHAR(500) DEFAULT NULL COMMENT '备注',
  PRIMARY KEY (`id`),
  KEY `idx_position` (`position`),
  KEY `idx_sort_order` (`sort_order`),
  KEY `idx_status` (`status`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='Banner表';
```

### 8.2 运营位表 (t_operation_position)

存储各页面的运营位配置。

```sql
CREATE TABLE `t_operation_position` (
  `id` BIGINT NOT NULL COMMENT '主键ID',
  `position_name` VARCHAR(100) NOT NULL COMMENT '运营位名称',
  `position_code` VARCHAR(50) NOT NULL COMMENT '运营位编码',
  `position_type` VARCHAR(20) NOT NULL COMMENT '运营位类型（recommend-推荐，hot-热门，new-新品）',
  `page_location` VARCHAR(50) NOT NULL COMMENT '页面位置（home-首页，category-分类页，detail-详情页）',
  `content_type` VARCHAR(20) NOT NULL COMMENT '内容类型（event-项目，activity-活动）',
  `content_ids` TEXT DEFAULT NULL COMMENT '内容ID列表（JSON数组）',
  `display_rule` VARCHAR(20) NOT NULL DEFAULT 'fixed' COMMENT '展示规则（fixed-固定展示，rotate-轮播展示，recommend-个性化推荐）',
  `max_display` INT NOT NULL DEFAULT 10 COMMENT '最大展示数量',
  `sort_order` INT NOT NULL DEFAULT 0 COMMENT '排序号',
  `valid_from` DATETIME DEFAULT NULL COMMENT '有效期开始时间',
  `valid_to` DATETIME DEFAULT NULL COMMENT '有效期结束时间',
  `status` VARCHAR(20) NOT NULL DEFAULT 'active' COMMENT '状态（active-启用，inactive-禁用）',
  `view_count` INT NOT NULL DEFAULT 0 COMMENT '浏览次数',
  `click_count` INT NOT NULL DEFAULT 0 COMMENT '点击次数',
  `create_by` BIGINT DEFAULT NULL COMMENT '创建人ID',
  `create_time` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `update_by` BIGINT DEFAULT NULL COMMENT '更新人ID',
  `update_time` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  `remark` VARCHAR(500) DEFAULT NULL COMMENT '备注',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_position_code` (`position_code`),
  KEY `idx_page_location` (`page_location`),
  KEY `idx_status` (`status`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='运营位表';
```

### 8.3 活动页面表 (t_activity_page)

存储专题活动页面配置。

```sql
CREATE TABLE `t_activity_page` (
  `id` BIGINT NOT NULL COMMENT '主键ID',
  `page_name` VARCHAR(100) NOT NULL COMMENT '页面名称',
  `page_code` VARCHAR(50) NOT NULL COMMENT '页面编码',
  `page_title` VARCHAR(200) NOT NULL COMMENT '页面标题',
  `page_config` LONGTEXT NOT NULL COMMENT '页面配置（JSON格式，包含模块布局）',
  `cover_image` VARCHAR(500) DEFAULT NULL COMMENT '封面图URL',
  `valid_from` DATETIME DEFAULT NULL COMMENT '有效期开始时间',
  `valid_to` DATETIME DEFAULT NULL COMMENT '有效期结束时间',
  `status` VARCHAR(20) NOT NULL DEFAULT 'draft' COMMENT '状态（draft-草稿，published-已发布，offline-已下线）',
  `view_count` INT NOT NULL DEFAULT 0 COMMENT '浏览次数',
  `create_by` BIGINT DEFAULT NULL COMMENT '创建人ID',
  `create_time` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `update_by` BIGINT DEFAULT NULL COMMENT '更新人ID',
  `update_time` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  `remark` VARCHAR(500) DEFAULT NULL COMMENT '备注',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_page_code` (`page_code`),
  KEY `idx_status` (`status`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='活动页面表';
```

---

## 九、通知相关表

### 9.1 通知记录表 (t_notification)

存储系统通知记录。

```sql
CREATE TABLE `t_notification` (
  `id` BIGINT NOT NULL COMMENT '主键ID',
  `user_id` BIGINT NOT NULL COMMENT '用户ID',
  `notification_type` VARCHAR(20) NOT NULL COMMENT '通知类型（sms-短信，email-邮件，push-推送，system-站内消息）',
  `title` VARCHAR(200) DEFAULT NULL COMMENT '通知标题',
  `content` TEXT NOT NULL COMMENT '通知内容',
  `template_code` VARCHAR(50) DEFAULT NULL COMMENT '模板编码',
  `template_params` TEXT DEFAULT NULL COMMENT '模板参数（JSON）',
  `receiver` VARCHAR(100) NOT NULL COMMENT '接收人（手机号、邮箱等）',
  `status` VARCHAR(20) NOT NULL DEFAULT 'pending' COMMENT '状态（pending-待发送，sent-已发送，failed-失败）',
  `send_time` DATETIME DEFAULT NULL COMMENT '发送时间',
  `read_time` DATETIME DEFAULT NULL COMMENT '阅读时间（站内消息）',
  `error_msg` VARCHAR(500) DEFAULT NULL COMMENT '错误信息',
  `retry_count` INT NOT NULL DEFAULT 0 COMMENT '重试次数',
  `create_by` BIGINT DEFAULT NULL COMMENT '创建人ID',
  `create_time` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `update_by` BIGINT DEFAULT NULL COMMENT '更新人ID',
  `update_time` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  `remark` VARCHAR(500) DEFAULT NULL COMMENT '备注',
  PRIMARY KEY (`id`),
  KEY `idx_user_id` (`user_id`),
  KEY `idx_notification_type` (`notification_type`),
  KEY `idx_status` (`status`),
  KEY `idx_create_time` (`create_time`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='通知记录表';
```

### 9.2 短信模板表 (t_sms_template)

存储短信模板配置。

```sql
CREATE TABLE `t_sms_template` (
  `id` BIGINT NOT NULL COMMENT '主键ID',
  `template_name` VARCHAR(100) NOT NULL COMMENT '模板名称',
  `template_code` VARCHAR(50) NOT NULL COMMENT '模板编码',
  `template_type` VARCHAR(20) NOT NULL COMMENT '模板类型（verify_code-验证码，order_notify-订单通知，ticket_notify-电子票通知）',
  `template_content` TEXT NOT NULL COMMENT '模板内容',
  `template_params` TEXT DEFAULT NULL COMMENT '模板参数说明（JSON）',
  `third_party_code` VARCHAR(100) DEFAULT NULL COMMENT '第三方平台模板编码',
  `status` VARCHAR(20) NOT NULL DEFAULT 'active' COMMENT '状态（active-启用，inactive-禁用）',
  `create_by` BIGINT DEFAULT NULL COMMENT '创建人ID',
  `create_time` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `update_by` BIGINT DEFAULT NULL COMMENT '更新人ID',
  `update_time` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  `remark` VARCHAR(500) DEFAULT NULL COMMENT '备注',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_template_code` (`template_code`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='短信模板表';
```

---

## 十、结算相关表

### 10.1 结算单表 (t_settlement)

存储商家结算单信息。

```sql
CREATE TABLE `t_settlement` (
  `id` BIGINT NOT NULL COMMENT '主键ID',
  `settlement_no` VARCHAR(50) NOT NULL COMMENT '结算单号',
  `merchant_id` BIGINT NOT NULL COMMENT '商家ID',
  `settlement_period` VARCHAR(20) NOT NULL COMMENT '结算周期（如2024-01）',
  `order_count` INT NOT NULL DEFAULT 0 COMMENT '订单数量',
  `order_amount` DECIMAL(10,2) NOT NULL DEFAULT 0.00 COMMENT '订单总金额',
  `service_fee` DECIMAL(10,2) NOT NULL DEFAULT 0.00 COMMENT '平台服务费',
  `refund_amount` DECIMAL(10,2) NOT NULL DEFAULT 0.00 COMMENT '退款金额',
  `settlement_amount` DECIMAL(10,2) NOT NULL DEFAULT 0.00 COMMENT '结算金额',
  `status` VARCHAR(20) NOT NULL DEFAULT 'pending' COMMENT '状态（pending-待结算，processing-结算中，completed-已完成）',
  `settlement_time` DATETIME DEFAULT NULL COMMENT '结算时间',
  `create_by` BIGINT DEFAULT NULL COMMENT '创建人ID',
  `create_time` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `update_by` BIGINT DEFAULT NULL COMMENT '更新人ID',
  `update_time` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  `remark` VARCHAR(500) DEFAULT NULL COMMENT '备注',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_settlement_no` (`settlement_no`),
  KEY `idx_merchant_id` (`merchant_id`),
  KEY `idx_status` (`status`),
  KEY `idx_settlement_period` (`settlement_period`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='结算单表';
```

### 10.2 提现记录表 (t_withdrawal)

存储商家提现记录。

```sql
CREATE TABLE `t_withdrawal` (
  `id` BIGINT NOT NULL COMMENT '主键ID',
  `withdrawal_no` VARCHAR(50) NOT NULL COMMENT '提现单号',
  `merchant_id` BIGINT NOT NULL COMMENT '商家ID',
  `withdrawal_amount` DECIMAL(10,2) NOT NULL COMMENT '提现金额',
  `bank_name` VARCHAR(100) NOT NULL COMMENT '开户银行',
  `bank_account` VARCHAR(255) NOT NULL COMMENT '银行账号（AES加密）',
  `bank_account_name` VARCHAR(100) NOT NULL COMMENT '开户名',
  `status` VARCHAR(20) NOT NULL DEFAULT 'pending' COMMENT '状态（pending-待审核，approved-已同意，rejected-已拒绝，processing-处理中，completed-已完成，failed-失败）',
  `audit_time` DATETIME DEFAULT NULL COMMENT '审核时间',
  `audit_by` BIGINT DEFAULT NULL COMMENT '审核人ID',
  `audit_opinion` VARCHAR(500) DEFAULT NULL COMMENT '审核意见',
  `transfer_time` DATETIME DEFAULT NULL COMMENT '转账时间',
  `transfer_voucher` VARCHAR(500) DEFAULT NULL COMMENT '转账凭证URL',
  `create_by` BIGINT DEFAULT NULL COMMENT '创建人ID',
  `create_time` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `update_by` BIGINT DEFAULT NULL COMMENT '更新人ID',
  `update_time` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  `remark` VARCHAR(500) DEFAULT NULL COMMENT '备注',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_withdrawal_no` (`withdrawal_no`),
  KEY `idx_merchant_id` (`merchant_id`),
  KEY `idx_status` (`status`),
  KEY `idx_create_time` (`create_time`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='提现记录表';
```

---

## 十一、系统配置表

### 11.1 系统参数表 (t_system_config)

存储系统全局配置参数。

```sql
CREATE TABLE `t_system_config` (
  `id` BIGINT NOT NULL COMMENT '主键ID',
  `config_key` VARCHAR(100) NOT NULL COMMENT '配置键',
  `config_value` TEXT NOT NULL COMMENT '配置值',
  `config_type` VARCHAR(20) NOT NULL COMMENT '配置类型（string-字符串，number-数字，json-JSON对象，boolean-布尔值）',
  `config_group` VARCHAR(50) NOT NULL COMMENT '配置分组（system-系统，order-订单，payment-支付，sms-短信）',
  `config_desc` VARCHAR(200) DEFAULT NULL COMMENT '配置说明',
  `is_encrypted` TINYINT NOT NULL DEFAULT 0 COMMENT '是否加密（0-否，1-是）',
  `status` VARCHAR(20) NOT NULL DEFAULT 'active' COMMENT '状态（active-启用，inactive-禁用）',
  `create_by` BIGINT DEFAULT NULL COMMENT '创建人ID',
  `create_time` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `update_by` BIGINT DEFAULT NULL COMMENT '更新人ID',
  `update_time` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  `remark` VARCHAR(500) DEFAULT NULL COMMENT '备注',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_config_key` (`config_key`),
  KEY `idx_config_group` (`config_group`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='系统参数表';
```

### 11.2 系统公告表 (t_system_notice)

存储系统公告信息。

```sql
CREATE TABLE `t_system_notice` (
  `id` BIGINT NOT NULL COMMENT '主键ID',
  `notice_title` VARCHAR(200) NOT NULL COMMENT '公告标题',
  `notice_content` TEXT NOT NULL COMMENT '公告内容',
  `notice_type` VARCHAR(20) NOT NULL COMMENT '公告类型（system-系统公告，maintenance-维护公告，activity-活动公告）',
  `target_user` VARCHAR(20) NOT NULL DEFAULT 'all' COMMENT '目标用户（all-全部，customer-C端用户，merchant-商家）',
  `display_position` VARCHAR(50) DEFAULT NULL COMMENT '展示位置（home-首页，my-个人中心）',
  `valid_from` DATETIME DEFAULT NULL COMMENT '有效期开始时间',
  `valid_to` DATETIME DEFAULT NULL COMMENT '有效期结束时间',
  `status` VARCHAR(20) NOT NULL DEFAULT 'draft' COMMENT '状态（draft-草稿，published-已发布，offline-已下线）',
  `publish_time` DATETIME DEFAULT NULL COMMENT '发布时间',
  `create_by` BIGINT DEFAULT NULL COMMENT '创建人ID',
  `create_time` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `update_by` BIGINT DEFAULT NULL COMMENT '更新人ID',
  `update_time` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  `remark` VARCHAR(500) DEFAULT NULL COMMENT '备注',
  PRIMARY KEY (`id`),
  KEY `idx_status` (`status`),
  KEY `idx_target_user` (`target_user`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='系统公告表';
```

---

## 十二、日志相关表

### 12.1 操作日志表 (t_operation_log)

存储用户操作日志。

```sql
CREATE TABLE `t_operation_log` (
  `id` BIGINT NOT NULL COMMENT '主键ID',
  `user_id` BIGINT DEFAULT NULL COMMENT '用户ID',
  `user_name` VARCHAR(50) DEFAULT NULL COMMENT '用户名',
  `operation_type` VARCHAR(50) NOT NULL COMMENT '操作类型',
  `operation_desc` VARCHAR(500) DEFAULT NULL COMMENT '操作描述',
  `request_method` VARCHAR(10) DEFAULT NULL COMMENT '请求方法（GET、POST等）',
  `request_url` VARCHAR(500) DEFAULT NULL COMMENT '请求URL',
  `request_params` TEXT DEFAULT NULL COMMENT '请求参数',
  `response_data` TEXT DEFAULT NULL COMMENT '响应数据',
  `ip_address` VARCHAR(50) DEFAULT NULL COMMENT 'IP地址',
  `user_agent` VARCHAR(500) DEFAULT NULL COMMENT '用户代理',
  `execute_time` INT DEFAULT NULL COMMENT '执行时长（毫秒）',
  `status` VARCHAR(20) NOT NULL COMMENT '状态（success-成功，failed-失败）',
  `error_msg` TEXT DEFAULT NULL COMMENT '错误信息',
  `create_by` BIGINT DEFAULT NULL COMMENT '创建人ID',
  `create_time` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `update_by` BIGINT DEFAULT NULL COMMENT '更新人ID',
  `update_time` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  `remark` VARCHAR(500) DEFAULT NULL COMMENT '备注',
  PRIMARY KEY (`id`),
  KEY `idx_user_id` (`user_id`),
  KEY `idx_operation_type` (`operation_type`),
  KEY `idx_create_time` (`create_time`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='操作日志表';
```

---

## 数据库初始化SQL

### 创建数据库

```sql
CREATE DATABASE IF NOT EXISTS `ticketing_system` 
DEFAULT CHARACTER SET utf8mb4 
COLLATE utf8mb4_unicode_ci;

USE `ticketing_system`;
```

### 初始化基础数据

```sql
-- 插入系统配置
INSERT INTO `t_system_config` (`id`, `config_key`, `config_value`, `config_type`, `config_group`, `config_desc`) VALUES
(1, 'service_fee_rate', '0.05', 'number', 'order', '平台服务费率'),
(2, 'delivery_fee_electronic', '0', 'number', 'order', '电子票配送费'),
(3, 'delivery_fee_express', '15', 'number', 'order', '快递配送费'),
(4, 'order_expire_minutes', '15', 'number', 'order', '订单超时时间（分钟）'),
(5, 'seat_lock_minutes', '5', 'number', 'order', '座位锁定时间（分钟）'),
(6, 'refund_fee_rate', '0.05', 'number', 'order', '退款手续费率');
```

---

## 索引优化建议

### 1. 高频查询字段添加索引
- 订单表的 `user_id`、`status`、`create_time`
- 项目表的 `city_id`、`category_id`、`status`
- 用户表的 `phone`、`status`

### 2. 联合索引
```sql
-- 订单表联合索引
ALTER TABLE `t_order` ADD INDEX `idx_user_status_time` (`user_id`, `status`, `create_time`);

-- 项目表联合索引
ALTER TABLE `t_event` ADD INDEX `idx_city_category_status` (`city_id`, `category_id`, `status`);
```

### 3. 分区表建议
对于数据量大的表（如订单表、日志表），建议按时间分区：

```sql
-- 订单表按月分区示例
ALTER TABLE `t_order` PARTITION BY RANGE (TO_DAYS(create_time)) (
    PARTITION p202401 VALUES LESS THAN (TO_DAYS('2024-02-01')),
    PARTITION p202402 VALUES LESS THAN (TO_DAYS('2024-03-01')),
    -- 继续添加分区...
);
```

---

## 数据库设计总结

### 表统计
- **用户相关**: 6张表
- **商家相关**: 1张表
- **基础数据**: 4张表
- **项目相关**: 5张表
- **订单相关**: 3张表
- **支付相关**: 1张表
- **营销相关**: 3张表
- **运营相关**: 3张表
- **通知相关**: 2张表
- **结算相关**: 2张表
- **系统配置**: 2张表
- **日志相关**: 1张表

**总计**: 33张表

### 设计特点
1. ✅ 所有表都包含标准字段（id、create_by、create_time、update_by、update_time、remark）
2. ✅ 所有字段都有中文注释
3. ✅ 合理的索引设计
4. ✅ 支持软删除和状态管理
5. ✅ 敏感信息加密存储
6. ✅ 支持分库分表扩展

---

**文档版本**: v1.0  
**更新日期**: 2025-11-17  
**编制人**: Kiro AI Assistant
