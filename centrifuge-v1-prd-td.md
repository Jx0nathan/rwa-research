# Centrifuge V1 (Tinlake) — PRD + TD

> 版本：V1 / Tinlake  
> 部署网络：Ethereum Mainnet  
> 时间线：2020-2022  
> 状态：已停止新池部署，存量池仍在运行

---

# Part 1: PRD — 产品需求文档

## 1. 产品定位

Centrifuge V1（Tinlake）是一个部署在 Ethereum 上的链上资产支持证券（ABS）协议。

**核心价值主张：**
- 资产发起方（Asset Originator）将现实世界资产（RWA，如应收账款、贸易融资、抵押贷款）代币化后，在链上募集资金
- 投资人通过购买分级代币参与投资，根据风险偏好选择高级（DROP）或劣后（TIN）档位
- 整个清结算、利息分配、赎回流程在链上完成，无需传统 SPV 的人工对账

**类比：** 链上版 CLO（担保贷款凭证）/ ABS，发行方是中小企业或 FinTech 借贷平台，投资人是 DeFi 机构。

---

## 2. 核心用户角色

| 角色 | 英文 | 职责 | 链上操作 |
|---|---|---|---|
| 资产发起方 | Asset Originator / Issuer | 创建资金池，将贷款铸造为 NFT，管理贷款生命周期 | 铸造 Asset NFT、提款（借款）、还款 |
| 高级投资人 | DROP Investor | 购买固定收益型代币 DROP，优先受偿 | 申购/赎回 DROP |
| 劣后投资人 | TIN Investor | 购买浮动收益型代币 TIN，首先承担损失 | 申购/赎回 TIN |
| 池管理员 | Pool Admin | 设置风险参数，管理白名单 | 设置利率、质押率、NAV 参数 |
| MakerDAO | 外部协议 | 作为高级流动性提供方（MKR 集成） | DAI 自动注入池子 |

---

## 3. 核心功能模块

### 3.1 资产代币化模块

| 功能 | 描述 |
|---|---|
| NFT 铸造 | 每笔贷款铸造为一个 ERC-721 NFT，元数据包含借款金额、到期日、利率、借款人信息 |
| NFT 抵押 | Issuer 将 NFT 锁入合约作为抵押，换取资金 |
| 贷款管理 | 支持还款、提前还款、违约标记操作 |
| 折现率设置 | 每个 NFT 可独立设置折现率（Discount Rate），用于 NAV 计算 |

### 3.2 分级（Tranching）模块

| 档位 | 代币 | 特性 |
|---|---|---|
| 高级档（Senior） | DROP | 固定利率（如 5% APR），优先分配现金流，最后承担损失 |
| 劣后档（Junior） | TIN | 浮动收益，首先吸收损失，超额收益归属 TIN 持有人 |

**关键约束：**
- `TIN 最小比例`（Min TIN Ratio）：TIN 在池中的占比不能低于某个阈值（如 10%），确保对 DROP 的保护缓冲
- 当 TIN 比例低于阈值时，新的 DROP 申购被暂停

### 3.3 Epoch 机制（定期结算）

**问题背景：** DeFi 流动性和链下资产流动性不匹配——投资人随时想进出，但贷款有固定期限。

**解决方案：** Epoch（时间窗口）机制

| 阶段 | 描述 |
|---|---|
| Epoch 开放期 | 投资人提交申购/赎回订单（不立即执行） |
| Epoch 关闭 | 任何人可调用 `closeEpoch()` 触发结算 |
| 求解阶段 | 链上优化器（Coordinator）计算最优执行比例 |
| 执行阶段 | 订单按计算结果部分或全部成交 |

**订单优先级（现金有限时）：**
1. DROP 赎回（最高优先）
2. TIN 赎回
3. DROP 申购
4. TIN 申购

### 3.4 NAV（净资产价值）计算模块

**NAV = 所有贷款的现值之和**

```
单笔贷款现值 = 预期偿还金额 / (1 + 折现率)^剩余天数/365
Pool NAV = Σ(各笔贷款现值) - 待偿还费用
```

**代币价格计算：**
```
Senior Token Price = (Senior NAV + Senior Cash) / Senior Token Supply
Junior Token Price = (Pool NAV - Senior NAV) / TIN Supply
```

当发生违约时，NAV 下降首先冲击 TIN 价格，DROP 价格不受影响直到 TIN 归零。

### 3.5 MakerDAO 集成模块

- Tinlake 池可接入 MKR 的 DAI 信用额度（Debt Ceiling）
- MKR 自动化库（Clip）根据 DROP 价值自动注入/回收 DAI
- 有效扩大高级档流动性，无需外部 DROP 投资人

---

## 4. 业务流程

### 4.1 资产生命周期

```
1. Issuer 在链下签署贷款合同
2. Issuer 铸造 NFT（代表该笔贷款）
3. NFT 锁入 Shelf 合约
4. Issuer 调用 borrow() 提取 DAI
5. 贷款到期后，Issuer 调用 repay()
6. NAV 更新，分配利息
7. NFT 解锁销毁
```

### 4.2 投资人流程

```
1. 投资人完成 KYC（链下，白名单机制）
2. Epoch 开放期内提交申购订单（transferIn）
3. Epoch 结束，Coordinator 计算成交量
4. 部分或全部成交，获得 DROP/TIN 代币
5. 到期赎回：提交赎回订单 → 等待下一 Epoch 结算 → 收回 DAI
```

---

## 5. 验收标准 / 核心指标

| 指标 | 说明 |
|---|---|
| NAV 准确性 | 链上 NAV 与链下 Excel 计算误差 < 0.01% |
| Epoch 结算时间 | Gas 优化后 < 500k gas per epoch |
| TIN 最小比例保护 | 低于阈值时自动暂停 DROP 申购 |
| 违约处理 | 违约 NFT 标记后，NAV 在下一次计算中扣除 |

---

# Part 2: TD — 技术方案文档

## 1. 系统架构

```
┌─────────────────────────────────────────────────────────┐
│                    Ethereum Mainnet                       │
│                                                           │
│  ┌──────────┐    ┌──────────┐    ┌──────────────────┐   │
│  │  Shelf   │    │  Pile    │    │   Assessor       │   │
│  │(NFT托管) │    │(贷款账本)│    │  (价格/NAV计算)  │   │
│  └────┬─────┘    └────┬─────┘    └────────┬─────────┘   │
│       │               │                    │              │
│  ┌────▼───────────────▼────────────────────▼──────────┐  │
│  │                  Pool (Root)                        │  │
│  │              核心路由 + 权限管理                     │  │
│  └───────────┬──────────────────────┬─────────────────┘  │
│              │                      │                     │
│  ┌───────────▼────┐      ┌──────────▼──────────────────┐ │
│  │  Tranche       │      │   Coordinator               │ │
│  │  (DROP/TIN     │      │  (Epoch 求解器)              │ │
│  │   ERC20 代币)  │      │                             │ │
│  └────────────────┘      └─────────────────────────────┘ │
│                                                           │
│  ┌──────────────────────────────────────────────────┐    │
│  │  Reserve (资金池)  ←→  MakerDAO DAI Adapter       │    │
│  └──────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────┘
```

## 2. 核心合约清单

| 合约 | 文件路径 | 职责 |
|---|---|---|
| `Shelf` | `src/lender/shelf.sol` | NFT 托管，贷款提款/还款入口 |
| `Pile` | `src/borrower/pile.sol` | 贷款利率计算（复利），贷款状态管理 |
| `Assessor` | `src/lender/assessor.sol` | NAV 计算，DROP/TIN 价格计算，权重计算 |
| `Tranche` | `src/lender/tranche.sol` | DROP/TIN ERC-20 代币，申购赎回订单 |
| `Coordinator` | `src/lender/coordinator.sol` | Epoch 求解器，线性规划优化 |
| `Reserve` | `src/lender/reserve.sol` | 资金池管理，资金调度 |
| `Root` | `src/root.sol` | 权限管理，合约注册中心 |
| `Feed` | `src/lender/nftfeed.sol` | NFT 价值预言机，设置折现率 |

## 3. 数据模型

### 3.1 贷款数据（Pile）

```solidity
struct Loan {
    uint256 principal;      // 本金（WAD = 1e18）
    uint256 interestRatePerSecond; // 每秒利率（RAY = 1e27）
    uint256 normalizedDebt; // 标准化债务（debt = normalizedDebt * rateAccumulator）
    uint256 rateGroup;      // 利率组ID（同一利率的贷款共享accumulator）
}

struct Rate {
    uint256 pie;            // 该利率组总债务（标准化）
    uint256 chi;            // 利率累加器（初始 1 RAY，随时间增长）
    uint256 ratePerSecond;  // 每秒利率
    uint48  lastUpdated;    // 上次更新时间
}
```

### 3.2 Epoch 数据（Coordinator）

```solidity
struct EpochOrder {
    uint256 dropInvest;   // DROP 申购总量（DAI）
    uint256 dropRedeem;   // DROP 赎回总量（DROP token）
    uint256 tinInvest;    // TIN 申购总量（DAI）
    uint256 tinRedeem;    // TIN 赎回总量（TIN token）
}

struct EpochResult {
    uint256 dropInvestFulfillment;  // DROP申购成交比例（RAY）
    uint256 dropRedeemFulfillment;  // DROP赎回成交比例（RAY）
    uint256 tinInvestFulfillment;   // TIN申购成交比例（RAY）
    uint256 tinRedeemFulfillment;   // TIN赎回成交比例（RAY）
}
```

## 4. 关键算法

### 4.1 利率计算（复利）

```solidity
// Pile.sol 中的复利累加
function drip(uint256 rateGroup) public {
    Rate storage rate = rates[rateGroup];
    uint256 elapsed = block.timestamp - rate.lastUpdated;
    // chi = chi * (1 + ratePerSecond)^elapsed
    rate.chi = rmul(rpow(rate.ratePerSecond, elapsed, ONE), rate.chi);
    rate.lastUpdated = block.timestamp;
}

// 当前债务 = normalizedDebt * chi
function debt(uint256 loan) public view returns (uint256) {
    return rmul(loans[loan].normalizedDebt, rates[loans[loan].rateGroup].chi);
}
```

### 4.2 Epoch 求解（线性规划）

Coordinator 使用启发式搜索（非标准 LP solver，因为链上 Gas 限制）：

```
目标：最大化资金利用率，同时满足约束条件

约束条件：
1. 池子 NAV 变化后，TIN 比例 ≥ minTINRatio
2. 可用现金 ≥ 0（不能透支）
3. DROP 赎回优先于 TIN 赎回

求解顺序：
Step 1: 尝试全量成交（100% fulfillment）
Step 2: 检查所有约束
Step 3: 若约束不满足，按优先级依次减少低优先级订单
Step 4: 直到找到满足所有约束的解
```

### 4.3 NAV 计算（Feed）

```solidity
// 单笔贷款现值
function calcDiscount(
    uint256 rate,       // 折现率（APR）
    uint256 futureValue, // 到期应收金额
    uint48  maturityDate // 到期日
) public view returns (uint256) {
    if (block.timestamp >= maturityDate) return futureValue;
    uint256 daysLeft = (maturityDate - block.timestamp) / 1 days;
    // PV = FV / (1 + rate)^days
    return rdiv(futureValue, rpow(rate, daysLeft, ONE));
}

// 池子总 NAV
function calcNAV() public view returns (uint256 nav) {
    for (uint i = 0; i < loans.length; i++) {
        nav += calcDiscount(rate[i], futureValue[i], maturity[i]);
    }
}
```

## 5. 安全考量

| 风险 | 缓解措施 |
|---|---|
| NFT 估值操纵 | Feed 合约设置白名单，只有授权 Oracle 可更新 NAV 参数 |
| 重入攻击 | 所有状态变更前先更新余额（Checks-Effects-Interactions） |
| 利率累加器溢出 | 使用 SafeMath + 限制最大利率 |
| Epoch 无人触发 | 任何人都可调用 `closeEpoch()`，无需权限 |
| 流动性危机 | TIN 最小比例保护 + Maker 集成作为流动性后盾 |

## 6. 已知局限性（V1 → V2 升级原因）

1. **只支持 DAI**：单币种限制了市场规模
2. **每池只有两档**：不支持三级以上分层
3. **NAV 需要链下人工输入**：折现率、未来价值需 Issuer 手动更新，存在操纵风险
4. **Gas 费用高**：Epoch 求解在链上运算复杂，高峰期 Gas 成本高
5. **流动性差**：DROP/TIN 二级市场几乎不存在

---
*文档版本：1.0 | 基于 Tinlake GitHub: https://github.com/centrifuge/tinlake*
