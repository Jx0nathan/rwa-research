# Centrifuge V3 — PRD + TD

> 版本：V3（当前生产版本）  
> 部署网络：Centrifuge Chain (Hub) + Ethereum / Base / Arbitrum / 更多 EVM 链 (Spokes)  
> 时间线：2023 至今  
> 状态：生产运行，持续迭代

---

# Part 1: PRD — 产品需求文档

## 1. 产品定位

V3 是 Centrifuge 的架构重构版本，从"以太坊上的 DeFi 协议"演变为"多链 RWA 基础设施层"。

**核心范式转变：**

| 维度 | V1/V2 | V3 |
|---|---|---|
| 架构模式 | 单链（Ethereum） | Hub-and-Spoke（多链） |
| 定位 | DeFi 协议 | 机构级 RWA 基础设施 |
| 标准 | 自定义接口 | ERC-4626 + ERC-7540 |
| 分档模型 | DROP/TIN 两档 | Share Class（多档，可配置） |
| 合规模型 | 白名单 | 基于 Hook 的可扩展合规层 |
| 流动性来源 | 单链 | 多链并行（资金在不同链上） |

**目标用户升级：**
V1/V2 主要服务中小型 DeFi 用户 → V3 主要面向**机构级投资人**（Aave、MakerDAO、养老基金等）

---

## 2. Hub-and-Spoke 架构详解

### 2.1 核心概念

```
                    ┌─────────────────────────────────┐
                    │     Centrifuge Chain (Hub)       │
                    │                                  │
                    │  - 所有 Pool 的权威状态           │
                    │  - Investment Manager            │
                    │  - Tranche Token 发行管理        │
                    │  - 跨链消息协调                   │
                    └──────────────┬──────────────────┘
                                   │
              ┌────────────────────┼──────────────────┐
              │                    │                  │
    ┌─────────▼──────┐   ┌─────────▼──────┐  ┌──────▼──────┐
    │  Ethereum      │   │   Base         │  │  Arbitrum   │
    │  Liquidity     │   │   Liquidity    │  │  Liquidity  │
    │  Pool          │   │   Pool         │  │  Pool       │
    │  (Spoke)       │   │   (Spoke)      │  │  (Spoke)    │
    └────────────────┘   └────────────────┘  └─────────────┘
    
    每条 Spoke 上有独立的 LiquidityPool 合约
    投资人在自己熟悉的链上操作
    资金/状态通过跨链消息同步到 Hub
```

### 2.2 Hub 职责

- 维护 Pool 的权威 NAV、总资产、总负债
- 管理所有 Tranche Token 的铸造/销毁
- 执行 Epoch 结算逻辑
- 存储合规规则（KYC/AML 状态）

### 2.3 Spoke 职责

- 接收投资人的申购/赎回请求（本链 ERC-20 资金）
- 将请求通过跨链消息传至 Hub
- 接收 Hub 返回的结算指令，完成代币发放或资金返还

---

## 3. 核心功能模块

### 3.1 Share Class（多档位系统）

V3 用 **Share Class** 替代固定的 DROP/TIN 概念：

| 特性 | 描述 |
|---|---|
| 数量灵活 | 一个 Pool 可以有 1-N 个 Share Class |
| 独立 NAV | 每个 Share Class 独立计算 NAV 和 Token Price |
| 独立权限 | 不同 Share Class 可以针对不同投资人群体开放 |
| 多货币 | 同一 Share Class 可以在不同链上以不同货币计价 |

**典型配置示例：**
```
Pool: Anemoy Treasury Fund
  Share Class A (Senior):
    - 目标收益：5% APR
    - 最低投资：$100,000
    - 允许投资人：KYC 白名单
  Share Class B (Junior):
    - 浮动收益
    - 最低投资：$500,000
    - 允许投资人：合格机构投资人（AI）
```

### 3.2 ERC-7540（异步 Vault）

**问题：** ERC-4626 是同步的（存入立即获得份额），但 RWA 需要等待 Epoch 结算，无法同步。

**ERC-7540 解决方案：** 将申购/赎回拆分为"请求"和"领取"两个步骤：

```
申购流程：
Step 1: investor.requestDeposit(assets, receiver)  → 生成 Request ID
  (资金锁定，等待 Epoch 结算)
Step 2: [Epoch 结算后] 
Step 3: investor.deposit(assets, receiver)          → 领取 Share

赎回流程：
Step 1: investor.requestRedeem(shares, receiver)   → 生成 Request ID
  (Share 锁定，等待 Epoch 结算)
Step 2: [Epoch 结算后]
Step 3: investor.redeem(shares, receiver)          → 领取资产
```

**标准兼容性：** ERC-7540 继承 ERC-4626，完全兼容 DeFi 生态。

### 3.3 Hook 合规框架

V3 用**可插拔 Hook** 替代硬编码白名单，实现合规逻辑的灵活扩展：

```
投资人操作 → LiquidityPool → Hook.checkBeforeTransfer()
                                     ↓
                           KYC Hook | AML Hook | Jurisdiction Hook
                                     ↓
                              通过 or 拒绝（附拒绝原因码）
```

**已有 Hook 类型：**
| Hook | 功能 |
|---|---|
| RestrictionManager | 基础白名单（地址级） |
| TransferRestrictions | 限制转让（锁定期、地区限制） |
| LimitHook | 限制单笔/累计投资额度 |

**自定义 Hook：** 机构可部署自己的 Hook 合约，满足特定监管要求。

### 3.4 跨链消息系统

V3 支持多种跨链桥接方案：

| 桥 | 特性 | 适用场景 |
|---|---|---|
| Axelar | 通用跨链，较成熟 | Ethereum ↔ Centrifuge Chain |
| Wormhole | 高速，支持多链 | Base、Arbitrum 等 |
| LayerZero | 轻量级，Gas 低 | L2 场景 |

**消息类型：**
```
Spoke → Hub:
  - Transfer(investorAddr, poolId, shareClassId, currency, amount) // 申购请求
  - RedeemRequest(investorAddr, poolId, shareClassId, shares)      // 赎回请求

Hub → Spoke:
  - FulfilledDepositRequest(requestId, mintedShares)               // 申购成交
  - FulfilledRedeemRequest(requestId, returnedAssets)              // 赎回成交
  - UpdatePrice(shareClassId, price)                               // 价格更新
```

### 3.5 Pool Manager（资金池管理）

V3 引入统一的 PoolManager 作为 Pool 生命周期管理层：

| 功能 | 描述 |
|---|---|
| Pool 创建 | 注册新 Pool，分配 Pool ID |
| Share Class 管理 | 添加/暂停/激活 Share Class |
| 链部署管理 | 为 Pool 开放新的 Spoke 链 |
| 资产托管 | 与 Escrow 合约交互，管理待结算资金 |

---

## 4. 业务流程（V3 完整）

### 4.1 Pool 创建与上线

```
1. Issuer 在 Centrifuge Chain 上提交 Pool 申请（治理投票或白名单授权）
2. 部署 PoolManager，注册 Pool ID
3. 配置 Share Class（N 档位，各自参数）
4. 为目标链（如 Ethereum、Base）部署 LiquidityPool Spoke 合约
5. 配置 Hook（合规规则）
6. 开放 KYC 流程（链下），将合格投资人添加到 Hook 白名单
7. Pool 上线，开放申购
```

### 4.2 投资人申购流程（跨链场景）

```
场景：投资人在 Ethereum 上用 USDC 投资部署在 Base 上的 Pool

1. 投资人调用 Ethereum LiquidityPool.requestDeposit(USDC, amount)
2. Spoke 合约锁定 USDC，发送跨链消息到 Centrifuge Chain
3. Hub 的 InvestmentManager 收到消息，记录 PendingDeposit
4. Epoch 触发（由 Pool Admin 或自动化调用）
5. Hub 执行 Epoch 结算：
   a. 聚合所有链的申购/赎回请求
   b. 计算最优成交比例（满足 Share Class 约束）
   c. 铸造/销毁对应数量的 Tranche Token
6. Hub 向各 Spoke 发送 FulfilledDepositRequest 消息
7. 投资人调用 EthereumLiquidityPool.deposit() → 获得 Tranche ERC-20
```

### 4.3 NAV 更新流程

```
1. Pool Admin 链下计算 NAV（或接入第三方估值服务）
2. 调用 Centrifuge Chain 上的 PoolManager.setNAV(poolId, nav)
3. 各 Share Class 的 Token Price 自动重算：
   Price = (Share Class NAV) / (Share Token Supply)
4. Hub 向所有 Spoke 广播新价格
5. 投资人可查询最新 Token Price
```

---

## 5. 产品约束与边界条件

| 约束 | 说明 |
|---|---|
| Epoch 周期 | 通常 24h-7天，Pool Admin 可配置 |
| 最小投资额 | Share Class 级别，机构池通常 $100K+ |
| KYC 必须 | 所有 Share Class 至少有基础白名单 Hook |
| 跨链延迟 | 跨链消息 1-10 分钟，Epoch 结算后才生效 |
| Gas 费用 | Spoke 链操作由投资人支付；Hub 操作由 Pool 账户支付 |

---

# Part 2: TD — 技术方案文档

## 1. 完整系统架构

```
┌────────────────────────────────────────────────────────────────────┐
│                    Centrifuge Chain (Hub)                           │
│                                                                     │
│  ┌──────────────────┐  ┌──────────────────┐  ┌─────────────────┐  │
│  │   PoolManager    │  │InvestmentManager │  │  ShareClass     │  │
│  │                  │  │                  │  │  Registry       │  │
│  │ - Pool CRUD      │  │ - 申购/赎回      │  │ - Token Mint    │  │
│  │ - Spoke管理      │  │   订单聚合       │  │ - Price Oracle  │  │
│  │ - 权限控制       │  │ - Epoch 执行     │  │ - NAV 计算      │  │
│  └────────┬─────────┘  └────────┬─────────┘  └────────┬────────┘  │
│           └───────────────────┬─┘                     │           │
│                               ▼                        │           │
│  ┌────────────────────────────────────────────────────┴─────────┐ │
│  │                   Message Router (Hub)                        │ │
│  │            接收/发送跨链消息，路由到目标 Spoke                  │ │
│  └───────────────────────────────────────────────────────────────┘ │
└───────────────────────────────┬────────────────────────────────────┘
                                │ 跨链桥（Axelar / Wormhole / LZ）
              ┌─────────────────┼───────────────────┐
              │                 │                   │
┌─────────────▼──┐  ┌──────────▼─────┐  ┌─────────▼──────┐
│  Ethereum      │  │   Base          │  │  Arbitrum      │
│                │  │                 │  │                │
│ LiquidityPool  │  │  LiquidityPool  │  │ LiquidityPool  │
│ (ERC-7540)     │  │  (ERC-7540)     │  │ (ERC-7540)     │
│                │  │                 │  │                │
│ TrancheToken   │  │  TrancheToken   │  │ TrancheToken   │
│ (ERC-20)       │  │  (ERC-20)       │  │ (ERC-20)       │
│                │  │                 │  │                │
│ Escrow         │  │  Escrow         │  │ Escrow         │
│ (资金暂存)     │  │  (资金暂存)     │  │ (资金暂存)     │
│                │  │                 │  │                │
│ Hook           │  │  Hook           │  │ Hook           │
│ (合规逻辑)     │  │  (合规逻辑)     │  │ (合规逻辑)     │
└────────────────┘  └─────────────────┘  └────────────────┘
```

## 2. 核心合约清单

### 2.1 Hub（Centrifuge Chain - Substrate Pallets）

| Pallet / 模块 | 职责 |
|---|---|
| `pallet-pool-system` | Pool CRUD，Share Class 管理，Epoch 状态机 |
| `pallet-investment` | 订单聚合，Epoch 结算，成交计算 |
| `pallet-order-book` | 订单簿，申购/赎回排队 |
| `pallet-foreign-investments` | 跨链申购/赎回的异步状态管理 |
| `pallet-liquidity-pools` | 向 EVM Spoke 发送指令，接收消息 |
| `pallet-token-mux` | 多链 Tranche Token 映射管理 |

### 2.2 Spoke（EVM 链 - Solidity 合约）

| 合约 | 文件 | 职责 |
|---|---|---|
| `LiquidityPool` | `src/LiquidityPool.sol` | ERC-7540 入口，申购/赎回请求处理 |
| `TrancheToken` | `src/token/Tranche.sol` | ERC-20 Tranche 代币，含转让限制 |
| `PoolManager` | `src/PoolManager.sol` | 接收 Hub 的池配置消息，管理 Spoke 状态 |
| `InvestmentManager` | `src/InvestmentManager.sol` | 投资逻辑，价格计算，与 Escrow 交互 |
| `Escrow` | `src/Escrow.sol` | 资金暂存（申购/赎回过渡期） |
| `RestrictionManager` | `src/token/RestrictionManager.sol` | 默认 Hook：白名单 + 转让限制 |
| `Gateway` | `src/gateway/Gateway.sol` | 跨链消息收发，与桥接层交互 |
| `Routers` | `src/gateway/routers/` | 各桥（Axelar/Wormhole/LZ）适配器 |

## 3. 数据模型

### 3.1 Pool（Hub 上的权威数据）

```rust
// Substrate Pallet 数据结构
struct Pool<PoolId, AccountId, Balance, Rate, MaxTranches> {
    pub tranches: Tranches<Balance, Rate, Weight, CurrencyId, MaxTranches>,
    pub parameters: PoolParameters,    // minEpochTime, maxNAVAge
    pub metadata: Option<Vec<u8>>,     // IPFS 哈希（链下文档）
    pub currency: CurrencyId,          // 池子计价货币
    pub epoch: EpochState,             // 当前 Epoch 状态
    pub reserve: ReserveDetails,       // 资金池状态
}

struct Tranche {
    pub tranche_type: TrancheType,     // Residual(劣后) or NonResidual{interest_rate, min_risk_buffer}(高级)
    pub seniority: u32,                // 0 = 最高级，越大越劣后
    pub currency: CurrencyId,
    pub debt: Balance,
    pub reserve: Balance,
    pub loss: Balance,
    pub ratio: Perquintill,            // 占总池比例
}
```

### 3.2 Investment Order（跨链申购状态）

```rust
struct InvestOrder<Balance> {
    pub invest: Balance,   // 待执行申购金额
    pub redeem: Balance,   // 待执行赎回 Token 数量
}

// 已成交但未领取（等待投资人调用 collect）
struct CollectInvestment<Balance, TrancheToken> {
    pub payout_invest_amount: Balance,      // 退还的未成交资金
    pub payout_redeem_amount: Balance,      // 赎回所得资金
    pub remaining_invest_amount: Balance,   // 剩余未成交申购
    pub remaining_redeem_amount: TrancheToken, // 剩余未成交赎回
}
```

### 3.3 LiquidityPool Spoke 数据（Solidity）

```solidity
// InvestmentManager.sol
struct LPValues {
    uint128 maxDeposit;      // 最大可申购金额（Epoch 结算后设置）
    uint128 maxMint;         // 最大可铸造 Share 数量
    uint128 maxWithdraw;     // 最大可赎回资产金额
    uint128 maxRedeem;       // 最大可赎回 Share 数量
    uint128 depositPrice;    // 申购成交价格（ray = 1e27）
    uint128 redeemPrice;     // 赎回成交价格
    uint128 pendingDepositRequest;  // 待处理申购请求
    uint128 pendingRedeemRequest;   // 待处理赎回请求
    uint128 claimableCancelDepositRequest;  // 可取消的申购
    uint128 claimableCancelRedeemRequest;   // 可取消的赎回
}
```

## 4. 关键算法

### 4.1 多档位 Epoch 求解

V3 支持 N 个 Share Class 同时结算，求解复杂度大幅上升：

```
输入：
  - 各 Share Class 的申购总量（totalInvest[i]）
  - 各 Share Class 的赎回总量（totalRedeem[i]）
  - 当前池子 NAV
  - 各 Share Class 的最小风险缓冲（minRiskBuffer[i]）
  - 当前池子可用现金（reserve）

约束：
  1. 高级档 Share Class 的风险缓冲比例 ≥ minRiskBuffer
  2. 总现金流出 ≤ 可用现金
  3. 各档位赎回优先级：按 seniority 从高到低

目标：最大化总成交量

求解（简化伪代码）：
For each epoch:
  1. 计算全量成交场景
  2. Check all constraints
  3. If violated:
     a. 按优先级减少劣后档赎回
     b. 再减少高级档新申购
     c. 直到约束满足
  4. 输出各 Share Class 的 fulfillmentRatio[i]
```

### 4.2 Tranche Token 价格计算

```
// Hub 上的计算逻辑（Substrate）
fn calculate_tranche_token_price(tranche: &Tranche, pool: &Pool) -> Price {
    if tranche.token_supply == 0 {
        return INITIAL_PRICE; // 1.0（以 ray 表示）
    }
    
    // NAV 分配：从最高级开始分配
    let tranche_nav = match tranche.tranche_type {
        TrancheType::NonResidual { interest_rate, .. } => {
            // 高级档：本金 + 应计利息
            tranche.debt + accrued_interest(tranche.debt, interest_rate, elapsed)
        },
        TrancheType::Residual => {
            // 劣后档：池子总 NAV - 所有高级档 NAV
            total_pool_nav - sum_of_senior_navs
        }
    };
    
    tranche_nav / tranche.token_supply
}
```

### 4.3 跨链消息编码（ABI）

```solidity
// Gateway 使用的消息格式（紧凑编码，节省跨链 Gas）
enum MessageType {
    TRANSFER                 = 1,  // 资金转移
    TRANSFER_TRANCHE_TOKENS  = 2,  // Tranche Token 跨链
    INCREASE_INVEST_ORDER    = 3,  // 增加申购订单
    DECREASE_INVEST_ORDER    = 4,  // 减少申购订单
    INCREASE_REDEEM_ORDER    = 5,  // 增加赎回订单
    DECREASE_REDEEM_ORDER    = 6,  // 减少赎回订单
    EXECUTED_COLLECT_INVEST  = 7,  // 成交通知（申购）
    EXECUTED_COLLECT_REDEEM  = 8,  // 成交通知（赎回）
    CANCEL_INVEST_ORDER      = 9,  // 取消申购
    CANCEL_REDEEM_ORDER      = 10, // 取消赎回
    UPDATE_TRANCHE_TOKEN_METADATA = 11,
    UPDATE_MEMBER            = 16, // KYC 更新
    UPDATE_TRANCHE_PRICE     = 17, // 价格更新
    ADD_POOL                 = 21, // 新 Pool 上线
    ADD_TRANCHE              = 23, // 新 Share Class
    ALLOW_INVESTMENT_CURRENCY = 27, // 允许新货币
}
```

## 5. 安全模型

### 5.1 多层安全架构

```
Layer 1: 跨链消息安全
  - 只有授权 Router（Axelar/Wormhole/LZ 合约）可以向 Gateway 发消息
  - Root 合约的 ward 机制控制所有关键权限
  - 多签（Multi-sig）控制 Root 的 ward 操作

Layer 2: 资金安全
  - Escrow 合约持有资金，只有 InvestmentManager 可以划拨
  - 申购/赎回资金在 Epoch 结算前锁定在 Escrow，不可挪用

Layer 3: 合规安全
  - Hook 在每次 Transfer 前触发，阻止非合规操作
  - TrancheToken 的 transfer 函数检查 RestrictionManager

Layer 4: 经济安全
  - 高级档最小风险缓冲（minRiskBuffer）保护
  - 赎回顺序优先级防止挤兑
  - NAV 最大时效（maxNAVAge）防止过期估值
```

### 5.2 已知安全假设

| 假设 | 说明 |
|---|---|
| 跨链桥安全 | 依赖 Axelar/Wormhole/LZ 的安全性，桥被攻击则资金有风险 |
| Issuer 诚信 | NAV 由 Pool Admin 提交，存在主观操纵空间（链上无法完全验证） |
| Centrifuge Chain 活性 | Hub 停机则所有 Spoke 的 Epoch 结算暂停（不影响资金安全，但影响流动性） |
| Oracle 准确性 | 第三方估值 Oracle 的数据质量影响 NAV 精确度 |

## 6. 部署清单（新 Pool 上线 CheckList）

```
Centrifuge Chain (Hub):
☐ 创建 Pool（poolId 分配）
☐ 配置 Share Class（N 档位，参数：利率/最小风险缓冲/货币）
☐ 部署跨链通道（授权目标 EVM 链）
☐ 设置 Epoch 参数（minEpochTime, maxNAVAge）

目标 EVM 链（Spoke）:
☐ Gateway 合约已部署并授权
☐ LiquidityPool 部署（绑定 Pool ID + Share Class ID）
☐ TrancheToken 部署（ERC-20，名称/符号）
☐ Escrow 部署
☐ InvestmentManager 配置
☐ Hook 配置（RestrictionManager 或自定义）
☐ 授权投资货币（允许哪些 ERC-20 作为申购货币）

KYC 准备:
☐ 链下 KYC 流程上线（Synaps / Fractal / 自建）
☐ 白名单管理后台就绪
☐ 合规文件（Subscription Agreement）准备

测试:
☐ 在测试网完整走通：申购 → Epoch → 领取 → 赎回
☐ 跨链延迟测试（记录各桥的实际延迟）
☐ Gas 成本估算（各链的 requestDeposit / deposit 成本）
```

## 7. V3 与 V1/V2 核心差异汇总

| 维度 | V1 | V2 | V3 |
|---|---|---|---|
| 链 | Ethereum only | Ethereum + CF Chain (有限) | 真正多链（Hub-Spoke） |
| 分层 | DROP / TIN | DROP / TIN (可配置) | Share Class（N档） |
| 标准 | 自定义 | 自定义 | ERC-4626 + ERC-7540 |
| 合规 | 白名单 | MemberList v2 | Hook 框架（可插拔） |
| 货币 | DAI | 任意 ERC-20 | 多链多货币 |
| 机构支持 | 弱 | 中 | 强（多 Share Class = 基金结构） |
| Gas 成本 | 高（ETH L1） | 高（ETH L1） | 低（L2 Spoke） |
| 升级性 | 困难（独立合约） | 困难 | 好（Pallet 升级 + 合约代理） |

---
*文档版本：1.0 | 基于 Centrifuge GitHub: https://github.com/centrifuge/centrifuge-chain + https://github.com/centrifuge/liquidity-pools*
