# Grove Finance — 技术架构

> 源码：https://github.com/grove-labs/grove-alm-controller
> 文档：https://docs.grove.finance/grove-allocator
> 调研时间：2026-03-22

---

## 核心设计哲学

Grove 的架构不是 RWA 代币化平台，而是**链上 ALM（资产负债管理）系统**。

传统银行的 ALM 是：管理负债端（存款/债券）和资产端（贷款/投资）之间的期限、利率、流动性错配。Grove 把这套逻辑搬到链上：

- **负债端：** USDS 稳定币（Sky 发行，零期限）
- **资产端：** CLO 票据、私募信贷（有期限、有信用风险）
- **ALM 系统：** Grove Allocator（管理两者之间的桥接、速率、风险）

---

## 合约架构：三层分离

```
┌─────────────────────────────────────────────────────┐
│                  Relayer（链下 ALM Planner）          │
│         （离链组件，由 Grove 团队运营，发起交易）         │
└───────────────────────┬─────────────────────────────┘
                        │ 调用
┌───────────────────────▼─────────────────────────────┐
│            Controller 层（业务逻辑）                  │
│  ┌──────────────────────┐ ┌──────────────────────┐  │
│  │  MainnetController   │ │  ForeignController   │  │
│  │  （Ethereum 主网）    │ │  （Avalanche 等 L2）  │  │
│  └──────────┬───────────┘ └──────────┬───────────┘  │
│             │ 验证速率限制              │              │
│  ┌──────────▼───────────────────────▼───────────┐   │
│  │            RateLimits 合约（风控层）            │   │
│  └────────────────────────────────────────────────┘  │
└───────────────────────┬─────────────────────────────┘
                        │ 执行
┌───────────────────────▼─────────────────────────────┐
│              ALMProxy（托管层）                       │
│         持有所有资金，无状态，仅授权 Controller 调用    │
└───────────────────────┬─────────────────────────────┘
                        │ 与外部协议交互
              ┌─────────┴──────────────────────┐
              │                                │
     ┌────────▼──────┐               ┌─────────▼──────┐
     │  DeFi 协议    │               │   RWA 金库      │
     │  Aave V3      │               │  Centrifuge     │
     │  Curve        │               │  ERC7540 异步   │
     │  Uniswap V3   │               │  Pendle Finance │
     │  Ethena       │               │                 │
     └───────────────┘               └────────────────-┘
```

---

## 核心合约详解

### 1. ALMProxy — 托管层

**设计原则：极简、无状态、只做执行**

```solidity
// ALMProxy.sol
contract ALMProxy is AccessControl {
    bytes32 public constant CONTROLLER = keccak256("CONTROLLER");

    // 三个执行函数，仅 CONTROLLER 可调
    function doCall(address target, bytes calldata data) external onlyRole(CONTROLLER)
        returns (bytes memory);

    function doCallWithValue(address target, bytes calldata data, uint256 value) external onlyRole(CONTROLLER)
        returns (bytes memory);  // 用于跨链 msg.value 费用

    function doDelegateCall(address target, bytes calldata data) external onlyRole(CONTROLLER)
        returns (bytes memory);  // 在 Proxy 上下文中执行

    receive() external payable;  // 接收 ETH 用于跨链费用
}
```

**关键设计决策：**
- ALMProxy 没有业务逻辑，只负责持有资产 + 转发调用
- Controller 可以独立升级：授予新 Controller CONTROLLER 角色 → 旧 Controller 撤权 → **资金一分钱不动**
- 一个漏洞不会传染整个系统（不像单合约架构，攻击一点就全盘崩）

---

### 2. MainnetController — 以太坊主网业务逻辑

支持的操作分类（所有操作须通过 RateLimits 检验）：

**稳定币操作**
```
USDS Mint / Burn    → Sky Allocation Vault（从 Sky 借/还 USDS）
DAI ↔ USDS 转换    → Maker PSM
USDS ↔ USDC 转换   → Mainnet PSM
```

**收益资产**
```
ERC4626 Vaults    → 标准收益金库（存入/赎回），含汇率验证（防操纵）
ERC7540 Vaults    → 异步金库（请求 + 等待 + claim），适配 CLO / 信贷等流动性差资产
Centrifuge RWA    → 管理异步存赎请求 + 跨链 Share Token 转移
```

**DeFi 操作**
```
Aave V3           → 存入/取出，含滑点检验
Curve / Uniswap   → Swap + LP（含 TWAP Oracle 验证、tick 范围检查）
Ethena            → USDe mint/burn、sUSDe cooldown
Pendle            → 到期后赎回本金 Token
```

**跨链操作**
```
CCTP              → 桥接 USDC 到外链（Circle 官方跨链）
LayerZero         → 通用 Token 跨链
双重速率限制       → 全局 cap + 单目标链 cap
```

---

### 3. ForeignController — 非主网操作

和 MainnetController 基本一致，差异点：

- 移除 USDS Mint/Burn、Ethena、DAI↔USDS 操作（仅主网有）
- 新增 PSM3（Spark PSM，外链稳定币 hub）替代主网 PSM
- 其余操作（ERC4626、ERC7540、Centrifuge、Aave V3、DEX、Pendle、跨链）完全一致

**跨链流程（以 Avalanche 为例）：**
```
主网 USDS
  → CCTP / LayerZero 桥接到 Avalanche
  → ForeignController 管理本地资产配置
  → PSM3 提供 USDC ↔ USDS 本地转换
```

---

### 4. RateLimits — 风控层（最有意思的设计）

**问题：** 如果 Relayer 被黑客控制，或 Grove 团队作恶，如何限制单次可提取的资金量？

**解法：线性恢复速率限制**

```
可用额度（当前）= min(maxAmount, lastAmount + slope × (now - lastUpdated))
```

**例子：**
- maxAmount = 10,000,000 USDC
- slope = 115.74 USDC/秒（约 10M/天）
- 若 Relayer 一次提走 10M → 额度归零 → 每秒只能追加 115 USDC
- 24小时后才能恢复满额
- 黑客即使控制 Relayer，24小时最多也只能转移 10M，而不是全部 TVL

**Key 的设计（防止碰撞）：**

```solidity
// 不同操作用不同 key 做速率限制，互不干扰
makeAssetKey(key, asset)                      // 单资产存入特定金库
makeAssetDestinationKey(key, asset, dest)     // 单资产 × 目标地址
makeDomainKey(key, domain)                    // 跨链到特定域
```

**两个状态更新函数：**
```solidity
triggerRateLimitDecrease(key, amount)  // 资金流出时调用（检查是否超限）
triggerRateLimitIncrease(key, amount)  // 资金回流时调用（还 USDS 增加铸造额度）
```

---

## 权限体系

四个角色，职责严格分离：

| 角色 | 权限 | 持有方 |
|------|------|--------|
| DEFAULT_ADMIN_ROLE | 配置参数、授权/撤权 | Sky 治理（多签） |
| RELAYER | 调用 DeFi 操作（存/取/Swap/桥接） | Grove 运营地址（ALM Planner） |
| FREEZER | 紧急撤销 RELAYER 权限，冻结所有操作 | 独立安全委员会 |
| CONTROLLER | 调用 ALMProxy 执行函数 | Controller 合约本身 |

**紧急流程：**
```
发现异常 → FREEZER 调用 removeRelayer() → RELAYER 权限撤销
→ 所有自动化操作立即停止 → ALMProxy 资金原地锁定
```

---

## 审计安全体系

每个主要版本至少两家独立审计公司，外加 Certora 形式化验证：

- **v1.6.0：** Spearbit（Cantina）+ ChainSecurity
- **v1.8.0：** Certora 形式化验证 + ChainSecurity
- **治理合约：** Spearbit + Certora + ChainSecurity

形式化验证覆盖关键不变量（速率限制不超上限、只有授权角色可执行等），数学证明级别，不是 fuzzing。

---

## ERC7540 异步金库：接入 RWA 的关键桥梁

RWA 资产（CLO 票据、私募信贷）的赎回周期可能是 T+30、T+90，和 DeFi 的即时赎回需求天然冲突。

ERC7540 解决了这个问题：

```
用户/协议请求存入 → requestDeposit(assets, operator, owner)
  ↓ 金库处理（可能是链下处理 T+N 天）
金库准备好 → 触发 DepositClaimable 事件
  ↓
用户 claim → claimDeposit(requestId, receiver, owner)
```

赎回流程类似（requestRedeem → 等待 → claimRedeem）。

**Grove 在 MainnetController 中支持完整的 ERC7540 生命周期，包括取消请求（cancelDepositRequest / cancelRedeemRequest）。**

---

## 对自建 RWA 协议的启示

| 维度 | Grove | 自建参考 |
|------|-------|---------|
| 资金托管 | ALMProxy 无状态，Controller 可升级 | 避免把业务逻辑和资产托管放在同一合约 |
| 速率限制 | 线性恢复，分操作分资产 | 所有大额资金操作都应该有速率限制 |
| 异步赎回 | ERC7540 标准，Centrifuge 已经支持 | 若接 TradFi 资产必须用异步金库 |
| 紧急熔断 | FREEZER 一键撤权 | 任何系统都要有独立的紧急停止机制 |
| 跨链 | CCTP + LayerZero 双通道 | 不要单依赖一个桥，双桥降低单点风险 |
| 形式化验证 | Certora 关键路径验证 | $1M+ TVL 就应该做形式化验证 |

---

## 参考资料

- 合约源码：https://github.com/grove-labs/grove-alm-controller
- 协议文档：https://docs.grove.finance/grove-allocator
- 安全审计：https://docs.grove.finance/security
- 数据仪表盘：https://data.grove.finance
- GROVE Token 合约：https://etherscan.io/token/0xB30FE1Cf884B48a22a50D22a9282004F2c5E9406
