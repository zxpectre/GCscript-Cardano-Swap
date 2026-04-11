# Source: A Peer-to-Peer DeFi Application on Cardano using GCscript

## General Description

This application is an off-chain implementation of a P2P DeFi protocol on Cardano using GCscript.

---

## Foundational Concepts

### Beacon Tokens & Distributed DApps (CIP-0089)

Reference: https://github.com/cardano-foundation/CIPs/blob/master/CIP-0089/README.md

The DeFi protocol this application interfaces with follows the CIP-0089 pattern for distributed DApps (dDApps).

#### Address Model

Every user gets a personal DApp address derived from two components:

```
User DApp Address = DApp spending script (payment credential)
                 + User's staking credential (pubkey, native script, or Plutus script)
```

There is no single shared contract address. Each user's funds live at their own unique address while still being governed by the same spending script. The user retains staking delegation control over their locked assets.

#### Beacon Token Model

Beacon tokens are native assets that tag valid protocol UTxOs, enabling on-chain discovery without centralized indexers.

Lifecycle:
- **Minted** when a DApp UTxO is created
- **Burned** when that UTxO is spent
- **Never circulate freely** — a UTxO carrying the beacon is by definition valid

This invariant means invalid UTxOs cannot carry the beacon token, so filtering by beacon token is sufficient to find all live, valid protocol UTxOs.

#### UTxO Structure

Each protocol UTxO contains:
- The beacon token (presence = validity signal)
- A datum encoding the user's intent (e.g. swap offer, loan terms)
- The locked asset(s)

#### Discovery Mechanism

Because the protocol is distributed (no single address), off-chain logic must discover UTxOs by querying the chain for the beacon token across all addresses, then filtering results client-side. There is no centralized indexer — the beacon token itself is the index.

#### Implications for This Application

- Transaction building must derive the correct personal DApp address per user (spending script + user stake key)
- UTxO discovery queries by beacon token policy ID, not by address
- The application is censorship-resistant and requires no backend infrastructure beyond chain access