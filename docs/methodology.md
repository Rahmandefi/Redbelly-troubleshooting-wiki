# Methodology: How Coverage Was Validated

[Back to index](../README.md)

This appendix documents how the wiki's entries were checked against the questions developers and users actually ask.

## Method

Two sources were used:

1. **Live reproduction.** Wherever practical, errors were reproduced against Redbelly Testnet in July 2026 and the documented fix was applied end to end. Entries carrying a "Verified on Redbelly Testnet" note are backed by real transactions, with contract addresses and transaction hashes included in the entry.
2. **Community support channels.** Support threads in the Redbelly Discord from late May to early July 2026 were reviewed to confirm the wiki reflects what people actually get stuck on, rather than hypothetical problems. Notably, the server has no dedicated developer support channel; technical questions surface in general discussion, which makes recurring problems easy to miss and is part of the motivation for this wiki.

## Recurring themes observed

The support threads reviewed clustered around the following themes:

- **Network configuration requests.** The RPC URL, chain ID and explorer details are asked for repeatedly, often answered by moderators pasting the same block each time. Addressed by the quick-reference table in the README and entries 4 and 5.
- **Native coin confusion.** Several users ask for an RBNT "contract address" to import into MetaMask; RBNT is the native gas coin and has none. Addressed in entry 5.
- **MetaMask display and state glitches.** Balances showing as unknown, or appearing for a moment and vanishing while the explorer shows the funds intact. Moderators suggest re-adding the network or switching to another wallet. Addressed in entry 6, which incorporates the community-confirmed fixes.
- **Gas estimation failures that are really contract reverts.** Reward claim attempts failing with "cannot estimate gas; transaction may fail or may require manual gas limit" where the underlying cause was an exhausted contract balance. Addressed in entry 9.
- **KYC and network access requirements.** Whether identity verification is needed to hold or move RBNT, and what access actually gates. Addressed in entry 21, together with the full enablement flow.
- **Token logos not displaying in wallets.** Raised directly during the community validation round: tokens on Redbelly (and RBNT itself on manually added networks) show placeholder icons. Addressed in entry 22, added from that feedback, together with a companion tool that performs the EIP-747 registration.

## Live verification summary

- Entry 21 was reproduced and resolved end to end: a fresh wallet's first deployment returned "Sender not authorised to write transactions"; after the credential and enablement flow, the same key deployed successfully.
- Entries 12 and 13 are backed by a deployed, source-verified contract on Routescan.
- Entry 14 was tested with three builds of the same contract (paris, shanghai and cancun targets); all three deployed and executed correctly on Testnet.
- Network endpoints and chain IDs in every entry were checked against the live network, which is how the retired RPC hostname still circulating in older material was caught.

## Out of scope and future entries

Some recurring questions fall outside developer troubleshooting and were deliberately left out:

- Bridging RBNT and stablecoins to and from other chains, and wrapped RBNT on those chains. These are ecosystem and third-party bridge questions rather than Redbelly developer errors, and the answers change as bridge support evolves.
- Staking platform behaviour and reward economics, which belong to the individual platforms.

If community feedback shows demand for a developer-facing entry in these areas (for example, integrating wrapped RBNT correctly in a dApp), they are natural candidates for future entries and will be tracked in the changelog.

---

[Back to index](../README.md)
