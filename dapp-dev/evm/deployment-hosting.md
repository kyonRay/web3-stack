# DApp 部署与托管

> **标签：** vercel、ipfs、arweave、github-pages、前端部署

DApp 前端需要托管在用户可访问的地址。选择传统中心化托管（Vercel/Netlify）还是去中心化托管（IPFS/Arweave）取决于去中心化程度需求。

---

## 1. Vercel（推荐，生产首选）

Vercel 与 Next.js 深度集成，一键部署，自动 CI/CD：

### 部署流程

```bash
# 安装 Vercel CLI
npm install -g vercel

# 登录
vercel login

# 部署
vercel

# 生产部署（绑定域名）
vercel --prod
```

### GitHub 集成（推荐）

1. 在 [vercel.com](https://vercel.com) 导入 GitHub 仓库
2. 每次 push 自动触发部署
3. PR 自动创建预览 URL

### 环境变量配置

```bash
# .env.local（本地开发，不提交 git）
NEXT_PUBLIC_ALCHEMY_KEY=your_key
NEXT_PUBLIC_WC_PROJECT_ID=your_project_id

# Vercel 控制台 → Settings → Environment Variables
# 或命令行
vercel env add NEXT_PUBLIC_ALCHEMY_KEY
```

**安全提醒**：
- `NEXT_PUBLIC_` 前缀的变量会暴露到客户端，不要放私钥
- RPC Key 等敏感信息考虑通过后端代理（API Route）转发

---

## 2. IPFS（去中心化存储）

IPFS 基于**内容寻址**，文件由其 CID（内容哈希）标识：

### 使用 Fleek 部署

```bash
# Fleek 是 IPFS + CDN 的一站式服务
npm install -g @fleekxyz/sdk

# 初始化
fleek sites init

# 部署
fleek sites deploy
```

### 手动 IPFS 部署

```bash
# 安装 IPFS 客户端
npm install -g ipfs-http-client

# 构建
npm run build

# 上传到 IPFS
ipfs add -r ./out  # Next.js 静态导出
# 输出：CID = QmXxx...

# 通过网关访问
# https://gateway.pinata.cloud/ipfs/QmXxx
# https://cloudflare-ipfs.com/ipfs/QmXxx
```

### Pinning 服务（持久化）

```typescript
// Pinata（IPFS Pinning 服务）
import PinataClient from '@pinata/sdk'

const pinata = new PinataClient({ pinataJWTKey: PINATA_JWT })

const result = await pinata.pinFromFS('./out')
console.log('IPFS CID:', result.IpfsHash)
```

### ENS + IPFS（去中心化域名）

```
1. 构建 DApp → 上传 IPFS → 获得 CID
2. 在 ENS 设置 contenthash（指向 IPFS CID）
3. 用户访问 mydapp.eth（需 ENS 浏览器插件或 eth.limo 网关）
```

---

## 3. Arweave（永久存储）

Arweave 承诺**永久存储**（一次付费，永久可访问）：

```bash
# Arweave 网关访问
# https://arweave.net/{TRANSACTION_ID}

# 使用 Bundlr/Turbo 上传（廉价，基于 Arweave）
npm install @irys/sdk

npx @irys/sdk upload-dir ./out \
  --wallet ./arweave-key.json \
  --url https://turbo.ardrive.io

# 上传后通过 ArNS（Arweave Name System）或直接用交易 ID 访问
```

**Arweave vs IPFS**：
- Arweave：永久存储，内容哈希，需付费（用 AR 代币）
- IPFS：内容寻址，需定期 pin 才能持久（Pinata 等服务需付费）

---

## 4. EthStorage（以太坊原生去中心化存储）

EthStorage 是基于以太坊 Blob（EIP-4844）扩展的去中心化存储方案，将文件存储与以太坊安全性绑定：

```
文件上传 → EthStorage 合约（以太坊） → 通过 Blob 存储数据
文件读取 → 通过 EthStorage 节点网络或 RPC 读取
```

**与 IPFS/Arweave 的区别：**
- 数据安全由以太坊 PoS 共识保证
- 不需要 Pinata 等第三方 Pin 服务
- 原生支持 DApp 前端文件去中心化托管（EIP-4804 Web3 URL）

```bash
# 通过 Web3 URL 访问 EthStorage 上的文件
# web3://CONTRACT_ADDRESS/path/to/file
```

---

## 5. 其他托管选项

| 服务 | 说明 | 适合场景 |
|------|------|---------|
| **Netlify** | 类 Vercel，插件丰富 | 非 Next.js 项目 |
| **GitHub Pages** | 免费静态托管 | 开源项目、文档 |
| **Cloudflare Pages** | 全球 CDN，免费层大 | 高流量项目 |
| **4everland** | IPFS + Arweave + 传统混合 | Web3 项目 |
| **EthStorage** | 以太坊原生去中心化 | 完全去中心化 DApp |

---

## 6. 多环境部署策略

```
dev（本地）:     http://localhost:3000
                 RPC: Anvil / 测试网公共 RPC

staging（预发）: vercel 预览 URL
                 RPC: Sepolia Alchemy
                 合约: Sepolia 测试网

production:      mydapp.xyz（Vercel）
                 RPC: Mainnet Alchemy + 备用
                 合约: 主网
```

### 环境变量按环境隔离

```typescript
// lib/config.ts
const config = {
    development: {
        rpcUrl: 'http://localhost:8545',
        contractAddress: '0xDev...',
    },
    production: {
        rpcUrl: `https://eth-mainnet.g.alchemy.com/v2/${process.env.NEXT_PUBLIC_ALCHEMY_KEY}`,
        contractAddress: '0xProd...',
    },
}

export const appConfig = config[process.env.NODE_ENV as keyof typeof config]
```

---

## 参考资源

- [Vercel 文档](https://vercel.com/docs)
- [Fleek（IPFS 部署）](https://fleek.xyz/)
- [Pinata 文档](https://docs.pinata.cloud/)
- [Arweave 文档](https://docs.arweave.org/)
- [IPFS 文档](https://docs.ipfs.tech/)
