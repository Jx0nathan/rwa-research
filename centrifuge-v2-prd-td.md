# Centrifuge V2 — PRD + TD

> 版本：V2（Tinlake 升级版 + Centrifuge Chain 引入）  
> 部署网络：Ethereum + Centrifuge Chain (Substrate)  
> 时间线：2022-2023  
> 状态：部分功能已迁移至 V3

---

# Part 1: PRD — 产品需求文档

## 1. 产品定位

V2 是对 V1 Tinlake 的系统性升级，主要解决三个核心问题：
1. **NAV 可信度**：V1 的 NAV 由 Issuer 手动更新，V2 引入更严格的估值框架
2. **多资产支持**：不再绑定 DAI，支持任意 ERC-20 作为池子货币
3. **跨链准备**：引入 Centrifuge Chain（基于 Substrate 的专用区块链），为 V3 Hub-and-Spoke 架构奠定基础

**类比：** 如果 V1 是手工作坊，V2 是引入了标准化流程和质检体系的工厂。

---

## 2. 核心改进点（相对 V1）

| 模块 | V1 | V2 改进 |
|---|---|---|
| NAV 计算 | Issuer 手动设置折现率 | 引入风险评分模型，限制 Issuer 权限 |
| 货币 | 仅 DAI | 任意 ERC-20（USDC、USDT、DAI 等） |
| 分档 | 固定 DROP/TIN 两档 | 支持参数化配置，灵活调整 |
| 合规 | 简单白名单 | 引入 Member List v2，支持机构 KYC 场景 |
| 链上基础设施 | 纯 Ethereum | 引入 Centrifuge Chain 平行链 |
| 利率模型 | 固定利率组 | 引入动态利率（基于风险评分） |

---

## 3. 核心用户角色（新增/变化）

| 角色 | V1 状态 | V2 变化 |
|---|---|---|
| Asset Originator | 完全控制 NAV | 受限：不能随意修改已设置的折现率 |
| DROP Investor | 仅白名单 | 引入更完整的 KYC 流程，支持机构准入 |
| TIN Investor | 同上 | 可设置 TIN 最高收益上限（cap） |
| 风险评估员 | 无此角色 | 新增：可独立于 Issuer 设置资产风险参数 |

---

## 4. 核心功能模块

### 4.1 增强型 NAV 框架

**V2 NAV 模型：两种模式**

| 模式 | 场景 | 计算方式 |
|---|---|---|
| 折现现金流（DCF） | 定期还款贷款（如抵押贷款） | 基于剩余现金流折现 |
| 外部价格（Oracle） | 流动性较好资产（如企业债） | 链下价格喂入，有效期限制 |

**NAV 更新流程改进：**
```
V1: Issuer 随时可更新任意资产折现率
V2: 
  - 初始折现率由 Risk Scoring Model 计算
  - 更新需满足：在有效期内 + 权限验证
  - 过期资产自动降级估值（penalty rate）
```

### 4.2 多货币资金池

```
V1: Pool Currency = DAI（硬编码）
V2: Pool Currency = 任意 ERC-20

技术实现：
- 每个池子部署时指定 currency 地址
- Reserve 合约适配任意 ERC-20 的 transfer/transferFrom
- 价格计算统一以 pool currency 计价
```

### 4.3 改进的 Epoch 机制

**V2 Epoch 主要变化：**

1. **订单有效期**：申购/赎回订单可设置过期时间
2. **部分成交优化**：更精确的成交比例计算（减少精度损失）
3. **批量结算**：支持多个用户订单在一次 Epoch 中批量处理
4. **Epoch 最短时间**：设置最小 Epoch 时长（防止频繁触发 Gas 浪费）

### 4.4 Centrifuge Chain 集成（预备）

V2 阶段开始引入 Centrifuge Chain 作为：
- RWA 资产的唯一标识注册中心（Asset Registry）
- 跨链消息传递的协调层
- 未来 V3 Hub 的早期形态

**V2 阶段的跨链范围：** 有限，主要是资产元数据同步，资金仍在 Ethereum 上流转

---

## 5. 业务流程（V2 完整流程）

### 5.1 Pool 创建流程

```
1. Issuer 在 Centrifuge Chain 上注册 Asset Originator 身份
2. 部署 Pool 合约集（Root, Shelf, Pile, Tranche×2, Coordinator, Reserve, Feed）
3. 配置参数：
   - Pool Currency（如 USDC）
   - DROP 利率（如 5% APR）
   - MIN_TIN_RATIO（如 10%）
   - 最大池规模（Max Pool Size）
   - Epoch 最短时长（Min Epoch Duration）
4. 通过 Centrifuge 治理审核
5. 开放投资人准入（KYC 白名单）
```

### 5.2 贷款生命周期（V2 增强）

```
1. 贷款评估：风险评分模型计算 PD（违约概率）、LGD（违约损失率）
2. 折现率设置：discountRate = riskFreeRate + spread(PD, LGD)
3. NFT 铸造：包含更丰富的元数据（借款人评分、到期日、还款计划）
4. 资金提取：Issuer 从 Reserve 提取对应金额
5. 还款跟踪：支持分期还款场景
6. 违约处理：
   - 标记 NFT 为违约状态
   - 触发 NAV 减记（write-down）
   - TIN 价格首先下降
```

### 5.3 投资人流程（V2 细节）

```
申购：
1. 投资人通过 KYC（链下）→ 添加到 MemberList
2. 调用 Tranche.submitOrder(投资金额) → 订单记入当前 Epoch
3. Epoch 关闭后，Coordinator 计算成交比例
4. 调用 Tranche.disburse() → 获得 DROP/TIN 代币

赎回：
1. 持有 DROP/TIN 代币的投资人调用 submitOrder(赎回金额)
2. 等待 Epoch 结算
3. 成交后收回 pool currency（如 USDC）

注意：赎回不是即时的，最快需要等待下一个 Epoch 周期（可能是 24h-7天）
```

---

## 6. 验收标准

| 模块 | 验收标准 |
|---|---|
| 多货币 | 用 USDC 部署池子，完整走通申购→贷款→还款→赎回流程 |
| NAV 过期保护 | 超过 NAV 更新有效期后，资产自动用 penalty rate 计算 |
| KYC 白名单 | 非白名单地址调用 submitOrder 应 revert |
| Epoch 最短时长 | 在 minEpochDuration 内调用 closeEpoch 应 revert |

---

# Part 2: TD — 技术方案文档

## 1. 系统架构

```
┌─────────────────────────────────────────────────────────────────┐
│                      Ethereum Mainnet                            │
│                                                                   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                    Pool Contract Suite                    │   │
│  │                                                           │   │
│  │  ┌─────────┐  ┌──────────┐  ┌──────────────────────┐   │   │
│  │  │  Shelf  │  │   Pile   │  │   NavFeed (V2)        │   │   │
│  │  │ NFT托管 │  │ 利率引擎 │  │ DCF + Oracle双模式   │   │   │
│  │  └────┬────┘  └────┬─────┘  └──────────┬───────────┘   │   │
│  │       └────────────┼──────────────────┘               │   │
│  │                    ▼                                    │   │
│  │  ┌─────────────────────────────────────────────────┐   │   │
│  │  │                  Assessor V2                     │   │   │
│  │  │  (支持多货币 + 改进NAV聚合 + 动态参数)           │   │   │
│  │  └─────────────────────────────────────────────────┘   │   │
│  │            ↑                        ↑                   │   │
│  │  ┌─────────┴──────┐      ┌──────────┴─────────────┐   │   │
│  │  │  Tranche DROP  │      │   Tranche TIN          │   │   │
│  │  │  (ERC-20)      │      │   (ERC-20)             │   │   │
│  │  └────────────────┘      └────────────────────────┘   │   │
│  │                    ↑                                    │   │
│  │  ┌─────────────────┴────────────────────────────────┐  │   │
│  │  │            Coordinator V2                        │  │   │
│  │  │     (改进Epoch求解 + 最短Epoch时长保护)          │  │   │
│  │  └──────────────────────────────────────────────────┘  │   │
│  │                    ↑                                    │   │
│  │  ┌─────────────────┴───────────────────────────────┐   │   │
│  │  │        Reserve (多货币 ERC-20 适配)              │   │   │
│  │  └─────────────────────────────────────────────────┘   │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                               │
└─────────────────────────────────────────────────────────────────┘
          ↕ 跨链消息（资产元数据同步，V2 有限使用）
┌───────────────────────────────────────────────────────────────┐
│               Centrifuge Chain (Substrate Parachain)          │
│                                                               │
│   Asset Registry  |  Identity (DID)  |  早期跨链协调          │
└───────────────────────────────────────────────────────────────┘
```

## 2. 核心合约变化（V1 → V2 diff）

### 2.1 NavFeed V2（核心升级）

```solidity
contract NavFeedV2 {
    // V2 新增：价值更新的有效期
    uint256 public constant PRICE_EXPIRY = 1 days;
    
    struct NFTValue {
        uint256 nftID;
        uint256 value;          // 当前估值
        uint256 futureValue;    // 到期应收
        uint48  maturityDate;   // 到期日
        uint256 risk;           // 风险分类（0-7级）
        uint256 updatedAt;      // 最后更新时间
    }
    
    // 风险分类 → 折现率映射
    mapping(uint256 => uint256) public riskGroupDiscountRate;
    
    // V2 关键：过期保护
    function currentNAV() public view returns (uint256 nav) {
        for (uint i = 0; i < activeLoans.length; i++) {
            NFTValue memory v = nftValues[activeLoans[i]];
            if (block.timestamp - v.updatedAt > PRICE_EXPIRY) {
                // 超期：使用 penalty rate（更高折现率）
                nav += calcDiscount(penaltyRate, v.futureValue, v.maturityDate);
            } else {
                nav += calcDiscount(riskGroupDiscountRate[v.risk], v.futureValue, v.maturityDate);
            }
        }
    }
}
```

### 2.2 Reserve V2（多货币支持）

```solidity
contract ReserveV2 {
    ERC20 public currency;  // V2: 不再硬编码 DAI
    
    constructor(address _currency) {
        currency = ERC20(_currency);
    }
    
    function deposit(uint256 amount) public {
        currency.transferFrom(msg.sender, address(this), amount);
        balance += amount;
    }
    
    function payout(address to, uint256 amount) public auth {
        require(balance >= amount, "insufficient reserve");
        currency.transfer(to, amount);
        balance -= amount;
    }
}
```

### 2.3 Coordinator V2（改进 Epoch）

```solidity
contract CoordinatorV2 {
    uint256 public minimumEpochTime;  // V2 新增：最短 Epoch 时长
    uint256 public lastEpochClosed;
    
    function closeEpoch() public {
        // V2 新增：防止过于频繁触发
        require(
            block.timestamp >= lastEpochClosed + minimumEpochTime,
            "min epoch time not reached"
        );
        lastEpochClosed = block.timestamp;
        // ... 执行求解逻辑
    }
}
```

## 3. 数据模型（V2 新增/变更）

### 3.1 风险分组

```solidity
// 7级风险分类（V2 引入）
// Risk 0 = 最低风险，折现率最低（如 3%）
// Risk 7 = 最高风险，折现率最高（如 15%）
mapping(uint256 => RiskGroup) public riskGroups;

struct RiskGroup {
    uint256 ceilingRatio;       // 最大质押率（贷款/资产价值）
    uint256 thresholdRatio;     // 清算触发阈值
    uint256 recoveryRatePD;     // 违约回收率（Probability × Rate）
    uint256 discountRate;       // NAV 折现率
}
```

### 3.2 MemberList V2（KYC 增强）

```solidity
struct Member {
    address investor;
    uint256 validUntil;    // V2 新增：KYC 有效期（需定期更新）
}

// 申购/赎回时校验
modifier onlyValidMember(address user) {
    require(memberList[user].validUntil >= block.timestamp, "member expired");
    _;
}
```

## 4. 关键算法（V2 改进）

### 4.1 改进的 Epoch 求解

V2 在 V1 启发式搜索基础上，增加了二分搜索：

```
V1 求解：顺序减少低优先级订单（线性，可能多次迭代）
V2 求解：
  1. 先尝试全量成交
  2. 如失败，对每个订单类型使用二分搜索找到最大可成交量
  3. 验证约束（TIN 最小比例 + 资金充足）
  4. 总体 Gas 消耗降低约 30%
```

### 4.2 动态折现率（基于风险组）

```
贷款折现率 = riskFreeRate + riskPremium[riskGroup]

例：
- 风险组 1（低风险）：5%（基础利率 3% + 风险溢价 2%）
- 风险组 5（中高风险）：12%（基础利率 3% + 风险溢价 9%）

NAV 对风险组变化敏感：
- 贷款从风险组 1 升至风险组 5 → NAV 立即重新计算，池子 NAV 下降
```

## 5. 安全模型（V2 增强）

| 攻击向量 | V1 防护 | V2 改进 |
|---|---|---|
| NAV 操纵 | 仅 Issuer 可更新（中心化风险） | 引入风险组约束，Issuer 只能在限定范围内调整 |
| 闪电贷攻击 | 无保护 | Epoch 机制天然防御（无法在单个区块内申购+赎回） |
| KYC 绕过 | 简单地址检查 | 加入有效期校验，过期 KYC 自动失效 |
| 极低 TIN 场景 | MIN_TIN_RATIO | 保持，增加了 TIN 赎回顺序优先级逻辑优化 |

## 6. V2 局限性（→ V3 升级原因）

1. **仍是单链架构**：所有资金在 Ethereum，无法利用 L2/其他链的流动性
2. **ERC-20 Tranche 代币不可组合**：KYC 限制导致 DROP/TIN 无法在 Uniswap 等 DEX 流通
3. **批量操作 Gas 成本高**：大型机构的批量操作在 Ethereum 主网成本高昂
4. **池子扩展性受限**：每个 Pool 是独立合约部署，代码升级困难
5. **机构需求未满足**：不支持多个 Share Class，不满足基金结构要求
6. **无法与 ERC-4626 生态集成**：V2 Tranche 接口与新兴标准不兼容

---
*文档版本：1.0 | 基于 Centrifuge Docs + Tinlake 合约分析*
