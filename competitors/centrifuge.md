# Centrifuge — 调研笔记

> 调研时间：2026-03-02
> 官网：https://centrifuge.io
> 文档：https://docs.centrifuge.io

---

## 一句话定位

**DeFi 原生的 RWA 协议层，把真实世界的信贷资产（发票、贷款、ABS）Token 化后接入 MakerDAO/Aave/Morpho，$2B+ TVL，Apollo/Janus Henderson 背书。**

---

## 产品概况

- **TVL：** $2B+（RWA 赛道最大之一）
- **客户：** Apollo Global、Janus Henderson、S&P Dow Jones Indices 等机构
- **DeFi 集成：** Sky（MakerDAO）、Aave Horizon、Morpho、Pendle
- **链：** Centrifuge Chain（自有，基于 Substrate）+ Ethereum + 7 条链
- **Token 标准：** ERC-4626 + ERC-7540（异步赎回）
- **审计：** 21+ 次审计

---

## 核心产品：Centrifuge Pool

**资产发行方（Issuer）视角：**

```
真实世界资产
（发票、贷款、ABS、基金份额）
    ↓
创建 Centrifuge Pool
    ↓
资产 Token 化（每笔资产 = 1 个 NFT）
    ↓
NFT 存入 Pool，换取流动性
    ↓
DeFi 用户提供 USDC 流动性
```

**DeFi 用户视角：**

```
存入 USDC
    ↓
获得 Pool Token（Junior / Senior Tranche）
    ↓
赚取比普通 DeFi 更高的固定收益
```

---

## 分级（Tranche）结构

这是 Centrifuge 最重要的技术设计，直接从传统 ABS 借鉴：

```
Senior Tranche（高级）
  → 优先还款，风险最低
  → 收益率：~5-8%
  → 面向机构 / 保守投资者

Junior Tranche（劣后）
  → 最后还款，风险最高（首亏）
  → 收益率：~10-20%
  → 面向高风险偏好 / 发行方自持（信用增强）
```

发行方通常自持 Junior Tranche，作为对 Senior 投资者的信用背书。

---

## ERC-7540：异步赎回的技术解决方案

Centrifuge 主导了 ERC-7540 标准（ERC-4626 的异步扩展），解决了 RWA 的核心流动性问题：

**问题：** RWA 的底层资产不能即时变现（贷款有锁定期、发票有账期），但 DeFi 用户期望随时赎回。

**ERC-7540 方案：**
```
用户发起赎回请求（异步）
    ↓
请求进入队列，等待底层资产到期
    ↓
资产到期后，用户 claim USDC
    ↓
用户等待期间，可以在二级市场出售 pending 状态的 Token
```

这比简单的 T+1 更复杂，但更真实地反映了 RWA 的流动性特征。

---

## 与机构合作模式（Centrifuge Prime）

**Apollo 合作案例：**

Apollo 通过 Centrifuge 把私信贷基金份额 Token 化，接入 DeFi 流动性：

```
Apollo 私信贷基金（链下，$500B AUM）
    ↓ Centrifuge Pool
链上 Token（aToken）
    ↓
Aave Horizon（借贷抵押品）
```

机构提供资产，Centrifuge 提供技术 + DeFi 分发渠道，双方都获益。

---

## 技术架构

```
Centrifuge Chain（Substrate 自有链）
  → 资产 NFT 登记
  → Pool 逻辑 / NAV 计算
  → 合规检查

Centrifuge Liquidity Pools（EVM 侧）
  → ERC-4626 / ERC-7540 接口
  → 接入 DeFi 协议

跨链消息（Wormhole / LayerZero）
  → Centrifuge Chain ↔ Ethereum 双向通信
```

---

## 竞争定位

Centrifuge 和 Ondo 是**完全不同的赛道**：

- Ondo → 标准化、流动性高的公开市场资产（美债、美股）
- Centrifuge → 非标准化、流动性低的私人信贷资产（发票、贷款、ABS）

两者的 DeFi 集成路径也不同：
- Ondo → Token 直接作为抵押品（高流动性）
- Centrifuge → Pool 提供 Senior/Junior Tranche，通过分级承担信用风险

---

## 关键数据（2025 年）

- 处理资产总额：$2B+
- 活跃 Pool 数量：60+
- 最大 Pool：Blocktower（信用基金）、New Silver（房地产过桥贷款）
- 接入 MakerDAO 借贷额度：$2.2 亿（最高时）

---

## 对你的项目的启示

**新加坡场景下，Centrifuge 模式比 Ondo 模式更适合你：**

原因：
- 新加坡有大量中小企业贸易融资需求（发票、信用证）
- SGX 上有大量房地产信托（REITs），可以做 REIT 份额 Token 化
- MAS 对私人信贷有完整的监管框架（Prescribed Capital Market Products）

**具体方向建议：**
- 新加坡 SME 贸易融资 Token 化（发票 → Pool → DeFi 流动性）
- 新加坡 REITs 的链上分级（Senior/Junior Tranche）
- 和 SGX 合作，把 SGX 上市产品的链上接口做出来

这比直接复制 Ondo 的美债路线更有本地差异化，也更契合你的支付/清算基础设施。

---

## 技术架构（源码级）

> 参考：https://github.com/centrifuge/liquidity-pools（EVM 合约）
> 参考：https://github.com/centrifuge/centrifuge-chain（Substrate 链）

### 整体分层架构

Centrifuge 采用双链架构：自有的 **Centrifuge Chain**（Substrate/Polkadot）负责资产登记和池子核心逻辑，**EVM Liquidity Pools**（Ethereum/Arbitrum 等 7 条链）作为用户交互和 DeFi 集成层。

```
┌─────────────────────────────────────────────────────────┐
│                   DeFi 协议层                            │
│    Aave Horizon / Morpho / Sky / Pendle                 │
└─────────────────────┬───────────────────────────────────┘
                      │ ERC-4626 / ERC-7540
┌─────────────────────▼───────────────────────────────────┐
│              EVM Liquidity Pools（7 条链）               │
│  LiquidityPool → InvestmentManager → Gateway → Router   │
│  Tranche Token（ERC-20 + 转让限制）                      │
└─────────────────────┬───────────────────────────────────┘
                      │ Wormhole / Axelar 跨链消息
┌─────────────────────▼───────────────────────────────────┐
│            Centrifuge Chain（Substrate/Polkadot）        │
│  pallet-pool-system / pallet-investments / pallet-loans  │
│  pallet-oracle / pallet-liquidity-pools                  │
│  资产 NFT 登记 / NAV 计算 / Pool 订单簿                   │
└─────────────────────────────────────────────────────────┘
```

---

### EVM 侧核心合约

#### LiquidityPool.sol — 用户入口

实现 ERC-4626（同步）+ ERC-7540（异步）双接口，是用户直接交互的合约。

```solidity
// ERC-7540 异步接口（RWA 标准路径）
function requestDeposit(uint256 assets, address controller, address owner) external;
function requestRedeem(uint256 shares, address controller, address owner) external;

// ERC-4626 同步接口（流动性充足时的快速路径）
function deposit(uint256 assets, address receiver) external returns (uint256 shares);
function redeem(uint256 shares, address receiver, address owner) external returns (uint256 assets);

// 关键：request 和 claim 分开，中间经过跨链消息
function pendingDepositRequest(uint256, address controller) external view returns (uint256);
function claimableDepositRequest(uint256, address controller) external view returns (uint256);
```

**资金隔离：** LiquidityPool 本身不持有资产。USDC 进来后转入 `Escrow.sol`，等跨链确认后才 mint Tranche Token。

#### InvestmentManager.sol — 状态机

管理从"用户发出请求"到"可 claim"的完整状态转换，是整个异步流程的核心协调者。

```
用户 requestDeposit
    → InvestmentManager 记录请求
    → 发消息给 Centrifuge Chain（via Gateway）
    → Centrifuge Chain 处理订单（下一个 epoch）
    → 回调消息：可 deposit 数量（epochOrder 处理结果）
    → InvestmentManager 标记为 claimable
    → 用户调 deposit() → mint Tranche Token
```

每个 LP 地址的请求状态：

```solidity
struct InvestmentState {
    uint128 maxDeposit;       // 当前 epoch 最大可 deposit 额度
    uint128 maxMint;          // 对应可 mint 的 shares
    uint128 maxWithdraw;      // 当前可 redeem 的 assets
    uint128 maxRedeem;        // 对应可销毁的 shares
    uint128 pendingDepositRequest;  // 等待处理的 deposit 请求
    uint128 pendingRedeemRequest;   // 等待处理的 redeem 请求
    bool pendingCancelDepositRequest;
    bool pendingCancelRedeemRequest;
}
```

#### PoolManager.sol — 池子注册表

管理所有 Pool 和 Tranche 的元数据，处理资产与 tranche 的关联。

```solidity
// Pool 基本信息
mapping(uint64 poolId => address[] tranches) public tranches;
mapping(uint64 poolId => mapping(address asset => bool)) public isPoolAsset;

// 关键函数
function deployPool(uint64 poolId) external;      // 链上创建 Pool
function deployTranche(uint64 poolId, bytes16 trancheId, ...) external;  // 创建 Tranche Token
function deployLiquidityPool(uint64 poolId, bytes16 trancheId, address asset) external;  // 部署用户入口
```

#### Tranche.sol — 分层 Token

ERC-20 + 转让限制，是 LP 持有的实际资产凭证。

```solidity
// 核心特性：转让前检查 RestrictionManager
function _update(address from, address to, uint256 value) internal override {
    // 检查 from/to 是否满足合规要求（KYC 状态）
    require(restrictionManager.canTransfer(from, to), "TRANSFER_BLOCKED");
    super._update(from, to, value);
}

// 新增：mint 和 burn 只允许 InvestmentManager 调用
function mint(address to, uint256 value) external onlyInvestmentManager;
function burn(address from, uint256 value) external onlyInvestmentManager;
```

**关键**：Tranche Token 自身有 KYC 门控，这正是 deRWA 要用 Wrapper 绕过限制的原因。

#### Gateway.sol + Router.sol — 跨链消息

```
EVM 发出消息流程：
InvestmentManager → Gateway（encode message）→ Router → Wormhole/Axelar Adapter → Centrifuge Chain

Centrifuge Chain 回调流程：
Centrifuge Chain → Wormhole/Axelar → Router → Gateway（decode message）→ InvestmentManager
```

消息格式：固定头部 + 操作码 + payload（ABI-like 紧凑编码，不用标准 ABI 避免冗余）

---

### Centrifuge Chain 侧核心 Pallet

| Pallet | 功能 |
|--------|------|
| `pallet-pool-system` | Pool 生命周期（创建/暂停/关闭），Tranche 定义 |
| `pallet-investments` | 订单簿，epoch 处理（按 NAV 折算 deposit/redeem） |
| `pallet-loans` | 每笔贷款的创建、还款、违约、NAV 估值 |
| `pallet-oracle` | 聚合多个 price feeder 的 NAV 数据，防止单点操纵 |
| `pallet-liquidity-pools` | 接收/发送跨链消息的网关 |

**Epoch 机制（关键）：**

```
每个 Pool 按固定 epoch（通常 24 小时）处理订单
    → epoch 结束时：锁定当前 NAV
    → 按 NAV 计算：
        deposit_orders → 应 mint 多少 Tranche Token
        redeem_orders  → 应返还多少 USDC
    → 可处理量受流动性约束：如果赎回超过可用 USDC，按比例处理
    → 发消息回 EVM → InvestmentManager 标记为 claimable
```

---

### ERC-7540 异步赎回完整状态机

```
┌──────────┐  requestRedeem()  ┌────────────┐
│  用户    │ ──────────────►  │  PENDING   │  等待 epoch 处理
└──────────┘                   └─────┬──────┘
                                     │ Centrifuge Chain epoch 完成
                                     ▼
                               ┌────────────┐
                               │ CLAIMABLE  │  shares 已锁定，USDC 可领
                               └─────┬──────┘
                                     │ 用户调 redeem()
                                     ▼
                               ┌────────────┐
                               │  COMPLETE  │  USDC 到账，shares 销毁
                               └────────────┘

取消路径：
PENDING → cancelRedeemRequest() → 等待确认 → 调 withdraw() 领回 shares
```

等待期间用户可在二级市场出售 Tranche Token（如果有流动性），无需等待 epoch。

---

### Tranche 分层机制（数学层面）

以两层 Tranche 为例（Senior + Junior）：

```
总 Pool 资产 = Principal + AccruedInterest - Defaults

优先分配（Senior Tranche）：
  Senior NAV = min(Senior 投入 × (1 + seniorRate × t), 总资产 × (1 - juniorRatio))

Junior Tranche 承接剩余：
  Junior NAV = max(0, 总资产 - Senior NAV)

关键：Junior NAV 可以为 0（全部亏损），Senior 优先偿付
```

Junior 投资者通常是资产发行方自持（信用增强，向 Senior 提供隐性担保）。

---

### deRWA 合规模型（混合合规，2025年新功能）

deRWA 是解决"KYC 门控 vs DeFi 可组合性"矛盾的核心创新：

```
架构设计：

Layer 1（Pool 入口，Hard Gate）：
  requestDeposit() → RestrictionManager.canDeposit(user) → KYC 检查
  → 通过 → 进入 epoch 队列
  → 拒绝 → revert（未 KYC 用户无法入池）

Layer 2（Tranche Token，有转让限制）：
  Tranche Token 的 transfer() 内置 KYC 检查
  → 只有白名单地址可以持有

Layer 3（deRWA Wrapper，无限制）：
  用户存 Tranche Token → deRWA 合约持有
  用户获得 deRWA Token（标准 ERC-20，无转让限制）
  → 可在 Uniswap/Aave/Morpho 中自由流通

赎回路径：
  deRWA Token → burn → 解锁对应 Tranche Token
  → 再走正常 requestRedeem() → Centrifuge Chain → USDC
```

**和 Ondo USDY 的本质区别：**
- Ondo：每次 transfer 触发合规检查（Soft Gate，检查在流通层）
- Centrifuge deRWA：KYC 检查只在入口（Hard Gate），流通层完全开放

监管风险：deRWA 的 KYC 一旦完成，后续持有人变化不受限，合规边界需要法律意见支撑。

---

### NAV Oracle 机制

```
链下：
  资产管理人每日计算 Pool NAV
  → 提交给 Centrifuge Oracle Feeder（多签或单签，取决于 Pool 配置）

链上（Centrifuge Chain）：
  pallet-oracle 聚合多个 feeder 的报价
  → 取中位数（防止单个 feeder 操纵）
  → 触发 epoch 处理时使用该 NAV

偏差保护：
  新 NAV 与上一个 epoch NAV 的偏差超过阈值 → 拒绝更新（需管理员确认）
  Staleness 检测：24 小时未更新 → Pool 进入暂停状态
```

---

### 与我们 rwa-platform 的技术差距

| 维度 | Centrifuge | 我们的 rwa-platform |
|------|-----------|-------------------|
| 赎回模式 | ERC-7540 异步 + epoch 机制 | 简单 T+N 队列 |
| 分层结构 | Senior/Junior Tranche，独立 token | 单层，无分层 |
| 跨链 | 7 条链，Wormhole/Axelar 消息层 | 单链 |
| deRWA 合规 | Pool 入口 KYC + 流通层无限制 wrapper | Whitelist 检查在每次 transfer |
| NAV Oracle | 多 feeder 聚合 + 中位数 | 单 operator 推送 |
| 可升级性 | UUPS proxy 全套 | 无代理，不可升级 |
| 审计 | 21+ 次，Trail of Bits 参与 | 未审计 |

**最值得借鉴的设计点：**

1. **ERC-7540 接口**：比我们自定义的赎回队列更标准，DeFi 协议认可度更高
2. **deRWA Wrapper 模式**：解决 KYC 和 DeFi 可组合性矛盾的最优解
3. **多 feeder NAV 聚合**：防止单点 Oracle 操纵，比我们的单 operator 推送更安全
