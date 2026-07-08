# Gas and Transaction Issues

[Back to index](../README.md)

Nonce errors, gas estimation failures, stuck and underfunded transactions. Entries 7 to 11.

---

### 7. "Nonce too low" transaction failures

**Symptom**

```
Error: nonce too low: next nonce 42, tx nonce 40
```

**Root Cause**

The transaction's nonce has already been used. Typical triggers: MetaMask's cached nonce is behind the chain (after scripted transactions from the same key), two scripts sending from the same account concurrently, or resubmitting an already-mined transaction.

**Solution**

1. Get the real next nonce from the chain:

```bash
curl -s -X POST https://governors.testnet.redbelly.network \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"eth_getTransactionCount","params":["0xYOUR_ADDRESS","pending"],"id":1}'
```

2. In MetaMask: Settings > Advanced > Clear activity tab data, so it refetches the nonce.

3. In scripts, fetch the pending nonce explicitly instead of trusting a cached signer:

```js
const nonce = await provider.getTransactionCount(wallet.address, "pending");
const tx = await wallet.sendTransaction({ ...txRequest, nonce });
```

4. If two processes share a key, serialise them or give each its own key.

**Prevention**

One key, one sender. Deployment scripts, keeper bots and your personal MetaMask should each use separate accounts. In bots, track nonces in a single queue rather than firing transactions in parallel.

---

### 8. "Replacement transaction underpriced"

**Symptom**

```
Error: replacement transaction underpriced
```
when re-sending a transaction that is already pending.

**Root Cause**

A transaction with the same nonce is already in the mempool. Nodes only accept a replacement if it bumps the gas price by a sufficient margin (typically at least 10 per cent over the pending one).

**Solution**

1. Check whether the original is still pending on the explorer. If it confirmed, just resend with a fresh nonce (or let your signer pick one).
2. To genuinely replace a stuck transaction, reuse the same nonce with a higher gas price:

```js
const pending = await provider.getTransactionCount(wallet.address, "pending");
const latest = await provider.getTransactionCount(wallet.address, "latest");
if (pending > latest) {
  const feeData = await provider.getFeeData();
  await wallet.sendTransaction({
    to: wallet.address,          // self-send 0 to cancel
    value: 0,
    nonce: latest,               // the stuck nonce
    gasPrice: (feeData.gasPrice * 125n) / 100n,  // +25%
  });
}
```

**Prevention**

Do not blind-retry sends on timeout; check pending state first. If your script retries, always bump the gas price by at least 25 per cent on each attempt.

---

### 9. Gas estimation returning null or "cannot estimate gas"

**Symptom**

```
Error: cannot estimate gas; transaction may fail or may require manual gas limit
(reason="execution reverted", method="estimateGas")
```
or `eth_estimateGas` returns null or empty in your tooling.

**Root Cause**

Nine times out of ten this is not a gas problem: `eth_estimateGas` simulates the transaction, and if the simulation reverts (failed `require`, missing approval, unverified wallet blocked by an eligibility check, wrong constructor arguments), estimation fails. The remaining cases are RPC hiccups or a signer with zero balance.

**Solution**

1. Find the actual revert reason by simulating with `eth_call`:

```js
try {
  await contract.myFunction.staticCall(arg1, arg2, { from: sender });
} catch (err) {
  console.log(err.reason || err.data || err);  // the real error
}
```

2. Common Redbelly-specific revert: contracts gated by the EligibilitySDK revert for wallets that have not completed KYC. Check permission first:

```js
const ok = await eligibilityContract.hasChainPermission(sender);
```

3. If the call genuinely should succeed, retry estimation once (transient RPC issue), and as a last resort set a manual gas limit to surface the real on-chain revert:

```js
await contract.myFunction(arg1, arg2, { gasLimit: 500_000 });
```

**Prevention**

Treat estimation failure as "this transaction would revert" and surface the decoded reason to users. In eligibility-gated dApps, check `hasChainPermission` in the UI before enabling action buttons.

---

### 10. Transaction pending indefinitely

**Symptom**

A transaction sits in "pending" in MetaMask or your script for many minutes. Redbelly's DBFT consensus gives fast, deterministic finality, so anything pending for more than a minute is stuck, not slow.

**Root Cause**

Either (a) the gas price is below what nodes currently accept, (b) an earlier nonce from the same account is stuck, blocking everything behind it, or (c) the transaction never actually reached the network (an RPC error swallowed by the app).

**Solution**

1. Check whether the network ever saw it: search the transaction hash on https://redbelly.testnet.routescan.io. Not found means it never landed; resend it.
2. Check for a nonce gap:

```bash
# compare "latest" vs "pending" counts; a gap means a stuck earlier tx
curl -s -X POST https://governors.testnet.redbelly.network -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"eth_getTransactionCount","params":["0xYOU","latest"],"id":1}'
curl -s -X POST https://governors.testnet.redbelly.network -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"eth_getTransactionCount","params":["0xYOU","pending"],"id":2}'
```

3. Clear the stuck nonce with a self-send replacement at a higher gas price (exact snippet in entry 8), after which the queued transactions behind it will process.
4. In MetaMask, use the built-in "Speed up" on the oldest pending transaction rather than sending new ones on top.

**Prevention**

Use the network's suggested gas price (`provider.getFeeData()`) instead of hardcoding one. In scripts, `await tx.wait()` before sending the next transaction from the same account.

---

### 11. "Insufficient funds for gas * price + value"

**Symptom**

```
Error: insufficient funds for gas * price + value
```
often during `npx hardhat run scripts/deploy.js --network redbellyTestnet`, sometimes even though you believe the wallet is funded.

**Root Cause**

The sending account cannot cover gas cost plus sent value on the network you are actually targeting. The three classic variants: (a) the account genuinely has 0 RBNT, (b) Hardhat derived a different address than the one you funded (wrong `PRIVATE_KEY` in `.env`), or (c) you funded the account on a different network than the one being used.

Note that on Redbelly a brand-new account fails earlier with `Sender not authorised to write transactions` (entry 21 in [Account Access and Permissioning](account-access.md)); this entry applies to accounts already enabled for write access.

**Solution**

1. Print which address your script is actually using:

```js
const [deployer] = await ethers.getSigners();
console.log("Deployer:", deployer.address);
console.log("Balance:", ethers.formatEther(await ethers.provider.getBalance(deployer.address)), "RBNT");
```

2. If the address is not the one you funded, fix `PRIVATE_KEY` in `.env` (no duplicated `0x` prefix, no whitespace) and confirm `dotenv` is loaded at the top of `hardhat.config.js`:

```js
require("dotenv").config();
```

3. If the address is right but the balance is 0, fund it from the faucet (entry 20 in [Faucet and Funding](faucet.md)) and verify on the explorer before retrying.

4. If the balance looks fine, check you are on the intended network: `console.log(await ethers.provider.getNetwork())` should report chain ID 153 for Testnet.

**Prevention**

Add a balance-and-network preflight to every deploy script that aborts with a clear message when the deployer has no RBNT or is on an unexpected chain ID.

---

[Back to index](../README.md) | Next category: [Contract Deployment](deployment.md)
