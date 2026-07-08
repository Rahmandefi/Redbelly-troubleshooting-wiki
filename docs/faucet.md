# Faucet and Funding Issues

[Back to index](../README.md)

Getting and keeping testnet RBNT. Entry 20.

---

### 20. Testnet faucet not distributing RBNT

**Symptom**

You request testnet RBNT but nothing arrives, the faucet reports you as rate-limited, or the faucet page errors out.

**Root Cause**

The official faucet (FAUCETME) requires joining with a Discord account and enforces a claim cooldown (one claim per 24 hours). Requests fail when the Discord link is not completed, the cooldown has not elapsed, the wrong address was pasted, or the faucet is temporarily out of funds.

**Solution**

1. Use the official faucet at https://redbelly.faucetme.pro and complete the "join with Discord" step; claims without the Discord link do not process.
2. Double-check the address you pasted and that you are watching the same address on the explorer: search it at https://redbelly.testnet.routescan.io to see whether the transfer actually landed (MetaMask can display stale balances; see entry 6 in [Wallet and MetaMask](wallet.md)).
3. If you claimed recently, wait out the 24-hour cooldown; repeat attempts inside the window are rejected.
4. Try the third-party thirdweb faucet at https://thirdweb.com/redbelly-network-testnet as a fallback (small amounts, also rate-limited).
5. If nothing arrives after the cooldown, ask in the Redbelly Discord developer channel with your address and timestamp; the faucet occasionally needs a refill by the team.

**Prevention**

Claim faucet funds before you need them, and consolidate testing on one or two funded addresses instead of spreading dust across many throwaway wallets. Deployment-heavy days may need more than one day's claim, so plan a day ahead.

---

[Back to index](../README.md)
