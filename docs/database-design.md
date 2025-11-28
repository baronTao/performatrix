# 演出票务系统数据库设计文档

## 文档信息

| 项目 | 内容 |
|------|------|
| 项目名称 | 演出票务系统 |
| 数据库类型 | MySQL 8.0 |
| 字符集 | utf8mb4 |
| 排序规则 | utf8mb4_general_ci |
| 文档版本 | v1.2 |
| 编制日期 | 2025-11-28 |

---

## 一、数据库概述

### 1.1 数据库架构

本系统采用单库设计，包含以下模块的数据表：

| 模块 | 表数量 | 说明 |
|------|--------|------|
| 用户模块 | 3 | 用户、观演人、收货地址 |
| 基础配置 | 3 | 城市、频道、场馆 |
| 项目模块 | 4 | 项目、场次、票档、想看列表 |
| 订单模块 | 3 | 订单、订单项、退款 |
| 运营模块 | 3 | Banner、黑名单、系统参数 |
| 系统管理 | 6 | 管理员、角色、权限、关联表、日志 |

**总计：22张表**

### 1.2 命名规范

- 表名：`t_` 前缀 + 模块名/实体名（小写下划线）
- 字段名：小写下划线命名
- 主键：统一使用 `id`，BIGINT 自增
- 时间字段：`created_at`、`updated_at`
- 操作人字段：`created_by`、`updated_by`
- 状态字段：TINYINT 类型，0/1 表示禁用/启用
- 外键：`xxx_id` 格式，不创建物理外键约束

### 1.3 公共字段说明

所有表统一包含以下公共字段：

| 字段名 | 数据类型 | 说明 |
|--------|----------|------|
| remark | VARCHAR(500) | 备注 |
| created_at | DATETIME | 创建时间，默认CURRENT_TIMESTAMP |
| created_by | BIGINT | 创建人ID |
| updated_at | DATETIME | 更新时间，自动更新 |
| updated_by | BIGINT | 更新人ID |

---

## 二、表结构详细设计

### 2.1 用户模块

#### 2.1.1 用户表 (t_user)

```sql
CREATE TABLE t_user (
  id BIGINT PRIMARY KEY AUTO_INCREMENT COMMENT '用户ID',
  phone VARCHAR(11) NOT NULL COMMENT '手机号',
  password VARCHAR(255) COMMENT '密码（BCrypt加密）',
  nickname VARCHAR(50) COMMENT '昵称',
  avatar VARCHAR(255) COMMENT '头像URL',
  gender TINYINT DEFAULT 0 COMMENT '性别：0-保密，1-男，2-女',
  birthday DATE COMMENT '生日',
  real_name VARCHAR(50) COMMENT '真实姓名',
  id_card VARCHAR(255) COMMENT '身份证号（AES加密）',
  real_name_status TINYINT DEFAULT 0 COMMENT '实名状态：0-未认证，1-认证中，2-已认证，3-认证失败',
  real_name_time DATETIME COMMENT '实名认证时间',
  status TINYINT DEFAULT 1 COMMENT '状态：0-禁用，1-启用',
  remark VARCHAR(500) COMMENT '备注',
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  created_by BIGINT COMMENT '创建人ID',
  updated_at DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  updated_by BIGINT COMMENT '更新人ID',
  UNIQUE KEY uk_phone (phone)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='用户表';
```

#### 2.1.2 观演人表 (t_attendee)

```sql
CREATE TABLE t_attendee (
  id BIGINT PRIMARY KEY AUTO_INCREMENT COMMENT '观演人ID',
  user_id BIGINT NOT NULL COMMENT '用户ID',
  real_name VARCHAR(50) NOT NULL COMMENT '真实姓名',
  id_card VARCHAR(255) NOT NULL COMMENT '身份证号（AES加密）',
  phone VARCHAR(11) COMMENT '联系电话',
  is_default TINYINT DEFAULT 0 COMMENT '是否默认：0-否，1-是',
  remark VARCHAR(500) COMMENT '备注',
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  created_by BIGINT COMMENT '创建人ID',
  updated_at DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  updated_by BIGINT COMMENT '更新人ID',
  KEY idx_user_id (user_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='观演人表';
```

#### 2.1.3 收货地址表 (t_address)

```sql
CREATE TABLE t_address (
  id BIGINT PRIMARY KEY AUTO_INCREMENT COMMENT '地址ID',
  user_id BIGINT NOT NULL COMMENT '用户ID',
  receiver_name VARCHAR(50) NOT NULL COMMENT '收货人姓名',
  receiver_phone VARCHAR(11) NOT NULL COMMENT '收货人电话',
  province VARCHAR(50) NOT NULL COMMENT '省份',
  city VARCHAR(50) NOT NULL COMMENT '城市',
  district VARCHAR(50) NOT NULL COMMENT '区县',
  detail_address VARCHAR(255) NOT NULL COMMENT '详细地址',
  is_default TINYINT DEFAULT 0 COMMENT '是否默认：0-否，1-是',
  remark VARCHAR(500) COMMENT '备注',
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  created_by BIGINT COMMENT '创建人ID',
  updated_at DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  updated_by BIGINT COMMENT '更新人ID',
  KEY idx_user_id (user_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='收货地址表';
```

---

### 2.2 基础配置模块

#### 2.2.1 城市表 (t_city)

```sql
CREATE TABLE t_city (
  id BIGINT PRIMARY KEY AUTO_INCREMENT COMMENT '城市ID',
  city_name VARCHAR(50) NOT NULL COMMENT '城市名称',
  city_code VARCHAR(20) NOT NULL COMMENT '城市编码',
  first_letter CHAR(1) NOT NULL COMMENT '首字母',
  is_hot TINYINT DEFAULT 0 COMMENT '是否热门：0-否，1-是',
  sort_order INT DEFAULT 0 COMMENT '排序权重',
  status TINYINT DEFAULT 1 COMMENT '状态：0-禁用，1-启用',
  remark VARCHAR(500) COMMENT '备注',
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  created_by BIGINT COMMENT '创建人ID',
  updated_at DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  updated_by BIGINT COMMENT '更新人ID',
  UNIQUE KEY uk_city_code (city_code),
  KEY idx_first_letter (first_letter),
  KEY idx_is_hot (is_hot),
  KEY idx_status (status)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='城市表';
```

#### 2.2.2 频道表 (t_channel)

```sql
CREATE TABLE t_channel (
  id BIGINT PRIMARY KEY AUTO_INCREMENT COMMENT '频道ID',
  channel_name VARCHAR(50) NOT NULL COMMENT '频道名称',
  channel_code VARCHAR(20) NOT NULL COMMENT '频道编码',
  icon_url VARCHAR(255) NOT NULL COMMENT '频道图标URL',
  sort_order INT DEFAULT 0 COMMENT '排序权重',
  status TINYINT DEFAULT 1 COMMENT '状态：0-禁用，1-启用',
  remark VARCHAR(500) COMMENT '备注',
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  created_by BIGINT COMMENT '创建人ID',
  updated_at DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  updated_by BIGINT COMMENT '更新人ID',
  UNIQUE KEY uk_channel_code (channel_code),
  KEY idx_status_sort (status, sort_order DESC)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='频道表';
```

#### 2.2.3 场馆表 (t_venue)

```sql
CREATE TABLE t_venue (
  id BIGINT PRIMARY KEY AUTO_INCREMENT COMMENT '场馆ID',
  venue_name VARCHAR(100) NOT NULL COMMENT '场馆名称',
  city_id BIGINT NOT NULL COMMENT '所在城市ID',
  address VARCHAR(255) NOT NULL COMMENT '详细地址',
  longitude DECIMAL(10,7) COMMENT '经度',
  latitude DECIMAL(10,7) COMMENT '纬度',
  capacity INT COMMENT '场馆容量',
  venue_image VARCHAR(255) COMMENT '场馆图片URL',
  status TINYINT DEFAULT 1 COMMENT '状态：0-禁用，1-启用',
  remark VARCHAR(500) COMMENT '备注',
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  created_by BIGINT COMMENT '创建人ID',
  updated_at DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  updated_by BIGINT COMMENT '更新人ID',
  KEY idx_city_id (city_id),
  KEY idx_status (status)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='场馆表';
```

---

### 2.3 项目模块

#### 2.3.1 项目表 (t_event)

```sql
CREATE TABLE t_event (
  id BIGINT PRIMARY KEY AUTO_INCREMENT COMMENT '项目ID',
  event_name VARCHAR(200) NOT NULL COMMENT '项目名称',
  channel_id BIGINT NOT NULL COMMENT '频道ID',
  cover_image VARCHAR(255) COMMENT '封面图URL',
  carousel_images TEXT COMMENT '轮播图（JSON数组）',
  description_images TEXT COMMENT '项目详情图片（JSON数组，存储图片URL列表）',
  purchase_notice TEXT COMMENT '购票须知',
  rush_strategy TEXT COMMENT '抢票攻略',
  duration INT COMMENT '演出时长（分钟）',
  artist_info TEXT COMMENT '艺人/演员信息',
  show_wish_count TINYINT DEFAULT 1 COMMENT '是否展示想看人数：0-否，1-是',
  wish_count INT DEFAULT 0 COMMENT '想看人数',
  limit_per_person INT DEFAULT 6 COMMENT '限购数量',
  allow_refund TINYINT DEFAULT 1 COMMENT '是否可退票：0-否，1-是',
  refund_rules TEXT COMMENT '退票规则（JSON格式，包含时间段和手续费配置）',
  sale_start_time DATETIME COMMENT '开售时间',
  sale_end_time DATETIME COMMENT '结束时间',
  require_real_name TINYINT DEFAULT 1 COMMENT '是否实名购票：0-否，1-是',
  status TINYINT DEFAULT 0 COMMENT '状态：0-下架，1-上架',
  remark VARCHAR(500) COMMENT '备注',
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  created_by BIGINT COMMENT '创建人ID',
  updated_at DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  updated_by BIGINT COMMENT '更新人ID',
  KEY idx_channel_id (channel_id),
  KEY idx_status (status),
  KEY idx_sale_start_time (sale_start_time)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='项目表';
```

#### 2.3.2 场次表 (t_session)

```sql
CREATE TABLE t_session (
  id BIGINT PRIMARY KEY AUTO_INCREMENT COMMENT '场次ID',
  event_id BIGINT NOT NULL COMMENT '项目ID',
  venue_id BIGINT NOT NULL COMMENT '场馆ID',
  session_date DATE NOT NULL COMMENT '演出日期',
  start_time TIME NOT NULL COMMENT '开始时间',
  end_time TIME COMMENT '结束时间',
  status TINYINT DEFAULT 1 COMMENT '状态：0-下架，1-上架',
  remark VARCHAR(500) COMMENT '备注',
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  created_by BIGINT COMMENT '创建人ID',
  updated_at DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  updated_by BIGINT COMMENT '更新人ID',
  KEY idx_event_id (event_id),
  KEY idx_venue_id (venue_id),
  KEY idx_session_date (session_date),
  KEY idx_event_date (event_id, session_date)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='场次表';
```

#### 2.3.3 票档表 (t_ticket_tier)

```sql
CREATE TABLE t_ticket_tier (
  id BIGINT PRIMARY KEY AUTO_INCREMENT COMMENT '票档ID',
  session_id BIGINT NOT NULL COMMENT '场次ID',
  tier_name VARCHAR(50) NOT NULL COMMENT '票档名称',
  price DECIMAL(10,2) NOT NULL COMMENT '票档价格',
  total_stock INT NOT NULL COMMENT '总库存',
  available_stock INT NOT NULL COMMENT '可售库存',
  sold_count INT DEFAULT 0 COMMENT '已售数量',
  status TINYINT DEFAULT 1 COMMENT '状态：0-下架，1-上架，2-售罄',
  remark VARCHAR(500) COMMENT '备注',
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  created_by BIGINT COMMENT '创建人ID',
  updated_at DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  updated_by BIGINT COMMENT '更新人ID',
  KEY idx_session_id (session_id),
  KEY idx_status (status)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='票档表';
```

#### 2.3.4 想看列表表 (t_wishlist)

```sql
CREATE TABLE t_wishlist (
  id BIGINT PRIMARY KEY AUTO_INCREMENT COMMENT '想看ID',
  user_id BIGINT NOT NULL COMMENT '用户ID',
  event_id BIGINT NOT NULL COMMENT '项目ID',
  remark VARCHAR(500) COMMENT '备注',
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  created_by BIGINT COMMENT '创建人ID',
  UNIQUE KEY uk_user_event (user_id, event_id),
  KEY idx_user_id (user_id),
  KEY idx_event_id (event_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='想看列表表';
```

---

### 2.4 订单模块

#### 2.4.1 订单表 (t_order)

```sql
CREATE TABLE t_order (
  id BIGINT PRIMARY KEY AUTO_INCREMENT COMMENT '订单ID',
  order_no VARCHAR(32) NOT NULL COMMENT '订单编号',
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
  expire_time DATETIME COMMENT '订单过期时间',
  remark VARCHAR(500) COMMENT '备注',
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  created_by BIGINT COMMENT '创建人ID',
  updated_at DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  updated_by BIGINT COMMENT '更新人ID',
  UNIQUE KEY uk_order_no (order_no),
  KEY idx_user_id (user_id),
  KEY idx_order_status (order_status),
  KEY idx_created_at (created_at),
  KEY idx_expire_time (expire_time)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='订单表';
```

#### 2.4.2 订单项表 (t_order_item)

```sql
CREATE TABLE t_order_item (
  id BIGINT PRIMARY KEY AUTO_INCREMENT COMMENT '订单项ID',
  order_id BIGINT NOT NULL COMMENT '订单ID',
  ticket_tier_id BIGINT NOT NULL COMMENT '票档ID',
  attendee_name VARCHAR(50) NOT NULL COMMENT '观演人姓名',
  attendee_id_card VARCHAR(255) NOT NULL COMMENT '观演人身份证（AES加密）',
  ticket_price DECIMAL(10,2) NOT NULL COMMENT '票价',
  seat_info VARCHAR(100) COMMENT '座位信息（人工录入，如：1排5座）',
  ticket_code VARCHAR(64) COMMENT '电子票码',
  qr_code_url VARCHAR(255) COMMENT '二维码URL',
  ticket_status TINYINT DEFAULT 1 COMMENT '票状态：1-有效，2-已使用，3-已退票，4-已过期',
  use_time DATETIME COMMENT '使用时间',
  remark VARCHAR(500) COMMENT '备注',
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  created_by BIGINT COMMENT '创建人ID',
  updated_at DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  updated_by BIGINT COMMENT '更新人ID',
  UNIQUE KEY uk_ticket_code (ticket_code),
  KEY idx_order_id (order_id),
  KEY idx_ticket_status (ticket_status)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='订单项表';
```

#### 2.4.3 退款表 (t_refund)

```sql
CREATE TABLE t_refund (
  id BIGINT PRIMARY KEY AUTO_INCREMENT COMMENT '退款ID',
  refund_no VARCHAR(32) NOT NULL COMMENT '退款单号',
  order_id BIGINT NOT NULL COMMENT '订单ID',
  refund_amount DECIMAL(10,2) NOT NULL COMMENT '退款金额',
  handling_fee DECIMAL(10,2) DEFAULT 0.00 COMMENT '手续费',
  refund_reason VARCHAR(500) COMMENT '退款原因',
  refund_status TINYINT DEFAULT 1 COMMENT '退款状态：1-待审核，2-已通过，3-已拒绝，4-已退款',
  auditor_id BIGINT COMMENT '审核人ID',
  audit_time DATETIME COMMENT '审核时间',
  audit_remark VARCHAR(500) COMMENT '审核备注',
  refund_time DATETIME COMMENT '退款时间',
  remark VARCHAR(500) COMMENT '备注',
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  created_by BIGINT COMMENT '创建人ID',
  updated_at DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  updated_by BIGINT COMMENT '更新人ID',
  UNIQUE KEY uk_refund_no (refund_no),
  KEY idx_order_id (order_id),
  KEY idx_refund_status (refund_status)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='退款表';
```

---

### 2.5 运营模块

#### 2.5.1 Banner表 (t_banner)

**城市配置说明**：
- `city_ids` 为 `NULL` 或空数组 `[]`：表示该Banner在所有城市展示
- `city_ids` 为 `[1, 2, 3]`：表示该Banner仅在指定城市（ID为1、2、3）展示

**数据示例**：
```json
// 全部城市可见
{"city_ids": null}
或
{"city_ids": []}

// 仅北京、上海、深圳可见
{"city_ids": [1, 2, 3]}
```

```sql
CREATE TABLE t_banner (
  id BIGINT PRIMARY KEY AUTO_INCREMENT COMMENT 'Banner ID',
  banner_name VARCHAR(100) NOT NULL COMMENT 'Banner名称',
  image_url VARCHAR(255) NOT NULL COMMENT 'Banner图片URL',
  jump_type TINYINT DEFAULT 0 COMMENT '跳转类型：0-无跳转，1-项目详情，2-外部链接',
  jump_target VARCHAR(255) COMMENT '跳转目标',
  city_ids TEXT COMMENT '展示城市ID（JSON数组，为空或null表示全部城市）',
  sort_order INT DEFAULT 0 COMMENT '排序权重',
  start_time DATETIME NOT NULL COMMENT '开始时间',
  end_time DATETIME NOT NULL COMMENT '结束时间',
  status TINYINT DEFAULT 1 COMMENT '状态：0-禁用，1-启用',
  remark VARCHAR(500) COMMENT '备注',
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  created_by BIGINT COMMENT '创建人ID',
  updated_at DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  updated_by BIGINT COMMENT '更新人ID',
  KEY idx_status_time (status, start_time, end_time)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='Banner表';
```

#### 2.5.2 黑名单表 (t_blacklist)

```sql
CREATE TABLE t_blacklist (
  id BIGINT PRIMARY KEY AUTO_INCREMENT COMMENT '黑名单ID',
  blacklist_type TINYINT NOT NULL COMMENT '类型：1-手机号，2-身份证号',
  blacklist_value VARCHAR(255) NOT NULL COMMENT '黑名单值（AES加密）',
  reason VARCHAR(500) NOT NULL COMMENT '拉黑原因',
  operator_id BIGINT NOT NULL COMMENT '操作人ID',
  status TINYINT DEFAULT 1 COMMENT '状态：0-已解除，1-生效',
  remark VARCHAR(500) COMMENT '备注',
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  created_by BIGINT COMMENT '创建人ID',
  updated_at DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  updated_by BIGINT COMMENT '更新人ID',
  KEY idx_type_value (blacklist_type, blacklist_value),
  KEY idx_status (status)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='黑名单表';
```

#### 2.5.3 系统参数表 (t_system_config)

```sql
CREATE TABLE t_system_config (
  id BIGINT PRIMARY KEY AUTO_INCREMENT COMMENT '参数ID',
  config_key VARCHAR(100) NOT NULL COMMENT '参数编码',
  config_name VARCHAR(100) NOT NULL COMMENT '参数名称',
  config_type VARCHAR(20) NOT NULL COMMENT '参数类型：text/number/boolean/enum',
  config_value TEXT COMMENT '参数值',
  description VARCHAR(500) COMMENT '参数说明',
  is_editable TINYINT DEFAULT 1 COMMENT '是否可编辑：0-否，1-是',
  remark VARCHAR(500) COMMENT '备注',
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  created_by BIGINT COMMENT '创建人ID',
  updated_at DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  updated_by BIGINT COMMENT '更新人ID',
  UNIQUE KEY uk_config_key (config_key)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='系统参数表';
```

---

### 2.6 系统管理模块

#### 2.6.1 管理员账号表 (t_admin_user)

```sql
CREATE TABLE t_admin_user (
  id BIGINT PRIMARY KEY AUTO_INCREMENT COMMENT '账号ID',
  username VARCHAR(50) NOT NULL COMMENT '账号（手机号）',
  password VARCHAR(255) NOT NULL COMMENT '密码（BCrypt加密）',
  real_name VARCHAR(50) NOT NULL COMMENT '真实姓名',
  status TINYINT DEFAULT 1 COMMENT '状态：0-禁用，1-启用',
  last_login_time DATETIME COMMENT '最后登录时间',
  last_login_ip VARCHAR(50) COMMENT '最后登录IP',
  remark VARCHAR(500) COMMENT '备注',
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  created_by BIGINT COMMENT '创建人ID',
  updated_at DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  updated_by BIGINT COMMENT '更新人ID',
  UNIQUE KEY uk_username (username)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='管理员账号表';
```

#### 2.6.2 角色表 (t_role)

```sql
CREATE TABLE t_role (
  id BIGINT PRIMARY KEY AUTO_INCREMENT COMMENT '角色ID',
  role_code VARCHAR(50) NOT NULL COMMENT '角色编码',
  role_name VARCHAR(50) NOT NULL COMMENT '角色名称',
  description VARCHAR(200) COMMENT '角色描述',
  status TINYINT DEFAULT 1 COMMENT '状态：0-禁用，1-启用',
  remark VARCHAR(500) COMMENT '备注',
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  created_by BIGINT COMMENT '创建人ID',
  updated_at DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  updated_by BIGINT COMMENT '更新人ID',
  UNIQUE KEY uk_role_code (role_code)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='角色表';
```

#### 2.6.3 权限表 (t_permission)

```sql
CREATE TABLE t_permission (
  id BIGINT PRIMARY KEY AUTO_INCREMENT COMMENT '权限ID',
  permission_code VARCHAR(100) NOT NULL COMMENT '权限编码',
  permission_name VARCHAR(100) NOT NULL COMMENT '权限名称',
  permission_type TINYINT NOT NULL COMMENT '权限类型：1-菜单，2-按钮',
  parent_id BIGINT DEFAULT 0 COMMENT '父级权限ID',
  sort_order INT DEFAULT 0 COMMENT '排序',
  remark VARCHAR(500) COMMENT '备注',
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  created_by BIGINT COMMENT '创建人ID',
  UNIQUE KEY uk_permission_code (permission_code),
  KEY idx_parent_id (parent_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='权限表';
```

#### 2.6.4 用户角色关联表 (t_admin_user_role)

```sql
CREATE TABLE t_admin_user_role (
  id BIGINT PRIMARY KEY AUTO_INCREMENT COMMENT 'ID',
  user_id BIGINT NOT NULL COMMENT '用户ID',
  role_id BIGINT NOT NULL COMMENT '角色ID',
  remark VARCHAR(500) COMMENT '备注',
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  created_by BIGINT COMMENT '创建人ID',
  updated_at DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  updated_by BIGINT COMMENT '更新人ID',
  UNIQUE KEY uk_user_role (user_id, role_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='用户角色关联表';
```

#### 2.6.5 角色权限关联表 (t_role_permission)

```sql
CREATE TABLE t_role_permission (
  id BIGINT PRIMARY KEY AUTO_INCREMENT COMMENT 'ID',
  role_id BIGINT NOT NULL COMMENT '角色ID',
  permission_id BIGINT NOT NULL COMMENT '权限ID',
  remark VARCHAR(500) COMMENT '备注',
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  created_by BIGINT COMMENT '创建人ID',
  updated_at DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  updated_by BIGINT COMMENT '更新人ID',
  UNIQUE KEY uk_role_permission (role_id, permission_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='角色权限关联表';
```

#### 2.6.6 操作日志表 (t_operation_log)

```sql
CREATE TABLE t_operation_log (
  id BIGINT PRIMARY KEY AUTO_INCREMENT COMMENT '日志ID',
  operator_id BIGINT COMMENT '操作人ID',
  operator_name VARCHAR(50) COMMENT '操作人姓名',
  operation_type VARCHAR(50) NOT NULL COMMENT '操作类型',
  operation_module VARCHAR(50) NOT NULL COMMENT '操作模块',
  operation_content TEXT COMMENT '操作内容',
  operation_data JSON COMMENT '操作数据',
  operation_ip VARCHAR(50) COMMENT '操作IP',
  operation_result TINYINT DEFAULT 1 COMMENT '操作结果：0-失败，1-成功',
  failure_reason VARCHAR(500) COMMENT '失败原因',
  remark VARCHAR(500) COMMENT '备注',
  operation_time DATETIME DEFAULT CURRENT_TIMESTAMP COMMENT '操作时间',
  KEY idx_operator_id (operator_id),
  KEY idx_operation_type (operation_type),
  KEY idx_operation_time (operation_time)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='操作日志表';
```

---

## 三、初始化数据

### 3.1 初始化管理员账号

```sql
-- 初始化admin管理员账号（密码：admin123，BCrypt加密）
INSERT INTO t_admin_user (username, password, real_name, status, created_by) VALUES 
('admin', '$2a$10$N.zmdr9k7uOCQb376NoUnuTJ8iAt6Z5EHsM8lE9lBOsl7iAt6Z5EH', '系统管理员', 1, 1);
```

### 3.2 初始化角色

```sql
INSERT INTO t_role (role_code, role_name, description, status, created_by) VALUES 
('SUPER_ADMIN', '超级管理员', '拥有所有权限', 1, 1),
('OPERATOR', '运营人员', '负责运营管理', 1, 1),
('CUSTOMER_SERVICE', '客服人员', '负责订单和用户管理', 1, 1);
```

### 3.3 初始化权限

```sql
INSERT INTO t_permission (permission_code, permission_name, permission_type, parent_id, sort_order, created_by) VALUES 
-- 一级菜单
('dashboard', '首页看板', 1, 0, 1, 1),
('event', '项目管理', 1, 0, 2, 1),
('order', '订单管理', 1, 0, 3, 1),
('operation', '运营工具', 1, 0, 4, 1),
('system', '系统管理', 1, 0, 5, 1),
-- 项目管理子菜单
('event:list', '项目列表', 1, 2, 1, 1),
('event:session', '场次管理', 1, 2, 2, 1),
('event:ticket', '票档管理', 1, 2, 3, 1),
('event:venue', '场馆管理', 1, 2, 4, 1),
-- 订单管理子菜单
('order:list', '订单列表', 1, 3, 1, 1),
('order:refund', '退款管理', 1, 3, 2, 1),
('order:sell', '后台售票', 1, 3, 3, 1),
-- 运营工具子菜单
('operation:city', '城市管理', 1, 4, 1, 1),
('operation:channel', '频道管理', 1, 4, 2, 1),
('operation:banner', 'Banner管理', 1, 4, 3, 1),
('operation:blacklist', '黑名单管理', 1, 4, 4, 1),
('operation:config', '系统参数', 1, 4, 5, 1),
-- 系统管理子菜单
('system:user', '账号管理', 1, 5, 1, 1),
('system:role', '角色管理', 1, 5, 2, 1),
('system:log', '操作日志', 1, 5, 3, 1);
```

### 3.4 初始化频道数据

```sql
INSERT INTO t_channel (channel_name, channel_code, icon_url, sort_order, status, created_by) VALUES 
('演唱会', 'CONCERT', '/icons/concert.png', 100, 1, 1),
('话剧歌剧', 'DRAMA', '/icons/drama.png', 90, 1, 1),
('音乐会', 'MUSIC', '/icons/music.png', 80, 1, 1),
('体育赛事', 'SPORTS', '/icons/sports.png', 70, 1, 1),
('儿童亲子', 'KIDS', '/icons/kids.png', 60, 1, 1),
('展览休闲', 'EXHIBITION', '/icons/exhibition.png', 50, 1, 1);
```

### 3.5 初始化系统参数

**参数类型说明**：
- `text`: 文本类型，config_value存储字符串
- `number`: 数字类型，config_value存储数字字符串
- `boolean`: 布尔类型，config_value存储 'true' 或 'false'
- `enum`: 枚举类型，config_value存储JSON格式的枚举配置

**枚举类型数据格式**：
```json
{
  "current": "alipay",
  "options": [
    {"value": "wechat", "label": "微信支付"},
    {"value": "alipay", "label": "支付宝"},
    {"value": "unionpay", "label": "银联支付"}
  ]
}
```

```sql
INSERT INTO t_system_config (config_key, config_name, config_type, config_value, description, is_editable, created_by) VALUES 
('order_timeout_minutes', '订单超时时间（分钟）', 'number', '15', '未支付订单自动取消时间', 1, 1),
('default_limit_per_person', '默认限购数量', 'number', '6', '每人每场次默认限购票数', 1, 1),
('sms_sign', '短信签名', 'text', '票务系统', '短信发送签名', 1, 1),
('customer_service_phone', '客服电话', 'text', '400-123-4567', 'C端展示的客服电话', 1, 1),
('enable_real_name', '是否开启实名认证', 'boolean', 'true', '全局实名认证开关', 1, 1),
('default_payment_method', '默认支付方式', 'enum', '{"current":"wechat","options":[{"value":"wechat","label":"微信支付"},{"value":"alipay","label":"支付宝"}]}', '系统默认支付方式', 1, 1),
('order_status_options', '订单状态配置', 'enum', '{"current":"all","options":[{"value":"1","label":"待支付"},{"value":"2","label":"已支付"},{"value":"3","label":"已取消"},{"value":"4","label":"已退款"},{"value":"5","label":"已完成"}]}', '订单状态枚举值', 0, 1);
```

---

## 四、表汇总

| 序号 | 表名 | 中文名 | 模块 |
|------|------|--------|------|
| 1 | t_user | 用户表 | 用户模块 |
| 2 | t_attendee | 观演人表 | 用户模块 |
| 3 | t_address | 收货地址表 | 用户模块 |
| 4 | t_city | 城市表 | 基础配置 |
| 5 | t_channel | 频道表 | 基础配置 |
| 6 | t_venue | 场馆表 | 基础配置 |
| 7 | t_event | 项目表 | 项目模块 |
| 8 | t_session | 场次表 | 项目模块 |
| 9 | t_ticket_tier | 票档表 | 项目模块 |
| 10 | t_wishlist | 想看列表表 | 项目模块 |
| 11 | t_order | 订单表 | 订单模块 |
| 12 | t_order_item | 订单项表 | 订单模块 |
| 13 | t_refund | 退款表 | 订单模块 |
| 14 | t_banner | Banner表 | 运营模块 |
| 15 | t_blacklist | 黑名单表 | 运营模块 |
| 16 | t_system_config | 系统参数表 | 运营模块 |
| 17 | t_admin_user | 管理员账号表 | 系统管理 |
| 18 | t_role | 角色表 | 系统管理 |
| 19 | t_permission | 权限表 | 系统管理 |
| 20 | t_admin_user_role | 用户角色关联表 | 系统管理 |
| 21 | t_role_permission | 角色权限关联表 | 系统管理 |
| 22 | t_operation_log | 操作日志表 | 系统管理 |

---

## 五、表关系说明

| 主表 | 从表 | 关系 | 关联字段 |
|------|------|------|----------|
| t_user | t_attendee | 1:N | user_id |
| t_user | t_address | 1:N | user_id |
| t_user | t_wishlist | 1:N | user_id |
| t_user | t_order | 1:N | user_id |
| t_city | t_venue | 1:N | city_id |
| t_channel | t_event | 1:N | channel_id |
| t_event | t_session | 1:N | event_id |
| t_event | t_wishlist | 1:N | event_id |
| t_venue | t_session | 1:N | venue_id |
| t_session | t_ticket_tier | 1:N | session_id |
| t_order | t_order_item | 1:N | order_id |
| t_order | t_refund | 1:1 | order_id |
| t_admin_user | t_admin_user_role | 1:N | user_id |
| t_role | t_admin_user_role | 1:N | role_id |
| t_role | t_role_permission | 1:N | role_id |
| t_permission | t_role_permission | 1:N | permission_id |

---

## 六、数据安全设计

### 6.1 敏感数据加密

| 字段 | 加密方式 | 说明 |
|------|----------|------|
| t_user.password | BCrypt | 密码单向加密 |
| t_user.id_card | AES-256 | 身份证号对称加密 |
| t_attendee.id_card | AES-256 | 观演人身份证对称加密 |
| t_order_item.attendee_id_card | AES-256 | 订单观演人身份证对称加密 |
| t_blacklist.blacklist_value | AES-256 | 黑名单值对称加密 |
| t_admin_user.password | BCrypt | 管理员密码单向加密 |

### 6.2 数据备份策略

- **全量备份**：每天凌晨2点执行
- **增量备份**：每小时执行
- **备份保留**：全量备份保留30天，增量备份保留7天

---

**文档版本**: v1.2  
**编制日期**: 2025-11-28  
**编制人**: Kiro AI Assistant  
**更新说明**: 所有表补充remark备注字段，关联表补充updated_at和updated_by字段
