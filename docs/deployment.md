# Contract Deployment Issues

[Back to index](../README.md)

Hardhat configuration, explorer verification and EVM compatibility. Entries 12 to 14.

---

### 12. Hardhat cannot connect (HH108) or misconfigured network

*Verified on Redbelly Testnet, July 2026: the config block below deployed a live contract to `0xd583cbaf72849Ad868445C4D025dc49dF84358d6` (tx `0x6b852e17bc77f0cad8a7e5b03e264a08eed9f297c20a72f1b8498c6ee51a40e9`).*

**Symptom**

```
Error HH108: Cannot connect to the network redbellyTestnet
```
or
```
Error HH100: Network redbellyTestnet doesn't exist
```

**Root Cause**

HH100: the `--network` name you passed does not match any key under `networks` in `hardhat.config.js`. HH108: the name matches but the RPC URL is wrong, unreachable, or the environment variable holding it is undefined (so Hardhat sees `url: undefined`).

**Solution**

1. Use a known-good config block:

```js
require("@nomicfoundation/hardhat-toolbox");
require("dotenv").config();

module.exports = {
  solidity: "0.8.24",
  networks: {
    redbellyTestnet: {
      url: "https://governors.testnet.redbelly.network",
      chainId: 153,
      accounts: process.env.PRIVATE_KEY ? [process.env.PRIVATE_KEY] : [],
    },
  },
};
```

2. Confirm the exact network name in the command matches the config key:

```bash
npx hardhat run scripts/deploy.js --network redbellyTestnet
```

3. If HH108 persists, run the `eth_chainId` curl from entry 2 in [Network and RPC](network-rpc.md). If curl succeeds but Hardhat fails, check proxies and the WSL2 IPv6 note in the same entry.

**Prevention**

Commit a working `hardhat.config.js` with the Redbelly network block to your project template, and validate required environment variables at config load with a clear throw if missing.

---

### 13. Contract verification failing on the block explorer

*Verified on Redbelly Testnet, July 2026: the custom chain config below verified the contract at `0xd583cbaf72849Ad868445C4D025dc49dF84358d6` on Routescan; the explorer now serves its full source and ABI.*

**Symptom**

`npx hardhat verify` fails with "network not supported", "unable to verify" or bytecode mismatch errors; or manual verification on the explorer rejects the source.

**Root Cause**

Redbelly's explorer is Routescan, which is not in hardhat-verify's built-in chain list, so it must be added as a custom chain. Bytecode mismatches come from verifying with different compiler settings (version, optimizer runs, evmVersion) or different constructor arguments than the deployed build.

**Solution**

1. Add Routescan's Etherscan-compatible API as a custom chain in `hardhat.config.js`:

```js
etherscan: {
  apiKey: { redbellyTestnet: "verifyContract" },  // Routescan accepts any non-empty string
  customChains: [
    {
      network: "redbellyTestnet",
      chainId: 153,
      urls: {
        apiURL: "https://api.routescan.io/v2/network/testnet/evm/153/etherscan",
        browserURL: "https://redbelly.testnet.routescan.io",
      },
    },
  ],
},
```

2. Verify with the exact constructor arguments used at deployment:

```bash
npx hardhat verify --network redbellyTestnet 0xDEPLOYED_ADDRESS "constructorArg1" "constructorArg2"
```

3. On bytecode mismatch: verify from the same commit and the same `hardhat.config.js` (solc version, optimizer settings, evmVersion) that produced the deployment. Run `npx hardhat clean && npx hardhat compile` first to rule out stale artefacts.

4. Manual alternative: on the contract's page at https://redbelly.testnet.routescan.io, use "Verify Contract" and paste the flattened source, selecting the exact compiler version, optimizer setting and EVM version used at deployment. Useful when the deployment was done through Remix or another tool outside Hardhat.

**Prevention**

Verify immediately after deploying, from the same machine and commit, ideally in the deploy script itself via `hardhat.run("verify:verify", {...})`. Record constructor arguments in your deployment output JSON.

---

### 14. "Invalid opcode" on deployment (EVM version mismatch)

*Tested on Redbelly Testnet, July 2026: the same contract was deployed three times with solc 0.8.24 targeting paris, shanghai and cancun. All three deployed and executed correctly, including the shanghai build carrying 27 `PUSH0` opcodes in its runtime bytecode. Testnet currently accepts post-paris builds.*

**Symptom**

A contract that compiles cleanly and deploys fine on another network reverts on Redbelly with `invalid opcode` during deployment or on first interaction.

**Root Cause**

Solidity 0.8.20 and later target the Shanghai EVM by default and emit the `PUSH0` opcode (Cancun targets add `MCOPY` and transient storage opcodes on top). Chains whose execution layer has not enabled a given fork reject those opcodes, and older reports of this error on Redbelly stem from that gap. As of July 2026, Redbelly Testnet executes shanghai and cancun builds correctly (see the test note above), so a fresh `invalid opcode` there usually has a different cause: stale build artefacts deployed from an old `artifacts/` directory, a constructor revert being misreported by the tooling, or a library compiled with different settings than the importing project.

Two version caveats worth knowing. Hardhat 2 pins the EVM target to paris for solc 0.8.20 to 0.8.24 precisely to avoid `PUSH0` incompatibilities, so a default Hardhat build is already pre-shanghai without you asking. And Mainnet enablement of a fork can lag Testnet, so retest there before assuming parity.

**Solution**

1. Rule out stale artefacts first: `npx hardhat clean && npx hardhat compile`, then redeploy.

2. Isolate the EVM version question: deploy a trivial contract compiled with `evmVersion: "paris"` (pre-PUSH0). If the paris build works while the default build fails, it is a fork mismatch on the network you are targeting; pin the version in `hardhat.config.js`:

```js
solidity: {
  version: "0.8.24",
  settings: {
    optimizer: { enabled: true, runs: 200 },
    evmVersion: "paris",
  },
},
```

Foundry equivalent in `foundry.toml`:

```toml
evm_version = "paris"
```

3. If the paris build fails the same way, the EVM version is not your problem: check that the deployer account is enabled for the target network (entry 21 in [Account Access](account-access.md)) and inspect the transaction on Routescan for the actual revert point.

**Prevention**

Set `evmVersion` explicitly in your project template rather than relying on compiler or framework defaults, retest after every Solidity version bump, and remember that the same settings must be used again at verification time (entry 13). When targeting Mainnet, confirm fork support there rather than extrapolating from Testnet.

---

[Back to index](../README.md) | Next category: [EligibilitySDK and Onboarding SDKs](eligibility-sdk.md)
