# Grove Finance 调研

> 官网：https://grove.finance
> 文档：https://docs.grove.finance
> GitHub：https://github.com/grove-labs
> X：https://x.com/grovedotfinance
> 调研时间：2026-03-22

## 一句话定位

**MakerDAO/Sky 生态孵化的机构信用协议，把 USDS 稳定币部署进 CLO、私募信贷等链下信用资产，TVL $2.38B，充当 DeFi 的固定收益底层。**

## 文件索引

- [01-business-model.md](./01-business-model.md) — 商业逻辑、产品线、Sky 生态关系、收入模型
- [02-technical-architecture.md](./02-technical-architecture.md) — 合约架构、ALM Proxy 设计、Rate Limits 风控

## 核心结论

**商业定位：** 不是 RWA 上链工具商，而是争夺 DeFi 抵押品席位。通过 Sky/MakerDAO 的 $10B+ 抵押基础设施背书，把机构信用资产做成 USDS 的底层储备，实现资产需求随 DeFi 杠杆放大。

**技术亮点：**
- ALMProxy 三层架构（托管 / 业务逻辑 / 风控分离），控制器可独立升级不迁移资金
- RateLimits 线性速率限制引擎，时间恢复机制，防止资本单点大量流出
- 支持 Centrifuge ERC7540 异步金库，兼容传统 RWA 上链流程
- 跨链部署（Ethereum + Avalanche），CCTP + LayerZero 双桥

**最关键的竞争逻辑：**
不是比谁的债券收益率更高，而是谁能成为 DeFi 协议（Aave、MakerDAO）的合格抵押品来源。
一旦 RWA 债券进入 DeFi 抵押品体系 → 需求被杠杆放大 → 资产规模是指数增长，不是线性增长。
