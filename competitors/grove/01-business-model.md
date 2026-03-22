# Grove Finance — 商业逻辑

## 一句话总结

**不是卖债券，是争夺 DeFi 抵押品的席位。**

Grove 把机构信用资产（CLO、私募信贷）注入 MakerDAO/Sky 的储备体系，让 USDS 稳定币有更高收益的底层支撑，同时让 RWA 债券获得 DeFi 杠杆需求的放大。

---

## Sky 生态关系（最关键的背景）

Grove 是 Sky（原 MakerDAO）生态的 **Star（SubDAO）**，不是独立协议：

- Sky 负责 $10B+ 抵押基础设施和 USDS 稳定币发行
- Grove 负责把 USDS 部署进机构信贷市场，赚取超额收益
- Sky 给 Grove 分配 USDS 额度（Allocation），Grove 向 Sky 汇报收益
- GROVE Token 70% 分配给 Sky 生态（Sky Farms 用 USDS/SKY 质押挖 GROVE）

**本质：** Grove 是 MakerDAO 的链上资产管理部门，背后的信用背书是 MakerDAO 十年的去中心化金融基础设施。

---

## 核心产品：Grove Allocator

Grove 只做一件事：**把 USDS 稳定币配置进收益更高的信用资产**

资金流向：
```
Sky 分配 USDS 额度
    ↓
Grove Allocator（ALMProxy 托管）
    ↓
分散配置至：
  ├── CLO 票据（Galaxy tokenized CLO，$1B 部署）
  ├── 私募信贷（Apollo ACRDX，$50M 部署）
  ├── Aave V3 借贷（RLUSD / USDC 流动性）
  ├── Centrifuge RWA 金库（异步赎回结构）
  ├── Pendle Finance（固定收益本金 Token）
  └── DEX 流动性（Curve / Uniswap V3）
```

---

## 关键里程碑（按时间线）

**2025年7月** — Grove 正式发布，定位 Sky 生态机构信用协议

**2025年7月** — Avalanche 部署，$250M 目标配置

**2025年8月** — $1B USDS 部署进 Galaxy tokenized CLO
- CLO = Collateralized Loan Obligation（抵押贷款凭证），传统金融最成熟的信用产品之一
- Galaxy 负责发行和评级，Grove 购买 B 类票据（次级，收益更高）

**2025年9月** — $50M 部署进 Apollo ACRDX（多元化全球信贷策略，Plume 链上）

**2025年10月** — 接入 Aave Horizon，供应 RLUSD / USDC 流动性

**2026年1月** — 又购买 $50M Galaxy tokenized CLO（B 类票据），在 Avalanche 上执行

**当前 TVL：$2.38B，30日 APY 2.41%**

---

## 竞争护城河

**一、Sky 生态背书**

MakerDAO 管理 $10B+ 抵押资产，是最大的去中心化稳定币发行方。Grove 作为其 SubDAO 获得：
- 几乎无限的 USDS 额度来源（只要 Sky 治理批准）
- MakerDAO 品牌信用（机构认知度远超新协议）
- 硬件：Sky 的 ALM 基础设施（Grove Allocator 直接复用）

**二、机构信用资产的议价能力**

$1B 规模一次性配置进 CLO → Galaxy 给 Grove 的条款远优于散户。
信用资产的规模效应比 DeFi 借贷池更明显，越大越有话语权。

**三、DeFi 杠杆需求放大**

一旦 Grove 的 RWA 债券被 Aave、Compound 等协议接受为抵押品：
```
用户存 Grove RWA 凭证
  → 借出 USDC/USDS
  → 再去其他 DeFi 协议生息
  → Grove RWA 的需求被杠杆放大
```
这不是线性增长，是乘数效应。Grove 现在和 Aave Horizon 的合作，正是在铺这条路。

---

## GROVE Token 经济

- 总量：100 亿枚
- Sky Farms（质押 USDS/SKY 挖矿）：70% = 70 亿枚，8年线性释放
- Grove Labs（团队）：25% = 25 亿枚
- Grove Foundation：5% = 5 亿枚

**年释放计划（前两年最多）：**
- Year 1：17.5 亿枚
- Year 2：17.5 亿枚
- Year 3：8.75 亿枚（减半）
- ... 每两年减半

**Token 用途：**
- 治理投票（草案阶段）
- 质押（后续解锁协议功能，具体机制待定）
- 由 MCD_PAUSE_PROXY（Sky 治理合约）控制，Grove 团队不能单方面修改 Token 合约

---

## 收入模型

Grove 的收入来自：
- 信用资产超额收益和 USDS 成本之间的**利差**（Spread）
- 规模越大，利差绝对值越高
- 有别于 Ondo 的 AUM × 0.15% 固定管理费模型，Grove 更像传统信贷基金

具体利差数据未公开披露，但 2.41% 的 30日 APY 已显示其产品有竞争力（美债利率 ~4.5%，说明底层资产在承担一定信用风险换取更高收益）。

---

## 和 Ondo 的本质差异

| 维度 | Ondo | Grove |
|------|------|-------|
| 核心产品 | 代币化美债（零信用风险） | 代币化 CLO / 私募信贷（有信用风险） |
| 目标用户 | 机构 + 零售 | 机构（DeFi 协议为主） |
| 生态依附 | 独立 + BlackRock 合作 | MakerDAO/Sky SubDAO |
| 抵押品逻辑 | USDY 进入 DeFi 作抵押品 | USDS 通过 Grove 部署进 TradFi |
| 收益来源 | 美债收益 - 0.15% 管理费 | 信用资产超额收益 - USDS 成本 |
| Token 策略 | ONDO → L1 网络 Token | GROVE → SubDAO 治理 Token |

---

## 参考资料

- 官网：https://grove.finance
- 文档：https://docs.grove.finance
- 博客：https://grove.finance/blog
- 数据仪表盘：https://data.grove.finance
- Token 合约：https://etherscan.io/token/0xB30FE1Cf884B48a22a50D22a9282004F2c5E9406
- GitHub：https://github.com/grove-labs
