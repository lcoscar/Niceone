# API 接口设计

## API 设计概览

本文档定义 NiceOne 系统的 REST API 接口，包括认证、订单、产品、客户管理等所有端点。

## API 基础信息

### 基础 URL

```
开发环境: http://localhost:8000/api
生产环境: https://api.niceone.com/api
```

### 版本管理

API 版本通过 URL 路径管理：`/api/v1/...`

### 认证方式

所有需要认证的接口都需要在请求头中携带 Firebase ID Token：

```
Authorization: Bearer {firebase_id_token}
```

后端使用 Firebase Admin SDK 验证 Token 的有效性。

### 响应格式

**成功响应**:
```json
{
  "success": true,
  "data": {...},
  "message": "操作成功"
}
```

**错误响应**:
```json
{
  "success": false,
  "error": {
    "code": "ERROR_CODE",
    "message": "错误描述",
    "details": {...}
  }
}
```

### HTTP 状态码

| 状态码 | 说明 |
|--------|------|
| 200 | 成功 |
| 201 | 创建成功 |
| 400 | 请求参数错误 |
| 401 | 未认证 |
| 403 | 无权限 |
| 404 | 资源不存在 |
| 500 | 服务器错误 |

## 认证 API

### 1. 验证 Firebase Token

验证前端传入的 Firebase ID Token 并返回用户信息。

**端点**: `POST /api/auth/verify`

**请求头**: `Authorization: Bearer {firebase_id_token}`

**响应示例**:
```json
{
  "success": true,
  "data": {
    "user": {
      "uid": "firebase_user_id",
      "email": "user@example.com",
      "name": "John Doe",
      "avatar_url": "https://..."
    },
    "is_authenticated": true
  }
}
```

**错误响应** (Token 无效):
```json
{
  "success": false,
  "error": {
    "code": "AUTH_INVALID_TOKEN",
    "message": "Firebase ID Token 无效或已过期"
  }
}
```

### 2. 获取当前用户信息

获取当前认证用户的信息（从 Token 中解析）。

**端点**: `GET /api/auth/user`

**请求头**: `Authorization: Bearer {firebase_id_token}`

**响应示例**:
```json
{
  "success": true,
  "data": {
    "uid": "firebase_user_id",
    "email": "user@example.com",
    "name": "John Doe",
    "avatar_url": "https://...",
    "email_verified": true
  }
}
```

**注意**: 前端使用 Firebase Authentication SDK 直接处理登录流程，无需后端提供 OAuth 回调端点。

### 3. 检查白名单状态

检查当前用户是否在白名单中。

**端点**: `GET /api/auth/check-whitelist`

**请求头**: `Authorization: Bearer {firebase_id_token}`

**响应示例**:
```json
{
  "success": true,
  "data": {
    "in_whitelist": true,
    "user_info": {
      "email": "user@example.com",
      "name": "John Doe"
    },
    "status": "active"
  }
}
```

**响应（不在白名单）**:
```json
{
  "success": true,
  "data": {
    "in_whitelist": false,
    "waitlist_status": "pending",
    "waitlist_id": 123
  }
}
```

### 4. 登出

登出当前用户（前端使用 Firebase SDK 登出）。

**端点**: `POST /api/auth/logout`

**请求头**: `Authorization: Bearer {firebase_id_token}`

**说明**: 登出操作主要由前端 Firebase SDK 处理，后端接口可选（用于清理服务器端会话）。

**响应示例**:
```json
{
  "success": true,
  "message": "登出成功"
}
```

## 产品 API

### 1. 获取产品列表

获取产品列表，支持分类和搜索。

**端点**: `GET /api/products`

**查询参数**:
- `category` (optional): 分类代码
- `is_hot` (optional): 是否热卖 (true/false)
- `search` (optional): 搜索关键词
- `page` (optional): 页码 (默认 1)
- `limit` (optional): 每页数量 (默认 20)

**响应示例**:
```json
{
  "success": true,
  "data": {
    "products": [
      {
        "id": 1,
        "product_code": "P1001",
        "name_cn": "力士香皂(混合)",
        "name_en": "Lux Soap (Mixed)",
        "category": {
          "id": 2,
          "category_code": "PERSONAL_CARE",
          "name_cn": "個人護理",
          "name_en": "Personal Care"
        },
        "spec": "80g x6s x24扎",
        "price": 264.00,
        "origin": "印尼",
        "is_hot": true
      }
    ],
    "pagination": {
      "page": 1,
      "limit": 20,
      "total": 100,
      "pages": 5
    }
  }
}
```

### 2. 获取产品详情

获取单个产品详情。

**端点**: `GET /api/products/{id}`

**路径参数**:
- `id`: 产品 ID 或产品编号

**响应示例**:
```json
{
  "success": true,
  "data": {
    "id": 1,
    "product_code": "P1001",
    "name_cn": "力士香皂(混合)",
    "name_en": "Lux Soap (Mixed)",
    "category": {...},
    "spec": "80g x6s x24扎",
    "price": 264.00,
    "origin": "印尼",
    "is_hot": true
  }
}
```

### 3. 获取分类列表

获取所有产品分类。

**端点**: `GET /api/categories`

**响应示例**:
```json
{
  "success": true,
  "data": {
    "categories": [
      {
        "id": 1,
        "category_code": "HOT_ITEMS",
        "name_cn": "熱門貨品",
        "name_en": "Hot Items",
        "icon": "🔥",
        "sort_order": 1
      }
    ]
  }
}
```

## 客户 API

### 1. 创建客户

创建新客户。

**端点**: `POST /api/customers`

**请求头**: `Authorization: Bearer {firebase_id_token}`

**请求体**:
```json
{
  "customer_id": "CUST001",
  "customer_name": "ABC Company",
  "phone": "12345678",
  "address": "Hong Kong",
  "region": "HK"
}
```

**响应示例**:
```json
{
  "success": true,
  "data": {
    "id": 1,
    "customer_id": "CUST001",
    "customer_name": "ABC Company",
    "phone": "12345678",
    "address": "Hong Kong",
    "region": "HK",
    "created_at": "2024-01-01T10:00:00Z"
  }
}
```

### 2. 获取客户列表

获取客户列表。

**端点**: `GET /api/customers`

**请求头**: `Authorization: Bearer {firebase_id_token}`

**查询参数**:
- `search` (optional): 搜索关键词
- `page` (optional): 页码
- `limit` (optional): 每页数量

**响应示例**:
```json
{
  "success": true,
  "data": {
    "customers": [
      {
        "id": 1,
        "customer_id": "CUST001",
        "customer_name": "ABC Company",
        "phone": "12345678",
        "address": "Hong Kong",
        "region": "HK"
      }
    ],
    "pagination": {...}
  }
}
```

### 3. 获取客户详情

获取单个客户详情。

**端点**: `GET /api/customers/{id}`

**请求头**: `Authorization: Bearer {firebase_id_token}`

**路径参数**:
- `id`: 客户 ID 或客户编号

**响应示例**:
```json
{
  "success": true,
  "data": {
    "id": 1,
    "customer_id": "CUST001",
    "customer_name": "ABC Company",
    "phone": "12345678",
    "address": "Hong Kong",
    "region": "HK"
  }
}
```

### 4. 更新客户信息

更新客户信息。

**端点**: `PUT /api/customers/{id}`

**请求头**: `Authorization: Bearer {firebase_id_token}`

**路径参数**:
- `id`: 客户 ID

**请求体**:
```json
{
  "customer_name": "ABC Company Ltd",
  "phone": "87654321",
  "address": "New Address",
  "region": "HK"
}
```

**响应示例**:
```json
{
  "success": true,
  "data": {
    "id": 1,
    "customer_id": "CUST001",
    "customer_name": "ABC Company Ltd",
    "phone": "87654321",
    "address": "New Address",
    "region": "HK",
    "updated_at": "2024-01-01T11:00:00Z"
  }
}
```

## 订单 API

### 1. 创建订单

创建新订单。

**端点**: `POST /api/orders`

**请求头**: `Authorization: Bearer {firebase_id_token}`

**请求体**:
```json
{
  "customer_id": "CUST001",
  "customer_name": "ABC Company",
  "phone": "12345678",
  "address": "Hong Kong",
  "region": "HK",
  "items": [
    {
      "product_id": "P1001",
      "quantity": 5
    },
    {
      "product_id": "P1002",
      "quantity": 3
    }
  ],
  "notes": "AI 生成的备注"
}
```

**响应示例**:
```json
{
  "success": true,
  "data": {
    "id": 1,
    "invoice_id": "INV-20240101-001",
    "order_number": "SO-123456",
    "customer_id": 1,
    "customer_name": "ABC Company",
    "total_amount": 2130.00,
    "status": "pending",
    "is_checked": false,
    "items": [
      {
        "id": 1,
        "product_code": "P1001",
        "product_name": "力士香皂(混合)",
        "product_spec": "80g x6s x24扎",
        "quantity": 5,
        "unit_price": 264.00,
        "subtotal": 1320.00
      },
      {
        "id": 2,
        "product_code": "P1002",
        "product_name": "力士香皂(米色)",
        "product_spec": "80g x6s x24扎",
        "quantity": 3,
        "unit_price": 270.00,
        "subtotal": 810.00
      }
    ],
    "created_at": "2024-01-01T10:00:00Z"
  }
}
```

### 2. 获取订单列表

获取订单列表。

**端点**: `GET /api/orders`

**请求头**: `Authorization: Bearer {firebase_id_token}`

**查询参数**:
- `status` (optional): 订单状态
- `is_checked` (optional): 是否确认 (true/false)
- `customer_id` (optional): 客户 ID
- `start_date` (optional): 开始日期
- `end_date` (optional): 结束日期
- `page` (optional): 页码
- `limit` (optional): 每页数量

**响应示例**:
```json
{
  "success": true,
  "data": {
    "orders": [
      {
        "id": 1,
        "invoice_id": "INV-20240101-001",
        "order_number": "SO-123456",
        "customer_name": "ABC Company",
        "total_amount": 2130.00,
        "status": "pending",
        "is_checked": false,
        "created_at": "2024-01-01T10:00:00Z"
      }
    ],
    "pagination": {...}
  }
}
```

### 3. 获取订单详情

获取单个订单详情。

**端点**: `GET /api/orders/{id}`

**请求头**: `Authorization: Bearer {firebase_id_token}`

**路径参数**:
- `id`: 订单 ID 或订单编号

**响应示例**:
```json
{
  "success": true,
  "data": {
    "id": 1,
    "invoice_id": "INV-20240101-001",
    "order_number": "SO-123456",
    "customer": {
      "id": 1,
      "customer_id": "CUST001",
      "customer_name": "ABC Company",
      "phone": "12345678",
      "address": "Hong Kong",
      "region": "HK"
    },
    "total_amount": 2130.00,
    "status": "pending",
    "is_checked": false,
    "notes": "AI 生成的备注",
    "items": [
      {
        "id": 1,
        "product": {
          "id": 1,
          "product_code": "P1001",
          "name_cn": "力士香皂(混合)",
          "name_en": "Lux Soap (Mixed)",
          "spec": "80g x6s x24扎",
          "price": 264.00,
          "origin": "印尼"
        },
        "quantity": 5,
        "unit_price": 264.00,
        "subtotal": 1320.00
      }
    ],
    "created_at": "2024-01-01T10:00:00Z",
    "updated_at": "2024-01-01T10:00:00Z"
  }
}
```

### 4. 更新订单状态

更新订单状态（如人工确认）。

**端点**: `PATCH /api/orders/{id}/status`

**请求头**: `Authorization: Bearer {firebase_id_token}`

**路径参数**:
- `id`: 订单 ID

**请求体**:
```json
{
  "status": "confirmed",
  "is_checked": true
}
```

**响应示例**:
```json
{
  "success": true,
  "data": {
    "id": 1,
    "status": "confirmed",
    "is_checked": true,
    "updated_at": "2024-01-01T11:00:00Z"
  }
}
```

### 5. 取消订单

取消订单。

**端点**: `POST /api/orders/{id}/cancel`

**请求头**: `Authorization: Bearer {firebase_id_token}`

**路径参数**:
- `id`: 订单 ID

**响应示例**:
```json
{
  "success": true,
  "data": {
    "id": 1,
    "status": "cancelled",
    "updated_at": "2024-01-01T12:00:00Z"
  }
}
```

## 转换器 API

### 1. 生成 WhatsApp 消息

为订单生成 WhatsApp 消息内容。

**端点**: `POST /api/orders/{id}/whatsapp`

**请求头**: `Authorization: Bearer {firebase_id_token}`

**路径参数**:
- `id`: 订单 ID

**响应示例**:
```json
{
  "success": true,
  "data": {
    "message": "*新訂單 New Order*\nOrder ID: SO-123456\n客戶: ABC Company (CUST001)\n電話: 12345678\n地址: Hong Kong\n----------------\n力士香皂(混合) x 5箱 = $1320\n力士香皂(米色) x 3箱 = $810\n----------------\n*預計總額: HK$2130.00*",
    "whatsapp_url": "https://wa.me/85265752256?text=..."
  }
}
```

### 2. 生成 Quickbooks IIF 文件

为订单生成 Quickbooks IIF 文件。

**端点**: `POST /api/orders/{id}/iif`

**请求头**: `Authorization: Bearer {firebase_id_token}`

**路径参数**:
- `id`: 订单 ID

**查询参数**:
- `download` (optional): 是否直接下载 (true/false, 默认 true)

**响应示例** (download=false):
```json
{
  "success": true,
  "data": {
    "filename": "Inv_SO-123456.iif",
    "content": "!TRNS\tTRNSID\tTRNSTYPE\tDATE\tACCNT\tNAME\tAMOUNT\tDOCNUM\tMEMO\n..."
  }
}
```

**响应** (download=true): 直接返回 IIF 文件下载

### 3. 生成打印标签

为订单生成打印标签文件或 QR Code。

**端点**: `POST /api/orders/{id}/print`

**请求头**: `Authorization: Bearer {firebase_id_token}`

**路径参数**:
- `id`: 订单 ID

**查询参数**:
- `format` (optional): 输出格式 (json/qrcode/file, 默认 json)

**响应示例** (format=json):
```json
{
  "success": true,
  "data": {
    "order_id": "SO-123456",
    "customer_name": "ABC Company",
    "labels": [
      {
        "product_name": "力士香皂(混合)",
        "product_spec": "80g x6s x24扎",
        "order_id": "SO-123456",
        "customer_name": "ABC Company",
        "index": 1,
        "total": 5
      }
    ],
    "print_url": "https://niceone.com/?printData=..."
  }
}
```

**响应示例** (format=qrcode):
```json
{
  "success": true,
  "data": {
    "qrcode_url": "data:image/png;base64,...",
    "print_url": "https://niceone.com/?printData=..."
  }
}
```

## 等待列表 API (管理员)

### 1. 获取等待列表

获取等待审核的用户列表。

**端点**: `GET /api/admin/waitlist`

**请求头**: `Authorization: Bearer {firebase_id_token}` (需要管理员权限)

**查询参数**:
- `status` (optional): 状态 (pending/approved/rejected)
- `page` (optional): 页码
- `limit` (optional): 每页数量

**响应示例**:
```json
{
  "success": true,
  "data": {
    "waitlist": [
      {
        "id": 1,
        "user_email": "newuser@example.com",
        "user_name": "Jane Smith",
        "status": "pending",
        "created_at": "2024-01-01T09:00:00Z"
      }
    ],
    "pagination": {...}
  }
}
```

### 2. 审核等待列表

审核等待列表中的用户。

**端点**: `POST /api/admin/waitlist/{id}/review`

**请求头**: `Authorization: Bearer {firebase_id_token}` (需要管理员权限)

**路径参数**:
- `id`: 等待列表 ID

**请求体**:
```json
{
  "action": "approve",
  "notes": "审核通过"
}
```

**响应示例**:
```json
{
  "success": true,
  "data": {
    "id": 1,
    "status": "approved",
    "whitelist_id": 10,
    "reviewed_at": "2024-01-01T12:00:00Z"
  }
}
```

## 错误码定义

### 认证错误 (AUTH_*)

| 错误码 | HTTP 状态码 | 说明 |
|--------|------------|------|
| AUTH_REQUIRED | 401 | 需要认证 |
| AUTH_INVALID_TOKEN | 401 | Firebase ID Token 无效 |
| AUTH_EXPIRED_TOKEN | 401 | Firebase ID Token 已过期 |
| AUTH_REVOKED_TOKEN | 401 | Firebase ID Token 已被撤销 |
| AUTH_NOT_IN_WHITELIST | 403 | 不在白名单中 |

### 订单错误 (ORDER_*)

| 错误码 | HTTP 状态码 | 说明 |
|--------|------------|------|
| ORDER_NOT_FOUND | 404 | 订单不存在 |
| ORDER_INVALID_STATUS | 400 | 订单状态无效 |
| ORDER_CART_EMPTY | 400 | 购物车为空 |
| ORDER_CUSTOMER_REQUIRED | 400 | 客户信息不完整 |

### 产品错误 (PRODUCT_*)

| 错误码 | HTTP 状态码 | 说明 |
|--------|------------|------|
| PRODUCT_NOT_FOUND | 404 | 产品不存在 |
| PRODUCT_OUT_OF_STOCK | 400 | 产品缺货 |

### 客户错误 (CUSTOMER_*)

| 错误码 | HTTP 状态码 | 说明 |
|--------|------------|------|
| CUSTOMER_NOT_FOUND | 404 | 客户不存在 |
| CUSTOMER_ID_EXISTS | 400 | 客户编号已存在 |

### 通用错误 (COMMON_*)

| 错误码 | HTTP 状态码 | 说明 |
|--------|------------|------|
| COMMON_VALIDATION_ERROR | 400 | 请求参数验证失败 |
| COMMON_INTERNAL_ERROR | 500 | 服务器内部错误 |
| COMMON_NOT_FOUND | 404 | 资源不存在 |
| COMMON_FORBIDDEN | 403 | 无权限访问 |

## API 限流

### 限流规则

- **认证接口**: 10 次/分钟
- **产品接口**: 60 次/分钟
- **订单接口**: 30 次/分钟
- **转换器接口**: 20 次/分钟

### 限流响应

当超过限流阈值时，返回 429 Too Many Requests：

```json
{
  "success": false,
  "error": {
    "code": "RATE_LIMIT_EXCEEDED",
    "message": "请求过于频繁，请稍后再试",
    "retry_after": 60
  }
}
```

## API 文档

建议使用 OpenAPI (Swagger) 自动生成 API 文档：

```yaml
openapi: 3.0.0
info:
  title: NiceOne API
  version: 1.0.0
servers:
  - url: http://localhost:8000/api
    description: 开发环境
  - url: https://api.niceone.com/api
    description: 生产环境
```

所有 API 端点都应提供 OpenAPI 规范的文档，包括：
- 请求参数说明
- 响应格式说明
- 错误码说明
- 示例请求/响应
