# Ondo Finance 调研

> 官网：https://ondo.finance
> 文档：https://docs.ondo.finance
> GitHub：https://github.com/ondoprotocol

## 文件索引

- [01-business-model.md](./01-business-model.md) — 商业逻辑、产品线、收入模型、合规结构
- [02-technical-architecture.md](./02-technical-architecture.md) — 源码级技术分析（PriceId 系统、零余额设计、三层合规）
- [03-ondo-chain-l1.md](./03-ondo-chain-l1.md) — Ondo Chain L1 分析及对自建项目的启示

## 核心结论

**商业模式：** AUM × 0.15% 管理费，实际现金流不大，估值靠生态叙事。

**技术亮点：**
- PriceId 快照定价 → 公平、防 operator 套利
- 合约零余额设计 → 最小攻击面
- Chainalysis 链上制裁名单 → OFAC 实时合规
- claimableTimestamp → T+1 链上强制

**竞争优势：** BlackRock + Coinbase 渠道，DeFi 深度集成（Pendle/Morpho/Curve）。
