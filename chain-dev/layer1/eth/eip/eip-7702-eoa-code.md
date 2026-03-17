# EIP-7702：EOA 账户代码（Set EOA Account Code）

> **标签：** eip-7702、EOA、账户抽象、Type-4 交易、委托、Pectra
> **所属升级：** Pectra（Prague-Electra，主网激活：2025.5.7）
> **状态：** 已上线

---

## 1. 要解决的问题

### 1.1 EOA 的代码执行能力缺失

以太坊的账户分为两类：
- **EOA（外部账户）**：有私钥，可发起交易，但**代码字段为空**，无法执行自定义逻辑
- **合约账户**：有代码，可执行逻辑，但**无法主动发起交易**（必须被调用）

这个二元对立造成了长期的使用限制：

```
场景 1：用户想用 USDC 支付 Gas（无 ETH）
  EOA 方案：不可能（EOA 必须有 ETH 才能发交易）
  EIP-4337 方案：需要在新地址部署智能钱包合约（原地址废弃或闲置）

场景 2：用户想批量执行多个操作（approve + swap）
  EOA 方案：两笔独立交易，两次等待
  EIP-4337 方案：可以，但需要新地址 + Bundler

场景 3：用户想授权某个会话密钥（游戏场景）
  EOA 方案：完全不可能
  EIP-4337 方案：可以，但成本高（~42k gas 基础开销）

场景 4：用户已有多年历史的 EOA 地址，不想放弃
  EIP-4337 方案：只能在新地址上使用 AA 功能，原地址无法"升级"
```

### 1.2 EIP-4337 的局限

EIP-4337 是纯合约层的账户抽象，设计精良，但有几个结构性缺陷：

1. **无法在原 EOA 地址上使用**：用户的历史 ENS、链上声誉、NFT 等都在旧地址，迁移成本高
2. **额外开销固定**：每笔 UserOp 有约 42,000 gas 的固定基础成本（EntryPoint 验证开销）
3. **Bundler 依赖**：依赖第三方 Bundler 服务，增加了技术复杂性和潜在审查风险
4. **与 EOA 交互限制**：不能直接利用 EOA 的原生交易发起能力

### 1.3 EIP-3074 的问题（被取代的前任）

EIP-3074 引入了 `AUTH` / `AUTHCALL` 操作码，允许 EOA 将签名权限授予合约（invoker）。但它有安全风险：一旦签名，invoker 合约可以以 EOA 身份做任何事情，无法撤销（除非 invoker 合约有撤销逻辑）。EIP-7702 取代并替换了 EIP-3074 的设计方向。

---

## 2. 核心变更

### 2.1 Type-4 交易

EIP-7702 引入了新的交易类型（`0x04`），在 EIP-1559 Type-2 基础上增加了 `authorization_list` 字段：

```
Type-4 交易结构
├── chain_id
├── nonce
├── max_priority_fee_per_gas
├── max_fee_per_gas
├── gas_limit
├── destination         # 目标地址（可以为空，但通常有值）
├── value
├── data
├── access_list
├── authorization_list  # 新增！授权列表
│   └── Authorization[]
│       ├── chain_id    # 0 = 全链有效
│       ├── address     # 要委托的合约地址
│       ├── nonce       # 防重放
│       └── (v, r, s)   # EOA 私钥签名
└── signature_values    # 交易发送者的签名（可以不是被委托的 EOA）
```

### 2.2 委托机制

当 `authorization_list` 中的 EOA 签名被验证后，EVM 将该 EOA 的代码字段设置为：

```
code = 0xEF0100 || address（23 字节）
```

- `0xEF0100` 是特殊前缀（"委托标志"），表示这是一个委托指针
- `address` 是被委托的合约地址（20 字节）

效果：**当有人调用这个 EOA 地址时，EVM 会将调用路由到委托合约的代码，但 `storage` 和 `address` 仍然是 EOA 自己的**。

```
调用流程：
  调用者 → EOA 地址 → EVM 发现 "0xEF0100 || delegateAddress"
                         → 加载 delegateAddress 的字节码
                         → 在 EOA 的 storage context 中执行
                         → 存储变更写入 EOA storage
```

### 2.3 撤销委托

EOA 可以随时通过新的 Type-4 交易覆盖委托：

```
# 设置新委托（替换旧委托）
authorization_list = [{ address: NEW_CONTRACT, nonce: n+1, sig }]

# 清除委托（恢复为普通 EOA）
authorization_list = [{ address: 0x0000...0000, nonce: n+1, sig }]
```

### 2.4 安全设计细节

**Nonce 防重放**：每个 authorization 有独立 nonce，防止相同签名被重复使用

**Chain ID**：可以指定特定链，也可以设置为 0（允许所有链），后者需谨慎使用

**存储隔离**：委托合约的 storage 不会被污染，EOA 的 storage 独立

**批量授权**：一笔 Type-4 交易可以包含多个授权（但实际中通常只有一个）

**交易发送者**：Type-4 交易的**发送者**（支付 Gas 的地址）可以不是被授权的 EOA。这意味着可以由赞助者为 EOA 完成委托设置，实现 Gas 代付初始化。

---

## 3. 生效前后对比

| 维度 | EIP-7702 之前 | EIP-7702 之后 |
|------|--------------|--------------|
| **EOA 代码执行** | 完全不可能 | 可通过委托合约执行 |
| **地址连续性** | EOA 升级为 AA 必须换地址 | 保留原 EOA 地址，功能升级 |
| **Gas 支付** | 只能 ETH | 可通过委托合约实现 ERC-20 支付 |
| **批量操作** | 多笔独立交易 | 通过委托合约一笔完成 |
| **初始化成本** | EOA 部署智能钱包：~200k gas | 委托设置：~25k gas（一次性） |
| **每笔操作开销** | 普通 EOA：21k gas 起 | 委托 EOA：21k gas 起（接近 EOA 原始成本） |
| **与 Bundler 关系** | 不需要 | 不需要（普通 L1 交易） |
| **持久性** | EOA 代码永远为空 | 委托可以设置/撤销/更换 |
| **安全风险** | 私钥全权 | 同上，但合约逻辑可以限制操作 |

### Gas 对比

| 操作 | EIP-4337（新 SmartWallet） | EIP-7702（EOA 委托） |
|------|--------------------------|-------------------|
| 初始化账户 | ~200k gas（合约部署） | ~25k gas（委托设置） |
| 基础批量操作 | ~42k gas 基础 + 操作 gas | ~21k gas 基础 + 操作 gas |
| 更换签名方案 | 升级合约（高成本） | 更换委托地址（低成本） |

---

## 4. 影响范围

### 对用户

- **现有用户**：可以"原地升级"长期使用的 EOA 地址，获得批量操作、Gas 代付、多签等功能
- **新用户体验**：DApp 可以为用户的 EOA 快速设置标准委托合约，实现无感 Gas 代付
- **过渡场景**：持有大量 NFT/DeFi 头寸的老用户无需迁移即可获得 AA 功能

### 对钱包开发者

```solidity
// 一个简单的批量执行委托合约示例
contract BatchDelegate {
    struct Call {
        address to;
        uint256 value;
        bytes data;
    }

    function execute(Call[] calldata calls) external payable {
        // msg.sender 是最终调用者（可能是 EOA 自己，或 EIP-4337 EntryPoint）
        for (uint i = 0; i < calls.length; i++) {
            (bool success, ) = calls[i].to.call{value: calls[i].value}(calls[i].data);
            require(success, "Call failed");
        }
    }
    
    // 可以添加：签名验证、限额控制、白名单等
}
```

### 对 DApp 开发者

```typescript
// 使用 viem 发送 EIP-7702 授权交易
import { createWalletClient, http } from 'viem'
import { mainnet } from 'viem/chains'

const authorization = await walletClient.signAuthorization({
  contractAddress: BATCH_DELEGATE_ADDRESS,
})

// 发送 Type-4 交易：设置委托并立即执行批量操作
const hash = await walletClient.sendTransaction({
  authorizationList: [authorization],
  data: encodeFunctionData({
    abi: batchDelegateAbi,
    functionName: 'execute',
    args: [[
      { to: TOKEN, value: 0n, data: approveCalldata },
      { to: ROUTER, value: 0n, data: swapCalldata },
    ]],
  }),
  to: walletClient.account.address,  // 调用 EOA 自身（转发到委托合约）
})
```

### 对合约开发者

- **委托合约设计**：需要考虑委托合约被多个 EOA 共享（不能依赖合约级别的持久化初始化）
- **重入风险**：委托合约在 EOA 的 storage 中执行，需要防止重入攻击
- **EIP-1271 集成**：委托合约应实现 `isValidSignature`，使 EOA 支持链上合约签名验证

---

## 5. 与其他 EIP 的关系

| 关联 EIP | 关系 | 说明 |
|---------|------|------|
| **EIP-4337** | 互补（双轨 AA） | 4337 适合"全功能智能钱包新账户"；7702 适合"现有 EOA 功能扩展" |
| **EIP-3074** | 替代 | 7702 在功能上覆盖并取代了 EIP-3074 的设计（AUTH/AUTHCALL 不再被纳入） |
| **EIP-7579** | 延伸标准 | 7702 的委托合约可以遵循 ERC-7579 模块化接口，与 4337 生态共享插件 |
| **EIP-2930** | Type-1 参考 | 类似 Type-1（access list 交易），7702 的 Type-4 也在现有交易格式基础上扩展 |
| **EIP-1271** | 配套需求 | 委托 EOA 对外展示为"智能合约账户"，需要 EIP-1271 支持 permit/签名验证 |

---

## 参考资源

- [EIP-7702 原文](https://eips.ethereum.org/EIPS/eip-7702)
- [Vitalik：EIP-7702 设计初衷](https://ethereum-magicians.org/t/eip-7702-set-eoa-account-code-for-one-transaction/19923)
- [viem EIP-7702 文档](https://viem.sh/docs/eip7702)
- [EIP-7702 与 EIP-4337 对比分析](https://www.alchemy.com/blog/eip-7702)
