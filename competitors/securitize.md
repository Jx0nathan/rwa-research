# Securitize — 调研笔记

> 调研时间：2026-03-02
> 官网：https://securitize.io
> 定位：RWA Token 化基础设施平台

---

## 一句话定位

**RWA 赛道的「AWS」，不自己发资产，专门为机构提供 Token 化的技术栈 + 合规基础设施，服务 BlackRock、KKR、Hamilton Lane 等头部资管机构。**

---

## 商业模式

B2B SaaS + 服务费，不靠 AUM 管理费：

```
收入来源：
  → Token 化平台服务费（按项目收）
  → KYC/AML 服务费（Securitize ID）
  → 二级市场交易撮合费（Securitize Markets）
  → 托管服务费（Securitize Trust Company）
```

Securitize 的定位是卖铲子，不挖矿。

---

## 产品矩阵

**Securitize Platform（发行平台）**
- Token 发行全流程：合规设计 → 合约部署 → 投资者管理
- 支持多链：Ethereum、Polygon、Avalanche、Aptos 等
- 内置 CAP Table 管理、股东通讯、分红分配

**Securitize ID（合规身份）**
- 通用 KYC/AML 身份凭证，做过一次可跨产品复用
- 与 BlackRock BUIDL、Hamilton Lane、KKR 的 Token 产品共用同一套 KYC
- 相当于链上「合规护照」

**Securitize Markets（二级市场）**
- SEC 注册的另类交易系统（ATS）
- 为 Token 化证券提供合规二级流动性
- 解决 RWA 流动性问题的关键一环

**Securitize Trust Company**
- 持牌信托公司，提供数字资产托管
- 监管背书，机构合规要求必备

---

## 核心客户

- **BlackRock BUIDL** — 最大的合作背书，BUIDL 的整个技术 + KYC 基础设施
- **KKR** — 私募股权基金 Token 化（K-PRIME）
- **Hamilton Lane** — 私募市场基金 Token 化
- **Bain Capital** — 私募基金份额 Token 化
- **Apollo Global** — 通过 Securitize 发行信贷产品

---

## 技术架构特点

**DS Token 协议（Digital Security Token）**

Securitize 自研的合规 Token 标准，ERC-20 超集：

```solidity
// 核心合规功能
interface IDSToken {
    function transfer(...) → 检查双方合规状态
    function isWhitelisted(address) → KYC 验证
    function freeze(address) → 临时冻结
    function seize(address, uint256) → 监管强制转移（法院命令执行）
    function forceTransfer(...) → 企业行动（如并购）
}
```

特有的 `seize` 功能支持法院命令强制执行，这是传统证券的法律要求，普通 ERC-20 没有。

**合规检查流程：**

```
每次 transfer
  → 查 Securitize Compliance Service
  → 检查 from/to 是否在白名单
  → 检查转账是否违反持有限制（如 Reg D 锁定期）
  → 检查额度限制
  → 通过 → 执行 transfer
```

---

## 合规资质（护城河）

- **SEC Transfer Agent 执照** — 可以合法记录证券所有权变更
- **SEC Registered Investment Adviser** — 可以提供投资建议
- **FINRA Broker-Dealer（Securitize Markets）** — 可以合法撮合证券交易
- **OCC 信托执照（Securitize Trust）** — 可以托管数字资产

这四张牌照组合，是 Securitize 真正的护城河，竞争对手拿到同样资质至少需要 3-5 年。

---

## 与 Ondo 的关系

不是竞争，是上下游：

```
Securitize（基础设施层）
    ↓
BlackRock BUIDL（Token 化产品）
    ↓
Ondo OUSG（持有 BUIDL 的 DeFi 接口层）
    ↓
DeFi 用户（Pendle/Morpho 的 Ondo 用户）
```

Ondo 是 Securitize + BlackRock 的下游分发渠道之一。

---

## 对你的项目的启示

新加坡的 Securitize 等价物 → **Hashstacs / iSTOX**

- iSTOX（现在的 ADDX）：MAS 持牌，新加坡本地 RWA 平台，Temasek 参投
- 如果你不想自建合规技术栈，和 ADDX 这类平台合作是最快路径
- 但合作意味着让渡控制权和收入分成，长期战略价值有限

**自建 vs 合作判断：**
- 第一个产品 → 考虑和 ADDX 合作，验证市场
- 有了 PMF → 自建技术栈，掌控完整链条

---

## 技术架构（源码级）

> 参考：Securitize DS Token Protocol 文档
> 参考：https://github.com/securitize-io（部分开源）

### DS Token 协议架构

DS（Digital Security）Token 是 Securitize 自研的合规 Token 标准，是 ERC-20 的严格超集。核心设计：把合规逻辑从 Token 合约分离到可插拔的 Service 层。

```
┌─────────────────────────────────────────────────────────────┐
│                     DSToken.sol（ERC-20）                   │
│  transfer / transferFrom → 调用 compliance hook             │
│  mint / burn → 仅 DSTokenManager 可调                       │
│  seize / freeze / forceTransfer → 仅 admin 可调             │
└─────────────────────────────┬───────────────────────────────┘
                              │ validate(from, to, amount)
┌─────────────────────────────▼───────────────────────────────┐
│              DSComplianceService.sol（合规编排层）           │
│   按顺序调用各 Service 插件                                   │
└──┬──────────────┬────────────────┬──────────────┬───────────┘
   │              │                │              │
   ▼              ▼                ▼              ▼
DSLock     DSTransfer       DSRegistry      DSCountry
Manager    Validator        Service         Compliance
（锁定期）  （额度/频率限制） （投资者名册）  （国家/地区限制）
```

---

### 合规检查完整调用链

每次 `transfer(to, amount)` 触发以下检查序列：

```solidity
// DSToken.transfer() 内部
function transfer(address to, uint256 amount) public override returns (bool) {
    // 1. 调用合规验证（任何失败都 revert）
    dsService.validate(msg.sender, to, amount);

    // 2. 通过验证后执行标准 ERC-20 transfer
    return super.transfer(to, amount);
}

// DSComplianceService.validate() 检查顺序：
// ① 发送方是否被冻结（freeze list）
// ② 接收方是否被冻结
// ③ 发送方 KYC 状态（在 DSRegistryService 中查询）
// ④ 接收方 KYC 状态
// ⑤ Reg D 锁定期（12 个月，起算自首次购买日）
// ⑥ 国家限制（Reg S 产品不能转给美国人）
// ⑦ 最大持有人数量限制（Reg D Rule 504: ≤ 2000 人）
// ⑧ 单笔转账金额限制（可选）
// 全部通过 → 返回 true；任意失败 → revert 含错误码
```

**合规错误码体系：**

```
WALLETS_ARE_NOT_WHITELISTED     → 接收方未通过 KYC
TOKENS_ARE_LOCKED               → 在 Reg D 锁定期内
COUNTRY_NOT_ALLOWED             → 国家/地区不在白名单
MAXIMUM_NUMBER_OF_HOLDERS       → 持有人数量超过上限
TRANSFER_AMOUNT_EXCEEDS_LIMIT   → 金额超过限制
```

---

### 投资者身份：Securitize ID（链上合规护照）

Securitize ID 是跨产品复用的 KYC 身份系统：

```
线下 KYC 流程：
  用户提交证件 → Securitize 人工/自动审核 → 生成 Securitize ID

链上表示：
  mapping(address wallet => InvestorRecord) registry;

  struct InvestorRecord {
      bytes32 investorId;     // Securitize 内部 ID
      uint8 countryCode;      // 所在国家
      uint64 kycExpiry;       // KYC 有效期（timestamp）
      InvestorType investorType; // INDIVIDUAL / ENTITY / QUALIFIED_PURCHASER
      bool accredited;        // 是否是 Accredited Investor（Reg D）
      bool qualified;         // 是否是 Qualified Purchaser（更高门槛）
  }
```

**复用机制：** 一次 KYC，可以访问所有使用 Securitize ID 的产品（BUIDL、KKR、Hamilton Lane）。每个产品合约查询同一个 `DSRegistryService`，无需重复 KYC。

---

### 特有的监管执行功能

这是 DS Token 区别于所有普通 ERC-20 最核心的合规功能：

```solidity
// 1. 冻结（临时，可解冻）
// 用途：监管调查、可疑交易暂停
function freeze(address wallet) external onlyRegulator;
function unfreeze(address wallet) external onlyRegulator;

// 2. 强制转移（seize）
// 用途：法院命令执行、判决执行（如离婚财产分割）
// 注意：bypasses 所有合规检查，强制从 from 转到 to
function seize(address from, address to, uint256 amount) external onlyRegulator;

// 3. 企业行动（forceTransfer）
// 用途：并购、股票分割、私有化
function forceTransfer(address from, address to, uint256 amount) external onlyAdmin;

// 4. 强制销毁（burnFrom）
// 用途：股票回购、赎回
function burnFrom(address from, uint256 amount) external onlyAdmin;
```

这套功能对应传统 Transfer Agent 的法律职责：DTCC（美国证券结算中心）可以在特定情况下强制调整账簿记录，DS Token 把这个功能以链上函数的形式实现。

---

### 锁定期实现（Reg D 合规核心）

Reg D Rule 144 要求非 Accredited Investor 购买的证券须锁定 12 个月：

```solidity
// DSLockManagerService.sol
struct LockRecord {
    uint64 lockedUntil;    // 解锁时间戳
    uint256 lockedAmount;  // 锁定数量
    uint8 lockType;        // REG_D / RULE_144 / CUSTOM
}

mapping(address => LockRecord[]) public locks;

// 检查发送方是否有锁定的 Token
function isTransferAllowed(address from, uint256 amount) external view returns (bool) {
    uint256 lockedTotal = 0;
    for (uint i = 0; i < locks[from].length; i++) {
        if (block.timestamp < locks[from][i].lockedUntil) {
            lockedTotal += locks[from][i].lockedAmount;
        }
    }
    // 可转金额 = 总余额 - 仍在锁定期内的金额
    return balanceOf(from) - lockedTotal >= amount;
}
```

锁定记录在 mint 时自动创建，锁定时长由产品层面配置（Reg D = 12 个月，自定义锁定 = 任意时长）。

---

### 多链部署策略

Securitize 在多条链部署相同的合规 Token：

```
核心原则：每条链上独立部署，共享同一套 KYC 数据

Ethereum（主链）：原始 Token，全功能
    ↓ 官方桥（Securitize 自有或第三方）
Polygon / Avalanche / Aptos：镜像 Token

合规数据同步：
  - KYC 状态存储在 Securitize 中心化服务器
  - 每条链上的 DSRegistryService 通过 API 查询（非链上同步）
  - 即：不同链上的合规状态更新有轻微延迟风险

跨链转账：
  → 在原链 burn Token
  → Securitize 桥接层验证并通知目标链
  → 目标链 mint 等量 Token
  → 全程需合规验证（不能通过跨链绕过 KYC）
```

---

### 与传统 Transfer Agent 的技术对比

| 功能 | 传统 TA（DTCC + 登记处） | Securitize DS Token |
|------|------------------------|---------------------|
| 所有权记录 | 中心化数据库（Cede & Co 名义持有） | 链上 mapping，透明可查 |
| KYC 验证 | 经纪商自行完成，TA 不直接接触 | Securitize ID 统一管理 |
| 转账执行 | T+2 结算，DTC 清算 | 链上即时（受合规检查约束） |
| 法院命令执行 | TA 接受法院文件，手动调账 | seize() 函数，链上执行 |
| 股东大会 | 邮件通知，代理投票 | 链上治理（可选，需额外模块） |
| 分红分配 | TA 批量发送支票/转账 | 合约自动按持股比例分配 |
| 合规成本 | 高（人工审查） | 低（规则编码化，链上自动执行） |

**Securitize 最大的价值不是技术，是合规资质。** DS Token 本身技术门槛不高，但 SEC Transfer Agent 执照 + FINRA Broker-Dealer + OCC 信托执照的组合，是任何新进者 3-5 年内无法复制的。

---

### 对我们项目的参考价值

**可借鉴的设计点：**

1. **合规检查插件化（Service 架构）**
   把 KYC、锁定期、国家限制分成独立 Service，未来可以独立升级某个合规逻辑，不需要修改核心 Token 合约。比我们目前的 hardcode whitelist 更灵活。

2. **seize/freeze 功能**
   如果未来面向机构客户或做 STO，这两个函数是监管合规必须的。MAS 在某些情况下有权要求冻结资产，链上 freeze 功能比联系合约 owner 手动处理更快、更有法律依据。

3. **Securitize ID 复用模式**
   类似思路：建立一个统一的"新加坡 RWA KYC 护照"，用户完成一次认证，可以访问多个 RWA 产品。这是 B2B 平台化的关键基础设施。

**不适合直接借鉴的：**

- DS Token 的多链策略依赖 Securitize 中心化 KYC 服务器，我们如果做去中心化程度更高的平台，不适合这种架构。
- seize() 等强制函数增加了 Token 持有人的信任风险，零售产品不宜暴露此类接口。
