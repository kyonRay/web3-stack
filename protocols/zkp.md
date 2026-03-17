# 零知识证明（ZKP）

> **标签：** zkp、zk-snark、zk-stark、隐私、扩容

零知识证明允许证明者向验证者证明「某件事是真的」，而**无需透露任何额外信息**。在区块链中，ZKP 用于**隐私保护**和**扩容（ZK Rollup）**。

---

## 1. 基本概念

```
传统证明：告诉别人你的密码，对方验证是否正确（泄露秘密）
零知识证明：用数学方法让对方相信你知道密码，但不透露密码本身

三个性质：
  完整性：真实的陈述总能被接受
  可靠性：虚假的陈述几乎不能被接受
  零知识性：验证者不获得任何额外信息
```

---

## 2. 主要证明系统

### zk-SNARK

- **Succinct Non-interactive Arguments of Knowledge**
- 证明小（~200 bytes），验证快（< 1ms）
- 需要**可信设置（Trusted Setup）**：公共参数（SRS/CRS）需要一个诚实的初始化仪式，"有毒废料"需被销毁
- 代表：Groth16、PLONK（通用 SRS）

### zk-STARK

- **Scalable Transparent Arguments of Knowledge**
- **无需可信设置**（基于哈希函数），后量子安全
- 证明较大（~50-100 KB），但扩展性更好
- 代表：StarkNet 的 STARK 证明

---

## 3. 隐私应用

### Tornado Cash 模式（已被制裁）

混币器的核心思路（了解原理，不要部署）：

```
1. 存入 1 ETH，获得一个秘密 note（随机数）
2. 合约将 note 的哈希存入 Merkle 树
3. 提款时提交 ZK 证明：「我知道某个在 Merkle 树中的 note」
4. 证明不揭示是哪个 note（隐私），但能让合约验证确实存过款
```

### Zcash / 隐私代币

- zk-SNARK 证明交易金额和发送者合法，而不揭示具体数值
- Shield 地址的交易对外不可见

---

## 4. ZK Rollup 中的 ZKP

见 [chain-dev/layer2/zk-rollup.md](../chain-dev/layer2/zk-rollup.md)：
- Prover 为每批 L2 交易生成有效性证明
- L1 验证合约验证证明（约 500K gas）
- 无需挑战期，快速提款

---

## 5. 开发入门

```bash
# Circom（ZK 电路语言）+ SnarkJS
npm install -g circom snarkjs

# 简单电路示例（证明知道 a+b=c，不揭示 a 和 b）
# circuit.circom
template Addition() {
    signal input a;
    signal input b;
    signal input c;
    c === a + b;
}
component main = Addition();
```

---

## 参考资源

- [ZKP 入门](https://zkproof.org/learn/)
- [Circom 文档](https://docs.circom.io/)
- [RISC Zero（ZK 通用计算）](https://dev.risczero.com/)
- [StarkNet 文档](https://docs.starknet.io/)
