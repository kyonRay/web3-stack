# Move 合约测试与部署

> **标签：** move测试、sui测试、aptos测试、部署

---

## 1. Move 测试框架

Move 内置测试框架，测试代码与源码写在同一文件（用 `#[test_only]` 标注）。

### 基本测试结构

```move
module my_addr::counter {
    // ... 正式代码 ...

    #[test_only]
    use aptos_framework::account;

    #[test]
    fun test_initialize() {
        let account = account::create_account_for_test(@0x1);
        initialize(&account);
        assert!(get_count(@0x1) == 0, 0); // 0 = error code if assertion fails
    }

    #[test]
    fun test_increment() {
        let account = account::create_account_for_test(@0x1);
        initialize(&account);
        increment(&account);
        assert!(get_count(@0x1) == 1, 1);
    }

    #[test]
    #[expected_failure(abort_code = 1)] // 期望 abort
    fun test_double_initialize() {
        let account = account::create_account_for_test(@0x1);
        initialize(&account);
        initialize(&account); // 应该 abort（资源已存在）
    }
}
```

---

## 2. Sui Move 测试

```move
module my_package::nft_tests {
    use sui::test_scenario::{Self, Scenario};
    use my_package::nft::{Self, NFT};

    #[test]
    fun test_mint_and_transfer() {
        let creator = @0xA;
        let recipient = @0xB;

        let mut scenario = test_scenario::begin(creator);
        {
            // 铸造 NFT
            let nft = nft::mint(
                string::utf8(b"Test NFT"),
                string::utf8(b"https://example.com/img.png"),
                test_scenario::ctx(&mut scenario),
            );
            transfer::public_transfer(nft, recipient);
        };

        // 切换到 recipient，验证收到了 NFT
        test_scenario::next_tx(&mut scenario, recipient);
        {
            let nft = test_scenario::take_from_sender<NFT>(&scenario);
            assert!(nft::name(&nft) == string::utf8(b"Test NFT"), 0);
            test_scenario::return_to_sender(&scenario, nft);
        };

        test_scenario::end(scenario);
    }

    #[test]
    fun test_shared_object() {
        let admin = @0xA;
        let user = @0xB;

        let mut scenario = test_scenario::begin(admin);
        {
            // 创建共享 counter
            counter::create(test_scenario::ctx(&mut scenario));
        };

        test_scenario::next_tx(&mut scenario, user);
        {
            // 修改共享 counter
            let mut c = test_scenario::take_shared<Counter>(&scenario);
            counter::increment(&mut c);
            test_scenario::return_shared(c);
        };

        test_scenario::end(scenario);
    }
}
```

---

## 3. 运行测试

```bash
# Aptos：运行所有测试
aptos move test

# 运行特定测试
aptos move test --filter test_initialize

# 详细输出
aptos move test --verbose

# Sui：运行测试
sui move test

# 运行特定模块的测试
sui move test --filter nft_tests
```

---

## 4. Aptos 部署

```bash
# 编译检查
aptos move compile \
  --named-addresses my_addr=$(cat .aptos/config.yaml | grep account | awk '{print $2}')

# 发布到 devnet
aptos move publish \
  --named-addresses my_addr=DEPLOYER_ADDRESS \
  --profile devnet

# 发布到主网
aptos move publish \
  --named-addresses my_addr=DEPLOYER_ADDRESS \
  --profile mainnet \
  --max-gas 10000

# 调用初始化函数
aptos move run \
  --function-id 'DEPLOYER_ADDRESS::my_module::initialize' \
  --profile mainnet
```

### 升级合约

Aptos 支持**升级兼容性检查**：

```bash
# 检查新版本是否与旧版本兼容（向后兼容约束）
aptos move publish \
  --named-addresses my_addr=DEPLOYER_ADDRESS \
  --upgrade-policy compatible  # 或 immutable（不可升级）
```

升级策略：
- `compatible`：允许添加新函数和模块，不可删改已有公开函数
- `immutable`：发布后不可升级（适合高度去中心化场景）

---

## 5. Sui 发布

```bash
# 编译
sui move build

# 发布到 devnet
sui client publish --gas-budget 100000000

# 发布后记录 Package ID（用于后续调用）
# 输出示例：Package published: 0x1234abcd...

# 调用函数
sui client call \
  --package 0x1234abcd \
  --module counter \
  --function create \
  --gas-budget 10000000

# 升级包（需要在发布时保留 upgrade capability）
sui client upgrade \
  --upgrade-capability UPGRADE_CAP_OBJECT_ID \
  --gas-budget 100000000
```

### Sui 升级机制

```move
// 发布时保存升级凭证
module my_package::main {
    use sui::package::{Self, Publisher, UpgradeCap};

    // 部署时：将 UpgradeCap 发送给部署者
    // 升级时：提供 UpgradeCap 签名

    // 限制升级权（仅允许 additive 升级）
    public entry fun make_compatible_only(cap: UpgradeCap, publisher: &Publisher) {
        package::only_additive_upgrades(cap, publisher);
    }
}
```

---

## 6. 部署最佳实践

- **测试覆盖**：正常路径 + abort 路径 + 边界情况
- **主网前**：在 testnet/devnet 充分测试
- **升级策略**：明确是否需要升级，高价值协议考虑 `immutable`
- **多签控制**：重要操作（如 mint authority）应由多签控制
- **合约审计**：主网大额资金合约须经过专业审计

---

## 参考资源

- [Aptos Move 测试文档](https://aptos.dev/move/move-on-aptos/testing)
- [Sui Move 测试](https://docs.sui.io/guides/developer/first-app/write-package#testing-with-the-sui-move-test-framework)
- [Move Prover（形式化验证）](https://aptos.dev/move/prover/intro)
