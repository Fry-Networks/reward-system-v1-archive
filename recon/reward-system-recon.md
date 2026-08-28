# Fry Networks Miner Reward System — Full Recon Synthesis
**Date**: 2026-06-08
**Hosts touched**: <internal-host-ares>, <internal-host-hermes>, <internal-host-zeus>, <internal-host-heph> (read-only)

---

## CRITICAL DISCOVERY: TWO SEPARATE REWARD SYSTEMS

| System | Location | Purpose | Scope |
|--------|----------|---------|-------|
| **dbrewards** (<internal-host-ares>) | `/home/fryadmin/projects/dbRewards` | Device-specific miner rewards (PoC-gated, weekly maturation) | THIS is the miner reward system being archived/replaced |
| **fry.farm daily claim** (<internal-host-heph>) | `/opt/fry-farm/backend/controllers/rewardsController.js` | User-level daily FRY claim (CAPTCHA, anti-sybil, capped-hybrid) | SEPARATE system — stays, not part of archive |

The operator wants to archive/replace **dbrewards** (the miner reward system). The fry.farm daily claim is a different system entirely.

**Current state**: Dashboard claim endpoints (`/api/rewards/claim`, `/api/rewards/confirm`) are **DISABLED at nginx level** since 2026-04-23 (503 maintenance). Users cannot currently claim miner rewards.

---

## 18a. ARCHIVE MANIFEST

Everything that IS the miner reward system — archive to its own GitHub repo before removal.

### <internal-host-ares> — dbrewards core

**Codebase**: `/home/fryadmin/projects/dbRewards/`
**Git**: `https://github.com/Fry-Foundation/dbRewards.git` (branch: `feat/virtual-mining-phase1`)

| File/Dir | Purpose |
|----------|---------|
| `src/main.ts` | Entry point, hourly scheduler (xx:15), daily maturation, weekly aggregation |
| `src/reward.ts` | Reward calculation + recording (`recordReward()` function) |
| `docker-compose.yml` | Service definitions: dbrewards-prod, dbrewards-weekly, dbrewards-sim, wix-sync |
| `Dockerfile` | Build definition |
| All `src/**/*.ts` | Full business logic |
| Config via Docker Compose env | 40+ env vars (see §8 below) |

**Docker services**:
- `dbrewards-prod` — hourly reward processing (xx:15), 461-527 devices/hour
- `dbrewards-weekly` — Friday 00:05-00:10 UTC, weekly maturation + rollup
- `dbrewards-sim` — simulation mode
- `wix-sync` — product sync from Wix (15 min interval, 1048 products)

**MongoDB collections** (all in `main` database on <internal-host-ares>):

| Collection | Count | Archive? | Purpose |
|------------|-------|----------|---------|
| `device-rewards` | 11,185 | YES | Primary reward ledger per device (daily_rewards[], weekly_rewards[], totals) |
| `poc_reward_dailies` | 494,253 | YES | Daily PoC slot data (slots_valid, slots_total, category, multiplier) |
| `rewards_archived` | 2,181,893 | YES | Legacy pre-aggregation daily records |
| `reward-boosts` | 17,216 | YES | Active boost/multiplier records |
| `counters` | 9,804 | YES | Sequence counters (reward_number increments) |
| `rewards_job_runs` | 30 | YES | Batch distribution job tracking |
| `products` | 42 | SHARED | Device type config — used by admin panel AND dbrewards |
| `devices` | 19,926 | SHARED | Device registry — used by dashboard AND dbrewards |

### <internal-host-zeus> — hardwareapi (PoC eligibility + device registration)

**Codebase**: `/home/fry/subdomains/hardware_exe_api/`
**Git**: `https://github.com/Fry-Foundation/hardwareapi-poc.git` (branch: `main`, clean)

| File | Lines | Purpose |
|------|-------|---------|
| `app.py` | 2,411 | FastAPI: 40+ endpoints, 4-tier auth, IP ban/whitelist |
| `models.py` | 542 | Pydantic models: MinerCode enum, HardwareDocument, etc. |
| `storage.py` | 1,298 | MongoDB backends (PoC, creds, measurements DBs) |
| `poc_eligibility.py` | 119 | INSTALLER reward_eligible 3-gate computation |
| `measurement_aggregator.py` | 248 | Daily measurement aggregation |
| `docker-compose.yml` | — | Service + env config (secrets via 1Password) |
| `Dockerfile` | — | Python 3.11-slim + 1Password CLI |
| `deployment/banned_ips.json` | — | Persistent IP ban list |

**Separate IoT pipeline**: `/home/fry/iot-slot-pipeline/main.py` (10-min async loop, writes directly to MongoDB)

**MongoDB collections** (on <internal-host-ares>, databases: PoC, creds, measurements):

| Collection | Database | Archive? | Purpose |
|------------|----------|----------|---------|
| `hardware` | PoC | YES | Device hardware docs with `reward_eligible` flag |
| `versions` | PoC | YES | Version requirements per miner code + reward multipliers |
| `installations` | PoC | YES | Installation heartbeat records |
| `leases` | PoC | YES | Installation leases (BM one-per-IP enforcement) |
| `hardware` | creds | SHARED | Miner profiles (address, type, verified flag) |
| `mysterium` | PoC | YES | Mysterium keystore storage (BM partner) |
| `presearch` | PoC | YES | Presearch node connectivity (RDN partner) |
| `*` | measurements | YES | Per-country measurement collections |

### <internal-host-hermes> — dashboard reward UI + claim API

**Dashboard codebase**: `/home/helpdesk/subdomains/dashb/`
**Git**: `https://github.com/Fry-Foundation/registration_portal.git` (branch: `main`, clean)

**Reward-specific files to archive**:

| Path (relative to dashb/) | Purpose |
|---------------------------|---------|
| `pages/api/rewards/claim.ts` | Instant claim + free withdrawal (~650 lines) |
| `pages/api/rewards/confirm.ts` | Claim confirmation |
| `pages/api/rewards/boost.ts` | Fee-based reward boost (~300 lines) |
| `pages/api/rewards/get-reward-summary.ts` | Single device reward summary (245 lines) |
| `pages/api/rewards/get-reward-summary-batch.ts` | Batch summary (200 devices in 5-7 calls) |
| `pages/api/rewards/get-rewards-page.ts` | Pagination/UI data |
| `pages/api/rewards/get-asset-totals.ts` | Asset-specific totals |
| `pages/api/fee/pay-withdraw.ts` | 30% fee payment + withdrawal |
| `pages/api/fee/verify-pay.ts` | On-chain fee verification |
| `pages/api/stake/verification.ts` | Verification stake (50 fNODE) |
| `pages/api/stake/registration.ts` | Registration stake (50 fNODE for AEM/nodes) |
| `pages/api/stake/node-staking.ts` | Node stake multiplier |
| `pages/api/stake/stake-withdraw.ts` | General stake withdrawal |
| `pages/api/stake/r-withdraw.ts` | Registration stake withdrawal |
| `pages/api/stake/n-withdraw.ts` | Node stake withdrawal |
| `pages/api/stake/withdrawable.ts` | Query withdrawable amount |
| `pages/api/stake/precheck.ts` | Pre-validation |
| `components/modals/Claim.tsx` | Claim UI (706 lines) |
| `components/modals/Stake.tsx` | Verification stake UI |
| `components/modals/StakeVerification.tsx` | Stake confirmation UI |
| `components/modals/Withdraw.tsx` | Free withdrawal UI |
| `components/modals/WithdrawAll.tsx` | Withdraw-all UI |
| `components/modals/Boost.tsx` | Boost UI |
| `components/modals/rewardWallet.tsx` | Reward wallet config |
| `components/modals/WithdrawStakeVerification.tsx` | Stake withdrawal confirmation |
| `lib/hooks/useRewardSummary.ts` | React hook for single device |
| `lib/hooks/useRewardSummaryBatch.ts` | React hook for batch fetch |
| `lib/types.ts` | Reward, RewardBoost, Product, Asset interfaces |
| `lib/utils.ts` | ASA ID constants |

**Admin panel codebase**: `/home/helpdesk/subdomains/admin_panel/`
**Git**: `https://github.com/Fry-Foundation/admin_panel.git` (branch: `main`, clean)

**Reward-config files to archive**:

| Path (relative to admin_panel/) | Purpose |
|---------------------------------|---------|
| `pages/api/edit-product.ts` | Modify product reward rates/tokens/stakes |
| `pages/api/update-multiplier.ts` | Adjust multipliers |
| `pages/api/update-rewards-enabled.ts` | Enable/disable rewards globally |
| `pages/api/reward-history.ts` | Historical distributions |
| `pages/rewards.tsx` | Reward management UI |
| `pages/stakes.tsx` | Stake configuration UI |
| `lib/products-schema.ts` | Product DB schema |
| `lib/reward-schema.ts` | Reward DB schema |
| `components/reward-history.tsx` | Historical display |
| `components/form-device.tsx` | Device edit form |

**nginx**: Claim endpoints disabled via 503 block in `/etc/nginx/sites-enabled/` (added 2026-04-23)

### On-chain contracts (for reference archive, not removal)

| App ID | Name | Chain | Purpose |
|--------|------|-------|---------|
| 3465579498 | FryStakingV2 Pool 1 | Algorand | Staking pool (backend claim routing) |
| 3468848937 | FryStakingV2 Pool 2 | Algorand | Staking pool |
| 3470020844 | FryStakingV2 Pool 3 | Algorand | Staking pool |
| 3476263283 | FryStakingV2 Pool 4 | Algorand | Staking pool |
| 3469720617 | FryStakingV2 Pool 5 | Algorand | Staking pool |
| 3476263325 | FryStakingV2 Pool 6 | Algorand | Staking pool |

### Cron jobs (part of reward system)

| Host | Schedule | Command | Purpose |
|------|----------|---------|---------|
| <internal-host-ares> | `0 3 * * *` | `/usr/local/bin/mongo_backup.sh` | Daily DB backup |
| <internal-host-ares> | `0 5 * * 0` | `/usr/local/bin/mongo_test_restore.sh` | Weekly restore test |
| <internal-host-ares> | `*/5 * * * *` | `/usr/local/bin/ares00_health_check.sh` | Health check |
| <internal-host-heph> | `30 0 * * *` | `feeDistributionCron.js` | Daily fee distribution (related) |

---

## 18b. DEPENDENCY MAP — Things That DEPEND ON the Reward System

These will break when the reward system is ripped out:

### Dashboard (<internal-host-hermes>) — direct consumers

| Component | Depends On | Impact |
|-----------|-----------|--------|
| Reward display pages (`pages/index.tsx`, `pages/devices.tsx`, `pages/history.tsx`) | `device-rewards` collection, `/api/rewards/get-reward-summary*` | No reward data to display |
| Claim modal (`Claim.tsx`) | `/api/rewards/claim` endpoint | Already disabled at nginx |
| Withdraw modal (`Withdraw.tsx`, `WithdrawAll.tsx`) | `/api/rewards/claim` endpoint | Already disabled |
| Boost modal (`Boost.tsx`) | `/api/rewards/boost` endpoint | No rewards to boost |
| Staking modals (`Stake.tsx`, etc.) | `/api/stake/*` endpoints | Need rewiring to new contract |
| `useRewardSummary` / `useRewardSummaryBatch` hooks | `device-rewards` collection schema | Must update data source |
| Floating totals widget | `device-rewards.total_claimable/total_claimed` | Must update data source |

### Admin panel (<internal-host-hermes>) — config writers

| Component | Depends On | Impact |
|-----------|-----------|--------|
| `edit-product.ts` | `products` collection | Products collection stays (shared), but reward fields become dead |
| `update-multiplier.ts` | `products.reward.verified/unverified` | Multiplier fields unused by new system |
| `update-rewards-enabled.ts` | `configs.rewards.enabled` flag | Flag irrelevant |
| `rewards.tsx` page | All reward API endpoints | Page becomes dead |
| `stakes.tsx` page | Staking API endpoints | Needs rewiring |

### hardwareapi (<internal-host-zeus>) — eligibility writer

| Component | Depends On | Impact |
|-----------|-----------|--------|
| `poc_eligibility.py` | `reward_eligible` flag consumed by dbrewards | Flag no longer read; harmless to leave |
| `PUT /PoC/{miner_key}/hardware` | Writes `reward_eligible` to MongoDB | Writes continue but nothing reads them |
| IoT pipeline (`/home/fry/iot-slot-pipeline/`) | Writes PoC slots to MongoDB | Writes continue but dbrewards gone |
| `/admin/versions/{miner_code}/rewards` | Sets multiplier_base/multiplier_per_tool | Values no longer consumed |

### fry.farm (<internal-host-heph>) — adjacent system

| Component | Depends On | Impact |
|-----------|-----------|--------|
| Device staking (`device_staking_func.ts`) | Registration stake → enables reward eligibility in dbrewards | Staking still works but doesn't affect rewards |
| Anti-sybil service | `hasActivePosition()` checks on-chain stakes | Unaffected (checks staking contracts, not dbrewards) |
| Fee distribution service | Collects fees from claims → distributes via DistPoolV2 | No claim fees if claims disabled; fee source dries up |

---

## 18c. DEPENDENCY MAP — Things the Reward System DEPENDS ON

The NEW system must interface with these:

### Device activity data

| Source | How | Collection/Endpoint | Shape |
|--------|-----|---------------------|-------|
| hardwareapi (<internal-host-zeus>) | INSTALLER PoC docs | `PUT /PoC/{miner_key}/hardware` → `PoC.hardware` | `{miner_key, reward_eligible: bool, lastUpdated, software: {poc_version_installed}}` |
| IoT pipeline (<internal-host-zeus>) | NON_INSTALLER polling | Direct MongoDB write → `PoC.hardware` | `{miner_key, online: bool, data: bool, slots array}` |
| hardwareapi leases | Installation heartbeats | `POST /installations/{miner_key}/installations/{id}` → `PoC.installations` | `{miner_key, install_id, last_seen_at, software_version}` |

### User/wallet identity

| Source | Collection | Key Fields |
|--------|-----------|------------|
| `main.devices` | <internal-host-ares> MongoDB | `{miner_key, address (wallet), reward_wallet, verified, isRegistered, staked: {type, amount, txId}}` |
| `main.registration-users` | <internal-host-ares> MongoDB | `{address, email, discordId (98.8% missing)}` |

### Miner type registry

| Source | Collection | Shape |
|--------|-----------|-------|
| `main.products` | <internal-host-ares> MongoDB | `{key, name, wix_id, reward: {unverified, verified, stake: {stake_one, stake_two, register, node, stake_one_usd, stake_two_usd}, tokens: {staked, reward, register, node}}}` |

42 products. Key types:
- **High-reward nodes** (fNODE): RDN, SDN, SVN, CN — 119.04/day
- **AI Edge Miners** (tFRY): AEM — 990/day
- **High-end miners** (tFRY): HWM, IHAQM, etc. — 59.52/day
- **Low-end miners** (tFRY): LWM, ILAQM, etc. — 22.89–29.76/day
- **Virtual nodes** (fNODE): VRDN, VSDN, VSVN — 119.04/day
- **Cameras** (tFRY): AIWCM, AOWCM, etc. — 29.76/day
- **Merchandise** (no reward): FNEB, FNWRT, etc. — 0/day

### Algorand node

| Endpoint | Host | Port | Used For |
|----------|------|------|----------|
| algod | <internal-host-atlas> | 4190 | Algorand mainnet queries + txn submission |
| algod | <internal-host-atlas> | 4191 | Voi mainnet queries |

### Verification staking

| Location | How Interface Works |
|----------|-------------------|
| `main.devices.staked` | `{type: "one"|"two", amount, txId, time, rewarded_time}` |
| Dashboard `/api/stake/verification.ts` | User builds 50 fNODE txn → confirms on-chain → stored in devices.staked |
| dbrewards reads | `device.verified === true && device.staked.type` → selects `product.reward.verified` (3x) vs `product.reward.unverified` (0.5x) |

### Token ASAs

| ASA ID | Symbol | Decimals | Role |
|--------|--------|----------|------|
| 2681521901 | tFRY | 6 | Primary miner reward token (most device types) |
| 2485202024 | fNODE | 6 | Node reward token (RDN/SDN/SVN/CN/AEM/virtual nodes) |
| 2485314946 | FRY 2.0 | 6 | Staking incentives, boost target |
| 924268058 | FRY 1.0 | 6 | Legacy (deprecated, conversion tracking only) |
| 48968653 | vFRY | 6 | FRY equivalent on Voi chain |

---

## 18d. INTERFACE CONTRACTS

Exact API/data shapes the new system must honor to be plug-and-play.

### Dashboard expects from reward backend

**GET reward summary** (used by `useRewardSummary` hook):
```typescript
// Request: POST /api/rewards/get-reward-summary
{ miner_key: string }

// Response:
{
  pending: number,        // Sum of 'pending' daily entries
  claimable: number,      // Sum of 'claimable' entries
  claimed: number,        // Sum of 'claimed' entries
  accruing: number,       // Sum of 'accruing' entries (current week)
  nextUnlockAt: string,   // ISO date of next Friday 00:05 UTC
  firstRewardAt: string,  // ISO date of first reward
  daily_rewards: [{
    date: string,           // "YYYY-MM-DD"
    amount: number,
    status: "accruing" | "pending" | "claimable" | "claimed",
    asset_id: string,       // ASA ID
    reward_number: number,
    poc_reward_factor: number,  // 0-1
    poc_slots_valid: number,
    poc_slots_total: number,
    poc_category: "INSTALLER" | "NON_INSTALLER"
  }],
  weekly_rewards: [{
    week_start: string,     // ISO date
    unlock_at: string,      // ISO date (Friday 00:05 UTC)
    status: "pending" | "claimable" | "claimed",
    amount: number,
    asset_id: string,
    reward_number: number
  }]
}
```

**Batch summary** (used by `useRewardSummaryBatch` hook):
```typescript
// Request: POST /api/rewards/get-reward-summary-batch
{ miner_keys: string[] }  // Up to 200

// Response: same shape as above, per device
{ summaries: { [miner_key: string]: RewardSummary } }
```

**Claim endpoint**:
```typescript
// Request: POST /api/rewards/claim
{
  miner_key: string,
  no?: number,          // Specific reward number (omit for claim-all)
  preview?: boolean     // True for preview only
}

// Response:
{
  txId: string,         // Algorand transaction ID
  total_claimable: number,
  total_claimed: number,
  fee_amount: number,   // 30% deducted
  net_amount: number    // 70% sent to user
}
```

### Admin panel writes to products collection

```typescript
// POST /api/edit-product
{
  productId: string,             // products._id
  unverifiedReward: number,      // Daily reward (no stake)
  verifiedReward: number,        // Daily reward (with stake, typically 3x)
  register_token: string,        // ASA ID for registration token
  register_price: number,        // Registration stake amount (50 fNODE typical)
  node_token: string,            // ASA ID for node stake token
  node_price: number,            // Node stake amount
  stake_one: number,             // Verification stake tier 1
  stake_two: number,             // Verification stake tier 2
  stake_one_usd: number,         // FIP-012: USD equivalent
  stake_two_usd: number,
  stake_token: string,           // ASA ID for staking
  reward_token: string           // ASA ID for reward token
}
```

### hardwareapi PoC data (activity reporting interface)

```typescript
// PUT /PoC/{miner_key}/hardware
// Payload: HardwareDocument
{
  document: {
    miner_key: string,
    lastUpdated: string,          // ISO date
    software: {
      poc_version_installed: string
    },
    // ... hardware-specific fields
    reward_eligible?: boolean     // Set by hardwareapi for INSTALLER only
  }
}
```

### Verification staking interface

```typescript
// devices collection document
{
  miner_key: string,
  address: string,                // Owner wallet
  reward_wallet: string,          // Reward destination wallet
  verified: boolean,              // Verification status
  isRegistered: boolean,
  staked: {
    type: "one" | "two",          // Stake tier
    amount: number,
    txId: string,
    time: string,                 // ISO date
    rewarded_time: string
  }
}
```

---

## 18e. THINGS TO DISABLE

### Node Staking — disable without breaking other things

| Location | File | What to Disable |
|----------|------|----------------|
| Dashboard API | `/home/helpdesk/subdomains/dashb/pages/api/stake/node-staking.ts` | Return 503/disabled response |
| Dashboard API | `/home/helpdesk/subdomains/dashb/pages/api/stake/n-withdraw.ts` | Return 503 (OR keep for existing stake withdrawals) |
| Dashboard UI | `components/modals/` — node staking components | Hide/disable UI elements |
| Admin panel | `pages/api/edit-product.ts` — node_price/node_token fields | Make read-only or hide |
| Products collection | `products.reward.stake.node` field | Set to 0 for all products |

**Safe approach**: Add nginx 503 block for `/api/stake/node-staking` (same pattern as disabled claim endpoints). Keep `/api/stake/n-withdraw` active for users to unstake existing positions.

### Registration Staking — disable without breaking other things

| Location | File | What to Disable |
|----------|------|----------------|
| Dashboard API | `/home/helpdesk/subdomains/dashb/pages/api/stake/registration.ts` | Return 503/disabled |
| Dashboard API | `/home/helpdesk/subdomains/dashb/pages/api/stake/r-withdraw.ts` | Keep for existing withdrawals |
| Dashboard UI | Registration stake components | Hide/disable |
| Products collection | `products.reward.stake.register` field | Set to 0 |

**Safe approach**: nginx 503 block for `/api/stake/registration`. Keep withdrawal endpoint.

### IoT Miner Reward Path — what happens if not wired up

| Component | If Not Wired | Impact |
|-----------|-------------|--------|
| IoT pipeline (`/home/fry/iot-slot-pipeline/`) | Continues running, writes PoC data to MongoDB | No impact — data just accumulates unused |
| NON_INSTALLER devices in dbrewards | dbrewards stops → no rewards calculated | IoT devices earn nothing (already unrewarded per current system) |
| `poc_reward_dailies` for NON_INSTALLER | Stops accumulating | Historical data preserved |

**Safe**: IoT path is already effectively unrewarded. Not wiring it up just means the 10-min pipeline writes go nowhere. No user-visible change.

---

## 18f. CURRENT REWARD FORMULA

### Base Reward Determination

```
amount = device.verified ? product.reward.verified : product.reward.unverified
```

Typical values:
- Verified AEM: 990 tFRY/day
- Unverified AEM: 990 * 0.5 = 495 tFRY/day (inferred from 0.5x multiplier)
- Verified SDN/RDN/SVN/CN: 119.04 fNODE/day
- Verified HWM/IDM/ODM/ISM/OSM/BM: 59.52 tFRY/day
- Verified LWM: 22.89 tFRY/day

### PoC Pro-Rating (slot-based)

```
poc_reward_factor = slots_valid / slots_total
```
- Typical range: 0.6–1.0
- `slots_valid`: Number of PoC slots where device passed validation
- `slots_total`: Total expected slots in period (typically 144/day = 10-min intervals)

### Final Daily Amount

```
daily_amount = base_amount * poc_reward_factor
```

Example: AEM verified, 128/144 slots valid:
```
990 * (128/144) = 990 * 0.8889 = 880.0 tFRY
```

### INSTALLER Eligibility (hardwareapi 3-gate)

All three must pass for `reward_eligible = true`:

| Gate | Check | Config |
|------|-------|--------|
| Cutoff | `now >= POC_LIVENESS_CUTOFF_DATE` | `2026-04-20T00:00:00Z` |
| Version | `poc_version_installed == poc_version_needed` | Per miner_code in versions collection |
| Liveness | `0 <= age(lastUpdated) <= POC_LIVENESS_STALENESS_SECONDS` | `86400` (24 hours) |

If `reward_eligible === false` → device gets **ZERO rewards** for that cycle.

### Multiplier Tiers

| Condition | Multiplier | Source |
|-----------|-----------|--------|
| FRY 2.0 staked (verification) | 3x | `product.reward.verified` |
| FRY 1.0 staked | 1x | (legacy, conversion path) |
| No stake | 0.5x | `product.reward.unverified` |

### Reward Lifecycle

```
Day 0: recordReward() → status="accruing"
Day 7 (Friday 00:05 UTC): weekly aggregation → status="aggregated" → weekly_rewards[]
Day 30: runDailyMaturationOnce() → status="claimable" (total_pending -= amount, total_claimable += amount)
Claim: dashboard /api/rewards/claim → 30% fee → 70% ASA transfer to reward_wallet
```

### Distribution Frequency

- **Accrual**: Hourly at xx:15 (dbrewards-prod Docker container)
- **Weekly rollup**: Friday 00:05 UTC (dbrewards-weekly container)
- **Maturation**: Daily run flips pending→claimable after 30 days
- **Claim**: On-demand (currently disabled at nginx)

### "Invalid Reward Amount" — Root Cause

**Origin**: dbrewards on <internal-host-ares> (NOT hardwareapi)
**Trigger**: INSTALLER devices with `reward_eligible === false` → dbrewards calculates 0 reward → validation rejects 0 as invalid
**264 affected INSTALLER devices**: Likely failing one of the 3 PoC gates:
- Version mismatch (poc_version_installed ≠ poc_version_needed)
- Stale lastUpdated (>24 hours)
- Before cutoff date (pre-2026-04-20)

### Distribution Mechanism

- dbrewards records rewards to `device-rewards` collection (status tracking)
- **No Algorand signing code in dbrewards** — distribution delegated to dashboard
- Dashboard `claim.ts` constructs ASA transfer from custodial wallet → user's `reward_wallet`
- Custodial wallet: `loadMnemonicAccountPair()` → REWARD_MNEMONIC/REWARD_REKEY
- Transaction: `algosdk.makeAssetTransferTxnWithSuggestedParamsFromObject()` → sign with REWARD_REKEY → submit → wait 4 rounds
- Idempotency: per `{action: 'claim', miner_key, address}`

---

## 18g. HOTWALLET INVENTORY

### Miner Reward Distribution Wallets

| Wallet Reference | Location | ASAs | Purpose | New System Replaces |
|-----------------|----------|------|---------|-------------------|
| `REWARD_MNEMONIC` | <internal-host-heph> `/opt/fry-farm/.env` | tFRY (2681521901), fNODE (2485202024) | Primary reward treasury (rekeyed) | YES — smart contract replaces |
| `REWARD_REKEY` | <internal-host-heph> `/opt/fry-farm/.env` | N/A (signing key only) | Rekey signing authority for REWARD_MNEMONIC | YES — contract-based signing |
| `VOI_REWARD_MNEMONIC` | <internal-host-heph> `/opt/fry-farm/.env` | vFRY (48968653) | Voi chain reward distribution | YES — if Voi rewards continue |
| `AUTOMATION_MNEMONIC` | <internal-host-heph> `/opt/fry-farm/.env` | ALGO | DistPoolV2 epoch automation + fee distribution | MAYBE — depends on if fee pipeline stays |

### Dashboard Custodial Signing

| Function | File | Purpose |
|----------|------|---------|
| `loadMnemonicAccountPair()` | dashb claim.ts | Loads REWARD_MNEMONIC + REWARD_REKEY pair |
| `signAndSubmitCustodialTransactions()` | dashb claim.ts | Signs + submits ASA transfers |

### What the New Smart Contract Replaces

Currently: Treasury wallet holds tFRY/fNODE → dashboard constructs transfer → signs with REWARD_REKEY → sends to user

New system: Smart contract holds tFRY/fNODE → user calls claim method → contract validates eligibility → contract sends directly

Eliminates:
- Custodial wallet risk (mnemonic in .env)
- Backend signing code
- Idempotency/locking complexity
- Manual fee deduction (contract enforces)

---

## CONSOLIDATED: What Stays vs What Goes vs What Interfaces

### STAYS UNTOUCHED
- fry.farm daily claim system (capped-hybrid, anti-sybil) — completely separate
- fry.farm staking pools (FryStakingV2/V3) — used for FRY staking, not miner rewards
- DistPoolV2 fee distribution — feeds staker rewards from fees, not miner rewards
- FryGovernance — governance system
- P2P swap contracts
- hardwareapi device registration endpoints (non-reward)
- hardwareapi measurements system
- All merchandise products (0 reward)

### GOES (archive then remove)
- dbrewards Docker stack on <internal-host-ares>
- dbrewards source code and all env config
- Dashboard reward API routes (`/api/rewards/*`, `/api/fee/*`)
- Dashboard reward UI components (Claim, Withdraw, Boost modals)
- Dashboard reward hooks (useRewardSummary*)
- Admin panel reward config routes and UI
- MongoDB: `device-rewards`, `poc_reward_dailies`, `rewards_archived`, `reward-boosts`, `counters`, `rewards_job_runs`
- Custodial wallet signing code
- nginx claim disable block (replaced by new contract flow)

### INTERFACES (new system must connect to)
- `main.devices` collection → user identity, wallet, verification status
- `main.products` collection → device type config, reward rates
- `PoC.hardware` collection → device activity, reward_eligible flag
- `PoC.versions` collection → version requirements
- hardwareapi PoC endpoint → writes activity data
- IoT pipeline → writes NON_INSTALLER activity (if IoT ever gets rewards)
- <internal-host-atlas> algod → Algorand mainnet queries + txn submission
- Verification staking → multiplier determination
- Dashboard UI → must display new contract-based reward data
