# Solidity 进阶主题

> **标签：** create2、delegatecall、代理、账户抽象、eip-4337、链上随机数

本文覆盖智能合约开发中的高级模式：确定性部署（Create2）、代理架构（Delegatecall）、账户抽象（EIP-4337），以及链上随机数。

---

## 1. CREATE2：确定性合约地址

`CREATE2` 允许在部署前预测合约地址，基于 `deployer + salt + bytecode hash` 计算：

```
address = keccak256(0xff ++ deployer ++ salt ++ keccak256(bytecode))[12:]
```

### 1.1 使用场景

- **工厂模式**：在链上预知即将部署的合约地址
- **反事实实例化**：地址可用于接收资金，之后再实际部署
- **跨链确定性**：同一 salt 在多链上部署到相同地址
- **Uniswap Pair**：pair 地址由两个 token 地址唯一确定

```solidity
// 预计算地址
function predictAddress(bytes32 salt, bytes memory bytecode) public view returns (address) {
    bytes32 hash = keccak256(
        abi.encodePacked(bytes1(0xff), address(this), salt, keccak256(bytecode))
    );
    return address(uint160(uint256(hash)));
}

// 部署
function deploy(bytes32 salt, bytes memory bytecode) external returns (address deployed) {
    assembly {
        deployed := create2(0, add(bytecode, 0x20), mload(bytecode), salt)
        if iszero(extcodesize(deployed)) { revert(0, 0) }
    }
}
```

---

## 2. DELEGATECALL 与代理模式

`DELEGATECALL` 在调用方合约的上下文（存储、`msg.sender`、`msg.value`）中执行被调用方的代码，是可升级合约的基础。

### 2.1 透明代理（Transparent Proxy）

```
用户 → Proxy（存储） → delegate → Logic（代码）
  ↑                        ↓
  管理员可升级 Logic 地址
```

```solidity
// 极简代理（EIP-1167 Clone）
contract MinimalProxy {
    address immutable implementation;

    fallback() external payable {
        address impl = implementation;
        assembly {
            calldatacopy(0, 0, calldatasize())
            let result := delegatecall(gas(), impl, 0, calldatasize(), 0, 0)
            returndatacopy(0, 0, returndatasize())
            switch result
            case 0 { revert(0, returndatasize()) }
            default { return(0, returndatasize()) }
        }
    }
}
```

### 2.2 UUPS（EIP-1822）

升级逻辑在**实现合约**中，而非代理合约：

```solidity
import "@openzeppelin/contracts-upgradeable/proxy/utils/UUPSUpgradeable.sol";

contract MyContractV2 is UUPSUpgradeable, OwnableUpgradeable {
    uint256 public value;
    uint256 public newValue; // 只在末尾新增变量

    function _authorizeUpgrade(address newImplementation) internal override onlyOwner {}

    function initializeV2(uint256 _newValue) external reinitializer(2) {
        newValue = _newValue;
    }
}
```

### 2.3 Transparent vs UUPS

| | Transparent | UUPS |
|--|-------------|------|
| 升级权限 | 代理合约 admin | 实现合约函数 |
| 选择器冲突 | 代理屏蔽管理员函数 | 无内置保护 |
| 部署成本 | 略高（代理更大） | 略低 |
| 推荐 | 旧项目 | 新项目（OpenZeppelin 推荐） |

### 2.4 存储冲突解决：EIP-1967

```solidity
// 使用固定 slot 存储实现地址（避免与业务变量冲突）
bytes32 constant IMPL_SLOT = bytes32(uint256(keccak256("eip1967.proxy.implementation")) - 1);
// = 0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc

assembly {
    sstore(IMPL_SLOT, impl) // 写实现地址到固定 slot
}
```

---

## 3. 账户抽象（EIP-4337）

EIP-4337 无需修改 L1 协议即实现账户抽象，核心组件：

```
用户 → UserOperation（类交易） → Bundler（打包者）
                                      ↓
                              EntryPoint（核心合约）
                                  ├── validateUserOp()  ← 账户合约
                                  ├── executeUserOp()   ← 账户合约
                                  └── Paymaster（Gas 代付，可选）
```

### 3.1 核心能力

| 能力 | 说明 |
|------|------|
| **自定义签名验证** | 支持多签、社交恢复、硬件密钥 |
| **Gas 代付** | Paymaster 允许 dApp 代付 Gas 或用 ERC-20 支付 |
| **批量操作** | 一次 UserOp 执行多步操作 |
| **会话密钥** | 临时授权特定操作，降低主私钥风险 |

### 3.2 最小 Smart Account 实现

```solidity
import "@account-abstraction/contracts/core/BaseAccount.sol";

contract SimpleAccount is BaseAccount {
    IEntryPoint private immutable _entryPoint;
    address public owner;

    constructor(IEntryPoint ep, address _owner) {
        _entryPoint = ep;
        owner = _owner;
    }

    function entryPoint() public view override returns (IEntryPoint) {
        return _entryPoint;
    }

    function _validateSignature(
        UserOperation calldata userOp,
        bytes32 userOpHash
    ) internal override returns (uint256 validationData) {
        bytes32 hash = MessageHashUtils.toEthSignedMessageHash(userOpHash);
        if (owner != ECDSA.recover(hash, userOp.signature)) {
            return SIG_VALIDATION_FAILED;
        }
        return 0;
    }

    function execute(address dest, uint256 value, bytes calldata func) external {
        _requireFromEntryPointOrOwner();
        (bool success, bytes memory result) = dest.call{value: value}(func);
        if (!success) assembly { revert(add(result, 32), mload(result)) }
    }
}
```

### 3.3 Paymaster（Gas 代付）

```solidity
contract TokenPaymaster is BasePaymaster {
    IERC20 public token;
    uint256 public tokenPerEth;

    function _validatePaymasterUserOp(
        UserOperation calldata userOp,
        bytes32,
        uint256 maxCost
    ) internal override returns (bytes memory context, uint256 validationData) {
        // 用 ERC-20 token 支付 gas
        uint256 tokenAmount = maxCost * tokenPerEth / 1e18;
        token.transferFrom(userOp.sender, address(this), tokenAmount);
        return (abi.encode(userOp.sender, tokenAmount), 0);
    }
}
```

---

## 4. 链上随机数

区块变量（`blockhash`、`block.timestamp`）可被验证者操纵，真随机需使用以下方案：

### 4.1 Chainlink VRF v2.5

```solidity
import "@chainlink/contracts/src/v0.8/vrf/VRFConsumerBaseV2Plus.sol";

contract RandomNFT is VRFConsumerBaseV2Plus {
    bytes32 keyHash;
    uint256 subId;

    mapping(uint256 => address) public requestToSender;

    function requestRandom() external returns (uint256 requestId) {
        requestId = s_vrfCoordinator.requestRandomWords(
            VRFV2PlusClient.RandomWordsRequest({
                keyHash: keyHash,
                subId: subId,
                requestConfirmations: 3,
                callbackGasLimit: 100_000,
                numWords: 1,
                extraArgs: VRFV2PlusClient._argsToBytes(VRFV2PlusClient.ExtraArgsV1({nativePayment: false}))
            })
        );
        requestToSender[requestId] = msg.sender;
    }

    function fulfillRandomWords(uint256 requestId, uint256[] calldata randomWords) internal override {
        address user = requestToSender[requestId];
        uint256 tokenId = randomWords[0] % 10000; // 用随机数决定 NFT ID
        _mint(user, tokenId);
    }
}
```

### 4.2 Commit-Reveal

适合低价值场景，无需外部预言机：

```solidity
// 阶段一：提交哈希
function commit(bytes32 commitment) external {
    commitments[msg.sender] = commitment;
    commitBlock[msg.sender] = block.number;
}

// 阶段二：揭示（至少等 N 块）
function reveal(uint256 secret) external {
    require(block.number > commitBlock[msg.sender] + 1);
    require(keccak256(abi.encodePacked(secret, msg.sender)) == commitments[msg.sender]);

    // 用 secret + 未来区块哈希 组合随机数
    uint256 rand = uint256(keccak256(abi.encodePacked(
        secret,
        blockhash(commitBlock[msg.sender] + 1)
    )));
    // 使用 rand
}
```

---

## 5. EIP-7702（EOA 临时委托）

EIP-7702（Pectra 硬分叉，2025 年）允许 EOA 在单笔交易中临时委托给合约代码：

```
EOA 发送交易时，附带 authorization_list：
  [{ chain_id, address(实现合约), nonce, signature }]

该交易期间，EOA 临时具备合约代码，可执行批量操作、Gas 代付等
```

```solidity
// 实现合约（被 EOA 临时委托的代码）
contract Delegation {
    // EOA 临时拥有此合约的所有函数
    function batchExecute(Call[] calldata calls) external {
        for (uint i = 0; i < calls.length; i++) {
            (bool success,) = calls[i].target.call{value: calls[i].value}(calls[i].data);
            require(success, "Call failed");
        }
    }
}
```

**EIP-7702 vs EIP-4337 对比：**

| 维度 | EIP-4337 | EIP-7702 |
|------|---------|---------|
| 账户类型 | 新建智能合约账户 | 现有 EOA 临时升级 |
| 改动范围 | 无协议修改（合约级） | 协议级（Pectra 硬分叉） |
| 兼容性 | 需迁移 | 现有 EOA 直接可用 |
| Paymaster | 支持 | 支持 |
| 持久性 | 永久智能账户 | 仅在单次授权有效期内 |

---

## 6. Diamond 模式（EIP-2535）

当合约逻辑超过 24 KB 限制时，Diamond 模式将多个实现合约（Facet）组合：

```
Diamond（代理）
├── Facet A（功能 1: token 操作）
├── Facet B（功能 2: 治理）
└── Facet C（功能 3: 奖励）
```

每个函数选择器路由到对应的 Facet，共享同一个存储空间（通过 EIP-1967 style 固定 slot）。

---

## 参考资源

- [EIP-1967（代理存储 slot）](https://eips.ethereum.org/EIPS/eip-1967)
- [EIP-4337（账户抽象）](https://eips.ethereum.org/EIPS/eip-4337)
- [EIP-2535（Diamond）](https://eips.ethereum.org/EIPS/eip-2535)
- [Chainlink VRF 文档](https://docs.chain.link/vrf)
- [OpenZeppelin Upgrades Plugin](https://docs.openzeppelin.com/upgrades-plugins/)
