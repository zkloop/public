# PoB Stablecoin Architecture (Presentation Doc)

## Executive Summary
Traditional blockchains trade throughput for global consensus: every transaction is broadcast, ordered, and validated by the whole network. This creates performance bottlenecks, variable confirmation times, and high gas fees that make micropayments impractical. They are also fully transparent by default, exposing business activity to competitors and revealing consumer purchase behavior.

Proof-of-Balance (PoB) addresses these limits by validating balance
transitions off-chain with zero-knowledge proofs and storing them in a write-once BFT (Byzantine Fault Tolerant) key-value store. Transactions no longer require global ordering or mining, so throughput scales with the quorum capacity rather than block size. This eliminates mining-driven gas costs and enables low-value commerce payments. Privacy is first-class: balances are encrypted, transactions contain no linkable addresses or signatures, and only correctness is proven. The result is a stablecoin system that keeps the on-chain settlement guarantees while delivering high throughput, low fees, and privacy suitable for real-world commerce.

## 1) Goal and Constraints
- Build a stablecoin stack that feels like a regular web API to merchants but still preserves on-chain settlement and auditability.
- Improve throughput and scalability beyond classic blockchain limits by moving balance proofs off-chain while keeping settlement on-chain.
- Keep balances confidential and unlinkable while ensuring integrity.

## 2) System Overview (High Level)
The system combines an EVM-compatible settlement layer with a Proof-of-Balance (PoB) layer that updates wallet balances in a BFT key-value store.

```
Web UI / Merchant Backend
        |
        | REST (web3-agnostic)
        v
Merchant API ----> RPC Proxy ----> EVM Exec (exec-v2) ----> Contracts
        |                |                  |
        |                | x-pob-token      | receipts
        |                v                  v
        +----------> PoB Service <---- Wallets (cloud/custodial)
```

## 3) Core On-Chain Contracts
- Registry: name -> address + ABI resolution for web3-agnostic clients.
- PoBERC20: ERC20-compatible token with PoB tagging.
- PoBERC20Escrow: escrow + hold/settle/release with transaction fee handling.
- MerchantBase: merchant contract that calls the escrow, handles rewards, and restricts settlement to merchant wallets.

Key escrow semantics:
- hold: customer approves merchant wallet; escrow moves funds to merchant wallet and records hold.
- settle: merchant wallet settles; escrow refunds hold to customer, then transfers finalAmount to merchant owner and tx fee to operator.
- release: merchant owner or customer can cancel, returning held funds to customer.

## 4) Wallet and PoB Layer
- Wallets live off-chain, protected by PAKE + Oblivious PRF (OPAQUE).
- PoB uses zero-knowledge proofs to show that balances are valid after a transfer, without revealing the balance itself.
- No long-term signing keys are required for PoB proofs; transactions do not contain linkable public keys.
- Balances are encrypted; only the validity of the before/after state is proven.
- Wallets issue one-time PoB tokens used to authorize PoB balance updates.

## 5) Authentication and Tokens
- Merchants log in via OPAQUE; the vault key is known only to the client.
- The client generates a long-term API key (for REST calls) and displays it to the merchant.
- The REST API accepts the API key in the Authorization header.
- For each on-chain transaction, the server generates a one-time PoB token and sends it to the RPC proxy using `x-pob-token`.

## 6) Transaction Flow (Hold -> Settle)
1) Customer wallet has PoB balance and ERC20 balance.
2) Customer approves merchant wallet for holdAmount.
3) Customer calls MerchantBase.hold(merchantWallet, holdAmount).
4) Escrow moves tokens to merchant wallet and records held balance.
5) Merchant calls REST /settle with txId and finalAmount.
6) Merchant API opens merchant wallet (child wallet), generates a one-time PoB token, and submits settle tx through RPC proxy.
7) Escrow settles:
   - Refunds held funds from merchant wallet to customer.
   - Transfers finalAmount - txFee to merchant owner.
   - Transfers txFee to operator.
   - Optional reward transfer from merchant wallet to customer via MerchantBase.
8) EVM exec detects transfers and calls PoB service to update wallet balances.

## 7) Why PoB Improves Scalability and Throughput
Traditional blockchains:
- Broadcast every transaction globally.
- Require total ordering and global consensus.
- Every node validates every transaction and balance.
- Throughput is bounded by consensus and network propagation.

PoB approach:
- Transactions are stored in a write-once BFT key-value store instead of a global chain.
- Participants do not need absolute consistency across all nodes.
- Proof-of-Balance uses ZKPs to validate balance correctness without revealing balances.
- No linkable addresses or signatures are required inside the PoB transaction.
- The "Chain of Balance" forms a DAG (balances are vertices, transactions are edges) rather than a single hash chain.
- Consensus is based on quorum acceptance of a balance proof, not global ordering.

Practical impact in this stack:
- On-chain state changes are minimal (ERC20 + escrow), while balance validity is enforced by PoB off-chain.
- PoB proofs can be verified and stored in parallel across the KV quorum, reducing contention.
- Each transaction only needs a local PoB proof and quorum acceptance, not global re-validation.
- Throughput scales with the quorum capacity rather than the L1 block size or consensus round time.

## 8) Security and Auditability
- PoB prevents double spend by proving balance transitions; KV entries are write-once.
- Equivocation is detectable by the KV quorum (conflicting proofs for the same balance).
- The Chain of Balance DAG can be audited by tracing balance transitions.
- On-chain settlement provides a final, public accounting of token ownership.

## 9) What Merchants See (Web3-Agnostic)
- REST endpoints with familiar inputs: contractName, txId, finalAmount.
- No need to manage nonces, gas, or wallet signing.
- API key in Authorization header; PoB token handled server-side.

## 10) Summary
- PoB decouples balance validation from global on-chain consensus.
- PoB provides scalable, privacy-preserving balance proofs.
- The EVM layer remains the settlement and escrow ledger.
- Combined, the system delivers higher throughput while preserving correctness and auditability.
