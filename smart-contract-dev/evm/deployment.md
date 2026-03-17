# 合约部署与验证

> **标签：** 部署、forge script、升级合约、多链、etherscan 验证

本文覆盖合约部署的完整流程：脚本化部署、测试网/主网部署、源码验证、可升级合约部署，以及多链部署策略。

---

## 1. Forge Script 部署

推荐使用 `forge script` 替代部署脚本（比 `cast send` 更结构化）：

```solidity
// script/Deploy.s.sol
pragma solidity ^0.8.20;
import "forge-std/Script.sol";
import "../src/MyToken.sol";
import "../src/Vault.sol";

contract DeployScript is Script {
    function run() external {
        uint256 deployerPrivateKey = vm.envUint("PRIVATE_KEY");
        address deployer = vm.addr(deployerPrivateKey);

        vm.startBroadcast(deployerPrivateKey);

        // 部署代币
        MyToken token = new MyToken(1_000_000 * 1e18);
        console.log("Token deployed to:", address(token));

        // 部署 Vault，传入代币地址
        Vault vault = new Vault(address(token));
        console.log("Vault deployed to:", address(vault));

        // 转移所有权
        token.transferOwnership(address(vault));

        vm.stopBroadcast();
    }
}
```

```bash
# 模拟运行（不广播）
forge script script/Deploy.s.sol --rpc-url $SEPOLIA_RPC -vvvv

# 广播到链
forge script script/Deploy.s.sol \
  --rpc-url $SEPOLIA_RPC \
  --private-key $PRIVATE_KEY \
  --broadcast

# 广播 + 自动验证
forge script script/Deploy.s.sol \
  --rpc-url $SEPOLIA_RPC \
  --private-key $PRIVATE_KEY \
  --broadcast \
  --verify \
  --etherscan-api-key $ETHERSCAN_KEY
```

---

## 2. 手动验证源码

```bash
# 标准验证
forge verify-contract \
  --chain sepolia \
  --etherscan-api-key $ETHERSCAN_KEY \
  $CONTRACT_ADDRESS \
  src/MyToken.sol:MyToken

# 带构造参数
forge verify-contract \
  --chain mainnet \
  --etherscan-api-key $ETHERSCAN_KEY \
  --constructor-args $(cast abi-encode "constructor(uint256)" 1000000) \
  $CONTRACT_ADDRESS \
  src/MyToken.sol:MyToken
```

---

## 3. 可升级合约部署

### UUPS 代理部署

```solidity
// script/DeployUpgradeable.s.sol
import "@openzeppelin/contracts/proxy/ERC1967/ERC1967Proxy.sol";

contract DeployUpgradeable is Script {
    function run() external {
        vm.startBroadcast(vm.envUint("PRIVATE_KEY"));

        // 1. 部署实现合约
        MyContractV1 impl = new MyContractV1();

        // 2. 编码初始化调用
        bytes memory initData = abi.encodeWithSelector(
            MyContractV1.initialize.selector,
            msg.sender  // initialOwner
        );

        // 3. 部署代理
        ERC1967Proxy proxy = new ERC1967Proxy(address(impl), initData);
        console.log("Proxy:", address(proxy));
        console.log("Implementation:", address(impl));

        vm.stopBroadcast();
    }
}
```

### 升级到 V2

```solidity
contract UpgradeScript is Script {
    address constant PROXY = 0x...; // 代理地址

    function run() external {
        vm.startBroadcast(vm.envUint("PRIVATE_KEY"));

        // 1. 部署新实现
        MyContractV2 implV2 = new MyContractV2();

        // 2. 通过代理调用升级函数
        MyContractV2(PROXY).upgradeToAndCall(
            address(implV2),
            abi.encodeWithSelector(MyContractV2.initializeV2.selector, newParam)
        );

        vm.stopBroadcast();
    }
}
```

---

## 4. CREATE2 确定性部署

跨链部署到相同地址：

```solidity
// 使用 CREATE2Factory（如 Nick's Factory: 0x4e59b44847b379578588920cA78FbF26c0B4956C）
import "forge-std/Script.sol";

contract Create2DeployScript is Script {
    address constant FACTORY = 0x4e59b44847b379578588920cA78FbF26c0B4956C;

    function run() external {
        bytes32 salt = bytes32(uint256(0x1234)); // 固定 salt
        bytes memory bytecode = type(MyContract).creationCode;

        // 预计算地址
        bytes32 hash = keccak256(abi.encodePacked(bytes1(0xff), FACTORY, salt, keccak256(bytecode)));
        address predicted = address(uint160(uint256(hash)));
        console.log("Will deploy to:", predicted);

        vm.startBroadcast(vm.envUint("PRIVATE_KEY"));
        // 发送带 salt 的交易到 CREATE2Factory
        (bool success,) = FACTORY.call(abi.encodePacked(salt, bytecode));
        require(success, "Deploy failed");
        vm.stopBroadcast();
    }
}
```

---

## 5. 多链部署

```bash
# foundry.toml 配置多链 RPC
[rpc_endpoints]
mainnet   = "${MAINNET_RPC}"
arbitrum  = "${ARBITRUM_RPC}"
optimism  = "${OPTIMISM_RPC}"
base      = "${BASE_RPC}"
polygon   = "${POLYGON_RPC}"

# 逐链部署
for CHAIN in mainnet arbitrum optimism base; do
  forge script script/Deploy.s.sol \
    --rpc-url $CHAIN \
    --private-key $PRIVATE_KEY \
    --broadcast \
    --verify \
    --etherscan-api-key $ETHERSCAN_KEY
done
```

---

## 6. 部署安全清单

- [ ] 使用硬件钱包或专用部署密钥（不用主要密钥）
- [ ] `.env` 不提交到 git（`.gitignore` 包含 `.env`）
- [ ] 先在测试网完整测试
- [ ] 合约地址记录到文档/配置文件
- [ ] 验证源码（Etherscan）
- [ ] 可升级合约：导出存储布局快照
- [ ] 主网部署后立即设置监控（Tenderly Alert）

---

## 参考资源

- [Forge Script 文档](https://book.getfoundry.sh/forge/deploying)
- [OpenZeppelin Upgrades Plugin](https://docs.openzeppelin.com/upgrades-plugins/)
- [CREATE2 工厂（Nick's Factory）](https://github.com/Arachnid/deterministic-deployment-proxy)
