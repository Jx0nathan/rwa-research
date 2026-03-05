# RWA Research

Real World Assets (RWA) 调研笔记，记录技术架构、商业逻辑、竞品分析及市场洞察。

## RWA 的业务逻辑

![RWA Overview](./image/RWA-Overview.png)

## 目录结构

```
rwa-research/
├── competitors/                       # 竞品分析
│   ├── ondo/                          # Ondo Finance 深度分析
│   │   ├── 01-business-model.md       #   商业模式
│   │   ├── 02-technical-architecture.md #   技术架构（源码级）
│   │   └── 03-ondo-chain-l1.md        #   Ondo Chain L1
│   ├── blackrock-buidl.md             # BlackRock BUIDL（机构美债基金）
│   ├── franklin-templeton-benji.md    # Franklin Templeton BENJI（零售货基）
│   ├── securitize.md                  # Securitize（RWA 基础设施平台）
│   ├── centrifuge.md                  # Centrifuge（私人信贷 + DeFi）
│   ├── maple-finance.md              # Maple Finance（机构信贷借贷）
│   └── openeden.md                    # OpenEden
├── concepts/                          # 核心概念
│   └── rwa-fundamentals.md            # RWA 基础知识
└── image/                             # 图表资源
```

## 竞品研究覆盖

| 项目 | 赛道 | 研究深度 |
|------|------|----------|
| [Ondo Finance](competitors/ondo/) | 美债代币化 | 商业逻辑 + 技术架构（源码级）+ L1 |
| [BlackRock BUIDL](competitors/blackrock-buidl.md) | 机构美债基金 | 商业逻辑 |
| [Franklin Templeton](competitors/franklin-templeton-benji.md) | 零售货基 | 商业逻辑 |
| [Securitize](competitors/securitize.md) | RWA 基础设施 | 商业逻辑 |
| [Centrifuge](competitors/centrifuge.md) | 私人信贷 + DeFi | 商业逻辑 |
| [Maple Finance](competitors/maple-finance.md) | 机构信贷借贷 | 商业逻辑 |
| [OpenEden](competitors/openeden.md) | — | 商业逻辑 |

## 核心研究结论

### 市场规模（2026-03-05）

- 链上真实资产（Distributed）：**$261.7 亿**
- 含稳定币总计（Represented）：**$3352.9 亿**
- 两者差距 = 稳定币 **$2990.9 亿**

> RWA 市场本质上是稳定币市场。非稳定币 RWA 仅占整体 7.8%，所谓"RWA 爆发"大部分是稳定币的功劳。

### 各品类现状

**美国国债代币化** — $109 亿 / 64 个产品 / 仅 52,487 个持有人

纯机构市场，最小投资额 $100 万~$500 万。代表产品：
- BUIDL（BlackRock）：$22.4 亿，仅 101 个钱包地址
- USYC（Circle）：$18.5 亿，仅 43 个持有人
- OUSG（Ondo）：$7.5 亿，机构门控

**私人信贷** — 活跃贷款 $203.6 亿，平均收益 10.21%

Figure Finance 独占 $155.9 亿（约 76%），但其做的是 HELOC（住房净值贷款），本质是传统抵押贷款。去掉 Figure 后的真正链上信贷：
- Maple：Syrup USDC $11 亿（5.17% APY）+ Syrup USDT $6.8 亿
- Tradable：$22.9 亿（11.38% APY）
- Centrifuge：$7100 万（8.7% APY）
- Goldfinch：$5700 万（12.42% APY）

> Maple 是链上私人信贷的真正王者，不是 Centrifuge。

**黄金代币化** — 价格驱动增长，非 TVL 增长
- XAUT（Tether Gold）：$29.4 亿，30 天 +22.65%
- PAXG（Paxos Gold）：$25.3 亿，30 天 +18.22%

**不动产 + 股票** — 几乎不存在
- 不动产总计：< $2.5 亿；代币化股票总计：< $5 亿
- 讲了 3 年的故事，规模连 BUIDL 一个产品的零头都不到

### 链选择

| 链 | RWA 规模 | 份额 | 趋势 |
|----|----------|------|------|
| Ethereum | $153 亿 | 57.9% | 下降 |
| Solana | $17 亿 | 6.5% | 30 天 +45.81% |
| Stellar | $13 亿 | — | 30 天 +24.51% |
| XRP Ledger | $4.5 亿 | — | 30 天 +32.76% |

> Franklin BENJI、USDY、BUIDL 都已部署到 Solana。Solana 正在成为新 RWA 战场。
