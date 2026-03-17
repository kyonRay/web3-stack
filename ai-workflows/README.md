# AI 辅助工作流

本目录收录面向 Web3 开发的 AI 提示词与工作流，覆盖合约审计、代码生成与代码审查等场景，适用于 Cursor、Claude、ChatGPT 等 AI 工具。

## 文档列表

| 文档 | 说明 |
|------|------|
| [smart-contract-audit.md](./smart-contract-audit.md) | 合约审计提示词：安全扫描、重入、访问控制 |
| [solidity-code-generation.md](./solidity-code-generation.md) | 代码生成提示词：ERC-20/721、多签、质押合约 |

## 使用建议

- 将提示词粘贴到 Cursor/ChatGPT/Claude，用 `[在此粘贴代码]` 占位符替换为实际合约代码。
- AI 辅助审计**不能替代**专业人工审计，高价值合约必须进行第三方审计。
- 生成的合约代码需在本地编译、运行测试后方可使用。

## 延伸阅读

- [smart-contract-dev/evm/solidity/security-patterns.md](../smart-contract-dev/evm/solidity/security-patterns.md) — 安全模式
- [protocols/security-and-auditing.md](../protocols/security-and-auditing.md) — 审计流程
