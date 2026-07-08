# Network and RPC Issues

[Back to index](../README.md)

Connection problems between your tooling and the Redbelly RPC endpoints. Entries 1 to 4.

---

### 1. RPC endpoint returns 429 (Too Many Requests)

**Symptom**

```
ProviderError: 429 Too Many Requests
```

Scripts that loop over many `eth_call` requests (indexers, test suites, dashboards) fail intermittently. Single requests work fine.

**Root Cause**

The public Redbelly RPC endpoints are shared infrastructure and rate-limit aggressive clients. Batch-heavy tools (Hardhat tests, multicall loops, polling dashboards) exceed the per-IP request budget.

**Solution**

1. Add retry with exponential backoff. With ethers v6:

```js
async function withRetry(fn, retries = 5) {
  for (let i = 0; i < retries; i++) {
    try {
      return await fn();
    } catch (err) {
      if (!String(err).includes("429") || i === retries - 1) throw err;
      await new Promise((r) => setTimeout(r, 1000 * 2 ** i));
    }
  }
}

const balance = await withRetry(() => provider.getBalance(address));
```

2. Throttle concurrent requests. Replace `Promise.all` over hundreds of calls with a small concurrency pool (for example `p-limit` set to 3 to 5 concurrent requests).

3. Cache anything static. Contract ABIs, decimals and historical blocks never change; fetch once and store.

**Prevention**

Design for a rate-limited RPC from day one: batch reads with `Multicall3` where deployed, poll no faster than the block time, and keep a client-side cache. For production dashboards, run your own node or request a dedicated endpoint from the Redbelly team.

---

### 2. RPC connection timeouts and "could not detect network"

**Symptom**

```
Error: could not detect network (event="noNetwork", code=NETWORK_ERROR)
```
or
```
FetchError: request to https://governors.testnet.redbelly.network failed, reason: connect ETIMEDOUT
```

**Root Cause**

One of three things: (a) the RPC URL is typed incorrectly or missing `https://`, (b) a local firewall, VPN or corporate proxy blocks the request, or (c) a transient outage of the shared endpoint.

**Solution**

1. Verify the endpoint responds before blaming your code:

```bash
curl -s -X POST https://governors.testnet.redbelly.network \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"eth_chainId","params":[],"id":1}'
```

Expected response: `{"jsonrpc":"2.0","id":1,"result":"0x99"}` (0x99 = 153, the Testnet chain ID).

2. If curl works but your app does not, the problem is your environment. Check for `HTTP_PROXY`/`HTTPS_PROXY` environment variables, VPN split tunnelling, and that your `.env` value has no trailing whitespace or quotes.

3. If curl also fails, test from another network (mobile hotspot). If it fails everywhere, check the Redbelly Discord announcements channel for maintenance notices before opening a support thread.

4. On WSL2 specifically, Node.js can pick a broken IPv6 route. Force IPv4-first resolution:

```bash
export NODE_OPTIONS="--no-network-family-autoselection"
```

**Prevention**

Add the `eth_chainId` curl check to your project README as a first diagnostic. In long-running services, wrap the provider in retry logic and alert on repeated network errors instead of crashing.

---

### 3. Wrong or deprecated RPC endpoint

*Verified on Redbelly Testnet, July 2026.*

**Symptom**

```
Error: getaddrinfo ENOTFOUND rpc-testnet.redbelly.network
```
or requests to a DevNet URL hang or return errors, even though the URL came from an older guide.

**Root Cause**

Redbelly's endpoints have changed over time. Older community guides still reference `rpc-testnet.redbelly.network` or DevNet URLs. DevNet is officially deprecated, the `rpc-testnet` hostname no longer resolves, and the current canonical endpoints (per the developer portal at vine.redbelly.network) are the `governors.*` URLs.

**Solution**

1. Replace any old endpoint with the current one:

- Testnet: `https://governors.testnet.redbelly.network`
- Mainnet: `https://governors.mainnet.redbelly.network`

2. Confirm with the chain ID check:

```bash
curl -s -X POST https://governors.testnet.redbelly.network \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"eth_chainId","params":[],"id":1}'
# expect "result":"0x99"
```

3. Update `hardhat.config.js`/`foundry.toml`, `.env` files and any wagmi/viem chain definitions in the same pass so no stale URL survives.

**Prevention**

Define the RPC URL once (an `.env` variable or a shared `chains.ts`) and import it everywhere. When following any tutorial older than a few months, cross-check endpoints against https://vine.redbelly.network/environments/ before running anything.

---

### 4. Chain ID mismatch ("network changed" or "Unrecognized chain")

**Symptom**

```
NetworkError: underlying network changed (event="changed", network={"chainId":153,...})
```
or MetaMask rejects a transaction with `Unrecognized chain ID`, or Hardhat aborts with a chain ID assertion error.

**Root Cause**

The chain ID configured in your tooling does not match what the RPC actually reports. Common causes: using Mainnet chain ID `151` with the Testnet RPC (or vice versa), a leftover DevNet chain ID, or MetaMask connected to a different network than your dApp expects.

**Solution**

1. Ask the RPC what it really is (see the curl in entry 3). `0x99` = 153 = Testnet, `0x97` = 151 = Mainnet.

2. Make your Hardhat config match:

```js
// hardhat.config.js
networks: {
  redbellyTestnet: {
    url: "https://governors.testnet.redbelly.network",
    chainId: 153,
    accounts: [process.env.PRIVATE_KEY],
  },
},
```

3. In the frontend, prompt a network switch instead of failing silently:

```js
await window.ethereum.request({
  method: "wallet_switchEthereumChain",
  params: [{ chainId: "0x99" }],
});
```

**Prevention**

Never hardcode chain IDs in more than one place. Keep a single chain definition object and reference it from Hardhat, wagmi/viem config and deployment scripts.

---

[Back to index](../README.md) | Next category: [Wallet and MetaMask](wallet.md)
