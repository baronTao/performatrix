# 演出票务系统API接口清单

## 文档信息
- **文件名**: apifox-openapi-complete.yaml
- **文件大小**: 约90KB
- **接口总数**: 80+
- **数据模型**: 40+
- **格式**: OpenAPI 3.0.3 (YAML)

---

## 一、用户服务 (10个接口)

### 1.1 认证相关
| 方法 | 路径 | 说明 | 需要认证 |
|------|------|------|---------|
| POST | /api/user/send-sms | 发送短信验证码 | ❌ |
| POST | /api/user/register | 用户注册 | ❌ |
| POST | /api/user/login | 用户登录 | ❌ |
| POST | /api/user/oauth/wechat | 微信登录 | ❌ |

### 1.2 用户信息
| 方法 | 路径 | 说明 | 需要认证 |
|------|------|------|---------|
| GET | /api/user/profile | 获取用户信息 | ✅ |
| PUT | /api/user/profile | 更新用户信息 | ✅ |

### 1.3 地址管理
| 方法 | 路径 | 说明 | 需要认证 |
|------|------|------|---------|
| GET | /api/user/addresses | 获取收货地址列表 | ✅ |
| POST | /api/user/addresses | 添加收货地址 | ✅ |

### 1.4 实名信息
| 方法 | 路径 | 说明 | 需要认证 |
|------|------|------|---------|
| GET | /api/user/real-names | 获取实名信息列表 | ✅ |
| POST | /api/user/real-names | 添加实名信息 | ✅ |

---

## 二、项目服务 (15个接口)

### 2.1 基础数据
| 方法 | 路径 | 说明 | 需要认证 |
|------|------|------|---------|
| GET | /api/cities | 获取城市列表 | ❌ |
| GET | /api/categories | 获取类目列表 | ❌ |

### 2.2 项目管理
| 方法 | 路径 | 说明 | 需要认证 |
|------|------|------|---------|
| GET | /api/events | 获取项目列表 | ❌ |
| POST | /api/events | 创建项目（B端） | ✅ |
| GET | /api/events/{eventId} | 获取项目详情 | ❌ |
| PUT | /api/events/{eventId} | 更新项目（B端） | ✅ |

### 2.3 场次管理
| 方法 | 路径 | 说明 | 需要认证 |
|------|------|------|---------|
| GET | /api/events/{eventId}/sessions | 获取项目场次列表 | ❌ |
| POST | /api/events/{eventId}/sessions | 创建场次（B端） | ✅ |

### 2.4 票档管理
| 方法 | 路径 | 说明 | 需要认证 |
|------|------|------|---------|
| GET | /api/sessions/{sessionId}/ticket-tiers | 获取场次票档列表 | ❌ |
| POST | /api/sessions/{sessionId}/ticket-tiers | 创建票档（B端） | ✅ |

### 2.5 想看功能
| 方法 | 路径 | 说明 | 需要认证 |
|------|------|------|---------|
| POST | /api/events/{eventId}/wishlist | 添加想看 | ✅ |
| DELETE | /api/events/{eventId}/wishlist | 取消想看 | ✅ |
| GET | /api/user/wishlist | 获取想看列表 | ✅ |

---

## 三、订单服务 (10个接口)

### 3.1 订单管理
| 方法 | 路径 | 说明 | 需要认证 |
|------|------|------|---------|
| GET | /api/orders | 获取订单列表 | ✅ |
| POST | /api/orders | 创建订单 | ✅ |
| GET | /api/orders/{orderId} | 获取订单详情 | ✅ |
| POST | /api/orders/{orderId}/cancel | 取消订单 | ✅ |

### 3.2 退款管理
| 方法 | 路径 | 说明 | 需要认证 |
|------|------|------|---------|
| POST | /api/refunds | 申请退款 | ✅ |
| GET | /api/refunds/{refundId} | 获取退款详情 | ✅ |

### 3.3 电子票
| 方法 | 路径 | 说明 | 需要认证 |
|------|------|------|---------|
| GET | /api/tickets | 获取电子票列表（票夹） | ✅ |

---

## 四、支付服务 (4个接口)

| 方法 | 路径 | 说明 | 需要认证 |
|------|------|------|---------|
| POST | /api/payment/create | 创建支付订单 | ✅ |
| GET | /api/payment/{paymentId}/status | 查询支付状态 | ✅ |
| POST | /api/payment/alipay/callback | 支付宝支付回调 | ❌ |
| POST | /api/payment/wechat/callback | 微信支付回调 | ❌ |

---

## 五、库存服务 (5个接口)

### 5.1 库存管理
| 方法 | 路径 | 说明 | 需要认证 |
|------|------|------|---------|
| POST | /api/inventory/check | 检查库存可用性 | ❌ |

### 5.2 座位管理
| 方法 | 路径 | 说明 | 需要认证 |
|------|------|------|---------|
| POST | /api/seats/allocate | 分配座位 | ✅ |
| POST | /api/seats/lock | 锁定座位 | ✅ |
| POST | /api/seats/unlock | 释放座位 | ✅ |

---

## 六、营销服务 (4个接口)

### 6.1 优惠券
| 方法 | 路径 | 说明 | 需要认证 |
|------|------|------|---------|
| GET | /api/coupons | 获取可领取优惠券列表 | ❌ |
| POST | /api/coupons/{couponId}/receive | 领取优惠券 | ✅ |
| GET | /api/user/coupons | 获取用户优惠券列表 | ✅ |

### 6.2 促销活动
| 方法 | 路径 | 说明 | 需要认证 |
|------|------|------|---------|
| GET | /api/promotions/applicable | 获取适用的促销活动 | ❌ |

---

## 七、搜索服务 (3个接口)

| 方法 | 路径 | 说明 | 需要认证 |
|------|------|------|---------|
| GET | /api/search/events | 搜索项目 | ❌ |
| GET | /api/search/suggest | 搜索建议 | ❌ |
| GET | /api/search/hot-keywords | 获取热门搜索词 | ❌ |

---

## 八、B端管理 (10个接口)

### 8.1 数据看板
| 方法 | 路径 | 说明 | 需要认证 |
|------|------|------|---------|
| GET | /api/merchant/dashboard | 获取商家首页看板数据 | ✅ |

### 8.2 项目管理
| 方法 | 路径 | 说明 | 需要认证 |
|------|------|------|---------|
| GET | /api/merchant/events | 获取商家项目列表 | ✅ |

### 8.3 订单管理
| 方法 | 路径 | 说明 | 需要认证 |
|------|------|------|---------|
| GET | /api/merchant/orders | 获取商家订单列表 | ✅ |
| GET | /api/merchant/orders/export | 导出订单数据 | ✅ |
| POST | /api/merchant/offline-sale | 后台售票 | ✅ |

### 8.4 退款管理
| 方法 | 路径 | 说明 | 需要认证 |
|------|------|------|---------|
| GET | /api/merchant/refunds | 获取退款申请列表 | ✅ |
| POST | /api/merchant/refunds/{refundId}/approve | 审核退款申请 | ✅ |

### 8.5 结算管理
| 方法 | 路径 | 说明 | 需要认证 |
|------|------|------|---------|
| GET | /api/merchant/settlement | 获取结算列表 | ✅ |
| POST | /api/merchant/settlement/withdraw | 申请提现 | ✅ |

---

## 九、运营后台 (10个接口)

### 9.1 用户管理
| 方法 | 路径 | 说明 | 需要认证 |
|------|------|------|---------|
| GET | /api/admin/users | 获取用户列表 | ✅ |
| POST | /api/admin/users/{userId}/freeze | 冻结/解冻用户 | ✅ |

### 9.2 商家管理
| 方法 | 路径 | 说明 | 需要认证 |
|------|------|------|---------|
| GET | /api/admin/merchants | 获取商家列表 | ✅ |
| POST | /api/admin/merchants/{merchantId}/audit | 审核商家 | ✅ |

### 9.3 营销管理
| 方法 | 路径 | 说明 | 需要认证 |
|------|------|------|---------|
| GET | /api/admin/coupons | 获取优惠券列表 | ✅ |
| POST | /api/admin/coupons | 创建优惠券 | ✅ |
| GET | /api/admin/banners | 获取Banner列表 | ✅ |
| POST | /api/admin/banners | 创建Banner | ✅ |

### 9.4 系统设置
| 方法 | 路径 | 说明 | 需要认证 |
|------|------|------|---------|
| GET | /api/admin/settings/cities | 获取城市列表（管理） | ✅ |
| POST | /api/admin/settings/cities | 添加城市 | ✅ |
| GET | /api/admin/settings/categories | 获取类目列表（管理） | ✅ |
| POST | /api/admin/settings/categories | 添加类目 | ✅ |

---

## 十、总代票务 (8个接口)

### 10.1 数据大屏
| 方法 | 路径 | 说明 | 需要认证 |
|------|------|------|---------|
| GET | /api/agent/dashboard | 获取数据大屏数据 | ✅ |

### 10.2 座位图管理
| 方法 | 路径 | 说明 | 需要认证 |
|------|------|------|---------|
| GET | /api/agent/seat-maps | 获取座位图列表 | ✅ |
| POST | /api/agent/seat-maps | 创建座位图 | ✅ |

### 10.3 座位管理
| 方法 | 路径 | 说明 | 需要认证 |
|------|------|------|---------|
| GET | /api/agent/seats/{sessionId} | 获取场次座位详情 | ✅ |
| POST | /api/agent/seats/batch-lock | 批量锁定座位 | ✅ |

### 10.4 票务操作
| 方法 | 路径 | 说明 | 需要认证 |
|------|------|------|---------|
| POST | /api/agent/tickets/print | 打印纸质票 | ✅ |
| POST | /api/agent/allocation | 配票给渠道 | ✅ |

### 10.5 检票统计
| 方法 | 路径 | 说明 | 需要认证 |
|------|------|------|---------|
| GET | /api/agent/check-in/stats | 获取检票统计 | ✅ |

---

## 接口统计

| 模块 | 接口数量 | 需要认证 | 公开接口 |
|------|---------|---------|---------|
| 用户服务 | 10 | 6 | 4 |
| 项目服务 | 15 | 5 | 10 |
| 订单服务 | 10 | 10 | 0 |
| 支付服务 | 4 | 2 | 2 |
| 库存服务 | 5 | 3 | 2 |
| 营销服务 | 4 | 2 | 2 |
| 搜索服务 | 3 | 0 | 3 |
| B端管理 | 10 | 10 | 0 |
| 运营后台 | 10 | 10 | 0 |
| 总代票务 | 8 | 8 | 0 |
| **总计** | **79** | **56** | **23** |

---

## 数据模型清单 (40+)

### 用户相关
- LoginResponse
- WechatLoginResponse
- UserProfile
- Address
- AddressRequest
- RealNameInfo

### 项目相关
- City
- Category
- EventSummary
- EventDetail
- CreateEventRequest
- PagedEventList
- Venue
- Session
- TicketTier
- Seat
- SeatMap

### 订单相关
- CreateOrderRequest
- OrderDetail
- OrderItem
- PagedOrderList
- Refund
- Ticket

### 支付相关
- PaymentInfo

### 营销相关
- Coupon
- UserCoupon
- CreateCouponRequest
- Promotion
- Banner
- CreateBannerRequest

### 管理相关
- MerchantDashboard
- SettlementList
- MerchantList
- AgentDashboard

### 通用
- ApiResponse

---

## 使用说明

1. **认证方式**: Bearer Token (JWT)
2. **响应格式**: 统一JSON格式
3. **分页参数**: page, size
4. **日期格式**: ISO 8601
5. **错误码**: 1000-6999

详细使用说明请参考：
- `API文档说明.md`
- `Apifox导入指南.md`
