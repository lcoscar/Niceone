# NiceOne 货物下单系统设计文档

## 系统概述

NiceOne 是一个面向批发业务的在线订单管理系统，主要用于恒立（香港）批发商的订单处理。系统支持用户认证、白名单管理、产品选择、订单提交、以及与第三方系统（WhatsApp、Quickbooks、打印机）的集成。

### 核心功能

- **用户认证与授权**: 基于 Firebase Authentication 的用户认证，支持白名单机制
- **产品浏览与选择**: 按类别浏览产品，支持购物车管理
- **订单管理**: 创建订单、查看订单历史、订单状态跟踪
- **数据转换**: 支持将订单数据转换为 WhatsApp 消息、Quickbooks IIF 文件、打印标签文件
- **等待列表管理**: 非白名单用户的申请和审核流程

### 技术栈

- **前端**: HTML5, JavaScript, Tailwind CSS
- **后端**: Python (FastAPI/Django)
- **认证**: Firebase Authentication (支持 Email/Password, Google, GitHub 等)
- **数据库**: 通用设计（支持 SQL/NoSQL）
- **集成**: WhatsApp Business API, Quickbooks, 标签打印机

## 文档导航

本文档集包含以下文档：

### 1. [系统架构设计](./01-system-architecture.md)
- 整体架构图
- 技术栈说明
- 组件交互流程
- 部署架构

### 2. [用户旅程设计](./02-user-journey.md)
- 用户认证流程
- 白名单检查机制
- 下单流程详细说明
- 等待列表处理流程
- 状态转换图

### 3. [数据模型设计](./03-data-model.md)
- 数据库表结构定义
- 实体关系图（ER Diagram）
- 字段说明和约束
- 索引设计

### 4. [API 接口设计](./04-api-design.md)
- REST API 端点定义
- 请求/响应格式
- 认证机制
- 错误处理
- API 示例

### 5. [转换器设计](./05-converters.md)
- IM Connector (WhatsApp 消息生成)
- Quickbooks Convertor (IIF 文件生成)
- Printer Convertor (标签打印文件生成)
- 输入输出格式说明

## 快速开始

### 系统流程概览

```
用户访问系统
    ↓
Firebase 认证
    ↓
检查白名单
    ├─ 在白名单中 → 进入主页面
    └─ 不在白名单中 → 加入等待列表
    ↓
选择产品并加入购物车
    ↓
提交订单
    ↓
订单存入数据库
    ↓
生成转换输出（可选）
    ├─ WhatsApp 消息
    ├─ Quickbooks IIF 文件
    └─ 打印标签文件
```

## 业务规则

1. **白名单机制**: 只有白名单中的用户可以直接下单
2. **客户信息**: 下单前必须填写完整的客户资料（名称、编号、电话、地址）
3. **订单编号**: 格式为 `SO-{timestamp}`（如 SO-123456）
4. **人工确认**: 订单需要人工确认后才能发送到 Quickbooks
5. **打印条件**: 只有当 `is_checked=true` 时，才生成打印文件

## 相关资源

- 前端原型: `../index.html`
- 后端代码: `../backend/`
- 转换器代码: `../convertor/`

## 版本历史

- **v1.0** (当前): 初始设计文档

## 联系方式

如有问题或建议，请联系开发团队。
