# Ondo Finance — 商业逻辑

## 一句话总结

**把美国国债的收益率卖给全球加密用户，收的是资产管理费（AUM × 0.15%）。**

---

## 产品线

**OUSG（机构产品）**
- 底层资产：BlackRock iShares 短期美债 ETF（SHV）
- 目标用户：机构 / 合规用户
- 最低门槛：$100,000
- 年化：~5%，管理费 0.15%
- 合规结构：Reg D / Reg S 豁免

**USDY（DeFi 零售产品）**
- 底层资产：短期美债 + 银行存款
- 目标用户：非美国零售用户 / DeFi 协议
- 最低门槛：无
- 年化：~5%，管理费 0.15%
- 合规结构：Reg S（仅非美国人）

**Ondo Global Markets（最新，2025年）**
- 底层资产：100+ 只美股 / ETF（TSLAon、AAPLon 等）
- 目标用户：非美国机构投资者
- 特点：Total Return Tracker，即时铸造 / 赎回

---

## 资金流向

```
用户 USDC
  → 换成 USD（Coinbase / Circle）
  → 通过 Clear Street 买入 SHV ETF
  → 产生利息（~5% / 年）
  → Ondo 留 0.15%
  → 剩余回流给用户（OUSG 价格上涨 / USDY rebase）
```

---

## 收入模型

- 收入来源：AUM × 管理费率（0.15%）
- AUM 约 $10 亿 → 年管理费收入约 $150 万
- ONDO Token 市值 $20 亿+ → 依赖「RWA 赛道龙头」品牌溢价

**核心洞察：** 链上管理费收入本身不大，估值靠的是生态叙事和 Token 经济，不是现金流。

---

## 合规结构（关键）

**OUSG 结构：**
```
Ondo Finance Inc.（运营公司）
  └── Ondo Capital Management LLC（投资管理）
        └── Ondo I LP（特拉华 LP 基金）← 用户成为 LP
              └── 持有 SHV ETF
```
- 走 Section 3(c)(7) 豁免（美国 : Investment Company Act）, 意思是这个基金不注册成公募基金、不接受普通散户
- 仅向 Qualified Purchaser (超高净值投资者) 销售

**USDY 结构：**
```
Ondo USDY LLC（特拉华 SPV）
  → 向用户发行 USDY Token（债务凭证）
  → 资产：美债 + 银行存款
  → 向 FinCEN 注册为 Money Services Business
```
- 走 Reg S（仅非美国人）
- Token 是 LLC 的债务，不是股权

**Ondo Global Markets 结构：**
```
Ondo Global Markets (BVI) Limited（BVI SPV）
  → 发行结构性票据（Structured Note）
  → 底层：对应股票 / ETF
  → 安全代理：Ankura Trust（持有第一顺位担保权益）
```

---

## 竞争护城河

1. **BlackRock 关系** — OUSG 底层是 BlackRock iShares，BUIDL 进一步深化合作
2. **Coinbase 渠道** — USDY 在 Coinbase 上架，获得零售分发
3. **DeFi 集成深度** — USDY 接入 Pendle、Morpho、Curve，二级流动性

---

## 关键数据（2025年）

- 总 AUM：约 $10 亿+
- 主要链：Ethereum、Polygon、Solana
- 审计次数：10+（Code4rena、Spearbit、Cyfrin、Halborn 等）
