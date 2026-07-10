# Wallet and MetaMask Issues

[Back to index](../README.md)

Problems connecting MetaMask and other wallets to Redbelly. Entries 5, 6 and 22.

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

A related point of confusion once the network is added: RBNT is the network's **native gas coin**, not an ERC-20 token, so there is no contract address to import and MetaMask's "Import tokens" flow is not needed. If a guide asks you for an RBNT contract address, it is describing wrapped RBNT on a different chain, which is a separate asset.

**Prevention**

Every Redbelly dApp should ship the `wallet_addEthereumChain` fallback above in its connect flow. Do not assume the user has the network configured.

---

### 6. MetaMask shows wrong balance or stuck account state

**Symptom**

The balance in MetaMask does not match the explorer, sent transactions do not appear, or every new transaction immediately fails even though the account has funds. A variant reported by RBNT holders: the balance appears for a split second when switching networks and then vanishes, while the explorer shows the funds are untouched.

**Root Cause**

MetaMask caches account state (nonce, balance, activity) per network. After a testnet reset, an RPC switch, or heavy scripted use of the same key outside MetaMask, that cache goes stale and MetaMask signs with wrong assumptions.

**Solution**

1. First confirm the truth on-chain: look the address up on https://redbelly.testnet.routescan.io. The explorer reflects actual state.
2. In MetaMask: Settings > Advanced > **Clear activity tab data**. This resets cached activity and nonce for the selected account and network. It does not touch your keys or funds.
3. If the balance is still wrong, remove the Redbelly network entry and re-add it with the current RPC URL (entry 5), then reselect the account. Community members have confirmed this fixes the disappearing-balance variant.
4. If MetaMask keeps misbehaving on Redbelly, an ERC-20 compatible alternative such as Rabby works with the same seed or key and the same network details, and is what network moderators suggest for persistent display glitches.

**Prevention**

Do not share one private key between MetaMask and automated scripts on the same network; the nonce cache will constantly desync (see entry 7 in [Gas and Transactions](gas-transactions.md)). Use a dedicated deployer key for scripts.

---

### 22. Token or RBNT logo not displaying in wallets

*Added from community feedback in the Redbelly Discord, July 2026.*

**Symptom**

RBNT or a project token deployed on Redbelly shows a generic placeholder icon in wallets such as MetaMask, OKX Wallet or Zerion, even though the balance is correct. The same asset may show its logo correctly on an exchange but not in a self-custody wallet.

**Root Cause**

Wallets do not read logos from the chain. Each wallet renders logos from its own asset registry, and most registries are populated from a small set of sources: CoinGecko and CoinMarketCap listings, the wallet vendor's own submission process, or community asset repositories such as `trustwallet/assets`. Two situations produce the placeholder icon:

1. The network was added to the wallet manually as a custom network. Most wallets render a generic icon for custom networks and their assets regardless of any listing, because the wallet has no registry entry to consult for that chain.
2. The token itself has no entry in the registry the wallet uses. This is the usual case for newly deployed project tokens.

RBNT itself is listed on CoinGecko and CoinMarketCap, so wallets and exchanges that support Redbelly natively display its logo; wallets where Redbelly was added as a custom network generally will not. The Redbelly chain entries in the [ethereum-lists/chains](https://github.com/ethereum-lists/chains) registry (chains 151 and 153) already carry the network icon, which is what Chainlist and wallets ingesting that registry display.

**Solution**

The immediate fix is EIP-747: a dApp can call `wallet_watchAsset` with the token's address, symbol, decimals and a logo image URL, and supported wallets (MetaMask, Rabby, OKX Wallet and others) register the token and display the logo straight away. Two ways to use it:

1. Use the [Redbelly Wallet Helper](https://rahmandefi.github.io/redbelly-wallet-helper/), a free hosted tool that reads the token's metadata from the chain and makes the `wallet_watchAsset` call for you; it also adds the Redbelly networks with one click. Source: [github.com/Rahmandefi/redbelly-wallet-helper](https://github.com/Rahmandefi/redbelly-wallet-helper).
2. Or put an "Add token to wallet" button on your own project site:

```js
await window.ethereum.request({
  method: "wallet_watchAsset",
  params: {
    type: "ERC20",
    options: {
      address: "0xYourTokenAddress",
      symbol: "TKN",
      decimals: 18,
      image: "https://yoursite.com/token-logo.png",
    },
  },
});
```

Note that `wallet_watchAsset` registers the token on the wallet's currently selected network, so switch (or `wallet_addEthereumChain`, entry 5) to Redbelly first.

To make the logo appear everywhere by default, without users clicking anything, work through the public registries in this order:

1. Verify the contract source on Routescan (entry 13), then submit the token information and logo to Routescan; their documentation directs token update requests to their support contact. Explorer listing is the foundation other registries check.
2. Apply for a CoinGecko or CoinMarketCap listing once the token meets their criteria. Many wallet registries ingest from these automatically.
3. Submit to the specific wallets that matter to your users: OKX Wallet has its own asset submission process, and Trust Wallet accepts pull requests to the `trustwallet/assets` repository for chains it supports.

There is no per-user fix for the custom-network placeholder icon itself; it resolves only when the wallet vendor adds native support for the chain.

**Prevention**

Treat registry submissions as part of the token launch checklist rather than an afterthought: contract verification and explorer token info at deployment, listing applications once eligible. Ship an "Add token" button (or link the Wallet Helper) so users get the branded token from day one, and set expectations that custom-network icons stay generic until wallets support Redbelly natively.

---

[Back to index](../README.md) | Next category: [Gas and Transactions](gas-transactions.md)
