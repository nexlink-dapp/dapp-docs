# NexLink Honor & Reputation

> **Status: Contract mechanics SDK-ready; honor-wall UX and issuance pipeline are Design / Proposed.** Soulbound ERC-721 tokens — the on-chain substrate for honors — **can be deployed and minted today** (see [NFT.md](NFT.md)). The first-party **honor wall**, the issuance pipeline for organizations, and the negative-record policy are proposed. This document is the product-level concept a dApp integrates with; [NFT.md](NFT.md) is the contract-level reference.

An **honor (荣誉)** is a **soulbound token (SBT, 灵魂代币)** an organization issues to a person to certify something they earned and cannot buy — a degree, a competition result, research or work output — or a **negative record** (unpaid debt, a violation). Together, a person's honors form a **reputation (信任体系)** rendered as an in-app **honor wall (荣誉墙)**.

---

## 1. What an honor is

| Property | Detail |
|---|---|
| **Non-transferable** | An honor is an SBT — it **cannot be sold, sent, or moved** ([NFT.md §4](NFT.md#4-soulbound-tokens-sbt)). This is what makes it credible: it proves the holder actually earned it. |
| **Issuer-revocable** | Only the **issuer** can revoke (burn) it — e.g. a debt is repaid → the issuer clears the negative record. |
| **Bound to an identity, aggregated to 主身份** | An honor binds to the identity that earns it, and **all of a person's honors aggregate to their 主身份** — see [IDENTITY.md §3](IDENTITY.md#3-honor-aggregation-to-the-main-identity). |

| Category (原文) | Example | Polarity |
|---|---|---|
| 学历证书 | diploma, certification | positive |
| 赛事名次与奖杯 | competition placement, trophy | positive |
| 科研成果 / 工作成果 | research output, work achievement | positive |
| **不良记录** | unpaid credit card, workplace violation | **negative** |

These are the canonical **Soulbound Token** use cases (identity verification, on-chain credit, DAO anti-Sybil, credentials) applied to NexLink.

---

## 2. Issuing an honor (as a dApp / organization)

An honor is a soulbound ERC-721 mint. **First obtain a Foundation root certificate** (§4) — without a root SBT your honors won't count. Then mint from your certified contract. The contract mechanics are in [NFT.md](NFT.md); the honor-specific rules:

```javascript
// Mint a soulbound honor to a recipient identity's address.
// (Use a soulbound collection — transfers revert; see NFT.md §4.)
const { txHash } = await NexlinkApp.contract.call({
  contract: HONOR_COLLECTION_ADDRESS,
  abi: SOULBOUND_NFT_ABI,
  method: "mint",
  args: [recipientAddress, metadataURI]   // recipient = the earning identity's address
});
```

**Honor metadata** extends the standard ERC-721 schema ([NFT.md §6](NFT.md#6-metadata)) with reputation fields:

```json
{
  "name": "MSc Computer Science",
  "image": "ipfs://…",
  "attributes": [
    { "trait_type": "category", "value": "degree|trophy|research|work|negative" },
    { "trait_type": "polarity", "value": "positive|negative" },
    { "trait_type": "issuer",   "value": "Tsinghua University" },
    { "trait_type": "issued_at","value": "2025-06-30" },
    { "trait_type": "evidence", "value": "ipfs://…" }
  ]
}
```

Host metadata on IPFS so it is tamper-evident after mint.

---

## 3. Negative records — the integrity rules

A negative record is an honor with `polarity: negative`. Three rules make it meaningful:

| Rule | Why |
|---|---|
| **Non-transferable** | can't be moved to escape it |
| **No holder self-burn** | the holder must not delete their own bad record — **only the issuer** may revoke |
| **Aggregates to 主身份** | surfaces at the one-per-person 主身份, so it **cannot be dodged by switching identity** ([IDENTITY.md §3](IDENTITY.md#3-honor-aggregation-to-the-main-identity)) |

> The generic soulbound contract in [NFT.md §4](NFT.md#4-soulbound-tokens-sbt) allows holder burn. **Negative-record collections must remove holder burn** and gate revoke to the issuer.

---

## 4. Trust anchoring — the Foundation is the Root CA

Honors are gated by an on-chain **certificate hierarchy** (a decentralized PKI) so the market can't be flooded with fake credentials.

- The **NexLink Foundation is the Root CA.** It maintains the single [Identity contract](IDENTITY.md) and anchors all honor trust.
- **Anyone may run their own honor contract, but must first obtain a root certificate (根证书 / 根 SBT / 通行证) from the Foundation.** You deploy your honor contract; the Foundation **verifies it, then issues the root certificate** — a root SBT your issuer wallet holds.
- The root certificate has **three types**, each a class of root SBT:

| Root-cert type | 原文 | Issued to |
|---|---|---|
| **Government** | 政府 | government bodies |
| **Foundation** | 基金会 | the Foundation itself |
| **Institution** | 其他机构 | other recognized organizations |

- Only a holder of a root SBT can mint honor SBTs from its verified contract. An honor's credibility comes from **which root-cert type its issuer holds** — the honor wall badges honors by tier, **read on-chain from the issuer's root SBT** (never from spoofable metadata).

```
NexLink Foundation (Root CA)
  └── Root SBT (通行证) — 3 types: 政府 / 基金会 / 其他机构
        └── your verified Honor contract → Honor SBTs → the identity's wallet
```

---

## 5. Proving honors without revealing them (ZK)

A dApp often needs to know a person **meets a bar** (creditworthy, no negative record, holds a credential) without seeing their whole résumé — especially when they act as an **匿名身份**. This is a **zero-knowledge proof** over the person's aggregated honors, issued by the **主身份**:

- The dApp sets the **constraint (约束条件)**; the user's 主身份 proves it in ZK; the identity is never revealed.
- See [IDENTITY.md §4](IDENTITY.md#4-trust-without-deanonymization--the-zk-proof) for the flow and the proposed `NexlinkApp.identity.prove()` surface.
- **Escrow ([ESCROW.md](ESCROW.md)) is the flagship consumer** — gate a guaranteed trade on a ZK credit/reputation proof.

---

## 6. Displaying honors

- **Two surfaces** — identity SBTs and honor SBTs show in **the wallet** (like any held token) *and* the **personal homepage (个人主页)** honor wall.
- **Honor wall** — positive and negative sections, grouped by category, each honor badged by its issuer's **root-cert tier** (政府/基金会/其他机构).
- **Per-identity vs aggregated** — a persona shows its own honors; the **主身份** shows the **aggregated** person-level set.
- **In a dApp** — read a collection's `balanceOf` / `tokenURI` for a recipient, or (recommended) query a backend honor indexer rather than enumerating on-chain.

---

## 7. What Needs Building

### Available today (via [NFT.md](NFT.md))
- [x] Soulbound ERC-721 collections on NEXLK (`2026777`)
- [x] Mint + read via `NexlinkApp.contract`

### Proposed
- [ ] **Foundation Root-CA / root-certificate contract** — issues root SBTs in 3 types (政府/基金会/其他机构) after verifying an issuer's deployed contract
- [ ] Honor metadata standard (category / polarity / issuer / evidence); honor mint gated on the issuer holding a root SBT
- [ ] Negative-record collection variant (issuer-only revoke, no holder burn)
- [ ] Root-certificate **application + verification flow** (replaces a permissionless issuer registry)
- [ ] Honor aggregation to 主身份 + honor indexer / listing endpoints
- [ ] Honor wall in the **个人主页** + identity/honor SBTs in the **wallet**; root-cert tier badge

### Documentation
- [x] HONOR.md — this document
- [ ] API.md — honor issuance / listing endpoints (mark proposed)
- [x] SUMMARY.md — Honor link

**See also:** [NFT Issuance](NFT.md) (contract mechanics) · [Identity System](IDENTITY.md) (binding + aggregation + ZK) · [Community Governance](GOVERNANCE.md) (Delegate-ID & expert-panel honors) · [Escrow](ESCROW.md) (ZK-gated trust).
