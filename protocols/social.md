# Web3 社交协议

> **标签：** social、lens、farcaster、ens、cyberconnect、去中心化社交

Web3 社交协议将社交图谱（关注关系、内容、身份）存储在链上或去中心化网络，用户真正拥有自己的数字身份与社交数据。

---

## 1. 去中心化社交的核心优势

- **数据所有权**：用户的关注关系、发帖内容不由平台控制
- **抗审查**：内容存储在链上或去中心化存储，不可被单一实体删除
- **可组合性**：任何应用都可以读取同一份社交图谱，无需重新积累粉丝
- **身份可携带**：换应用不丢失粉丝与内容

---

## 2. ENS（以太坊域名服务）

ENS（Ethereum Name Service）是 Web3 最广泛使用的去中心化命名系统：

**核心功能：**
- 将 `0x1234...abcd` 映射为 `alice.eth`
- 支持反向解析（地址 → 域名）
- 可存储头像、社交链接、内容哈希等元数据

**开发集成：**

```typescript
import { createPublicClient, http } from 'viem'
import { mainnet } from 'viem/chains'
import { normalize } from 'viem/ens'

const client = createPublicClient({ chain: mainnet, transport: http() })

// 正向解析：name → address
const address = await client.getEnsAddress({
  name: normalize('alice.eth'),
})

// 反向解析：address → name
const name = await client.getEnsName({
  address: '0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045',
})

// 读取头像
const avatar = await client.getEnsAvatar({
  name: normalize('vitalik.eth'),
})
```

**ENS 子域名（SubNaming）：**
协议可以为用户发放 ENS 子域名，如 `username.myprotocol.eth`。

---

## 3. Lens Protocol

Lens Protocol 是构建在 Polygon PoS（v1）/ ZKsync（v2）上的去中心化社交图谱协议：

**核心概念：**
- **Profile NFT**：用户身份是 NFT，可转让
- **Publications**：帖子、评论、Mirror（转发）存储在链上
- **Follow NFT**：关注行为铸造 Follow NFT，支持变现
- **Collect**：任何人可收藏（collect）帖子，类似付费转发

**开发（Lens SDK）：**

```typescript
import { PublicClient, production } from '@lens-protocol/client'

const client = PublicClient.create({
  environment: production,
  storage: window.localStorage,
})

// 获取用户 profile
const profile = await client.profile.fetch({
  forHandle: 'lens/stani',
})

// 获取 Feed
const feed = await client.feed.fetch({
  where: { for: profile.id },
})
```

---

## 4. Farcaster

Farcaster 是以以太坊为注册层、去中心化 Hubs 为传播层的社交协议：

**架构：**
- **链上注册**：用户 ID（FID）注册在以太坊合约
- **Hubs 网络**：去中心化节点同步消息，类似 P2P
- **Frames**：在 Feed 中嵌入可交互的迷你应用（类似小程序）

**主要客户端：**Warpcast

**开发（Frames）：**

```typescript
// Farcaster Frame（Next.js API Route）
export async function GET(req: Request) {
  return new Response(
    `<!DOCTYPE html>
    <html>
      <head>
        <meta property="fc:frame" content="vNext" />
        <meta property="fc:frame:image" content="https://example.com/image.png" />
        <meta property="fc:frame:button:1" content="点击交互" />
        <meta property="fc:frame:post_url" content="https://example.com/api/frame" />
      </head>
    </html>`,
    { headers: { 'Content-Type': 'text/html' } }
  )
}
```

---

## 5. CyberConnect（Link3）

CyberConnect 是多链社交图谱协议，产品 Link3 提供 Web3 社交主页：

**核心特性：**
- 统一多链身份（EVM + Solana）
- CyberProfile：SBT（灵魂绑定代币）形式的身份
- CyberGraph：链上社交关系
- 跨链社交内容

---

## 6. 去中心化社交协议对比

| 协议 | 链 | 内容存储 | 代表客户端 | 状态 |
|------|-----|---------|-----------|------|
| **ENS** | 以太坊 | 链上 + IPFS | 多个集成 | 成熟 |
| **Lens v2** | zkSync | Arweave + 链上 | Hey.xyz | 活跃 |
| **Farcaster** | 以太坊注册 + Hubs | Hub 网络 | Warpcast | 高速增长 |
| **CyberConnect** | 多链 | Arweave | Link3 | 活跃 |
| **XMTP** | 以太坊 | 去中心化消息网络 | Converse 等 | 活跃 |

---

## 7. 核心挑战

- **内容审核**：去中心化内容无法统一审核，各客户端自行过滤
- **垃圾信息**：链上注册成本低可能导致 spam
- **用户体验**：链上操作（发帖、关注）有 Gas 成本，新用户门槛高
- **规模化**：链上存储全量内容成本极高，通常只存哈希/索引

---

## 参考资源

- [ENS 开发文档](https://docs.ens.domains/)
- [Lens Protocol 文档](https://docs.lens.xyz/)
- [Farcaster 文档](https://docs.farcaster.xyz/)
- [CyberConnect 文档](https://docs.cyberconnect.me/)
- [XMTP 文档](https://xmtp.org/docs/)
