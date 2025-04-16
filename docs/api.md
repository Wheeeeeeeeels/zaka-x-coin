# API 文档

## 1. 概述
本文档描述了 Zaka-X-Coin 交易平台的 REST API 接口规范。所有 API 请求都需要进行身份验证，使用 JWT Token 进行认证。

## 2. 通用说明

### 2.1 基础信息
- 基础URL: `https://api.zaka-x-coin.com/v1`
- 所有请求都需要在 Header 中包含 `Authorization: Bearer {token}`
- 所有响应都使用 JSON 格式

### 2.2 响应格式
```json
{
    "code": 0,           // 状态码，0 表示成功
    "message": "success", // 响应消息
    "data": {}           // 响应数据
}
```

### 2.3 错误码
| 错误码 | 说明 |
|--------|------|
| 0 | 成功 |
| 1001 | 参数错误 |
| 1002 | 认证失败 |
| 1003 | 权限不足 |
| 1004 | 资源不存在 |
| 1005 | 系统错误 |

## 3. 用户相关接口

### 3.1 用户注册
```http
POST /users/register
```

请求参数：
```json
{
    "username": "string",
    "email": "string",
    "password": "string",
    "phone": "string"
}
```

响应：
```json
{
    "code": 0,
    "message": "success",
    "data": {
        "id": 1,
        "username": "string",
        "email": "string",
        "created_at": "2024-01-01T00:00:00Z"
    }
}
```

### 3.2 用户登录
```http
POST /users/login
```

请求参数：
```json
{
    "username": "string",
    "password": "string"
}
```

响应：
```json
{
    "code": 0,
    "message": "success",
    "data": {
        "token": "string",
        "expires_in": 3600
    }
}
```

### 3.3 获取用户信息
```http
GET /users/me
```

响应：
```json
{
    "code": 0,
    "message": "success",
    "data": {
        "id": 1,
        "username": "string",
        "email": "string",
        "phone": "string",
        "status": "active",
        "created_at": "2024-01-01T00:00:00Z"
    }
}
```

## 4. 资产相关接口

### 4.1 获取资产列表
```http
GET /assets
```

响应：
```json
{
    "code": 0,
    "message": "success",
    "data": [
        {
            "currency": "BTC",
            "balance": "1.00000000",
            "frozen_balance": "0.00000000",
            "available_balance": "1.00000000"
        }
    ]
}
```

### 4.2 获取资产详情
```http
GET /assets/{currency}
```

响应：
```json
{
    "code": 0,
    "message": "success",
    "data": {
        "currency": "BTC",
        "balance": "1.00000000",
        "frozen_balance": "0.00000000",
        "available_balance": "1.00000000",
        "deposit_address": "string",
        "withdraw_address": "string"
    }
}
```

## 5. 交易相关接口

### 5.1 获取交易对列表
```http
GET /trading_pairs
```

响应：
```json
{
    "code": 0,
    "message": "success",
    "data": [
        {
            "id": 1,
            "base_currency": "BTC",
            "quote_currency": "USDT",
            "min_price": "0.00000001",
            "max_price": "100000.00000000",
            "min_amount": "0.00010000",
            "max_amount": "1000.00000000",
            "price_precision": 8,
            "amount_precision": 8,
            "status": "active"
        }
    ]
}
```

### 5.2 获取市场深度
```http
GET /trading_pairs/{id}/depth
```

请求参数：
```json
{
    "limit": 20
}
```

响应：
```json
{
    "code": 0,
    "message": "success",
    "data": {
        "bids": [
            ["10000.00000000", "1.00000000"],
            ["9999.00000000", "2.00000000"]
        ],
        "asks": [
            ["10001.00000000", "1.00000000"],
            ["10002.00000000", "2.00000000"]
        ]
    }
}
```

### 5.3 创建订单
```http
POST /orders
```

请求参数：
```json
{
    "trading_pair_id": 1,
    "side": "buy",
    "type": "limit",
    "price": "10000.00000000",
    "amount": "1.00000000"
}
```

响应：
```json
{
    "code": 0,
    "message": "success",
    "data": {
        "id": 1,
        "trading_pair_id": 1,
        "side": "buy",
        "type": "limit",
        "price": "10000.00000000",
        "amount": "1.00000000",
        "filled_amount": "0.00000000",
        "status": "pending",
        "created_at": "2024-01-01T00:00:00Z"
    }
}
```

### 5.4 获取订单列表
```http
GET /orders
```

请求参数：
```json
{
    "trading_pair_id": 1,
    "status": "pending",
    "page": 1,
    "limit": 20
}
```

响应：
```json
{
    "code": 0,
    "message": "success",
    "data": {
        "total": 100,
        "page": 1,
        "limit": 20,
        "items": [
            {
                "id": 1,
                "trading_pair_id": 1,
                "side": "buy",
                "type": "limit",
                "price": "10000.00000000",
                "amount": "1.00000000",
                "filled_amount": "0.00000000",
                "status": "pending",
                "created_at": "2024-01-01T00:00:00Z"
            }
        ]
    }
}
```

## 6. 钱包相关接口

### 6.1 获取充值地址
```http
GET /wallet/deposit_address/{currency}
```

响应：
```json
{
    "code": 0,
    "message": "success",
    "data": {
        "currency": "BTC",
        "address": "string",
        "tag": "string"
    }
}
```

### 6.2 发起提现
```http
POST /wallet/withdraw
```

请求参数：
```json
{
    "currency": "BTC",
    "amount": "1.00000000",
    "address": "string",
    "tag": "string"
}
```

响应：
```json
{
    "code": 0,
    "message": "success",
    "data": {
        "id": 1,
        "currency": "BTC",
        "amount": "1.00000000",
        "address": "string",
        "status": "pending",
        "created_at": "2024-01-01T00:00:00Z"
    }
}
```

## 7. WebSocket 接口

### 7.1 市场数据订阅
```javascript
// 连接
const ws = new WebSocket('wss://api.zaka-x-coin.com/v1/ws');

// 订阅消息
ws.send(JSON.stringify({
    "op": "subscribe",
    "args": ["market.ticker.BTC-USDT", "market.depth.BTC-USDT"]
}));

// 接收消息
ws.onmessage = (event) => {
    const data = JSON.parse(event.data);
    console.log(data);
};
```

### 7.2 订单更新订阅
```javascript
// 连接
const ws = new WebSocket('wss://api.zaka-x-coin.com/v1/ws');

// 订阅消息
ws.send(JSON.stringify({
    "op": "subscribe",
    "args": ["order.update"]
}));

// 接收消息
ws.onmessage = (event) => {
    const data = JSON.parse(event.data);
    console.log(data);
};
```

## 8. 错误处理

### 8.1 错误响应示例
```json
{
    "code": 1001,
    "message": "参数错误",
    "data": {
        "field": "price",
        "reason": "价格不能为空"
    }
}
```

### 8.2 常见错误
- 参数错误：检查请求参数是否符合要求
- 认证失败：检查 token 是否有效
- 权限不足：检查用户权限
- 资源不存在：检查请求的资源是否存在
- 系统错误：联系技术支持 