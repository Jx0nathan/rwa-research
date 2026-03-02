# RWA 核心概念

> 整理时间：2026-03-02

---

## 什么是 RWA

Real World Assets（真实世界资产）的链上 Token 化。把链下传统金融资产（债券、股票、房地产、商品等）的所有权或收益权，通过智能合约表示在区块链上。

---

## 主要资产类型

**固定收益类（目前最成熟）**
- 美国国债 / 货币市场基金 → Ondo OUSG/USDY, Franklin BENJI
- 新加坡政府债券 SGS / MAS Bills
- 企业债、ABS

**权益类**
- 美股 Token → Ondo Global Markets（TSLAon、AAPLon）
- 私募股权

**不动产类**
- 商业地产份额化 → RealT、Lofty
- REIT Token 化

**大宗商品**
- 黄金 Token → PAX Gold、Tether Gold

---

## Token 类型对比

**份额型（Fund Share Token）**
- 代表基金 LP 份额
- 法律结构：LP / VCC 基金
- 代表：OUSG（Ondo I LP 份额）
- 定价：按 NAV 每日更新

**债务型（Debt / Note Token）**
- Token 是发行方的债务凭证
- 法律结构：SPV 发行结构性票据
- 代表：USDY（Ondo USDY LLC 债务）、Ondo GM（BVI SPV 结构性票据）
- 特点：持有人无股东权利，但有第一顺位担保

**收益权型**
- 代表未来现金流的收益权
- 代表：Centrifuge（发票、贷款收益权）

---

## 关键合规概念

**Reg D（美国）**
- 面向 Accredited Investor 的私募豁免
- 可向美国投资者销售
- 有持有期限制（Rule 144）

**Reg S（美国）**
- 面向非美国人的境外发行豁免
- 不能向美国人销售
- Ondo USDY、Ondo GM 都走这条路

**MAS 框架（新加坡）**
- CMS License（资本市场服务牌照）
- VCC（Variable Capital Company）— 2020 年推出，专为基金设计
- MAS Sandbox — 可在监管沙盒内测试创新产品

---

## NAV（净资产价值）机制

```
NAV = 底层资产总价值 / Token 总供应量

每日更新流程：
  底层资产收盘价 → 基金管理人计算 NAV → Oracle 推送链上
  → Token 价格更新（价格型）或 Rebase（数量型）
```

**价格型（OUSG 模式）**
- Token 数量不变，单价每天上涨
- 适合机构（会计处理简单）

**数量型 Rebase（USDY 模式）**
- Token 价格始终 ~$1，数量每天增加
- 适合 DeFi 集成（Uniswap 等协议价格稳定）

---

## T+1 / T+0 结算

**T+1（标准）**
- 用户今天申购，明天处理（买入底层资产）后 mint token
- 安全，链下可操作
- 用户体验差（需要等待）

**T+0（即时）**
- 用户申购立即 mint token
- 需要流动性缓冲池（operator 提前备足资金）
- 或通过报价锁定 + 原子交换实现（Ondo GM 模式）

---

## 主要风险

- **Oracle 风险**：NAV 数据被篡改或延迟
- **托管风险**：底层资产托管方破产
- **流动性风险**：底层资产无法快速变现满足赎回
- **监管风险**：所在司法管辖区监管政策变化
- **智能合约风险**：合约漏洞导致资产损失
