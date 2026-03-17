# EVM Token 标准

> **标签：** erc-20、erc-721、erc-1155、erc-2612、erc-4626、token

以太坊通过 EIP/ERC 定义了一系列代币与 NFT 标准，使钱包、交易所、DApp 能以统一接口与任意合规实现交互。

---

## 1. ERC-20（同质化代币）

### 接口

```solidity
interface IERC20 {
    function totalSupply() external view returns (uint256);
    function balanceOf(address account) external view returns (uint256);
    function transfer(address to, uint256 amount) external returns (bool);
    function allowance(address owner, address spender) external view returns (uint256);
    function approve(address spender, uint256 amount) external returns (bool);
    function transferFrom(address from, address to, uint256 amount) external returns (bool);

    event Transfer(address indexed from, address indexed to, uint256 value);
    event Approval(address indexed owner, address indexed spender, uint256 value);
}
```

### OpenZeppelin 实现

```solidity
import "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import "@openzeppelin/contracts/access/Ownable.sol";

contract MyToken is ERC20, Ownable {
    constructor(uint256 initialSupply) ERC20("MyToken", "MTK") Ownable(msg.sender) {
        _mint(msg.sender, initialSupply * 10 ** decimals());
    }

    function mint(address to, uint256 amount) external onlyOwner {
        _mint(to, amount);
    }
}
```

### 注意事项

- `approve` 竞态：先设 0 再设新值，或改用 `increaseAllowance`/`decreaseAllowance`
- 不规范 ERC-20（不返回 bool）：使用 `SafeERC20`
- `type(uint256).max` 授权：许多协议用此表示无限授权

---

## 2. ERC-721（非同质化代币）

### 关键接口

```solidity
interface IERC721 {
    function ownerOf(uint256 tokenId) external view returns (address);
    function balanceOf(address owner) external view returns (uint256);
    function transferFrom(address from, address to, uint256 tokenId) external;
    function safeTransferFrom(address from, address to, uint256 tokenId) external;
    function approve(address to, uint256 tokenId) external;
    function setApprovalForAll(address operator, bool approved) external;
    function getApproved(uint256 tokenId) external view returns (address);
    function isApprovedForAll(address owner, address operator) external view returns (bool);
}
```

### 实现要点

```solidity
import "@openzeppelin/contracts/token/ERC721/ERC721.sol";
import "@openzeppelin/contracts/token/ERC721/extensions/ERC721URIStorage.sol";

contract MyNFT is ERC721URIStorage, Ownable {
    uint256 private _nextTokenId;

    constructor() ERC721("MyNFT", "MNFT") Ownable(msg.sender) {}

    function safeMint(address to, string memory uri) external onlyOwner {
        uint256 tokenId = _nextTokenId++;
        _safeMint(to, tokenId); // 检查接收方是否能处理 NFT
        _setTokenURI(tokenId, uri);
    }
}
```

**safeTransferFrom vs transferFrom**：
- `safeTransferFrom` 会检查接收方合约是否实现 `IERC721Receiver`，防止 NFT 被锁死
- 向合约转 NFT 时务必使用 `safeTransferFrom`

### Gas 优化：ERC-721A

批量 Mint 时，ERC-721A 通过延迟写入 ownership 将 N 次 mint 的成本从 O(N) 降至接近 O(1)：

```bash
forge install chiru-labs/ERC721A
```

---

## 3. ERC-1155（多合一代币）

单合约内同时管理多种同质化与非同质化资产：

```solidity
import "@openzeppelin/contracts/token/ERC1155/ERC1155.sol";

contract GameItems is ERC1155 {
    uint256 public constant GOLD = 0;    // 同质化（大量）
    uint256 public constant SWORD = 1;   // 半同质化（有限量）
    uint256 public constant SHIELD = 2;

    constructor() ERC1155("https://game.example/api/item/{id}.json") {
        _mint(msg.sender, GOLD, 10**18, "");
        _mint(msg.sender, SWORD, 1000, "");
    }

    // 批量转账（一笔交易）
    // safeBatchTransferFrom(from, to, [GOLD, SWORD], [100, 1], "")
}
```

**优势**：批量操作节省 gas；适合游戏道具、票证、多种资产场景。

---

## 4. ERC-2612（Permit，无需链上 Approve）

基于签名的授权，用户无需发送 `approve` 交易，只需签名后 DApp 代为提交：

```solidity
// EIP-2612 permit 函数
function permit(
    address owner,
    address spender,
    uint256 value,
    uint256 deadline,
    uint8 v, bytes32 r, bytes32 s
) external {
    require(block.timestamp <= deadline, "Permit: expired");
    bytes32 structHash = keccak256(abi.encode(PERMIT_TYPEHASH, owner, spender, value, nonces[owner]++, deadline));
    bytes32 hash = _hashTypedDataV4(structHash);
    address signer = ECDSA.recover(hash, v, r, s);
    require(signer == owner, "Permit: invalid signature");
    _approve(owner, spender, value);
}
```

**使用场景**：DeFi 协议的「一键操作」（approve + deposit 合并为单步）。

---

## 5. ERC-4626（代币化金库标准）

统一收益型 vault 的接口（如 Yearn、Aave 的收益代币）：

```solidity
interface IERC4626 is IERC20 {
    function asset() external view returns (address);
    function totalAssets() external view returns (uint256);
    function deposit(uint256 assets, address receiver) external returns (uint256 shares);
    function withdraw(uint256 assets, address receiver, address owner) external returns (uint256 shares);
    function mint(uint256 shares, address receiver) external returns (uint256 assets);
    function redeem(uint256 shares, address receiver, address owner) external returns (uint256 assets);
    // ...previewDeposit, previewWithdraw, convertToShares, convertToAssets
}
```

---

## 6. 其他常用 ERC

| 标准 | 说明 |
|------|------|
| **ERC-165** | 接口检测（`supportsInterface`），NFT/多接口合约必备 |
| **ERC-712** | 结构化数据签名，防钓鱼签名标准（配合 ERC-2612 使用） |
| **ERC-777** | ERC-20 增强版（hooks），因历史漏洞现较少使用 |
| **ERC-1271** | 合约账户签名验证（`isValidSignature`），智能钱包必备 |
| **ERC-1363** | 可支付代币（transferAndCall） |
| **ERC-1967** | 代理合约存储槽标准，避免升级时存储冲突 |
| **ERC-2981** | NFT 版税标准（`royaltyInfo`） |
| **ERC-6551** | Token Bound Accounts（NFT 拥有以太坊账户） |
| **ERC-7579** | 模块化智能账户标准（Modular Smart Accounts） |

### ERC-165 接口检测示例

```solidity
// 检查合约是否实现 ERC-721
bool isERC721 = IERC165(contractAddress).supportsInterface(type(IERC721).interfaceId);

// 实现 ERC-165
function supportsInterface(bytes4 interfaceId) external pure override returns (bool) {
    return interfaceId == type(IERC165).interfaceId
        || interfaceId == type(IERC721).interfaceId;
}
```

### ERC-1271 合约签名验证

```solidity
// 智能钱包实现 ERC-1271
function isValidSignature(bytes32 hash, bytes calldata signature)
    external view returns (bytes4 magicValue)
{
    address recovered = ECDSA.recover(hash, signature);
    if (recovered == owner) {
        return 0x1626ba7e; // IERC1271.isValidSignature.selector
    }
    return 0xffffffff;
}
```

---

## 参考资源

- [OpenZeppelin Contracts — Tokens](https://docs.openzeppelin.com/contracts/tokens)
- [ERC-20 原文](https://eips.ethereum.org/EIPS/eip-20)
- [ERC-721 原文](https://eips.ethereum.org/EIPS/eip-721)
- [ERC-1155 原文](https://eips.ethereum.org/EIPS/eip-1155)
- [ERC-165 原文](https://eips.ethereum.org/EIPS/eip-165)
- [ERC-712 原文](https://eips.ethereum.org/EIPS/eip-712)
- [ERC-1271 原文](https://eips.ethereum.org/EIPS/eip-1271)
- [ERC-721A](https://www.erc721a.org/)
