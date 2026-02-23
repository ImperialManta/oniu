# OniU — Decentralized Content Creator Platform

> Built on BNB Chain · AES-256-GCM End-to-End Encryption · Non-Custodial

**OniU** is an on-chain content monetization platform where creators retain full ownership of their content and earnings. All subscriptions, purchases, and revenue distributions are transparently recorded on the BNB Chain smart contract — the platform cannot tamper with them.

🌐 **Live**: https://oniu.pages.dev

---

## Features

| Feature | Description |
|---------|-------------|
| 🔐 **E2E Encrypted Content** | Files are AES-256-GCM encrypted in the browser before upload. The server only ever stores ciphertext. |
| ⛓️ **On-Chain Payments** | Subscriptions, PPV purchases, tips, and withdrawals all go through the smart contract — no intermediary holds funds. |
| 🗂️ **Decentralized Storage** | Encrypted content is stored on IPFS via each creator's own Pinata account. |
| 💬 **Encrypted DMs** | Premium creators can send one-to-one encrypted private messages with a custom unlock price. |
| 🏆 **Super Fan Board** | On-chain tip records power a leaderboard for each creator's top fans. |
| 🌍 **Multilingual** | Full Traditional Chinese (zh-TW) and English (en) support. |

---

## Creator Tiers

| | Basic | Pro | Premium |
|-|-------|-----|---------|
| Platform Fee | 5% | 8% | 12% |
| Subscription Revenue | ✅ | ✅ | ✅ |
| PPV Content | — | ✅ | ✅ |
| Tips | — | ✅ | ✅ |
| Encrypted DM | — | — | ✅ |

---

## Smart Contract

| | |
|-|-|
| **Network** | BNB Smart Chain (Chain ID: 56) |
| **Contract** | [`0x5D6741386FFeC7AD7BeE9382CE3589e0319b2a80`](https://bscscan.com/address/0x5D6741386FFeC7AD7BeE9382CE3589e0319b2a80#code) |
| **USDT (BEP-20)** | `0x55d398326f99059fF775485246999027B3197955` |
| **Compiler** | Solidity `0.8.24`, Optimizer 200 runs |
| **License** | MIT |

The contract is **verified on BSCScan** — source code is publicly auditable.

### Key Design Decisions

- **Non-upgradeable**: No proxy pattern. Logic is immutable after deployment.
- **Non-custodial**: The platform has no ability to move user or creator funds.
- **cancelSubscription has no `whenNotPaused`**: Users can always cancel even during an emergency pause, preventing lock-in.
- **2-step ownership transfer**: Prevents accidental transfer to the wrong address.
- **Reentrancy protection**: Manual lock + Check-Effects-Interactions pattern on all withdrawal functions.

---

## Architecture

```
Browser (MetaMask)
    │
    ├──── on-chain txs ────▶  CreatorPlatform.sol  (BNB Chain)
    │                          subscriptions / PPV / tips / withdrawals
    │
    └──── REST API ────────▶  Cloudflare Worker  (oniu-api)
                                    │
                                    ├── Cloudflare D1 (SQLite)
                                    │     creator profiles / content metadata / sub cache
                                    │
                                    └── BNB Chain RPC
                                          event sync (every 5 min) / subscription verification
```

### Tech Stack

| Layer | Technology |
|-------|-----------|
| Smart Contract | Solidity 0.8.24, Foundry |
| Backend API | Cloudflare Workers (TypeScript) |
| Database | Cloudflare D1 (SQLite) |
| Frontend | Vanilla HTML / CSS / JS, Cloudflare Pages |
| Encryption | Web Crypto API — AES-256-GCM |
| Blockchain library | ethers.js v6 |
| File Storage | IPFS via Pinata (creator-managed) |

---

## Repository Structure

```
├── contracts/
│   └── CreatorPlatform.sol     # Main contract
├── script/
│   └── DeployBSC.s.sol         # Mainnet deploy script
├── test/
│   └── CreatorPlatform.t.sol   # Foundry tests
├── workers/
│   └── src/
│       ├── index.ts            # Worker entry + CORS + cron scheduler
│       └── creator.ts          # All API routes + chain sync logic
├── frontend/
│   ├── index.html              # Single-page app
│   ├── app.js                  # Main logic
│   ├── crypto.js               # AES-256-GCM helpers
│   ├── i18n.js                 # zh-TW / en translations
│   └── styles.css
├── schema.sql                  # D1 database schema
└── foundry.toml
```

---

## Security

### Content Encryption Flow

```
Upload:
  Browser generates random AES-256-GCM key
      → encrypts file locally
      → uploads ciphertext to IPFS (creator's Pinata)
      → encrypts the AES key with MetaMask session signature
      → stores { encrypted_cid, encrypted_key } in D1

Decrypt:
  API verifies subscription / PPV purchase on-chain (source of truth)
      → returns { encrypted_key, encrypted_cid }
      → browser recovers AES key via MetaMask session signature
      → decrypts content locally — server is never involved
```

### API Authentication

All write endpoints require an **EIP-191 session signature**:

```
Message: "creator-platform:session:{address}:{unix_timestamp}"
Valid for: 4 hours
```

The server verifies the recovered address matches the claimed address. No JWT, no cookies — just a signed message.

---

## Local Development

### Prerequisites

```bash
# Foundry
curl -L https://foundry.paradigm.xyz | bash && foundryup

# Wrangler (Cloudflare Workers CLI)
npm install -g wrangler

# Worker dependencies
cd workers && npm install
```

### Start local chain and deploy contract

```bash
# Terminal 1 — local EVM
anvil

# Terminal 2 — deploy + seed test data
forge script script/LocalSetup.s.sol --rpc-url http://localhost:8545 --broadcast
```

### Start the Worker (local)

```bash
cd workers
npx wrangler dev   # http://localhost:8787
```

### Start the frontend (local)

```bash
cd frontend
npx serve . -p 8080
```

---

## Deployment

### Contract (already deployed)

```bash
forge script script/DeployBSC.s.sol \
  --rpc-url https://bsc-dataseed.binance.org/ \
  --broadcast \
  --private-key 0xYOUR_KEY
```

### Worker

```bash
cd workers
npx wrangler deploy --env production
```

### Frontend

```bash
cd frontend
npx wrangler pages deploy . --project-name oniu
```

---

## API Overview

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| GET | `/creators` | — | List creators (`q`, `tier`, `limit`) |
| GET | `/creators/:address` | — | Get creator profile |
| GET | `/content/:creator` | — | List creator's content |
| GET | `/content/:id/decrypt-key` | session sig | Get decryption key (checks subscription/purchase on-chain) |
| POST | `/content/upload` | session sig | Upload content metadata |
| PATCH | `/content/:id` | session sig | Edit content |
| DELETE | `/content/:id` | session sig | Soft-delete content |
| GET | `/subscriptions/:address` | — | Get subscription records |
| GET | `/fans/:creator` | — | Super fan leaderboard |
| GET | `/dm/:address` | — | DM list |
| GET | `/dm/:id/decrypt-key` | — | Get DM decryption key |
| POST | `/dm/store-key` | session sig | Store DM encrypted key after IPFS upload |
| POST | `/sync-creator` | — | Manually trigger chain sync |

---

## License

MIT © 2026 OniU

---

<div align="center">
  <a href="https://github.com/ImperialManta">GitHub</a> ·
  <a href="https://x.com/ImperialManta">X (Twitter)</a> ·
  <a href="https://oniu.pages.dev">Live App</a> ·
  <a href="https://bscscan.com/address/0x5D6741386FFeC7AD7BeE9382CE3589e0319b2a80#code">BSCScan</a>
</div>
