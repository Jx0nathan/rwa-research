# OpenEden — 深度分析

> 调研时间：2026-03-02
> 官网：https://openeden.com
> GitHub：https://github.com/OpenEdenHQ
> 定位：**你最直接的本地竞争对手**，新加坡，MAS 合规，T-Bill Token 化

---

## 基本信息

- **创始团队：** 前 Gemini 亚太区团队（CEO Jeremy Ng、CTO Ivan Tong）
- **成立时间：** 2022 年
- **所在地：** 新加坡
- **监管：** MAS 持牌（CMS License 下的豁免框架）
- **主要产品：** TBILL（T-Bill Vault）、USDO（收益稳定币）
- **底层资产：** 美国国库券（T-Bills）+ 部分 BUIDL（BlackRock）
- **TVL：** $3 亿+（2025 年）
- **链：** Ethereum（主链）

---

## 产品线

**TBILL（主力产品）**
- 认购美国国库券的链上份额
- 价格上涨模型（非 Rebase）：1 TBILL 价格每日小幅上涨
- 最低认购：$1,000
- 管理费：~0.5% / 年
- T+1 结算（队列赎回）+ 即时赎回（收额外费用）

**USDO（第二产品）**
- 由 TBILL 背书的收益型稳定币
- 锚定 $1，持有者不需要自己操作，自动获得收益
- 类似 Ondo USDY 的思路，但底层是 TBILL 而非直接持有美债

**Prism（最新产品，2024年）**
- 下一代多资产 Vault 框架
- 支持多种 RWA 资产（TBILL、BUIDL 等）接入同一个 Vault 体系
- 更灵活的赎回队列（RedemptionQueue.sol）
- 即时赎回功能增强（Express.sol）

---

## 技术架构（源码级分析）

### 核心合约：OpenEdenVaultV5

**Token 设计：** TBILL = ERC-20 + UUPS 可升级，价格上涨模型

```
1 TBILL 的价值 = tbillUsdcRate()
             = tbillUSD 价格 / USDC价格（Chainlink）
```

每次 deposit/redeem 都按当前 tbillUsdcRate 计算份额数量，不 rebase，Token 数量不变，价值增长。

**资金流向：**

```
用户 USDC
  → 扣除交易费（→ oplTreasury）
  → 剩余 USDC → treasury（链下托管）
  → 按当前 tbillUsdcRate mint TBILL Token 给用户
```

关键点：USDC 进来后**先进合约，再 offRamp 到 treasury**，不像 Ondo 直接转给 Coinbase。这是和 Ondo 最大的资金流设计差异。

**赎回流：**

```
用户 redeem(_shares)
  → Token 转入合约（锁住）
  → 进入 withdrawalQueue（双端队列）
  → operator 调 processWithdrawalQueue() 批量处理
  → 按当时 tbillUsdcRate 计算 USDC 数量
  → 发 USDC 给用户 + 烧掉 Token
```

---

### KycManager：五角色权限分离

```solidity
GRANT_ROLE   → 授予 KYC（独立账号）
REVOKE_ROLE  → 撤销 KYC（独立账号）
BAN_ROLE     → 封禁用户（独立账号）
UNBAN_ROLE   → 解封用户（独立账号）
UPGRADE_ROLE → 升级合约（独立账号）
```

**重要差异：** OpenEden 把 KYC 的授予、撤销、封禁、解封拆成四个独立角色，最小权限原则。这比你的 rwa-protocol 更安全——你目前所有操作都集中在一个 OPERATOR_ROLE。

**两种 KYC 类型：**
- `US_KYC` — 美国用户（受更严格限制）
- `GENERAL_KYC` — 非美国用户

`strictOn` 开关：strict 模式下只有 `GENERAL_KYC` 可以使用，用于紧急合规切换。

---

### TBillPriceOracle：双价格轨道

**两个价格概念：**

```
currentPrice（实时价格）
  → 由 updatePrice() 每日更新
  → 有偏差保护：新价格与 closeNavPrice 偏差不超过阈值

closeNavPrice（收盘 NAV）
  → 前一交易日的收盘价，作为偏差校验基准
  → 每日同步更新
  → 可被 admin 手动覆盖（紧急情况）
```

**偏差保护算法：**

```solidity
// 偏差 = |新价格 - closeNavPrice| / ((新价格 + closeNavPrice) / 2)
// 偏差 > maxPriceDeviation → revert
uint256 numerator = closeNavPrice > newPrice
    ? closeNavPrice - newPrice
    : newPrice - closeNavPrice;
uint256 denominator = (closeNavPrice + newPrice) / 2;
```

这比你的 rwa-protocol 的涨跌幅限制更精确——用中间价作为分母，避免单边趋势时的保护失效。

**Staleness 保护：**

```solidity
// Vault 读价格时检查：超过 7 天没更新 → revert
if (block.timestamp - updatedAt > 7 days)
    revert TBillPriceOutdated(updatedAt);
```

7 天是考虑了公共假期（最多连续 3 天不更新）后的宽松设置。

---

### 赎回队列：DoubleQueueModified

赎回请求用**双端队列**存储，关键特性：

```
pushBack(新请求) → 加入队尾
front() → 查看队头
popFront() → 处理队头请求

支持 cancel：只能取消队头前 N 个
```

**周末处理：**

```solidity
bool public isWeekend;

// 周末期间：不更新 epoch，不处理赎回队列
// updateEpoch(true) → 标记为周末
// setWeekendFlag() → 手动覆盖
```

T-Bill 在非交易日没有价格更新，OpenEden 做了明确的周末/公共假期处理逻辑。

---

### 即时赎回：redeemIns + 可插拔赎回合约

```solidity
// redeemIns 调用可插拔的 redemptionContract
function _processWithdrawIns(...) internal {
    _transfer(_sender, address(this), _shares); // 锁定 shares
    uint256 usdcReceived = redemptionContract.redeem(_assets + 1e6); // 用 BUIDL 赎回
    // 超额赎回的 USDC 会被 offRamp 到 treasury
}
```

**即时赎回路径：**

```
TBILL → 按当前价格计算 USDC 数值
     → 调 redemptionContract（BUIDL/USYC 赎回合约）
     → 赎回对应价值的 BUIDL/USYC → 换成 USDC
     → 扣除即时赎回费
     → USDC 发给用户
```

关键设计：即时流动性来自 BUIDL 或 USYC 这类链上可赎回资产，而不是合约自己囤 USDC。流动性来源可配置（`setRedemption()`），可随时切换。

---

### 费用体系：多层次费率 + 合作伙伴折扣

```
交易费（每笔）
  → 按金额 × feePct（bps）计算
  → 有最低费用（默认 $25）
  → 周末/工作日不同费率

管理费（每日累积）
  → 在 updateEpoch() 时按 AUM × 年化费率 / 365 计算
  → 累积在 unClaimedFee，operator 手动提取

合作伙伴费率（Partnership.sol）
  → 被推荐用户享受费率折扣（pFee 可为负数 = 补贴）
  → 推荐人和被推荐人都有收益调整
```

这个 Partnership 设计非常聪明——可以给 B2B 合作伙伴（交易所、钱包）做定制费率，支持渠道分销。

---

## Prism：下一代多资产架构

新的 Prism 产品解决了 V5 的核心局限——单一资产（TBILL）。

**Prism 核心合约：**

```
AssetRegistry.sol  → 多资产注册表，每种资产有独立 priceFeed
Vault.sol          → ERC-4626 标准，多资产 Vault
Token.sol          → 每个资产池对应一个 Token
RedemptionQueue.sol → 独立赎回队列合约
Express.sol         → 即时赎回（约 26KB，最大的合约）
MintRedeemLimiter.sol → 每日铸造/赎回限额
```

Prism 的核心思路：把资产、Vault、赎回队列全部模块化，一套框架支持任意 RWA 资产。

---

## 与你的 rwa-protocol 逐项对比

**Token 设计**
- OpenEden：价格上涨模型，decimals = USDC decimals（6 位）
- 你的：价格上涨模型，18 位（ERC-20 默认）。需要明确 decimals 设计

**KYC 架构**
- OpenEden：5 个独立角色，最小权限，可升级合约
- 你的：单 OPERATOR_ROLE 混合所有权限，不可升级。安全性低

**价格 Oracle**
- OpenEden：双轨道（currentPrice + closeNavPrice），7 天 staleness 检测，偏差用中间价算法
- 你的：单轨道，5% 涨跌幅，无 staleness 检测。差距较大

**赎回模式**
- OpenEden：T+1 队列（DoubleQueueModified）+ 即时赎回（接 BUIDL 流动性）
- 你的：T+1 pending → operator fulfill + 即时赎回（需合约持有 USDC buffer）。流动性来源不同

**费用体系**
- OpenEden：交易费 + 管理费 + 合作伙伴折扣三层
- 你的：仅 instantRedemptionFeeBps。管理费和合作伙伴体系缺失

**Factory**
- OpenEden：无 Factory（单一产品）
- 你的：有 Factory（多产品），但有 Critical Bug

**周末/假期处理**
- OpenEden：`isWeekend` + `updateEpoch` 显式处理
- 你的：无。T-Bill 在交易日外没有 NAV，这个逻辑不能缺

---

## 关键竞争差距总结

OpenEden 领先你的核心点（按重要性排序）：

**第一：MAS 合规资质**
他们有 CMS License，你还没有。这是最难追的差距，不是技术问题。

**第二：底层资产来源**
OpenEden 直接对接美国 T-Bill 托管商，你还没有资产合作方。

**第三：即时赎回流动性**
他们接了 BUIDL（BlackRock）作为即时流动性来源，你的即时赎回依赖合约 USDC 余额。

**第四：费用体系**
他们有完整的管理费日历 + 合作伙伴分发体系，你还没有。

---

## 差异化机会（你可以超越他们的地方）

**机会一：本地资产**
OpenEden 做的是美国 T-Bill，新加坡 SGS Bonds / MAS Bills 是空白。本地资产 = MAS 审查更友好 + 本地机构投资者偏好。

**机会二：工厂化多资产**
你的 RWAFactory 设计支持一键部署多个产品，OpenEden 的 V5 是单一资产。Prism 在做这件事但还不成熟。

**机会三：支付基础设施整合**
你在 DCS Card Centre 的稳定币收单 + 卡支付基础设施，可以做「消费 → RWA 收益」的完整闭环，OpenEden 没有这个资源。

**机会四：B2B 白标（CaaS 模式）**
你做过 U 卡白标，同样的思路可以做「RWA 发行即服务」，让其他金融机构用你的平台发行自己的 RWA 产品。OpenEden 是纯 B2C。

---

## 参考资料

- GitHub: https://github.com/OpenEdenHQ
- 主合约: openeden.vault.audit / openeden.prism.audit
- 已通过多轮审计（Trail of Bits、Sigma Prime 等）
