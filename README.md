# RWA Research

Real World Assets (RWA) 调研笔记，记录技术架构、商业逻辑、竞品分析及市场洞察。

## 目录结构

```
rwa-research/
├── competitors/          # 竞品分析
│   └── ondo/             # Ondo Finance 深度分析
├── concepts/             # 核心概念
└── architecture/         # 技术架构参考
```

## 调研进度

| 主题 | 状态 |
|------|------|
| Ondo Finance 商业逻辑 | ✅ 完成 |
| Ondo 技术架构（源码级） | ✅ 完成 |
| Ondo Chain（L1）分析 | ✅ 完成 |
| RWA 核心概念 | ✅ 完成 |
| MAS 合规框架 | 🔄 进行中 |
| Securitize 分析 | 📋 待开始 |
| Centrifuge 分析 | 📋 待开始 |
| Franklin Templeton BENJI | 📋 待开始 |

## 核心结论（持续更新）

**方向判断：** Protocol（RWA 发行平台）优于 Product（单一资产发行方），与现有支付基础设施结合形成差异化。

**底层资产首选：** 新加坡 SGS Bonds / MAS Bills，匹配 MAS 监管经验和本地资源。

**合规结构：** Singapore VCC 或 LP 基金，比开曼更便宜且 MAS 认可。

**Token 标准：** ERC-3643（T-REX）+ ERC-4626 组合，合规友好 + 收益分配。
