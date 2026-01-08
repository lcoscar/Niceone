# 用户旅程设计

## 用户旅程概览

本文档详细描述 NiceOne 系统中用户从访问到完成订单的完整流程，包括认证、白名单检查、下单和等待列表处理。

## 主要用户角色

1. **白名单用户**: 已通过审核，可以直接下单的客户
2. **等待列表用户**: 新申请用户，等待管理员审核
3. **管理员**: 管理系统、审核用户、处理订单

## 完整用户旅程

### 流程图

```mermaid
flowchart TD
    Start([用户访问系统]) --> Auth{是否已登录?}
    Auth -->|否| FirebaseAuth[Firebase 认证流程]
    Auth -->|是| CheckWhitelist{检查白名单}
    
    FirebaseAuth --> GetUserInfo[获取用户信息]
    GetUserInfo --> CheckWhitelist
    
    CheckWhitelist -->|在白名单中| MainPage[进入主页面]
    CheckWhitelist -->|不在白名单中| CheckWaitlist{是否在等待列表?}
    
    CheckWaitlist -->|否| JoinWaitlist[加入等待列表]
    CheckWaitlist -->|是| WaitlistPage[显示等待审核页面]
    JoinWaitlist --> WaitlistPage
    
    WaitlistPage --> AdminReview[管理员审核]
    AdminReview -->|通过| AddToWhitelist[加入白名单]
    AdminReview -->|拒绝| RejectNotification[发送拒绝通知]
    AddToWhitelist --> MainPage
    
    MainPage --> BrowseProducts[浏览产品]
    BrowseProducts --> SelectProducts[选择产品加入购物车]
    SelectProducts --> CheckCart{购物车有商品?}
    
    CheckCart -->|否| BrowseProducts
    CheckCart -->|是| CheckCustomer{客户信息完整?}
    
    CheckCustomer -->|否| FillCustomerInfo[填写客户资料]
    CheckCustomer -->|是| ReviewOrder[确认订单]
    
    FillCustomerInfo --> SaveCustomer[保存客户信息]
    SaveCustomer --> ReviewOrder
    
    ReviewOrder --> SubmitOrder[提交订单]
    SubmitOrder --> SaveToDB[保存订单到数据库]
    SaveToDB --> OrderSuccess[订单创建成功]
    
    OrderSuccess --> ChooseAction{选择操作}
    ChooseAction -->|发送 WhatsApp| GenerateWhatsApp[生成 WhatsApp 消息]
    ChooseAction -->|下载 IIF| GenerateIIF[生成 Quickbooks IIF]
    ChooseAction -->|打印标签| GeneratePrint[生成打印文件]
    ChooseAction -->|继续购物| BrowseProducts
    
    GenerateWhatsApp --> End1([完成])
    GenerateIIF --> End2([完成])
    GeneratePrint --> End3([完成])
    
    style Start fill:#e1f5ff
    style OrderSuccess fill:#c8e6c9
    style WaitlistPage fill:#fff9c4
    style AdminReview fill:#ffccbc
```

## 详细流程说明

### 1. 用户认证流程

#### 1.1 Firebase 认证步骤

```mermaid
sequenceDiagram
    participant User as 用户
    participant Web as 前端应用
    participant Firebase as Firebase Auth
    participant Backend as 后端 API
    participant AdminSDK as Firebase Admin SDK
    participant DB as 数据库
    
    User->>Web: 点击"登录"按钮
    Web->>Firebase: 调用 signInWithPopup/signInWithEmail
    Firebase->>User: 显示登录界面（或使用已保存凭据）
    User->>Firebase: 输入凭据并授权
    Firebase-->>Web: 返回 Firebase ID Token
    Web->>Web: 保存 ID Token
    Web->>Backend: API 请求（携带 ID Token）
    Backend->>AdminSDK: 验证 ID Token
    AdminSDK->>Firebase: 验证 Token 有效性
    Firebase-->>AdminSDK: 返回用户信息
    AdminSDK-->>Backend: 验证成功，返回用户数据
    Backend->>DB: 保存/更新用户信息（如需要）
    Backend-->>Web: 返回请求结果
    Web->>User: 显示主页面
```

#### 1.2 支持的认证方式

- **Email/Password 认证**
  - 用户注册和登录
  - 密码重置功能
  
- **Google 登录**
  - 通过 Firebase Google Provider
  - 自动获取用户信息
  
- **GitHub 登录**（可选）
  - 通过 Firebase GitHub Provider
  - 自动获取用户信息

#### 1.3 Token 管理

- **Firebase ID Token**: Firebase Authentication 返回的 ID Token
- **Token 验证**: 后端使用 Firebase Admin SDK 验证 Token
- **Token 自动刷新**: Firebase SDK 自动处理 Token 刷新
- **Token 有效期**: 
  - ID Token: 1 小时（Firebase 自动刷新）
  - Refresh Token: Firebase 自动管理

### 2. 白名单检查流程

#### 2.1 白名单验证逻辑

```mermaid
flowchart LR
    A[获取用户信息] --> B{查询白名单表}
    B -->|找到记录| C{状态 = active?}
    B -->|未找到| D[不在白名单]
    C -->|是| E[允许访问]
    C -->|否| D
    D --> F[检查等待列表]
    F --> G{是否在等待列表?}
    G -->|是| H[显示等待页面]
    G -->|否| I[加入等待列表]
```

#### 2.2 白名单表结构

| 字段 | 类型 | 说明 |
|------|------|------|
| id | INTEGER | 主键 |
| user_email | VARCHAR | 用户邮箱（唯一） |
| user_name | VARCHAR | 用户名称 |
| status | ENUM | 'active', 'inactive', 'suspended' |
| created_at | TIMESTAMP | 创建时间 |
| updated_at | TIMESTAMP | 更新时间 |
| approved_by | INTEGER | 审核人 ID |

#### 2.3 API 端点

```
GET /api/auth/check-whitelist
Headers: Authorization: Bearer {firebase_id_token}

Response:
{
  "in_whitelist": true,
  "user_info": {
    "email": "user@example.com",
    "name": "User Name"
  },
  "status": "active"
}
```

### 3. 等待列表流程

#### 3.1 加入等待列表

当用户不在白名单时，系统自动将其加入等待列表：

```mermaid
sequenceDiagram
    participant User as 用户
    participant Web as 前端
    participant API as 后端 API
    participant DB as 数据库
    participant Admin as 管理员
    
    User->>Web: 访问系统
    Web->>API: 检查白名单状态
    API->>DB: 查询白名单表
    DB-->>API: 未找到记录
    API->>DB: 检查等待列表
    DB-->>API: 未找到记录
    API->>DB: 插入等待列表记录
    DB-->>API: 创建成功
    API-->>Web: 返回等待状态
    Web->>User: 显示等待审核页面
    
    Note over Admin: 管理员收到通知
    Admin->>API: 审核用户
    API->>DB: 更新等待列表状态
    API->>DB: 加入白名单（如通过）
    API->>User: 发送审核结果通知
```

#### 3.2 等待列表页面内容

- 友好的提示信息
- 显示申请状态
- 说明审核流程
- 联系信息（可选）

#### 3.3 管理员审核流程

1. 管理员登录后台
2. 查看等待列表
3. 审核用户信息
4. 决定通过或拒绝
5. 系统自动发送通知

### 4. 下单流程

#### 4.1 产品浏览与选择

```mermaid
stateDiagram-v2
    [*] --> 浏览产品
    浏览产品 --> 按类别筛选: 选择类别
    浏览产品 --> 搜索产品: 输入关键词
    按类别筛选 --> 查看产品详情
    搜索产品 --> 查看产品详情
    查看产品详情 --> 加入购物车
    加入购物车 --> 浏览产品
    加入购物车 --> 查看购物车
    查看购物车 --> 修改数量
    修改数量 --> 查看购物车
    查看购物车 --> 继续购物
    查看购物车 --> 检查客户信息
    继续购物 --> 浏览产品
```

#### 4.2 购物车管理

**购物车数据结构**:
```javascript
{
  "P1001": 5,  // product_id: quantity
  "P1002": 3,
  "P1005": 10
}
```

**操作**:
- 添加产品: 增加数量
- 移除产品: 减少数量或删除
- 清空购物车: 清除所有商品

#### 4.3 客户信息管理

**必填字段**:
- 客户名称 (customer_name)
- 客户编号 (customer_id) - Quickbooks ID
- 电话 (phone)
- 地址 (address)

**保存机制**:
- 前端 localStorage 临时保存
- 提交订单时验证完整性
- 订单成功后保存到数据库

#### 4.4 订单提交流程

```mermaid
sequenceDiagram
    participant User as 用户
    participant Web as 前端
    participant API as 后端 API
    participant DB as 数据库
    participant Validator as 订单验证器
    
    User->>Web: 点击"提交订单"
    Web->>Web: 验证购物车不为空
    Web->>Web: 验证客户信息完整
    Web->>API: POST /api/orders
    
    API->>Validator: 验证订单数据
    Validator->>API: 验证通过
    
    API->>API: 生成订单号 (SO-{timestamp})
    API->>DB: 保存订单主表
    API->>DB: 保存订单明细表
    DB-->>API: 订单 ID
    
    API->>API: 设置 is_checked = false
    API-->>Web: 返回订单信息
    Web->>User: 显示订单成功页面
```

#### 4.5 订单数据结构

**订单主表**:
```
order_id: SO-123456
customer_id: CUST001
customer_name: ABC Company
total_amount: 5000.00
status: 'pending'
is_checked: false
created_at: 2024-01-01 10:00:00
```

**订单明细表**:
```
order_id: SO-123456
product_id: P1001
product_name: 力士香皂(混合)
product_spec: 80g x6s x24扎
quantity: 5
unit_price: 264.00
subtotal: 1320.00
```

### 5. 订单后处理

#### 5.1 可选操作

订单创建成功后，用户可以选择以下操作：

1. **发送 WhatsApp 消息**
   - 通过 IM Connector 生成消息
   - 打开 WhatsApp 应用发送

2. **下载 Quickbooks IIF 文件**
   - 通过 Quickbooks Convertor 生成
   - 下载到本地，导入 Quickbooks

3. **生成打印标签**
   - 通过 Printer Convertor 生成
   - 显示 QR Code 供扫描
   - 或直接下载打印文件

#### 5.2 人工确认流程

```mermaid
flowchart TD
    A[订单创建] --> B[is_checked = false]
    B --> C{管理员审核}
    C -->|通过 WhatsApp| D[管理员回复确认]
    C -->|通过后台| E[管理员点击确认]
    D --> F[更新 is_checked = true]
    E --> F
    F --> G[允许生成 IIF]
    F --> H[允许打印标签]
    
    style F fill:#c8e6c9
    style B fill:#fff9c4
```

### 6. 异常流程处理

#### 6.1 认证失败
- 显示错误信息
- 提供重新登录选项
- 记录错误日志

#### 6.2 白名单检查失败
- 自动加入等待列表
- 显示友好提示
- 记录检查日志

#### 6.3 订单提交失败
- 显示具体错误原因
- 保留购物车数据
- 允许重新提交

#### 6.4 网络错误
- 显示重试选项
- 本地数据备份
- 自动重试机制（可选）

## 状态转换图

### 用户状态转换

```mermaid
stateDiagram-v2
    [*] --> 未认证
    未认证 --> 已认证: Firebase 登录成功
    已认证 --> 白名单用户: 在白名单中
    已认证 --> 等待列表用户: 不在白名单中
    等待列表用户 --> 白名单用户: 管理员审核通过
    等待列表用户 --> 已拒绝: 管理员审核拒绝
    白名单用户 --> 下单中: 开始下单
    下单中 --> 订单完成: 提交订单成功
    订单完成 --> 白名单用户: 继续购物
    订单完成 --> [*]: 退出系统
```

### 订单状态转换

```mermaid
stateDiagram-v2
    [*] --> pending: 创建订单
    pending --> confirmed: 人工确认 (is_checked=true)
    pending --> cancelled: 取消订单
    confirmed --> processing: 生成 IIF/打印
    confirmed --> completed: 完成处理
    processing --> completed: 处理完成
    cancelled --> [*]
    completed --> [*]
```

## 用户体验优化

1. **加载状态**: 显示加载动画，避免用户等待焦虑
2. **错误提示**: 清晰的错误信息和解决建议
3. **数据持久化**: 购物车和客户信息本地保存
4. **响应式设计**: 支持移动端和桌面端
5. **快捷操作**: 常用功能快捷入口
6. **操作反馈**: 成功/失败操作的即时反馈

## 性能要求

1. **页面加载**: < 2 秒
2. **API 响应**: < 500ms (95 分位)
3. **订单提交**: < 1 秒
4. **文件生成**: < 5 秒

## 可访问性

1. 键盘导航支持
2. 屏幕阅读器兼容
3. 色彩对比度符合 WCAG 标准
4. 多语言支持（中文/英文）
