# 数据库设计文档

## 1. 数据库概述
本系统使用多种数据库来满足不同的业务需求：
- PostgreSQL：核心业务数据存储
- MongoDB：文档型数据存储
- Redis：缓存和会话管理
- 时序数据库：市场数据存储

## 2. PostgreSQL 数据库设计

### 2.1 用户相关表
```sql
-- 用户表
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    username VARCHAR(50) UNIQUE NOT NULL,
    email VARCHAR(100) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    phone VARCHAR(20),
    status VARCHAR(20) DEFAULT 'active',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- 用户认证表
CREATE TABLE user_auth (
    id SERIAL PRIMARY KEY,
    user_id INTEGER REFERENCES users(id),
    auth_type VARCHAR(20) NOT NULL,
    auth_value VARCHAR(255) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- 用户资产表
CREATE TABLE user_assets (
    id SERIAL PRIMARY KEY,
    user_id INTEGER REFERENCES users(id),
    currency VARCHAR(20) NOT NULL,
    balance DECIMAL(30,8) DEFAULT 0,
    frozen_balance DECIMAL(30,8) DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### 2.2 交易相关表
```sql
-- 交易对表
CREATE TABLE trading_pairs (
    id SERIAL PRIMARY KEY,
    base_currency VARCHAR(20) NOT NULL,
    quote_currency VARCHAR(20) NOT NULL,
    min_price DECIMAL(30,8),
    max_price DECIMAL(30,8),
    min_amount DECIMAL(30,8),
    max_amount DECIMAL(30,8),
    price_precision INTEGER,
    amount_precision INTEGER,
    status VARCHAR(20) DEFAULT 'active',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- 订单表
CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    user_id INTEGER REFERENCES users(id),
    trading_pair_id INTEGER REFERENCES trading_pairs(id),
    order_type VARCHAR(20) NOT NULL,
    side VARCHAR(10) NOT NULL,
    price DECIMAL(30,8),
    amount DECIMAL(30,8) NOT NULL,
    filled_amount DECIMAL(30,8) DEFAULT 0,
    status VARCHAR(20) DEFAULT 'pending',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- 交易记录表
CREATE TABLE trades (
    id SERIAL PRIMARY KEY,
    order_id INTEGER REFERENCES orders(id),
    trading_pair_id INTEGER REFERENCES trading_pairs(id),
    price DECIMAL(30,8) NOT NULL,
    amount DECIMAL(30,8) NOT NULL,
    taker_side VARCHAR(10) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### 2.3 钱包相关表
```sql
-- 钱包地址表
CREATE TABLE wallet_addresses (
    id SERIAL PRIMARY KEY,
    user_id INTEGER REFERENCES users(id),
    currency VARCHAR(20) NOT NULL,
    address VARCHAR(255) NOT NULL,
    private_key_encrypted TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- 转账记录表
CREATE TABLE transfers (
    id SERIAL PRIMARY KEY,
    user_id INTEGER REFERENCES users(id),
    currency VARCHAR(20) NOT NULL,
    amount DECIMAL(30,8) NOT NULL,
    from_address VARCHAR(255),
    to_address VARCHAR(255) NOT NULL,
    tx_hash VARCHAR(255),
    status VARCHAR(20) DEFAULT 'pending',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

## 3. MongoDB 数据库设计

### 3.1 市场数据集合
```javascript
// K线数据
{
    trading_pair: String,
    interval: String,  // 1m, 5m, 15m, 1h, 4h, 1d
    open: Number,
    high: Number,
    low: Number,
    close: Number,
    volume: Number,
    timestamp: Date
}

// 深度数据
{
    trading_pair: String,
    bids: [{price: Number, amount: Number}],
    asks: [{price: Number, amount: Number}],
    timestamp: Date
}
```

### 3.2 用户行为数据集合
```javascript
// 用户操作日志
{
    user_id: ObjectId,
    action: String,
    ip: String,
    user_agent: String,
    timestamp: Date
}

// 用户偏好设置
{
    user_id: ObjectId,
    theme: String,
    language: String,
    notifications: {
        email: Boolean,
        sms: Boolean,
        push: Boolean
    },
    updated_at: Date
}
```

## 4. Redis 数据结构设计

### 4.1 缓存数据
```redis
# 用户会话
SET user:session:{session_id} {user_data} EX 3600

# 市场数据缓存
SET market:{trading_pair}:ticker {ticker_data} EX 1
SET market:{trading_pair}:depth {depth_data} EX 1

# 限流计数器
INCR rate_limit:{ip}:{endpoint}
EXPIRE rate_limit:{ip}:{endpoint} 60
```

### 4.2 消息队列
```redis
# 订单队列
LPUSH order:queue {order_data}

# 交易队列
LPUSH trade:queue {trade_data}
```

## 5. 时序数据库设计

### 5.1 市场数据
```sql
-- 创建连续查询
CREATE CONTINUOUS QUERY market_data_cq ON crypto_db
BEGIN
    SELECT mean(price) AS price,
           sum(volume) AS volume,
           max(high) AS high,
           min(low) AS low
    INTO market_data_1h
    FROM market_data
    GROUP BY time(1h), trading_pair
END

-- 创建保留策略
CREATE RETENTION POLICY market_data_rp ON crypto_db
DURATION 30d REPLICATION 1
```

## 6. 索引设计

### 6.1 PostgreSQL 索引
```sql
-- 用户表索引
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_username ON users(username);

-- 订单表索引
CREATE INDEX idx_orders_user_id ON orders(user_id);
CREATE INDEX idx_orders_trading_pair_id ON orders(trading_pair_id);
CREATE INDEX idx_orders_status ON orders(status);
CREATE INDEX idx_orders_created_at ON orders(created_at);

-- 交易记录表索引
CREATE INDEX idx_trades_trading_pair_id ON trades(trading_pair_id);
CREATE INDEX idx_trades_created_at ON trades(created_at);
```

### 6.2 MongoDB 索引
```javascript
// K线数据索引
db.market_data.createIndex({trading_pair: 1, interval: 1, timestamp: -1})

// 用户操作日志索引
db.user_logs.createIndex({user_id: 1, timestamp: -1})
```

## 7. 数据库优化策略

### 7.1 查询优化
- 使用适当的索引
- 优化复杂查询
- 使用物化视图
- 定期分析表统计信息

### 7.2 性能优化
- 数据库分片
- 读写分离
- 连接池优化
- 定期维护

### 7.3 备份策略
- 全量备份
- 增量备份
- 异地备份
- 定期恢复测试 