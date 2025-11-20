# 演出票务系统设计文档（更新版）

## 概述

本文档描述演出票务系统的技术架构和设计方案。系统采用微服务架构，前后端分离，支持高并发场景下的票务交易。整体架构包括C端应用（PC Web/H5/App）、B端管理平台、运营后台、主办后台，以及支撑这些应用的后端服务集群。

## 技术栈

### 前端技术栈
- **C端 Web/H5/App**: uni-app框架
  - **核心技术**: Vue 3 + TypeScript + uni-app
  - **UI组件库**: uni-ui
  - **一套代码多端运行**: H5、Android App、iOS App、PC Web
  - **状态管理**: Pinia
  - **数据请求**: uni.request封装
  
- **B端/运营后台/主办后台**: Vue 3 + TypeScript + Element Plus + Vite
  - **状态管理**: Pinia
  - **数据请求**: Axios
  - **图表库**: ECharts
  - **富文本编辑器**: TinyMCE / WangEditor

### 后端技术栈
- **开发框架**: Spring Boot 3.x + Spring Cloud Alibaba
- **微服务框架**: 
  - Spring Cloud Gateway (API网关)
  - Nacos (服务注册与配置中心)
  - Sentinel (流量控制、熔断降级)
- **ORM框架**: MyBatis-Plus
- **数据库**: MySQL 8.0 + Redis 7.x
- **搜索引擎**: Elasticsearch 8.x
- **消息队列**: RocketMQ
- **文件存储**: 阿里云OSS / MinIO
- **支付**: 支付宝SDK + 微信支付SDK
- **短信**: 阿里云短信服务
- **安全认证**: Spring Security + JWT

### 基础设施
- **容器化**: Docker + Kubernetes
- **监控**: Prometheus + Grafana
- **日志**: ELK Stack
- **链路追踪**: SkyWalking

---

## 系统架构

### 整体架构图

```
┌─────────────────────────────────────────────────────────┐
│                      客户端层                            │
│  C端Web/H5/App  │  B端管理  │  运营后台  │  主办后台    │
└─────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────┐
│                    API网关层                             │
│              Spring Cloud Gateway                        │
└─────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────┐
│                   应用服务层                             │
│  用户服务 │ 项目服务 │ 订单服务 │ 支付服务 │ 库存服务  │
│  营销服务 │ 搜索服务 │ 通知服务                         │
└─────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────┐
│                     数据层                               │
│    MySQL    │    Redis    │  Elasticsearch  │   OSS     │
└─────────────────────────────────────────────────────────┘
```

### 微服务划分

1. **用户服务**: 用户注册、登录、个人信息管理、实名认证
2. **项目服务**: 演出项目、场次、票档、座位图管理
3. **订单服务**: 订单创建、支付、退款、订单查询
4. **支付服务**: 第三方支付集成、支付回调处理
5. **库存服务**: 票档库存、座位分配、库存锁定、防超卖
6. **营销服务**: 优惠券、促销活动管理
7. **搜索服务**: 项目搜索、筛选功能
8. **通知服务**: 短信、邮件、站内消息发送

---

## 核心模块设计

### 1. 城市数据筛选设计

**设计原则**: 城市作为主要数据筛选条件，而非数据隔离标准

**核心概念**：
- **数据隔离**：商家维度（商家A看不到商家B的数据）
- **数据筛选**：城市维度（用户选择城市后过滤演出场次）

**实现方案**:

1. **数据库设计**: 
   - 项目表不直接关联城市（支持多城市巡演）
   - 场次表关联城市（一个项目可以有多个城市的场次）
   - 场馆表关联城市（场馆属于特定城市）
   
2. **查询过滤**: 
   - C端查询：通过场次的 `city_id` 过滤演出
   - B端查询：通过 `merchant_id` 隔离商家数据
   
3. **缓存策略**: Redis缓存按城市维度分片存储热门数据

4. **前端状态**: 使用Pinia全局状态管理当前选定城市

**查询示例**：

```java
// C端：用户选择城市后查询演出
@Service
public class EventService {
    
    public List<EventVO> getEventsByCity(Long cityId) {
        // 通过场次的city_id筛选该城市的演出
        return eventMapper.selectEventsByCity(cityId);
    }
}

// Mapper SQL
SELECT e.*, s.*, v.*
FROM t_event e
JOIN t_session s ON e.id = s.event_id
JOIN t_venue v ON s.venue_id = v.id
WHERE s.city_id = #{cityId}
  AND s.status = 'on_sale'
  AND s.start_time > NOW()
ORDER BY s.start_time;

// B端：商家管理自己的项目（商家隔离）
@Service
public class MerchantEventService {
    
    public List<EventVO> getMerchantEvents(Long merchantId) {
        // 通过merchant_id隔离商家数据
        return eventMapper.selectByMerchantId(merchantId);
    }
}

// Mapper SQL - 查询商家的项目及其覆盖的城市
SELECT e.*, 
       COUNT(DISTINCT s.city_id) as city_count,
       GROUP_CONCAT(DISTINCT c.name) as cities
FROM t_event e
LEFT JOIN t_session s ON e.id = s.event_id
LEFT JOIN t_city c ON s.city_id = c.id
WHERE e.merchant_id = #{merchantId}
GROUP BY e.id;
```

**前端状态管理**：

```typescript
// 城市状态管理
export const useCityStore = defineStore('city', {
  state: () => ({
    currentCity: null as City | null,
    locationCity: null as City | null,
    cities: [] as City[]
  }),
  
  actions: {
    setCity(city: City) {
      this.currentCity = city
      localStorage.setItem('cityId', city.id)
      // 触发全局数据刷新
      this.refreshAllData()
    }
  }
})
```

---

### 2. 用户模块

#### 数据模型

```sql
-- 用户表
CREATE TABLE t_user (
  id BIGINT PRIMARY KEY,
  phone VARCHAR(20) NOT NULL UNIQUE,
  password VARCHAR(255),
  nickname VARCHAR(50),
  avatar VARCHAR(255),
  city_id BIGINT,
  status VARCHAR(20) DEFAULT 'active',
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  INDEX idx_phone (phone),
  INDEX idx_city (city_id)
);

-- 实名信息表
CREATE TABLE t_real_name_info (
  id BIGINT PRIMARY KEY,
  user_id BIGINT NOT NULL,
  real_name VARCHAR(50) NOT NULL,
  id_card VARCHAR(255) NOT NULL, -- 加密存储
  is_default BOOLEAN DEFAULT FALSE,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  INDEX idx_user (user_id)
);

-- 收货地址表
CREATE TABLE t_address (
  id BIGINT PRIMARY KEY,
  user_id BIGINT NOT NULL,
  receiver_name VARCHAR(50) NOT NULL,
  receiver_phone VARCHAR(20) NOT NULL,
  province VARCHAR(50),
  city VARCHAR(50),
  district VARCHAR(50),
  detail_address VARCHAR(255),
  is_default BOOLEAN DEFAULT FALSE,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  INDEX idx_user (user_id)
);
```

#### 核心接口

- `POST /api/user/register` - 用户注册
- `POST /api/user/login` - 用户登录
- `POST /api/user/send-sms` - 发送验证码
- `GET /api/user/profile` - 获取用户信息
- `PUT /api/user/profile` - 更新用户信息
- `GET /api/user/addresses` - 获取收货地址列表
- `POST /api/user/addresses` - 添加收货地址
- `GET /api/user/real-names` - 获取实名信息列表
- `POST /api/user/real-names` - 添加实名信息

---

### 3. 项目模块

#### 数据模型

```sql
-- 项目表（不直接关联城市，支持多城市巡演）
CREATE TABLE t_event (
  id BIGINT PRIMARY KEY,
  merchant_id BIGINT NOT NULL, -- 商家隔离
  name VARCHAR(255) NOT NULL,
  category_id BIGINT NOT NULL,
  cover_image VARCHAR(255),
  images TEXT, -- JSON数组
  description TEXT,
  notice TEXT,
  duration INT,
  min_price DECIMAL(10,2),
  max_price DECIMAL(10,2),
  status VARCHAR(20) DEFAULT 'draft',
  tags TEXT, -- JSON数组
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  INDEX idx_merchant (merchant_id), -- 商家隔离索引
  INDEX idx_category (category_id),
  INDEX idx_status (status)
);

-- 场次表（关联城市，一个项目可以有多个城市的场次）
CREATE TABLE t_session (
  id BIGINT PRIMARY KEY,
  event_id BIGINT NOT NULL,
  city_id BIGINT NOT NULL, -- 城市筛选条件
  venue_id BIGINT NOT NULL,
  start_time DATETIME NOT NULL,
  end_time DATETIME,
  status VARCHAR(20) DEFAULT 'on_sale',
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  INDEX idx_event (event_id),
  INDEX idx_city (city_id), -- 城市筛选索引
  INDEX idx_start_time (start_time),
  INDEX idx_city_time (city_id, start_time) -- 组合索引优化查询
);

-- 票档表
CREATE TABLE t_ticket_tier (
  id BIGINT PRIMARY KEY,
  session_id BIGINT NOT NULL,
  name VARCHAR(100) NOT NULL,
  price DECIMAL(10,2) NOT NULL,
  total_stock INT NOT NULL,
  available_stock INT NOT NULL,
  seat_area VARCHAR(100),
  status VARCHAR(20) DEFAULT 'on_sale',
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  INDEX idx_session (session_id)
);

-- 座位表
CREATE TABLE t_seat (
  id BIGINT PRIMARY KEY,
  session_id BIGINT NOT NULL,
  ticket_tier_id BIGINT NOT NULL,
  seat_map_id BIGINT,
  row_num VARCHAR(10),
  seat_num VARCHAR(10),
  status VARCHAR(20) DEFAULT 'available',
  locked_until DATETIME,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  INDEX idx_session (session_id),
  INDEX idx_tier (ticket_tier_id),
  INDEX idx_status (status)
);

-- 场馆表（场馆属于特定城市）
CREATE TABLE t_venue (
  id BIGINT PRIMARY KEY,
  name VARCHAR(255) NOT NULL,
  city_id BIGINT NOT NULL, -- 场馆所在城市
  address VARCHAR(255),
  latitude DECIMAL(10,6),
  longitude DECIMAL(10,6),
  capacity INT,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  INDEX idx_city (city_id)
);

-- 座位图表
CREATE TABLE t_seat_map (
  id BIGINT PRIMARY KEY,
  venue_id BIGINT NOT NULL,
  name VARCHAR(255),
  config TEXT, -- JSON配置
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  INDEX idx_venue (venue_id)
);
```

#### 核心接口

- `GET /api/events` - 获取项目列表（支持城市筛选）
- `GET /api/events/:id` - 获取项目详情
- `GET /api/events/:id/sessions` - 获取项目场次列表
- `GET /api/sessions/:id/ticket-tiers` - 获取场次票档列表
- `POST /api/events` - 创建项目（B端）
- `PUT /api/events/:id` - 更新项目（B端）
- `POST /api/sessions` - 创建场次（B端）
- `POST /api/ticket-tiers` - 创建票档（B端）

---

### 4. 搜索模块

#### Elasticsearch索引设计

```json
{
  "mappings": {
    "properties": {
      "id": { "type": "long" },
      "name": { 
        "type": "text",
        "analyzer": "ik_max_word",
        "search_analyzer": "ik_smart"
      },
      "category_id": { "type": "long" },
      "category_name": { "type": "keyword" },
      "city_id": { "type": "long" },
      "city_name": { "type": "keyword" },
      "venue_name": { 
        "type": "text",
        "analyzer": "ik_max_word"
      },
      "min_price": { "type": "double" },
      "max_price": { "type": "double" },
      "start_time": { "type": "date" },
      "tags": { "type": "keyword" },
      "status": { "type": "keyword" },
      "sales_volume": { "type": "integer" },
      "created_at": { "type": "date" }
    }
  }
}
```

#### 搜索功能实现

```java
@Service
public class SearchService {
    
    @Autowired
    private ElasticsearchRestTemplate elasticsearchTemplate;
    
    public Page<EventSearchResult> searchEvents(SearchRequest request) {
        NativeSearchQueryBuilder queryBuilder = new NativeSearchQueryBuilder();
        
        // 城市过滤（必须条件）
        queryBuilder.withFilter(QueryBuilders.termQuery("city_id", request.getCityId()));
        
        // 关键词搜索
        if (StringUtils.hasText(request.getKeyword())) {
            queryBuilder.withQuery(
                QueryBuilders.multiMatchQuery(request.getKeyword())
                    .field("name", 3.0f)  // 项目名称权重最高
                    .field("venue_name", 2.0f)
                    .field("category_name", 1.0f)
            );
        }
        
        // 日期筛选
        if (request.getStartDate() != null && request.getEndDate() != null) {
            queryBuilder.withFilter(
                QueryBuilders.rangeQuery("start_time")
                    .gte(request.getStartDate())
                    .lte(request.getEndDate())
            );
        }
        
        // 价格筛选
        if (request.getMinPrice() != null || request.getMaxPrice() != null) {
            RangeQueryBuilder priceQuery = QueryBuilders.rangeQuery("min_price");
            if (request.getMinPrice() != null) {
                priceQuery.gte(request.getMinPrice());
            }
            if (request.getMaxPrice() != null) {
                priceQuery.lte(request.getMaxPrice());
            }
            queryBuilder.withFilter(priceQuery);
        }
        
        // 排序
        if ("hot".equals(request.getSortBy())) {
            queryBuilder.withSort(SortBuilders.fieldSort("sales_volume").order(SortOrder.DESC));
        } else if ("price_asc".equals(request.getSortBy())) {
            queryBuilder.withSort(SortBuilders.fieldSort("min_price").order(SortOrder.ASC));
        }
        
        // 分页
        queryBuilder.withPageable(PageRequest.of(request.getPage(), request.getSize()));
        
        return elasticsearchTemplate.queryForPage(queryBuilder.build(), EventSearchResult.class);
    }
}
```

---

### 5. 库存模块

#### 防超卖设计

**核心策略**:
1. Redis分布式锁
2. Lua脚本原子操作
3. 库存预扣机制
4. 定时释放过期锁定

```lua
-- Redis Lua脚本：原子性库存扣减
local key = KEYS[1]
local quantity = tonumber(ARGV[1])
local ttl = tonumber(ARGV[2])

local current = tonumber(redis.call('get', key) or 0)
if current >= quantity then
    redis.call('decrby', key, quantity)
    redis.call('expire', key, ttl)
    return 1
else
    return 0
end
```

```java
@Service
public class InventoryService {
    
    @Autowired
    private RedisTemplate<String, String> redisTemplate;
    
    @Autowired
    private RedisScript<Long> lockInventoryScript;
    
    /**
     * 锁定库存
     */
    public boolean lockInventory(Long ticketTierId, Integer quantity) {
        String key = "inventory:" + ticketTierId;
        Long result = redisTemplate.execute(
            lockInventoryScript,
            Collections.singletonList(key),
            quantity.toString(),
            "900" // 15分钟TTL
        );
        return result != null && result == 1;
    }
    
    /**
     * 释放库存
     */
    public void releaseInventory(Long ticketTierId, Integer quantity) {
        String key = "inventory:" + ticketTierId;
        redisTemplate.opsForValue().increment(key, quantity);
    }
    
    /**
     * 确认扣减库存
     */
    @Transactional
    public void confirmInventory(Long ticketTierId, Integer quantity) {
        // 从数据库扣减库存
        ticketTierMapper.decreaseStock(ticketTierId, quantity);
    }
}
```

#### 座位自动分配算法

```java
@Service
public class SeatAllocationService {
    
    /**
     * 随机分配座位
     */
    public List<Seat> allocateSeats(Long sessionId, Long ticketTierId, Integer quantity) {
        // 1. 查询该票档下的可用座位
        List<Seat> availableSeats = seatMapper.findAvailableSeats(sessionId, ticketTierId);
        
        if (availableSeats.size() < quantity) {
            throw new BusinessException("可用座位不足");
        }
        
        // 2. 随机选择座位
        Collections.shuffle(availableSeats);
        List<Seat> selectedSeats = availableSeats.subList(0, quantity);
        
        // 3. 锁定座位（5分钟）
        LocalDateTime lockedUntil = LocalDateTime.now().plusMinutes(5);
        for (Seat seat : selectedSeats) {
            seat.setStatus("locked");
            seat.setLockedUntil(lockedUntil);
        }
        seatMapper.batchUpdate(selectedSeats);
        
        return selectedSeats;
    }
}
```

---

### 6. 订单模块

#### 订单创建流程

```java
@Service
public class OrderService {
    
    @Autowired
    private InventoryService inventoryService;
    
    @Autowired
    private SeatAllocationService seatAllocationService;
    
    @Transactional
    public Order createOrder(OrderCreateRequest request) {
        // 1. 验证库存
        if (!inventoryService.lockInventory(request.getTicketTierId(), request.getQuantity())) {
            throw new BusinessException("库存不足");
        }
        
        try {
            // 2. 分配座位
            List<Seat> seats = seatAllocationService.allocateSeats(
                request.getSessionId(),
                request.getTicketTierId(),
                request.getQuantity()
            );
            
            // 3. 计算金额
            BigDecimal totalAmount = calculateAmount(request);
            
            // 4. 创建订单
            Order order = new Order();
            order.setOrderNo(generateOrderNo());
            order.setUserId(request.getUserId());
            order.setEventId(request.getEventId());
            order.setSessionId(request.getSessionId());
            order.setTotalAmount(totalAmount);
            order.setStatus("pending");
            orderMapper.insert(order);
            
            // 5. 创建订单项
            for (Seat seat : seats) {
                OrderItem item = new OrderItem();
                item.setOrderId(order.getId());
                item.setTicketTierId(request.getTicketTierId());
                item.setSeatId(seat.getId());
                item.setPrice(request.getPrice());
                item.setTicketCode(generateTicketCode());
                orderItemMapper.insert(item);
            }
            
            // 6. 设置订单超时（15分钟）
            scheduleOrderTimeout(order.getId(), 15);
            
            return order;
            
        } catch (Exception e) {
            // 失败时释放库存
            inventoryService.releaseInventory(request.getTicketTierId(), request.getQuantity());
            throw e;
        }
    }
}
```

---

### 7. 前端架构设计

#### uni-app多端适配

**PC Web和移动端导航差异处理**:

```vue
<!-- App.vue -->
<template>
  <view class="app">
    <!-- 移动端：底部导航 -->
    <view v-if="isMobile" class="mobile-layout">
      <view class="content">
        <router-view />
      </view>
      <tabbar />
    </view>
    
    <!-- PC端：顶部导航 -->
    <view v-else class="pc-layout">
      <pc-header />
      <view class="content">
        <router-view />
      </view>
      <pc-footer />
    </view>
  </view>
</template>

<script setup lang="ts">
import { ref, onMounted } from 'vue'

const isMobile = ref(true)

onMounted(() => {
  const systemInfo = uni.getSystemInfoSync()
  isMobile.value = systemInfo.windowWidth < 768
})
</script>
```

#### 城市状态管理

```typescript
// stores/city.ts
import { defineStore } from 'pinia'

export const useCityStore = defineStore('city', {
  state: () => ({
    currentCity: null as City | null,
    locationCity: null as City | null,
    cities: [] as City[],
    hotCities: [] as City[]
  }),
  
  getters: {
    currentCityId: (state) => state.currentCity?.id,
    currentCityName: (state) => state.currentCity?.name || '选择城市'
  },
  
  actions: {
    // 设置当前城市
    setCity(city: City) {
      this.currentCity = city
      uni.setStorageSync('cityId', city.id)
      uni.setStorageSync('cityName', city.name)
      
      // 触发全局事件，通知其他页面刷新数据
      uni.$emit('cityChanged', city)
    },
    
    // 获取城市列表
    async fetchCities() {
      const res = await getCitiesApi()
      this.cities = res.cities
      this.hotCities = res.hotCities
    },
    
    // 定位当前城市
    async locateCity() {
      try {
        const location = await getLocation()
        const city = await getCityByLocation(location)
        this.locationCity = city
        
        // 如果用户未选择城市，使用定位城市
        if (!this.currentCity) {
          this.setCity(city)
        }
      } catch (error) {
        console.error('定位失败', error)
      }
    }
  }
})
```

---

## 运营后台设计

### 用户管理

**功能模块**:
- 用户列表（增删改查）
- 用户状态管理（启用/禁用/冻结）
- 用户订单查询

**页面设计**:
```vue
<!-- 用户列表页 -->
<template>
  <div class="user-management">
    <!-- 搜索栏 -->
    <el-form inline>
      <el-form-item label="手机号">
        <el-input v-model="searchForm.phone" />
      </el-form-item>
      <el-form-item label="状态">
        <el-select v-model="searchForm.status">
          <el-option label="全部" value="" />
          <el-option label="正常" value="active" />
          <el-option label="禁用" value="disabled" />
          <el-option label="冻结" value="frozen" />
        </el-select>
      </el-form-item>
      <el-form-item>
        <el-button type="primary" @click="search">搜索</el-button>
        <el-button @click="reset">重置</el-button>
      </el-form-item>
    </el-form>
    
    <!-- 操作栏 -->
    <el-button type="primary" @click="handleAdd">新增用户</el-button>
    
    <!-- 表格 -->
    <el-table :data="userList" border>
      <el-table-column prop="id" label="ID" width="80" />
      <el-table-column prop="phone" label="手机号" width="120" />
      <el-table-column prop="nickname" label="昵称" />
      <el-table-column prop="status" label="状态" width="100">
        <template #default="{ row }">
          <el-tag :type="getStatusType(row.status)">
            {{ getStatusText(row.status) }}
          </el-tag>
        </template>
      </el-table-column>
      <el-table-column prop="createdAt" label="注册时间" width="180" />
      <el-table-column label="操作" width="250" fixed="right">
        <template #default="{ row }">
          <el-button size="small" @click="handleView(row)">查看</el-button>
          <el-button size="small" @click="handleEdit(row)">编辑</el-button>
          <el-button size="small" @click="handleStatus(row)">
            {{ row.status === 'active' ? '禁用' : '启用' }}
          </el-button>
          <el-button size="small" type="danger" @click="handleDelete(row)">删除</el-button>
        </template>
      </el-table-column>
    </el-table>
    
    <!-- 分页 -->
    <el-pagination
      v-model:current-page="pagination.page"
      v-model:page-size="pagination.size"
      :total="pagination.total"
      @current-change="fetchUsers"
    />
  </div>
</template>
```

### 供应商管理

**功能模块**:
- 供应商列表（增删改查）
- 供应商资质管理（上传/查看/审核）
- 供应商状态管理（启用/禁用/待审核）
- 供应商项目查询

---

## 性能优化设计

### 1. 缓存策略

```java
@Service
public class EventCacheService {
    
    @Autowired
    private RedisTemplate<String, Object> redisTemplate;
    
    /**
     * 获取项目详情（带缓存）
     */
    @Cacheable(value = "event", key = "#eventId")
    public Event getEventById(Long eventId) {
        return eventMapper.selectById(eventId);
    }
    
    /**
     * 获取热门项目列表（按城市缓存）
     */
    public List<Event> getHotEvents(Long cityId) {
        String key = "hot_events:" + cityId;
        List<Event> events = (List<Event>) redisTemplate.opsForValue().get(key);
        
        if (events == null) {
            events = eventMapper.selectHotEvents(cityId);
            redisTemplate.opsForValue().set(key, events, 10, TimeUnit.MINUTES);
        }
        
        return events;
    }
}
```

### 2. 数据库优化

- 读写分离：主库写，从库读
- 分表策略：订单表按月分表
- 索引优化：为常用查询字段添加索引
- 慢查询优化：定期分析慢查询日志

---

**文档版本**: v2.0  
**编制日期**: 2025-11-18  
**编制人**: Kiro AI Assistant  
**审核状态**: 待审核
