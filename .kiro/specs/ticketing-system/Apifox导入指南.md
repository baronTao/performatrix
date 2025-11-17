# Apifox导入指南

## 快速开始

### 第一步：准备文件
确保你有以下文件：
```
.kiro/specs/ticketing-system/apifox-openapi-complete.yaml
```

### 第二步：打开Apifox
1. 启动Apifox应用
2. 创建新项目或打开现有项目

### 第三步：导入API文档
1. 点击左侧菜单栏的"导入"按钮
2. 在弹出的对话框中选择"OpenAPI/Swagger"
3. 选择"从文件导入"选项
4. 浏览并选择 `apifox-openapi-complete.yaml` 文件
5. 点击"确认导入"按钮

### 第四步：验证导入结果
导入成功后，你应该能看到：
- ✅ 11个API分组（用户服务、项目服务、订单服务等）
- ✅ 80+个API接口
- ✅ 40+个数据模型（Schemas）
- ✅ 3个服务器环境配置

## 导入后配置

### 1. 设置环境变量
在Apifox中创建以下环境变量：

**开发环境**
```
baseUrl: http://localhost:8080
token: (登录后获取的JWT Token)
```

**测试环境**
```
baseUrl: https://test-api.ticketing.com
token: (登录后获取的JWT Token)
```

**生产环境**
```
baseUrl: https://api.ticketing.com
token: (登录后获取的JWT Token)
```

### 2. 配置全局认证
1. 进入"项目设置" → "认证"
2. 选择"Bearer Token"
3. 在Token字段中输入：`{{token}}`
4. 这样所有需要认证的接口都会自动使用环境变量中的token

### 3. 启用Mock服务（可选）
1. 点击"Mock服务"
2. 启用Mock服务器
3. 前端开发时可以使用Mock数据进行开发

## 测试流程建议

### 基础流程测试
```
1. 发送短信验证码
   POST /api/user/send-sms

2. 用户注册
   POST /api/user/register
   → 获取token并保存到环境变量

3. 获取城市列表
   GET /api/cities

4. 获取项目列表
   GET /api/events?cityId=110000

5. 获取项目详情
   GET /api/events/{eventId}

6. 创建订单
   POST /api/orders

7. 创建支付
   POST /api/payment/create

8. 查询订单详情
   GET /api/orders/{orderId}
```

### 完整购票流程测试
```
用户注册/登录 
  ↓
浏览项目列表
  ↓
查看项目详情
  ↓
选择场次和票档
  ↓
检查库存
  ↓
分配座位（如需选座）
  ↓
创建订单
  ↓
创建支付
  ↓
查询支付状态
  ↓
查看电子票
```

## 常见问题

### Q1: 导入后接口显示不全？
**A:** 检查YAML文件是否完整，确保文件大小约为150KB左右。

### Q2: 如何测试需要认证的接口？
**A:** 
1. 先调用登录接口获取token
2. 将token保存到环境变量
3. 在全局认证中配置Bearer Token使用 `{{token}}`

### Q3: 如何切换不同环境？
**A:** 在Apifox右上角的环境下拉菜单中选择对应环境即可。

### Q4: Mock数据如何使用？
**A:** 
1. 启用Mock服务
2. 使用Mock服务器地址替代真实API地址
3. Apifox会根据Schema自动生成Mock数据

### Q5: 如何导出接口文档？
**A:** 
1. 点击"分享" → "导出数据"
2. 选择导出格式（Markdown、HTML、PDF等）
3. 可以生成在线文档或离线文档

## 接口分组说明

| 分组 | 接口数量 | 说明 |
|------|---------|------|
| 用户服务 | 10 | 注册、登录、个人信息管理 |
| 项目服务 | 15 | 项目、场次、票档管理 |
| 订单服务 | 10 | 订单创建、查询、退款 |
| 支付服务 | 4 | 支付、回调处理 |
| 库存服务 | 5 | 库存检查、座位分配 |
| 营销服务 | 4 | 优惠券、促销活动 |
| 搜索服务 | 3 | 项目搜索、建议 |
| B端管理 | 10 | 商家后台功能 |
| 运营后台 | 10 | 平台运营管理 |
| 总代票务 | 8 | 票务代理系统 |

## 数据模型说明

文档包含以下核心数据模型：

**用户相关**
- UserProfile: 用户信息
- Address: 收货地址
- RealNameInfo: 实名信息

**项目相关**
- Event: 项目信息
- Session: 场次信息
- TicketTier: 票档信息
- Seat: 座位信息
- Venue: 场馆信息

**订单相关**
- Order: 订单信息
- OrderItem: 订单项
- Refund: 退款信息
- Ticket: 电子票

**营销相关**
- Coupon: 优惠券
- UserCoupon: 用户优惠券
- Promotion: 促销活动
- Banner: 轮播图

## 技术支持

如遇到问题，请查看：
1. [Apifox官方文档](https://www.apifox.cn/help/)
2. [OpenAPI规范](https://swagger.io/specification/)
3. 项目API文档说明：`API文档说明.md`

---

**文档版本**: 1.0.0  
**最后更新**: 2024-01-01  
**维护团队**: 票务系统开发组
