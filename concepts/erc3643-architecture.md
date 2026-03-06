# ERC-3643（T-REX）合约架构与交互逻辑

> GitHub: https://github.com/ERC-3643/ERC-3643

## 一、整体架构：5个核心合约

```
┌─────────────────────────────────────────────────────────────┐
│                        Token.sol                            │
│              （ERC-20 + 合规钩子 + 冻结管理）                │
└──────────────┬─────────────────────┬───────────────────────┘
               │                     │
               ▼                     ▼
  IdentityRegistry.sol      ModularCompliance.sol
  （KYC 身份注册）            （合规规则容器）
       │                          │
       ▼                          ▼
  ┌────────────────┐     ┌────────────────────────┐
  │ Identity（链上 │     │  Module 1（国家限制）   │
  │ 身份 NFT）     │     │  Module 2（持仓上限）   │
  └────────────────┘     │  Module 3（投资者限额） │
       │                 │  Module N（可插拔...）  │
       ▼                 └────────────────────────┘
  ┌──────────────────────────────┐
  │ TrustedIssuersRegistry.sol   │  ← 谁有资格颁发 KYC 证明
  │ ClaimTopicsRegistry.sol      │  ← 需要哪些 KYC 证明类型
  └──────────────────────────────┘
```

---

## 二、transfer 完整交互流程

```
Alice.transfer(Bob, 100 TOKEN)
         │
         ▼
Token.transfer()
  ①  require(!frozen[Bob] && !frozen[Alice])     ← 检查是否被冻结
  ②  require(balance[Alice] - frozenTokens >= 100) ← 检查可用余额
  ③  identityRegistry.isVerified(Bob)            ← Bob 是否通过 KYC？
  ④  compliance.canTransfer(Alice, Bob, 100)      ← 合规规则是否通过？
         │
    ①②③④全部通过 → 执行转账
    任意一个失败  → revert
```

---

## 三、KYC 验证：IdentityRegistry.isVerified() 内部逻辑

这是 ERC-3643 最复杂的部分，引入了"声明（Claim）"机制。

### 概念解释

| 概念 | 含义 | 类比 |
|------|------|------|
| ONCHAINID | 每个投资者有一个链上身份合约 | 链上护照 |
| Claim | 身份合约里存储的声明（如"已通过 KYC"） | 护照上的签证 |
| ClaimTopic | 声明的类型（如 Topic 1 = KYC，Topic 2 = 合格投资者） | 签证类型 |
| ClaimIssuer | 有资格颁发声明的机构（如 KYC 服务商） | 使馆 |
| TrustedIssuers | 合约信任的 ClaimIssuer 列表 | 承认的使馆列表 |

### isVerified() 执行步骤

```
isVerified(Bob)
   │
   ├── 1. Bob 是否有 ONCHAINID？→ 没有直接返回 false
   │
   ├── 2. 从 ClaimTopicsRegistry 获取"需要的声明类型"
   │       例如：[Topic 1: KYC认证, Topic 7: 合格投资者]
   │
   └── 3. 对每个 Topic，逐一检查：
           │
           ├── 从 TrustedIssuersRegistry 获取"可信颁发机构"列表
           │       例如：[Onfido, Jumio, SingPass]
           │
           ├── 在 Bob 的 ONCHAINID 中查找对应 Claim
           │
           └── 调用 ClaimIssuer.isClaimValid() 验证签名有效性
                   → 有效：该 Topic 通过
                   → 无效或找不到：返回 false
```

### 代码核心片段

```solidity
function isVerified(address _userAddress) external view returns (bool) {
    // 没有链上身份 → 不通过
    if (address(identity(_userAddress)) == address(0)) return false;

    // 获取该 token 要求的声明类型列表
    uint256[] memory requiredTopics = _tokenTopicsRegistry.getClaimTopics();

    // 对每个声明类型，找一个可信机构颁发的有效声明
    for (uint i = 0; i < requiredTopics.length; i++) {
        IClaimIssuer[] memory issuers =
            _tokenIssuersRegistry.getTrustedIssuersForClaimTopic(requiredTopics[i]);

        // 遍历可信机构，找到一个有效声明就算通过
        bool found = false;
        for (uint j = 0; j < issuers.length; j++) {
            bytes32 claimId = keccak256(abi.encode(issuers[j], requiredTopics[i]));
            (uint topic,,address issuer, bytes memory sig, bytes memory data,) =
                identity(_userAddress).getClaim(claimId);

            if (topic == requiredTopics[i]) {
                // 验证签名是否有效（KYC 机构的私钥签名）
                if (IClaimIssuer(issuer).isClaimValid(identity(_userAddress), topic, sig, data)) {
                    found = true;
                    break;
                }
            }
        }
        if (!found) return false; // 某个 Topic 找不到有效声明
    }
    return true;
}
```

---

## 四、合规检查：ModularCompliance.canTransfer() 内部逻辑

模块化设计：每条规则是一个独立的 Module 合约，可动态增删。

```solidity
function canTransfer(address _from, address _to, uint256 _value)
    external view returns (bool)
{
    // 遍历所有已安装的 Module，全部通过才放行
    for (uint256 i = 0; i < _modules.length; i++) {
        if (!IModule(_modules[i]).moduleCheck(_from, _to, _value, address(this))) {
            return false;
        }
    }
    return true;
}
```

### 每个 Module 实现 moduleCheck()

```solidity
// 以国家限制 Module 为例
function moduleCheck(
    address /*_from*/,
    address _to,
    uint256 /*_value*/,
    address _compliance
) external view returns (bool) {
    uint16 receiverCountry = _getCountry(_compliance, _to);
    return !_isCountryRestricted(msg.sender, receiverCountry);
}
```

### 常见内置 Module

```
CountryRestrictions   → 禁止特定国家地区投资者
MaxBalance            → 单地址持仓上限
SupplyLimit           → token 总供应量上限
ExchangeMonthlyLimits → 月度交易所交互限额
DayMonthLimits        → 日/月转账金额限制
```

---

## 五、完整合约交互调用图

```
用户调用 Token.transfer(to, amount)
           │
           ├──① Token → IdentityRegistry.isVerified(to)
           │                │
           │                ├── IdentityRegistry → Identity(to).getClaim(claimId)
           │                │
           │                └── IdentityRegistry → ClaimIssuer.isClaimValid(sig)
           │                         ▲
           │                         │ 可信机构列表来自
           │                    TrustedIssuersRegistry
           │                    ClaimTopicsRegistry
           │
           └──② Token → ModularCompliance.canTransfer(from, to, amount)
                              │
                              ├── Module1.moduleCheck(from, to, amount)
                              ├── Module2.moduleCheck(from, to, amount)
                              └── ModuleN.moduleCheck(from, to, amount)
```

---

## 六、运营方（Agent）日常操作入口

```
向 IdentityRegistry 注册用户身份
  → registerIdentity(userAddress, onchainID, country)

KYC 过期后更新
  → updateIdentity(userAddress, newOnchainID)

封禁地址
  → Token.setAddressFrozen(userAddress, true)

部分冻结（锁仓）
  → Token.freezePartialTokens(userAddress, amount)

添加新合规规则
  → ModularCompliance.addModule(moduleAddress)

移除合规规则
  → ModularCompliance.removeModule(moduleAddress)

铸造 token
  → Token.mint(userAddress, amount)  // 先检查 isVerified + canTransfer
```

---

## 七、与 Ondo/USDY 合规层对比

```
维度              USDY（Ondo）          ERC-3643（T-REX）
────────────────────────────────────────────────────────
KYC 存储          简单 mapping          链上 ONCHAINID + Claim
身份验证          address whitelist     链上 Claim 签名验证
合规规则          硬编码在合约中        模块化，可热插拔
规则扩展          需要升级合约          addModule() 即可
制裁名单          Chainalysis 接口      需要自建 Module 接入
适合场景          单产品，规则固定      多产品，规则复杂多变
复杂度            低                    高
```

---

## 八、学习要点总结

**ERC-3643 最值得借鉴的两个设计：**

**① 身份与 Token 解耦**
IdentityRegistry 独立部署，多个 Token 可共用同一份 KYC 数据，不用重复管理白名单。

**② 合规模块化**
每条业务规则是一个独立合约，上线后可以 `addModule()` 添加新规则，不需要升级 Token 合约本身。这在监管要求频繁变化的金融场景下非常有价值。

**不需要照抄的部分：**
链上 ONCHAINID + Claim 这套身份体系过于复杂，对于 T-Bills 这类规则固定的产品，直接用 address mapping 白名单反而更安全、更简单。
