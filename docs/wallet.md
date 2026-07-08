# Wallet and MetaMask Issues

[Back to index](../README.md)

Problems connecting MetaMask and other wallets to Redbelly. Entries 5 and 6.

---

### 5. MetaMask not detecting Redbelly Network

**Symptom**

Redbelly does not appear in MetaMask's network list, or the "Add network" search finds nothing, so the user cannot connect to your dApp at all.

**Root Cause**

Redbelly is not in MetaMask's built-in popular networks list, so it must be added manually or programmatically. Users following a dApp prompt may also hit this if the dApp only calls `wallet_switchEthereumChain`, which fails with error `4902` when the chain has never been added.

**Solution**

Manual (for users):

1. MetaMask > Networks > Add a custom network, and enter:
   - Network name: `Redbelly Testnet`
   - RPC URL: `https://governors.testnet.redbelly.network`
   - Chain ID: `153`
   - Currency symbol: `RBNT`
   - Explorer: `https://redbelly.testnet.routescan.io`

Programmatic (for dApp developers), handling the 4902 case:

```js
try {
  await window.ethereum.request({
    method: "wallet_switchEthereumChain",
    params: [{ chainId: "0x99" }],
  });
} catch (err) {
  if (err.code === 4902) {
    await window.ethereum.request({
      method: "wallet_addEthereumChain",
      params: [{
        chainId: "0x99",
        chainName: "Redbelly Testnet",
        rpcUrls: ["https://governors.testnet.redbelly.network"],
        nativeCurrency: { name: "RBNT", symbol: "RBNT", decimals: 18 },
        blockExplorerUrls: ["https://redbelly.testnet.routescan.io"],
      }],
    });
  } else {
    throw err;
  }
}
```

Users can also add the chain in one click from https://chainlist.org/chain/153.

**Prevention**

Every Redbelly dApp should ship the `wallet_addEthereumChain` fallback above in its connect flow. Do not assume the user has the network configured.

---

### 6. MetaMask shows wrong balance or stuck account state

**Symptom**

The balance in MetaMask does not match the explorer, sent transactions do not appear, or every new transaction immediately fails even though the account has funds.

**Root Cause**

MetaMask caches account state (nonce, balance, activity) per network. After a testnet reset, an RPC switch, or heavy scripted use of the same key outside MetaMask, that cache goes stale and MetaMask signs with wrong assumptions.

**Solution**

1. First confirm the truth on-chain: look the address up on https://redbelly.testnet.routescan.io. The explorer reflects actual state.
2. In MetaMask: Settings > Advanced > **Clear activity tab data**. This resets cached activity and nonce for the selected account and network. It does not touch your keys or funds.
3. If the balance is still wrong, remove the Redbelly network entry and re-add it with the current RPC URL (entry 5), then reselect the account.

**Prevention**

Do not share one private key between MetaMask and automated scripts on the same network; the nonce cache will constantly desync (see entry 7 in [Gas and Transactions](gas-transactions.md)). Use a dedicated deployer key for scripts.

---

[Back to index](../README.md) | Next category: [Gas and Transactions](gas-transactions.md)
