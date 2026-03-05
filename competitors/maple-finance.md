# Maple Finance — 调研笔记

> 调研时间：2026-03-02
> 官网：https://maple.finance
> 文档：https://docs.maple.finance

---

## 一句话定位

**链上机构信贷市场，把高收益固定利率机构贷款 Token 化，通过 Syrup（零售产品）让 DeFi 用户分享机构贷款收益。**

---

## 产品架构

**两层产品：**

```
Maple（机构层）
  → 向机构借款方提供 USDC / USDT 贷款
  → 借款方：做市商、加密基金、量化机构
  → 贷款类型：超额抵押 + 固定利率 + 短期（7-90 天）

Syrup（零售 DeFi 层）
  → 把 Maple 机构贷款池开放给零售 DeFi 用户
  → 用户存 USDC 赚取机构贷款收益（约 8-12%）
  → SYRUP Token 激励 + 25% 协议收入回购
```

---

## 商业模式

```
收入来源：
  → 贷款起始费（1-2%，一次性）
  → 协议管理费（占利息 10-20%）
  → Syrup 平台费

费用分配：
  → 25% 回购 SYRUP Token
  → 75% 用于运营 + 储备
```

与 Ondo 不同，Maple 的收益来自「信用利差」而非「国债收益」：

```
借款机构支付 10-15% 利率
  - 平台费 ~2%
  = 投资者获得 8-12%
```

这比美债收益（5%）高得多，但信用风险也更高。

---

## 核心技术

**Pool Delegate 模式：**

```
Pool Delegate（信贷管理人）
  → 负责对借款机构进行尽职调查
  → 决定是否批准贷款
  → 承担第一损失（需自持 Pool 的 10%+）

流动性提供者（LP）
  → 存 USDC 进 Pool
  → 享受 Pool Delegate 筛选后的收益
  → 有流动性窗口期（通常 30-90 天提前申请赎回）
```

**风险隔离：**

- 每个 Pool 独立，风险不互相传染
- Pool Delegate 自持资金，保证尽职调查质量
- 超额抵押贷款：抵押率 130-150%

---

## 历史事件（重要）

**2022 年 FTX 崩盘冲击**

Maple 在 2022 年遭受重大损失：
- FTX / Alameda 是 Maple 的主要借款方之一
- FTX 崩盘后，多个 Pool 出现坏账，损失约 $54M
- Maple 的应对：暂停受影响 Pool、引入超额抵押要求、更严格的尽调流程

这次事件是 Maple 从「信用贷款（无抵押）」转向「超额抵押贷款」的转折点，也奠定了现在更稳健的产品架构。

---

## 当前状态（2025 年）

- TVL：$3亿+
- 累计放贷：$40亿+
- Syrup 用户：50,000+
- 主要贷款方：加密做市商（已知机构）、量化基金
- 链：Ethereum、Solana（Maple Direct）

---

## 与 Ondo / Centrifuge 的差异

- Ondo → 公开市场证券（美债 / 美股），低风险低收益，监管最复杂
- Centrifuge → 私人信贷 / 实体资产，中等风险，DeFi 集成最深
- Maple → 加密机构借贷，高风险高收益，最接近 TradFi 信贷业务

---

## 对你的项目的启示

Maple 模式在新加坡有一个特别有趣的应用场景：

**新加坡加密机构借贷市场**

- 新加坡有大量持牌加密交易所（Coinbase SG、Crypto.com、Independent Reserve）
- 这些机构有短期 USDC/USD 流动性需求
- DCS Card Centre 本身就在金融服务领域，有机构信任背书

**具体方向：**
- 面向持牌 crypto 机构做超额抵押 USDC 借贷
- 用 Maple 类似的 Pool Delegate 模式分散信用风险
- 与你的稳定币清算基础设施结合，形成完整的机构流动性服务

这个方向比 Ondo 的美债 Token 化竞争更小，且与你现有的支付/清算资源高度互补。

---

## 技术架构（源码级）

> 参考：https://github.com/maple-labs/maple-core-v2
> 参考：https://github.com/maple-labs/syrup-contracts

### 整体架构

Maple 是纯 EVM 架构（无自有链），所有逻辑在 Ethereum + Solana 上运行。核心设计：Pool 作为流动性聚合层，LoanManager 跟踪贷款状态，WithdrawalManager 管理 LP 退出。

```
┌─────────────────────────────────────────────────────────────┐
│                        用户层                                │
│   DeFi 用户（Syrup）          机构 LP（Maple Direct）        │
└──────────────┬──────────────────────────┬───────────────────┘
               │ ERC-4626                 │ 直接存入
┌──────────────▼──────────────────────────▼───────────────────┐
│                     Pool.sol（ERC-4626）                     │
│   poolToken（mapleUSDC）= LP 凭证                            │
│   totalAssets() = cash + outstandingLoans - unrealizedLoss   │
└──────┬────────────────────────────┬────────────────────────-─┘
       │ fund(loan)                  │ requestRedeem()
┌──────▼──────────┐     ┌───────────▼───────────────┐
│  LoanManager.sol │     │  WithdrawalManager.sol    │
│  贷款台账         │     │  周期赎回窗口              │
│  利息累积         │     │  流动性约束计算            │
└──────┬──────────┘     └───────────────────────────┘
       │ 放款 / 收款
┌──────▼──────────┐
│  MapleLoan.sol  │   每笔贷款独立合约
│  固定条款       │   borrower / lender / terms
│  还款 / 违约    │
└─────────────────┘
```

---

### 核心合约详解

#### Pool.sol — ERC-4626 流动性池

Pool 继承 ERC-4626，LP Token 代表 Pool 的份额。核心资产计算：

```solidity
// 总资产 = 合约现金 + 贷款应收 - 未实现损失
function totalAssets() public view returns (uint256) {
    return cash() + loanManager.assetsUnderManagement()
           - unrealizedLosses();
}

// unrealizedLosses 在 impairLoan 时增加，还款时减少
// 这保证了坏账在 NAV 中实时反映，而非等违约才扣减
```

**重要设计：** Pool 的 `totalAssets()` 是实时变化的（随利息累积而增加），所以 LP Token 的价值每秒都在微增——类似 aToken 的机制，但基于实际贷款收益而非流动性挖矿。

#### PoolManager.sol — 管理层权限

PoolManager 是 Pool Delegate（信贷管理人）的链上代理，封装了所有管理操作：

```solidity
// Pool Delegate 专属权限（需要在 MapleGlobals 中注册）
function fund(address loanAddress) external onlyPoolDelegate;       // 向贷款合约拨款
function impairLoan(address loan) external onlyPoolDelegate;        // 标记贷款为减值
function triggerDefault(address loan, ...) external onlyPoolDelegate; // 触发违约清算

// 紧急权限（Protocol Admin，多签）
function setPendingPoolDelegate(address) external onlyGovernor;      // 更换 Delegate
function setLiquidityCap(uint256) external onlyGovernorOrDelegate;   // 设置最大 TVL
```

**Pool Delegate 的信任模型：**
- Delegate 必须自持 Pool 资产的一定比例（"skin in the game"）
- Delegate 无法直接提走 LP 资金，只能通过 `fund(loan)` 发放给借款人合约
- 违约损失由 LP 承担，但 Delegate 自持部分先亏损（激励尽调）

#### LoanManager.sol — 贷款台账

LoanManager 是 Pool 的"影子记账本"，跟踪所有活跃贷款的状态：

```solidity
struct LoanState {
    uint112 incomingNetInterest;  // 应收利息（按秒累积）
    uint112 issuanceRate;         // 每秒利息增长率
    uint40  startDate;            // 贷款开始时间
    uint40  paymentDueDate;       // 下次还款日
    uint40  impairedDate;         // 减值日期（0 = 未减值）
    address loan;                 // 对应的 MapleLoan 合约地址
}

// 利息累积（任何人可调用，触发状态更新）
function updateAccounting() public {
    // 遍历所有活跃贷款，按时间差累积利息
    // 更新 Pool.totalAssets()
}
```

**关键：** 利息不需要借款人还款才会体现——只要贷款在进行中，每秒都累积到 `totalAssets()`。这使 LP Token 的价值时刻上涨，类似 yield-bearing token 的效果。

#### WithdrawalManager.sol（Cyclical）— 赎回窗口

Maple 采用 Cyclical Withdrawal（周期性赎回窗口），不是无限制的即时赎回：

```
时间轴示意：
│← 请求期(7天) →│← 提款窗口(3天) →│← 请求期 →│...
│              │                  │
│ LP提交请求   │ LP执行提款        │
│              │（按比例分配流动性）│

关键设计：
  - 请求期内：LP 调 requestRedeem(shares) → 预约赎回，shares 被锁定
  - 窗口开放时：processRedemptions() 计算可用流动性
    → 如果请求量 ≤ 可用现金：全部满足
    → 如果请求量 > 可用现金：按比例部分满足（剩余自动顺延下个窗口）
  - LP 调 withdraw()/redeem() → 领取 USDC，解锁 shares
```

```solidity
// 周期参数（由 Pool Delegate 配置）
uint256 public windowDuration;    // 提款窗口时长（如 3 天）
uint256 public cycleDuration;     // 完整周期时长（如 10 天）

// LP 的预约请求
mapping(address => uint256) public exitCycleId;  // 该 LP 的赎回周期
mapping(address => uint256) public lockedShares;  // 已锁定 shares
```

**流动性约束：** Pool 中的 `cash()` 是真实可用流动性（贷款中的钱取不出来），窗口处理时优先满足先到的请求。

#### MapleLoan.sol — 单笔贷款合约

每笔贷款都是独立合约，在放款时由 Pool Delegate 部署：

```solidity
// 贷款核心参数（创建时固定，不可变）
struct LoanParameters {
    address borrower;
    address asset;            // 贷款币种（USDC/WETH）
    uint256 principalRequested;
    uint128 interestRate;     // 年化利率（bps）
    uint128 closingFeeRate;   // 提前还款费率
    uint40  gracePeriod;      // 宽限期（秒）
    uint40  paymentInterval;  // 还款间隔（秒）
    uint32  paymentsRemaining; // 剩余还款期数
}

// 还款计算（每个 paymentInterval 调用一次）
function getPaymentBreakdown() external view returns (
    uint256 principal,    // 本期应还本金
    uint256 interest,     // 本期应还利息
    uint256 fees          // 协议手续费
);

// 关键：借款人直接向此合约转 USDC 还款
// 合约收到款项后通知 LoanManager 更新台账
```

**违约处理流程：**

```
1. paymentDueDate 过期 + gracePeriod 耗尽
2. Pool Delegate 调 impairLoan(loan) → unrealizedLoss 增加 → Pool NAV 下降
3. Pool Delegate 调 triggerDefault(loan, ...) → 触发清算
4. 清算人（Liquidator 合约）以折扣价获取抵押品
5. 清算所得返回 Pool → LoanManager 更新：本金亏损转为 realizedLoss
```

---

### Syrup 架构（零售 DeFi 层）

Syrup 是 Maple 的 DeFi 前端，把机构级贷款收益开放给散户：

```
Syrup 用户 ──USDC──► SyrupRouter.sol
                            │
                            │ depositWithPermit() / deposit()
                            ▼
                    Maple Pool（syrupUSDC pool）
                            │ 赚取贷款利息
                            ▼
                    syrupUSDC Token（ERC-4626）
                            │ 持有赚利息（NAV 每秒上涨）

另外：
  syrupUSDC 质押 ──► SyrupDrip.sol → 每秒释放 SYRUP token 奖励
  25% 协议收入 → 市场回购 SYRUP token（收入捕获）
```

**SyrupRouter.sol** 简化了 ERC-4626 的复杂性：用户只需一笔交易完成 approve + deposit，并自动处理 Withdrawal Manager 的窗口期预约。

---

### 风险参数体系

```
MapleGlobals.sol 管理全局风险参数：

╔══════════════════════════════════════════╗
║  超额抵押贷款（Secured）                  ║
║  maxLTV = 133%（即最多借价值的 75%）      ║
║  liquidationThreshold = 110%（抵押率低于  ║
║    110% 触发清算）                        ║
╠══════════════════════════════════════════╣
║  无抵押贷款（Unsecured，已基本废弃）       ║
║  完全依赖 Pool Delegate 尽调              ║
║  FTX 事件后已大幅收紧                     ║
╠══════════════════════════════════════════╣
║  池子级别参数（Pool Delegate 配置）        ║
║  liquidityCap：最大 TVL 上限              ║
║  delegateManagementFeeRate：管理费（%）   ║
║  withdrawCycleDuration：赎回窗口时长      ║
╚══════════════════════════════════════════╝
```

---

### 与我们 rwa-platform 的技术差距

| 维度 | Maple Finance | 我们的 rwa-platform |
|------|--------------|-------------------|
| 资产类型 | 机构贷款（现金流型） | NAV 基金份额（价格型） |
| 赎回机制 | Cyclical Window（周期赎回） | 简单 T+N 队列 |
| 收益来源 | 贷款利息（链上实时累积） | NAV 增值（oracle 推送） |
| 信用风险管理 | Pool Delegate 模型 + 抵押品清算 | 无信用管理（不做贷款） |
| TVL 限制 | liquidityCap 机制 | 无上限控制 |
| 利率模型 | 固定利率 per loan | 无利率（NAV 增值） |
| 违约处理 | impairLoan → triggerDefault → Liquidator | 不适用 |

**最值得借鉴的设计点：**

1. **totalAssets() 实时累积**：比我们的 epoch NAV 更精确，用户每秒都能看到收益增长
2. **Cyclical Withdrawal Manager**：比简单 T+N 队列更灵活，可以根据流动性调整窗口
3. **Pool Delegate 模型**：如果未来做私人信贷，Delegate 的"skin in the game"机制是关键风险控制手段
