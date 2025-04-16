# 智能合约设计文档

## 1. 概述
本文档描述了 Zaka-X-Coin 交易平台的智能合约系统设计。智能合约使用 Solidity 语言开发，运行在 EVM 兼容的区块链上。

## 2. 合约架构

### 2.1 核心合约
```
contracts/
├── core/
│   ├── ZakaToken.sol           # 平台代币合约
│   ├── ZakaFactory.sol         # 工厂合约
│   └── ZakaRouter.sol          # 路由合约
├── exchange/
│   ├── OrderBook.sol           # 订单簿合约
│   ├── MatchingEngine.sol      # 撮合引擎合约
│   └── TradeExecutor.sol       # 交易执行合约
├── staking/
│   ├── StakingPool.sol         # 质押池合约
│   └── RewardDistributor.sol   # 奖励分发合约
└── governance/
    ├── ZakaDAO.sol             # DAO 治理合约
    └── Voting.sol              # 投票合约
```

## 3. 核心合约设计

### 3.1 ZakaToken (ERC20)
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import "@openzeppelin/contracts/access/Ownable.sol";

contract ZakaToken is ERC20, Ownable {
    constructor() ERC20("Zaka Token", "ZAKA") {
        _mint(msg.sender, 1000000000 * 10 ** decimals());
    }

    function mint(address to, uint256 amount) public onlyOwner {
        _mint(to, amount);
    }

    function burn(uint256 amount) public {
        _burn(msg.sender, amount);
    }
}
```

### 3.2 OrderBook
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract OrderBook {
    struct Order {
        address user;
        uint256 price;
        uint256 amount;
        bool isBuy;
        uint256 timestamp;
    }

    mapping(uint256 => Order) public orders;
    uint256 public orderCount;

    event OrderCreated(uint256 orderId, address user, uint256 price, uint256 amount, bool isBuy);
    event OrderCancelled(uint256 orderId);
    event OrderMatched(uint256 buyOrderId, uint256 sellOrderId, uint256 price, uint256 amount);

    function createOrder(uint256 price, uint256 amount, bool isBuy) public returns (uint256) {
        uint256 orderId = orderCount++;
        orders[orderId] = Order({
            user: msg.sender,
            price: price,
            amount: amount,
            isBuy: isBuy,
            timestamp: block.timestamp
        });

        emit OrderCreated(orderId, msg.sender, price, amount, isBuy);
        return orderId;
    }

    function cancelOrder(uint256 orderId) public {
        require(orders[orderId].user == msg.sender, "Not order owner");
        delete orders[orderId];
        emit OrderCancelled(orderId);
    }
}
```

### 3.3 MatchingEngine
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "./OrderBook.sol";

contract MatchingEngine {
    OrderBook public orderBook;

    constructor(address _orderBook) {
        orderBook = OrderBook(_orderBook);
    }

    function matchOrders(uint256 buyOrderId, uint256 sellOrderId) public {
        OrderBook.Order memory buyOrder = orderBook.orders(buyOrderId);
        OrderBook.Order memory sellOrder = orderBook.orders(sellOrderId);

        require(buyOrder.isBuy && !sellOrder.isBuy, "Invalid order types");
        require(buyOrder.price >= sellOrder.price, "Price mismatch");

        uint256 matchAmount = buyOrder.amount <= sellOrder.amount ? buyOrder.amount : sellOrder.amount;
        uint256 matchPrice = (buyOrder.price + sellOrder.price) / 2;

        // Execute trade
        // ...

        emit OrderMatched(buyOrderId, sellOrderId, matchPrice, matchAmount);
    }
}
```

## 4. 质押系统设计

### 4.1 StakingPool
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/security/ReentrancyGuard.sol";

contract StakingPool is ReentrancyGuard {
    IERC20 public stakingToken;
    IERC20 public rewardToken;

    uint256 public totalStaked;
    uint256 public rewardPerToken;
    uint256 public lastUpdateTime;

    mapping(address => uint256) public stakedBalance;
    mapping(address => uint256) public rewards;

    event Staked(address indexed user, uint256 amount);
    event Withdrawn(address indexed user, uint256 amount);
    event RewardPaid(address indexed user, uint256 reward);

    constructor(address _stakingToken, address _rewardToken) {
        stakingToken = IERC20(_stakingToken);
        rewardToken = IERC20(_rewardToken);
    }

    function stake(uint256 amount) external nonReentrant {
        require(amount > 0, "Cannot stake 0");
        
        updateReward(msg.sender);
        stakedBalance[msg.sender] += amount;
        totalStaked += amount;

        stakingToken.transferFrom(msg.sender, address(this), amount);
        emit Staked(msg.sender, amount);
    }

    function withdraw(uint256 amount) external nonReentrant {
        require(amount > 0, "Cannot withdraw 0");
        require(stakedBalance[msg.sender] >= amount, "Insufficient balance");

        updateReward(msg.sender);
        stakedBalance[msg.sender] -= amount;
        totalStaked -= amount;

        stakingToken.transfer(msg.sender, amount);
        emit Withdrawn(msg.sender, amount);
    }

    function getReward() external nonReentrant {
        updateReward(msg.sender);
        uint256 reward = rewards[msg.sender];
        if (reward > 0) {
            rewards[msg.sender] = 0;
            rewardToken.transfer(msg.sender, reward);
            emit RewardPaid(msg.sender, reward);
        }
    }

    function updateReward(address account) internal {
        rewardPerToken = rewardPerToken + (
            (block.timestamp - lastUpdateTime) * 1e18 / totalStaked
        );
        lastUpdateTime = block.timestamp;
        
        if (account != address(0)) {
            rewards[account] = stakedBalance[account] * (rewardPerToken - userRewardPerTokenPaid[account]) / 1e18;
            userRewardPerTokenPaid[account] = rewardPerToken;
        }
    }
}
```

## 5. 治理系统设计

### 5.1 ZakaDAO
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "@openzeppelin/contracts/governance/Governor.sol";
import "@openzeppelin/contracts/governance/extensions/GovernorSettings.sol";
import "@openzeppelin/contracts/governance/extensions/GovernorCountingSimple.sol";
import "@openzeppelin/contracts/governance/extensions/GovernorVotes.sol";

contract ZakaDAO is Governor, GovernorSettings, GovernorCountingSimple, GovernorVotes {
    constructor(IVotes _token)
        Governor("ZakaDAO")
        GovernorSettings(1 days, 7 days, 1000e18)
        GovernorVotes(_token)
    {}

    function votingDelay() public view override(IGovernor, GovernorSettings) returns (uint256) {
        return super.votingDelay();
    }

    function votingPeriod() public view override(IGovernor, GovernorSettings) returns (uint256) {
        return super.votingPeriod();
    }

    function quorum(uint256 blockNumber) public view override returns (uint256) {
        return (token.getPastTotalSupply(blockNumber) * 10) / 100;
    }

    function proposalThreshold() public view override(Governor, GovernorSettings) returns (uint256) {
        return super.proposalThreshold();
    }
}
```

## 6. 安全考虑

### 6.1 常见攻击防护
- 重入攻击防护：使用 ReentrancyGuard
- 整数溢出防护：使用 SafeMath
- 权限控制：使用 OpenZeppelin 的访问控制
- 闪电贷攻击防护：添加时间锁

### 6.2 审计要点
- 代码审查
- 单元测试
- 集成测试
- 形式化验证
- 第三方审计

## 7. 部署策略

### 7.1 部署流程
1. 测试网部署
2. 安全审计
3. 主网部署
4. 监控和维护

### 7.2 升级策略
- 使用代理模式
- 版本控制
- 紧急暂停机制
- 升级投票机制

## 8. 测试方案

### 8.1 单元测试
```javascript
const { expect } = require("chai");
const { ethers } = require("hardhat");

describe("ZakaToken", function () {
  let ZakaToken;
  let zakaToken;
  let owner;
  let addr1;
  let addr2;

  beforeEach(async function () {
    [owner, addr1, addr2] = await ethers.getSigners();
    ZakaToken = await ethers.getContractFactory("ZakaToken");
    zakaToken = await ZakaToken.deploy();
    await zakaToken.deployed();
  });

  describe("Deployment", function () {
    it("Should set the right owner", async function () {
      expect(await zakaToken.owner()).to.equal(owner.address);
    });

    it("Should assign the total supply of tokens to the owner", async function () {
      const ownerBalance = await zakaToken.balanceOf(owner.address);
      expect(await zakaToken.totalSupply()).to.equal(ownerBalance);
    });
  });
});
```

### 8.2 集成测试
```javascript
describe("StakingPool", function () {
  let StakingPool;
  let stakingPool;
  let stakingToken;
  let rewardToken;

  beforeEach(async function () {
    [owner, addr1] = await ethers.getSigners();
    
    // Deploy tokens
    const Token = await ethers.getContractFactory("ZakaToken");
    stakingToken = await Token.deploy();
    rewardToken = await Token.deploy();
    
    // Deploy staking pool
    StakingPool = await ethers.getContractFactory("StakingPool");
    stakingPool = await StakingPool.deploy(stakingToken.address, rewardToken.address);
    await stakingPool.deployed();
  });

  describe("Staking", function () {
    it("Should stake tokens correctly", async function () {
      await stakingToken.approve(stakingPool.address, 1000);
      await stakingPool.stake(1000);
      expect(await stakingPool.stakedBalance(owner.address)).to.equal(1000);
    });
  });
});
``` 