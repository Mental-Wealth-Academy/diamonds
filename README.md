# 💎 Diamonds — Currency Documentation

*Mental Wealth Academy Platform · v1.0 · June 2026*

---

## Overview

Diamonds are the Academy's in-platform credit currency — earned by completing quests, reviewed and distributed by Blue, and spendable within the platform ecosystem. They are not a cryptocurrency and they don't require a wallet to earn. They do carry real weight: Diamonds track your contribution, unlock platform features, and feed into the broader reward system that connects quests to stablecoins.

The economy is intentionally two-layer:

| Currency | Symbol | What It Is | How You Get It | What It's For |
| --- | --- | --- | --- | --- |
| **Diamonds** | 💎 | In-platform credits | Quest completion, approved submissions | Platform items, Blue trades, unlock gates, reputation |
| **USDC** | 💵 | Stablecoin, real value | Achievement milestones, approved submissions | Stakeholder participation, treasury coordination, real-world reward |

Diamonds are the everyday currency — the thing you earn by showing up and doing the work. USDC is what that work converts to when it reaches the threshold. Think of Diamonds as proof of effort; USDC as proof of outcome.

---

## How Diamonds Are Earned

Every Diamond is issued in response to a real action inside the Academy. There are no passive accrual mechanics, no sign-up bonuses, and no diamond purchases. You earn them by doing things Blue can review.

### Quest Completion

The primary earning path. Each quest in the 12-week cohort program has a defined Diamond reward that is released when Blue approves the submission.

```
Member submits quest  →  Blue reviews submission  →  Approval unlocks Diamond reward
```

Blue scores submissions across six dimensions (clarity, impact, feasibility, ingenuity, depth, and format). Partial approvals can issue partial rewards — a submission flagged for revision earns nothing until the revision is approved.

### Milestone Seals

The `EtherealHorizonPathway` contract tracks 14 on-chain milestones across the 12-week program. Reaching specific seals triggers automatic Diamond bonuses on top of per-quest rewards. These are non-discretionary — if the seal condition is met, the bonus issues.

### Community Participation

Verified participation in governance votes, feedback surveys, and cohort discussions can issue small Diamond amounts. These are capped per period and are not a substitute for quest completion.

---

## How Diamonds Are Spent

Diamonds have two spending contexts: within the platform (UI unlocks, virtual items, Blue trade activations) and as an input toward USDC payouts.

### Platform Items & Unlocks

Non-essential digital items — cosmetics, profile markers, early access to case-study content — are purchasable with Diamonds through the platform store. These items do not affect research outcomes or voting weight; they're acknowledgement of contribution, not advantage purchases.

### Blue Trade Activations

Blue can facilitate structured exchanges: a Diamond cost to unlock a deeper feedback review, a targeted prompt session, or a collaborative quest variant. These are configured per-case-study and represent Blue's time and compute as a real resource.

### USDC Conversion Path

Diamonds are not directly convertible to USDC at a fixed rate. Conversion happens through **achievement gates**: defined milestones that require a minimum Diamond balance plus a qualifying completion record. When both conditions are met, USDC is released from Blue's wallet (`0x0920…4f8a`) to the member's connected wallet on Base.

```
Diamond balance ≥ threshold
  + qualifying quest completions met
  → Blue wallet releases USDC to member wallet on Base
```

The conversion is not automatic — it is triggered by a governance-compatible payout event, meaning the amounts and thresholds for each season are visible to the community.

---

## Technical Implementation

### Storage

Diamonds are stored off-chain in the platform's PostgreSQL database. Each member record holds a `diamond_balance` field. Mutations are write-logged with source type, quest ID, amount, timestamp, and Blue's approval signature.

All balance reads and writes go through the API layer — never directly from the client. Balance cannot be incremented client-side.

### Issuance

Diamond issuance is triggered server-side by Blue's approval response from the Eliza Cloud API. The flow:

```
Blue returns approval → API route validates approval signature
  → sqlQuery increments diamond_balance for wallet address
  → issuance logged with quest_id, amount, approved_at
```

Disputed approvals (e.g. a user appealing a rejection) are escalated to admin review. Admin overrides are logged separately and require `ADMIN_SECRET` auth.

### Rate Limiting

Diamond issuance per address is rate-limited to prevent abuse. Limits are enforced in-memory via `lib/rate-limit.ts`. For the current Vercel Hobby deployment, this resets on cold starts — a Redis-backed limiter is on the roadmap for multi-instance production.

### USDC Payouts

USDC payouts use the Scatter collection contract (`NEXT_PUBLIC_SCATTER_COLLECTION_ADDRESS`). Blue's signing key (`AZURA_PRIVATE_KEY`) authorizes the payout transaction. Payouts are submitted to Base Mainnet and are publicly verifiable on Basescan.

The contract address for the pathway milestone system:
`NEXT_PUBLIC_PATHWAY_CONTRACT_ADDRESS` (see `.env.example`)

---

## Diamond Economy Principles

These are the editorial and design constraints that govern any changes to the Diamond system.

**Diamonds track contribution, not wealth.** There is no way to buy Diamonds with money. The only path to a high Diamond balance is genuine participation over time.

**Blue is the sole issuer.** No system event or admin action issues Diamonds without Blue's approval signal, except pathway milestone bonuses (which are contract-triggered). This keeps issuance honest and auditable.

**Scarcity is seasonal.** Each 12-week cohort season has a defined Diamond emission ceiling. Unclaimed quest rewards do not roll over. This prevents inflation and keeps each season's economy meaningful.

**Transparency before spending.** All item costs and USDC conversion thresholds are published before a season opens. Members know what they're working toward from day one.

**Diamonds don't confer governance weight.** Voting weight in the community governance system is token-based (Blue's approval level), not Diamond-based. High Diamond balances signal reputation but don't control outcomes.

---

## Diamond Amounts by Activity

*Default values for Season 1. Amounts are set per season and published before enrollment opens.*

| Activity | Diamond Reward | Notes |
| --- | --- | --- |
| Quest completion (standard) | 50 💎 | Requires Blue approval |
| Quest completion (research depth) | 100 💎 | Extended submissions, Blue-scored |
| Pathway milestone seal | 75 💎 | Contract-triggered on Base |
| Governance vote participation | 10 💎 | Capped at 30/week |
| Community feedback survey | 15 💎 | Capped at 1 per survey |
| Revision approval (after rejection) | 25 💎 | Replaces the full reward |

---

## For Developers

### Reading a member's balance

```typescript
import { sqlQuery } from '@/lib/db';

const result = await sqlQuery(
  'SELECT diamond_balance FROM members WHERE wallet_address = :address',
  { address: walletAddress }
);
const balance = result.rows[0]?.diamond_balance ?? 0;
```

### Issuing Diamonds (server-side only)

```typescript
import { sqlQuery } from '@/lib/db';

await sqlQuery(
  `UPDATE members
   SET diamond_balance = diamond_balance + :amount
   WHERE wallet_address = :address`,
  { amount: questReward, address: walletAddress }
);

// Always log the issuance
await sqlQuery(
  `INSERT INTO diamond_ledger (wallet_address, amount, source_type, quest_id, approved_at)
   VALUES (:address, :amount, :sourceType, :questId, NOW())`,
  { address: walletAddress, amount: questReward, sourceType: 'quest', questId }
);
```

### Checking conversion eligibility

```typescript
// Example gate: 500 diamonds + at least 6 quests completed
const eligible =
  member.diamond_balance >= 500 &&
  member.quests_completed >= 6;
```

### Auth

All Diamond-mutating API routes require authentication via `getCurrentUserFromRequestCookie()` from `lib/auth.ts`. Never expose balance mutation endpoints to unauthenticated requests.

---

## Frequently Asked Questions

**Can I transfer Diamonds to another member?**
Not in Season 1. Diamonds are non-transferable between accounts. The on-chain USDC reward is what becomes portable.

**What happens to my Diamonds at the end of a season?**
Unspent Diamonds expire at season close. USDC earned during the season remains in your wallet permanently.

**Can I lose Diamonds?**
Not through normal use. Platform items are purchased, not staked — buying an item reduces your balance, it doesn't put it at risk. There is no slashing mechanism.

**What if Blue incorrectly rejects my submission?**
Contact the team at research@mentalwealthacademy.net with your quest ID. Admin review is available for disputed rejections. If overturned, the full quest reward is issued and logged.

**Are Diamonds visible on-chain?**
No. Diamond balances live in the platform database, not on Base. Only the USDC payouts and pathway milestone seals are on-chain. This is intentional — keeping soft currency off-chain keeps gas costs zero for members.

---

*Mental Wealth Foundation — open AI infrastructure, built by everyone, for everyone.*
*Platform docs · Apache-2.0 · © 2026*
