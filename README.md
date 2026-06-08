# Miner Reward System v1 — Archive

**Archived**: 2026-06-08
**Reason**: Replaced by smart-contract-based reward system v2
**Status**: Code remains on production servers as reference. No code was deleted.

---

## Tagged Source Repositories

Each source repo is tagged at the commit deployed on production at archive time.

| Repo | Tag | Commit | Host | Purpose |
|------|-----|--------|------|---------|
| [Fry-Foundation/dbRewards](https://github.com/Fry-Foundation/dbRewards/tree/v1-archive-20260608) | `v1-archive-20260608` | `6bed8d49` | ARES00 | Core reward calculation + distribution engine |
| [Fry-Foundation/hardwareapi-poc](https://github.com/Fry-Foundation/hardwareapi-poc/tree/v1-archive-20260608) | `v1-archive-20260608` | `241bbe54` | ZEUS00 | PoC eligibility gate + device activity tracking |
| [Fry-Foundation/registration_portal](https://github.com/Fry-Foundation/registration_portal/tree/v1-archive-20260608) | `v1-archive-20260608` | `b222158d` | HERMES00 | Dashboard reward UI + claim API |
| [Fry-Foundation/admin_panel](https://github.com/Fry-Foundation/admin_panel/tree/v1-archive-20260608) | `v1-archive-20260608` | `0c17aa27` | HERMES00 | Admin panel reward configuration |

### Git State at Archive Time

- **dbRewards**: Deployed commit `6bed8d49` is on branch `feat/virtual-mining-phase1`, ahead of default branch HEAD `7396edcc`. Tag captures the deployed state.
- **hardwareapi-poc**: Deployed = GitHub HEAD. Clean.
- **registration_portal**: Deployed = GitHub HEAD. Clean.
- **admin_panel**: Deployed = GitHub HEAD. Clean.

---

## MongoDB Collection Archive

All collections from the `main` database on ARES00. BSON (full) + JSON (100-doc samples).

| Collection | Documents | BSON Size | Purpose |
|------------|-----------|-----------|---------|
| `device-rewards` | 11,185 | 713 MB | Primary reward ledger per device |
| `poc_reward_dailies` | 494,419 | 128 MB | Daily PoC slot validation data |
| `rewards_archived` | 2,181,893 | 493 MB | Legacy pre-aggregation daily records |
| `reward-boosts` | 17,216 | 5.8 MB | Active boost/multiplier records |
| `counters` | 9,804 | 748 KB | Sequence counters (reward_number) |
| `rewards_job_runs` | 30 | 85 KB | Batch distribution job tracking |
| `products` | 42 | 14 KB | Device type config (FULL dump) |
| `devices` | 19,926 | 13 MB | Device registry (shared reference) |

### BSON Full Dumps

Full BSON dumps (1.4 GB) exceed GitHub file limits. Available at:
- **ARES00**: `/home/fry/backups/reward-archive-20260608/bson/`
- **GitHub Release**: attached as `reward-archive-20260608.tar.gz`

This repo contains JSON samples (100 docs each) and BSON metadata (index definitions).

### Restore from BSON

```bash
mongorestore --ssl --tlsInsecure -u <admin> -p <pass> --authenticationDatabase admin \
  --db main <path-to-bson-dir>/
```

---

## Reward Formula (v1)

```
daily_amount = base_reward * poc_reward_factor

where:
  base_reward = product.reward.verified   (if device has verification stake, ~3x)
              | product.reward.unverified  (if no stake, ~0.5x)
  poc_reward_factor = slots_valid / slots_total  (typically 0.6-1.0)
```

### Example

AEM verified, 128/144 slots valid: `990 tFRY * (128/144) = 880.0 tFRY/day`

### INSTALLER Eligibility (hardwareapi 3-gate)

All must pass for `reward_eligible = true`:

| Gate | Check | Fail = zero reward |
|------|-------|-------------------|
| Cutoff | `now >= 2026-04-20T00:00:00Z` | Yes |
| Version | `poc_version_installed == poc_version_needed` | Yes |
| Liveness | `age(lastUpdated) <= 24h` | Yes |

---

## Distribution Chain

```
hardwareapi (ZEUS00)       writes reward_eligible flag to PoC.hardware
       |
dbrewards (ARES00)         reads flag, calculates reward, records to device-rewards
  - Hourly at xx:15        (accrual)
  - Friday 00:05 UTC       (weekly rollup)
  - Daily                  (30-day maturation: pending -> claimable)
       |
dashboard (HERMES00)       claim API signs custodial ASA transfer
  - 30% fee instant claim
  - Free after 1 month
  - Custodial: REWARD_MNEMONIC / REWARD_REKEY
       |
user receives 70% in tFRY or fNODE to their reward_wallet
```

---

## Token ASAs

| ASA ID | Symbol | Role |
|--------|--------|------|
| 2681521901 | tFRY | Primary miner reward (most device types) |
| 2485202024 | fNODE | Node reward (RDN/SDN/SVN/CN/AEM/virtual) |
| 2485314946 | FRY 2.0 | Staking incentives, boost target |
| 924268058 | FRY 1.0 | Legacy (deprecated) |

---

## Hotwallet Inventory (replaced by smart contracts in v2)

| Reference | Location | Purpose |
|-----------|----------|---------|
| `REWARD_MNEMONIC` | HEPH00 /opt/fry-farm/.env | Primary reward treasury (rekeyed) |
| `REWARD_REKEY` | HEPH00 /opt/fry-farm/.env | Signing authority for treasury |
| `VOI_REWARD_MNEMONIC` | HEPH00 /opt/fry-farm/.env | Voi chain reward distribution |
| `AUTOMATION_MNEMONIC` | HEPH00 /opt/fry-farm/.env | DistPoolV2 epoch automation |

---

## What Was Disabled (2026-06-08)

| Endpoint | Action | Date |
|----------|--------|------|
| `/api/rewards/claim` | nginx 503 | 2026-04-23 (pre-existing) |
| `/api/rewards/confirm` | nginx 503 | 2026-04-23 (pre-existing) |
| `/api/stake/node-staking` | nginx 503 | 2026-06-08 (v2 migration) |
| `/api/stake/registration` | nginx 503 | 2026-06-08 (v2 migration) |

### Still Active

| Endpoint | Status |
|----------|--------|
| `/api/stake/n-withdraw` | Active (unstake existing node positions) |
| `/api/stake/r-withdraw` | Active (unstake existing registration positions) |
| `/api/stake/verification` | Active (verification staking unchanged) |
| `/api/stake/stake-withdraw` | Active (general stake withdrawal) |

---

## Interface Contracts (what v2 must honor)

See `recon/reward-system-recon.md` section 18d for exact API response shapes, MongoDB document schemas, and data flow diagrams.

Key interfaces:
- `main.devices` -- user identity, wallet, verification status
- `main.products` -- 42 device types with reward rates
- `PoC.hardware` -- device activity + eligibility flag
- ATLAS00 algod (port 4190) -- Algorand mainnet
- Verification staking -- multiplier (3x staked, 0.5x unstaked)

---

## What Stays Untouched

- fry.farm daily claim system (capped-hybrid, anti-sybil) -- completely separate
- FryStakingV2/V3 pools -- FRY staking, not miner rewards
- DistPoolV2 fee distribution -- staker rewards from fees
- FryGovernance -- governance system
- hardwareapi device registration (non-reward endpoints)
- All existing withdrawal endpoints

---

## Full Recon

See `recon/reward-system-recon.md` for the complete dependency map, interface contracts, and architectural documentation.
