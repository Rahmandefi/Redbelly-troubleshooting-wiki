# EligibilitySDK and Onboarding SDK Issues

[Back to index](../README.md)

Installation, widget rendering, cross-origin behaviour and verification checks for Redbelly's compliance tooling. Entries 15 to 19.

---

### 15. npm install fails with 401 for @redbellynetwork packages

**Symptom**

```
npm ERR! code E401
npm ERR! 401 Unauthorized - GET https://npm.pkg.github.com/@redbellynetwork%2feligibility-sdk
```
or `404 Not Found` for the same package.

**Root Cause**

Redbelly SDK packages are hosted on **GitHub Packages**, not the public npm registry. Without an `.npmrc` pointing the `@redbellynetwork` scope at GitHub's registry plus a valid GitHub token, npm either looks in the wrong registry (404) or is rejected (401).

**Solution**

1. Create a GitHub Personal Access Token (classic) with the `read:packages` scope: GitHub > Settings > Developer settings > Personal access tokens.

2. Create `.npmrc` in your project root:

```
@redbellynetwork:registry=https://npm.pkg.github.com
//npm.pkg.github.com/:_authToken=${GITHUB_TOKEN}
```

3. Export the token and install:

```bash
export GITHUB_TOKEN=your_token_here
npm i @redbellynetwork/eligibility-sdk
```

4. In CI, set `GITHUB_TOKEN` as a secret; never commit a literal token in `.npmrc`.

**Prevention**

Commit the `.npmrc` with the `${GITHUB_TOKEN}` placeholder (safe, contains no secret) and document the token requirement in your README so every new contributor and CI runner sets it before `npm install`.

---

### 16. EligibilitySDK widget not rendering

**Symptom**

The onboarding widget area is blank, stuck on a spinner, or the component mounts with no iframe. The browser console may show SDK initialisation errors or 401/403 responses from the verifier service.

**Root Cause**

The widget aborts silently when its required configuration is missing or wrong. The usual suspects: `REDBELLY_API_KEY` or `ALLOWED_ISSUER_DID` unset (or not exposed to the browser build), the SDK initialised for the wrong environment, or the component rendered server-side where `window` does not exist.

**Solution**

1. Check the browser console and network tab first: a 401 from the verifier API means the API key is missing or invalid; contact the Redbelly team if you have not been issued one.

2. Confirm the environment variables are present in `.env.local` and actually reach the client. In Next.js, server-only names are invisible to browser code; the widget config must come from `NEXT_PUBLIC_`-prefixed variables or be passed down from a server component:

```bash
# .env.local
REDBELLY_API_KEY=your_key           # server-side verification
ALLOWED_ISSUER_DID=did:receptor:redbelly:testnet:31K82iKCtE6ciDc7oAr3T5EpjZb4S1EFM7c4xJaWkM2
```

3. In the Next.js App Router, render the widget client-side only:

```tsx
"use client";
import dynamic from "next/dynamic";
const OnboardingWidget = dynamic(() => import("./OnboardingWidget"), { ssr: false });
```

4. Restart the dev server after any `.env.local` change; Next.js only reads env files at startup.

**Prevention**

Validate SDK config at startup and render a visible error state ("Verification service not configured") instead of an empty div. Keep testnet and mainnet issuer DIDs in separate env files.

---

### 17. Cross-origin and iframe issues with the SDK

**Symptom**

```
Blocked a frame with origin "https://..." from accessing a cross-origin frame
```
or the SDK iframe refuses to load with a CSP / `X-Frame-Options` error in the console, or `postMessage` events from the widget never arrive.

**Root Cause**

The onboarding widget runs in an iframe served from a Redbelly domain and communicates via `postMessage`. Your site's Content-Security-Policy can block the frame from loading (`frame-src`), and overly strict message-origin filtering (or third-party cookie blocking in the browser) can break the callback channel.

**Solution**

1. Read the exact console error. `Refused to frame ...` means your own CSP is the blocker. Allow the SDK origin in `frame-src`. In Next.js:

```js
// next.config.js
headers: async () => [{
  source: "/(.*)",
  headers: [{
    key: "Content-Security-Policy",
    value: "frame-src 'self' https://*.redbelly.network;",
  }],
}],
```

2. If you listen for widget events yourself, do not drop messages by checking the wrong origin; log `event.origin` once to see the real value, then allowlist exactly that.

3. Test in a normal browser profile. Extensions and "block third-party cookies" settings break iframe sessions; if it works in a clean profile, it is a browser privacy setting, not your code.

4. Serve your app over HTTPS (or localhost); mixed-content rules block an HTTPS iframe inside an HTTP page.

**Prevention**

If you deploy a CSP, include `frame-src` for Redbelly domains from the start, and test the full onboarding flow in Chrome, Firefox and Safari before release; Safari's tracking prevention is the strictest.

---

### 18. Onboarding wallet callback never arrives in local development

**Symptom**

You scan the QR code or open the identity wallet (Keyper or Privado), complete the credential step on your phone, but your locally running dApp never receives the completion callback. The flow works when deployed.

**Root Cause**

The onboarding flow depends on the identity wallet calling your application back over the network. `http://localhost:3000` is not reachable from your phone or from Redbelly's services, so the callback dies. This is why the official getting-started guide requires a tunnel (ngrok) for local development.

**Solution**

1. Start a tunnel to your dev server:

```bash
ngrok http 3000
```

2. Use the generated `https://xxxx.ngrok-free.app` URL as the app URL in your SDK configuration and open the app through that URL (not localhost) while testing.

3. Restart the flow from scratch after switching URLs; sessions started on localhost will not resume on the tunnel URL.

4. If the phone and callback still fail, confirm the phone has internet access (not just isolated local Wi-Fi) and that the tunnel is still alive; free ngrok URLs change on every restart.

**Prevention**

Script the tunnel into your dev workflow (for example an npm script that starts ngrok and prints the URL to paste into `.env.local`). For team testing, use a deployed preview environment instead of individual tunnels.

---

### 19. hasChainPermission returns false for a verified user

**Symptom**

A user completed the KYC flow in the onboarding widget, but `hasChainPermission(address)` (contract call or `useHasChainPermission` hook) still returns `false`, so your gated contract reverts or your UI keeps them locked out.

**Root Cause**

Verification status is per-address, per-environment and per-issuer. The mismatch is usually one of: the user verified a different wallet address than the one connected, they verified on a different environment (testnet vs mainnet) than your app queries, your app trusts a different issuer DID than the one that issued the credential, or the credential has expired.

**Solution**

1. Log exactly what you are checking:

```js
console.log("checking address:", address);
const ok = await eligibility.hasChainPermission(address);
console.log("hasChainPermission:", ok);
```

Compare that address character for character with the wallet the user verified. Multiple accounts in MetaMask is the most common source of this bug.

2. Confirm your app and the verification flow point at the same environment: chain ID 153 config with a testnet issuer DID, or 151 with mainnet. A testnet credential never validates on mainnet.

3. Check `ALLOWED_ISSUER_DID` matches the issuer used by the onboarding widget configuration.

4. If everything matches and it worked before, the credential may have expired; run the user through the onboarding flow again.

**Prevention**

In the UI, always display which connected address is being checked, and gate actions on the same address object you pass to the hook. Keep environment config (chain plus issuer DID) in one shared module so testnet and mainnet cannot mix.

---

[Back to index](../README.md) | Next category: [Faucet and Funding](faucet.md)
