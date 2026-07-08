# Redbelly Network Developer Troubleshooting Wiki

**Practical fixes for the 21 most common errors when building on Redbelly Testnet and Mainnet.**

Redbelly's official documentation covers architecture and SDK reference material well, but when a deployment fails late at night you need the exact error message, the reason it happened, and the command that fixes it. This wiki documents the recurring roadblocks developers raise in the Redbelly Developer Discord and Telegram channels, each with a concrete, tested fix.

Every entry follows the same structure: **Symptom** (the exact error or behaviour), **Root Cause** (why it happens), **Solution** (step-by-step fix with exact commands), and **Prevention** (how to avoid it next time).

> **New to Redbelly?** Unlike most EVM chains, write access is permissioned per account: a brand-new wallet cannot send any transaction until it is enabled through the official access dApp, no matter how much RBNT it holds. If your first deployment fails with `Sender not authorised to write transactions`, start with [entry 21](docs/account-access.md#21-sender-not-authorised-to-write-transactions).

## Network Quick Reference

| | Testnet | Mainnet |
|---|---|---|
| Chain ID | `153` (hex `0x99`) | `151` (hex `0x97`) |
| RPC URL | `https://governors.testnet.redbelly.network` | `https://governors.mainnet.redbelly.network` |
| Explorer | https://redbelly.testnet.routescan.io | https://redbelly.routescan.io |
| Gas token | RBNT | RBNT |
| Faucet | https://redbelly.faucetme.pro (Discord login) | n/a |

Redbelly DevNet is deprecated. If a guide references DevNet endpoints, use Testnet instead. Note that some older guides list `rpc-testnet.redbelly.network` as the testnet RPC; that hostname no longer resolves. See [entry 3](docs/network-rpc.md#3-wrong-or-deprecated-rpc-endpoint).

## Quick-Reference Index

Search this table by the error text you are seeing.

| Error message / symptom | Entry |
|---|---|
| `Sender not authorised to write transactions` | [21. Account not enabled for write access](docs/account-access.md#21-sender-not-authorised-to-write-transactions) |
| `429 Too Many Requests` | [1. RPC rate limiting](docs/network-rpc.md#1-rpc-endpoint-returns-429-too-many-requests) |
| `ETIMEDOUT`, `ECONNREFUSED`, `could not detect network` | [2. RPC connection timeouts](docs/network-rpc.md#2-rpc-connection-timeouts-and-could-not-detect-network) |
| `ENOTFOUND rpc-testnet.redbelly.network`, DevNet errors | [3. Wrong or deprecated RPC endpoint](docs/network-rpc.md#3-wrong-or-deprecated-rpc-endpoint) |
| `network changed`, `Unrecognized chain ID`, `chainId mismatch` | [4. Chain ID mismatch](docs/network-rpc.md#4-chain-id-mismatch-network-changed-or-unrecognized-chain) |
| MetaMask cannot find Redbelly Network | [5. Adding Redbelly to MetaMask](docs/wallet.md#5-metamask-not-detecting-redbelly-network) |
| MetaMask balance wrong, transactions missing, account stuck | [6. Stale MetaMask state](docs/wallet.md#6-metamask-shows-wrong-balance-or-stuck-account-state) |
| `nonce too low` | [7. Nonce too low](docs/gas-transactions.md#7-nonce-too-low-transaction-failures) |
| `replacement transaction underpriced` | [8. Replacement underpriced](docs/gas-transactions.md#8-replacement-transaction-underpriced) |
| `cannot estimate gas`, `eth_estimateGas` returns null or reverts | [9. Gas estimation failures](docs/gas-transactions.md#9-gas-estimation-returning-null-or-cannot-estimate-gas) |
| Transaction pending indefinitely | [10. Stuck transactions](docs/gas-transactions.md#10-transaction-pending-indefinitely) |
| `insufficient funds for gas * price + value` | [11. Insufficient funds](docs/gas-transactions.md#11-insufficient-funds-for-gas--price--value) |
| `HH108`, `HH100`, Hardhat cannot connect | [12. Hardhat network config](docs/deployment.md#12-hardhat-cannot-connect-hh108-or-misconfigured-network) |
| Contract verification failing on explorer | [13. Routescan verification](docs/deployment.md#13-contract-verification-failing-on-the-block-explorer) |
| `invalid opcode` on deploy of a contract that compiles fine | [14. EVM version / PUSH0](docs/deployment.md#14-invalid-opcode-on-deployment-evm-version-mismatch) |
| `npm ERR! 401 Unauthorized @redbellynetwork/...` | [15. GitHub Packages auth](docs/eligibility-sdk.md#15-npm-install-fails-with-401-for-redbellynetwork-packages) |
| EligibilitySDK widget renders blank | [16. Widget not rendering](docs/eligibility-sdk.md#16-eligibilitysdk-widget-not-rendering) |
| CORS / `X-Frame-Options` / iframe blocked | [17. Cross-origin issues](docs/eligibility-sdk.md#17-cross-origin-and-iframe-issues-with-the-sdk) |
| KYC wallet callback never arrives in local dev | [18. Wallet callbacks / ngrok](docs/eligibility-sdk.md#18-onboarding-wallet-callback-never-arrives-in-local-development) |
| `hasChainPermission` returns false for a verified user | [19. Eligibility check false](docs/eligibility-sdk.md#19-haschainpermission-returns-false-for-a-verified-user) |
| Faucet not sending RBNT | [20. Faucet problems](docs/faucet.md#20-testnet-faucet-not-distributing-rbnt) |

## Categories

- [Account Access and Permissioning](docs/account-access.md) (entry 21, but read it first)
- [Network and RPC](docs/network-rpc.md) (entries 1 to 4)
- [Wallet and MetaMask](docs/wallet.md) (entries 5 and 6)
- [Gas and Transactions](docs/gas-transactions.md) (entries 7 to 11)
- [Contract Deployment](docs/deployment.md) (entries 12 to 14)
- [EligibilitySDK and Onboarding SDKs](docs/eligibility-sdk.md) (entries 15 to 19)
- [Faucet and Funding](docs/faucet.md) (entry 20)

## Verification

Network endpoints, chain IDs and explorer details in this guide were checked against the live network in July 2026 (`eth_chainId` on the testnet RPC returns `0x99`, and the retired `rpc-testnet` hostname no longer resolves). Entries marked "Verified on Redbelly Testnet" have been reproduced and fixed end to end. Highlights:

- Entry 21 was reproduced live and then resolved: a freshly generated wallet's first deployment returned `Sender not authorised to write transactions`; after the credential and enablement flow, the same key deployed a contract in one attempt.
- Entries 12 and 13 are backed by a live deployment: [`0xd583cbaf72849Ad868445C4D025dc49dF84358d6`](https://redbelly.testnet.routescan.io/address/0xd583cbaf72849Ad868445C4D025dc49dF84358d6), deployed with the entry 12 config and source-verified on Routescan with the entry 13 config.
- Entry 14's guidance was tested by deploying the same contract three times with solc 0.8.24 targeting paris, shanghai and cancun: all three run correctly on Testnet, including the shanghai build carrying `PUSH0` opcodes.

How coverage was validated against real community support questions is documented in the [methodology appendix](docs/methodology.md).

## Contributing

Spotted an error pattern that is not covered, or a fix that no longer works? Open an issue or a pull request, or raise it in the Redbelly Discord developer channel. New entries should follow the same structure: Symptom, Root Cause, Solution, Prevention, with exact commands rather than vague advice.

## Sources

- Redbelly Developer Portal: https://vine.redbelly.network (environments, faucet, support)
- Redbelly Documentation: https://docs.redbelly.network (Receptor, EligibilitySDK, Onboarding SDKs)
- Redbelly Testnet Explorer: https://redbelly.testnet.routescan.io
- Chainlist entry for Redbelly Testnet: https://chainlist.org/chain/153
- Recurring developer questions from the Redbelly Discord and Telegram channels
