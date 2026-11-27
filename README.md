# indexd Smart Autopilot

English | [Українська](README_UA.md) | [Русский](README_RU.md)

**indexd Smart Autopilot** is an experimental evolution of the Sia indexer that continues the Smart Autopilot work started in `renterd`.

If a host develops a problem (an overpriced host, unavailability, insufficient collateral), the standard logic withdraws the contract and moves its data to other hosts, and moving data costs money. Smart Autopilot puts such a contract **on pause** and moves the data only when a replacement is economically justified (details in the "Core idea" section below).

In this architecture `indexd` provides placement policies, host selection, admission rules, host scores and the contract lifecycle management (above all pauses), while `s3d` receives these policies and applies them when transferring data. In addition, `indexd` can send hosts short encrypted signals on the blockchain (the reason for a pause, the host's own scores).

The goal is not to replace the standard mechanisms with an arbitrary algorithm, but to add a configurable intelligent layer for managing the host portfolio and data placement.

> **Status: release candidate.** Tested on a single Windows node. Always back up the database before updating and read the release notes: in `Smart Active` mode the autopilot carries out its plan. Sending signals to hosts and the host work share report are **off by default**. Settings and the database schema may change before the final release.

## Core idea: pause instead of migration

**A contract that has a problem does not have to be withdrawn and moved immediately. Smart Autopilot puts it on pause and moves the data only when that is really justified.**

### The problem

In the standard logic, a host that went beyond the allowed prices, became unavailable or fails the checks leads to the withdrawal of the contract and the move of its data to other hosts. Moving costs money: the data has to be read from the remaining hosts, reconstructed, written again, and the transfer and storage paid at the prices of the new hosts. A price jump or a short outage can start such moves on several hosts at once, and the expense becomes out of proportion to the problem that started it.

### What a pause does

- A contract on pause **receives no new data**, and the data already on the host **stays in place**: no move is started just because of a price change or a temporary failure.
- A contract on pause **occupies a slot** of the portfolio, so the system does not create extra contracts when several hosts go on pause at the same time.
- A replacement is made **only when it is justified**: the cost of keeping is compared with the cost of moving. If it is not justified, the contract stays on an "economic pause".
- A replacement is safe: **the new contract is formed first, and only then the old host is released**; one host is released at a time.
- Before any data is withdrawn, it is checked that enough of it remains for recovery (Reed-Solomon).
- The host can be **told the reason for the pause** by a `0x10` signal on the blockchain (if it agreed to receive signals).

### How it looks

```text
Good
  → Pause (price / availability / collateral / economics)
      → the problem went away                       → Good again, no move was needed
      → a candidate is found, replacement justified → new contract → then the old host is released
      → replacement not justified                   → stays on the economic pause
      → pause timeout                               → replacement by the slot rules (N+2)
```

### Reasons for a pause

1. **Prices above the allowed** (Smart Gouging Pause).
2. **Unavailability** of the host (low uptime, `Host grace period`).
3. **Insufficient collateral**.
4. **Economic reason**: a better candidate exists, but moving the data still costs more than keeping it.

### What is configurable

- Autopilot Mode: only `Smart Active` starts pauses; `Smart Shadow` only writes to the log what the autopilot would have done; pauses that have already started are served in all modes.
- `Keep Gouging Contract Until Replacement` and `Allow Existing Data on Overpriced Contracts`.
- The pause timeout and `Host grace period`.
- The replacement threshold (15% on Economics) on the **Scoring** page.

Where to see the result: `smart-autopilot.log`, alerts, the Scoring page, the `Paused` column and the contract panel on the Contracts page (the reason for the pause). Detailed description: "Contract pauses" in "Implemented in this release candidate".

## Relation to Smart Autopilot for renterd

The project continues the same line of development and keeps the main principles laid down in `renterd`.

- **A profile instead of one magic Score.** A host should be evaluated on separate dimensions so that strength in one does not hide weakness in another.
- **Seven score groups:** reliability, performance, resources, economics, risk/collateral, technical compatibility and network independence.
- **Measurements separate from policy.** The actual characteristics of a host should be kept apart from user-defined thresholds, weights and decision rules.
- **Host portfolio management.** What matters is not only the score of a single host but also its effect on the reliability and independence of the placement as a whole.
- **Safe lifecycle management.** A problem contract is first put on pause, and replacement and data migration are performed only when justified and always with the data kept available.
- **Transparency and configurability.** Decisions have clear reasons, and users keep control over policies.
- **Gradual rollout.** New decision mechanisms are verified separately from the standard behavior before they become active.

The seven-group model and full portfolio management remain architectural goals. This release candidate implements the working core: groups **G2–G5**, their aggregate **Smart Overall** and the portfolio engine ported from `renterd`, all controlled by a single **Autopilot Mode** switch (`Original`, `Smart Shadow`, `Smart Active`). Groups G1, G6 and G7 are for the future.

## Current version

- Release: **v0.4.1-rc.1**
- Build version: `v0.4.1-rc.1` (commit `0c5dd34c`)
- Base: `indexd v0.6.5`
- Fork migrations: up to **#25** (upstream schema: `32`, was `28`)
- Web UI: `feat/indexd-ui-econ-e` (commit `faad444a`)
- Companion products: **s3d v0.3.2-rc.1** (transfer summaries), **hostd-extended v0.2.0-alpha.1** (receives signals)
- Previous release: v0.4.0-rc.1

This is an independent fork, not an official Sia Foundation release.

## What's new in v0.4.1-rc.1

- **Merge with upstream `v0.6.5`** (v0.6.2–v0.6.5): fixes for funding application accounts on hosts, shared spaces and quotas, contract checks. **With the same `fund_target_bytes` setting the funding target is now twice as large** (every host in the pool holds twice as many funds; nothing extra is taken from the wallet); to get the previous size, halve `fund_target_bytes`.
- **Database migrations: upstream schema 28 → 32.** Rolling back to the previous exe is possible only by restoring a dump.
- **Economics E.** The storage price is corrected by the collateral multiple, and the scale is anchored to the 10th percentile of the cost in the pool (see G4 below).
- **New contract replacement rule** (the worst contract by Economics, see "Portfolio plan and execution").
- **Candidate pause after a failed contract formation** and a common candidate selection using the same conditions as execution.
- **Host card fix:** "Max Collateral" showed the collateral price instead of the per-contract collateral limit (always `$0.00`); it now shows `settings.maxCollateral`, and the collateral price has its own "Collateral" row.

Details, the list of files with hashes and the update procedure: the release notes [Release indexD Smart Autopilot v0.4.1-rc.1](https://github.com/MainQuestion/indexd-smart-autopilot/releases/tag/v0.4.1-rc.1).

## Implemented in this release candidate

### Autopilot Mode

One switch controls the whole Smart layer.

- `Original`: standard indexd behavior. The default.
- `Smart Shadow`: Original keeps working; Smart Autopilot only computes decisions and writes them to `smart-autopilot.log`.
- `Smart Active`: Smart Autopilot manages the contract portfolio and places new uploads by Smart Overall.

API values: `original`, `shadow` and `active` (`/settings/smart`). Pauses that have already started are served in all modes.

### Contract pauses

Pauses are the main mechanism that prevents unnecessary data migrations (see "Core idea"). They are controlled by Autopilot Mode: `Smart Active` starts new pauses, while `Original` and `Smart Shadow` keep the standard decision logic (`Smart Shadow` additionally records the decisions for analysis).

#### Price pause (Smart Gouging Pause)

A pause by the price policy can leave the data on the host instead of immediately starting a migration just because of a price change.

- `Keep Gouging Contract Until Replacement` and `Allow Existing Data on Overpriced Contracts`.
- A configurable pause timeout.
- Checks of the collateral-to-price ratio and safe recovery that takes Reed-Solomon into account.
- `smart-autopilot.log`, alerts, the Scoring page and the `Paused` column.

#### Availability pause

A host with low availability (uptime) falls into the same pause mechanism as Smart Gouging Pause, using `Host grace period`.

#### Economic pause

If a replacement is not yet justified (moving the data still costs more than keeping it), the contract gets an **economic pause** (one at a time, with an alert). It is lifted when nothing else fits or Smart leaves Active. More about the replacement rule: "Portfolio plan and execution".

#### Slots and replacement of hosts on pause

- Contracts of hosts in a Smart pause **occupy slots**: Good + Paused, except a pause for which a replacement already exists.
- A host on pause is replaced only when Good is below N+2, where N is the number of shards of the redundancy scheme: never at N+2 or more, immediately below N, in between after the pause timeout; at most one replacement per pass, only in `Smart Active`.
- This keeps the system from forming extra contracts when several hosts go on pause at the same time because of overpriced settings. The candidate preview uses the same rules.

#### Signal to the host

When a pause starts, a host that has agreed can be sent a `0x10` signal with the reason (see "Signals to hosts").

### Signals to hosts (new)

`indexd` can tell a host something the host cannot see itself. A signal is a short encrypted record sent **on the Sia blockchain**; it does not trigger any automatic action on the host side and is only shown to the operator.

- **`0x10` — pause.** Sent when a Smart pause **begins** (only in `Smart Active`) with a reason: prices above the allowed limits, unavailability, insufficient collateral, an economic reason.
- **`0x20` — scores.** The host's four own scores (Performance, Economics, Resources, Risk) and the number of days in the pool. The message **contains no data about other hosts or about the pool** (no share, no rank, no number of hosts).
- **Only to hosts that opted in.** The host operator publishes a `signals/v0` statement on the blockchain (done by hostd-extended); a withdrawal or a block rollback removes the consent. Nothing is sent to a host without consent.
- **Delivery.** The queue is checked every 10 minutes; events younger than 1 hour, one per host and signal type, go out in **a single v2 transaction** with a single `ArbitraryData` field (`SIG0` and fixed-size records of 88 bytes each). The only cost is the transaction fee (about 0.011 SC for one record on mainnet).
- **Protection.** Records are encrypted and authenticated (ed25519 → X25519, HKDF-SHA256, XChaCha20-Poly1305) and addressed with a tag that changes in every batch, so an observer cannot see which hosts received records or what they contain. A host accepts at most 12 signals per hour from one renter and only with a strictly increasing sequence number.
- **Control.** A single **Send signals automatically** switch in the **Scoring** window enables both signals (**off by default**) and shows how many hosts consented and the last send. `POST /settings/signals/test-share-message` sends a test `0x20` record to one consenting host (administrator only; intended for verification in the release candidate and may be removed in the final release).
- **Where the results are visible.** The `host-signals.log` journal in the data directory (one `SENT` / `FAILED` line per send with the full host key and the transaction ID, height, sequence number and content; rotation 10 MB × 3), an Info line in the main log, a yellow envelope next to the check mark in the **Accepted** column of the Hosts table, and `GET /settings/signals`.

Code `0x02` is reserved and not used.

### Host work share report (new)

A service report that **affects nothing** (neither Smart Overall, nor the queue, pauses or contracts): it determines which hosts should be sent the informational `0x20` signal.

- The measure is the volume of our data on a host per day in the pool relative to the median of the evaluated hosts; the time in the pool is counted from the root of the contract renewal chain.
- Candidates: the bottom tenth of the evaluated hosts (rounded up, at least one) with a share below 60% of the median; repeated no more often than once in 10 days, after a random delay of up to 6 hours.
- Only eligible hosts are evaluated (usable, not paused, not blocked, with a good contract, at least 2 days old). Skipped if fewer than three hosts are evaluated.
- Files in the data directory: `host-work-share-current.json` and `host-work-share.jsonl` (rotation 20 MB × 5). Settings: `GET/PUT /settings/work-share-report`. **Off by default.**

### Transfer events from clients (new)

Clients (for example, the `s3d` fork) send summaries of transfer attempts per host: attempts, lost races, timeouts, connection failures and **host error responses** (`host_error`). They are stored by the hour (`host_usage_hourly`, `usage_batch_hosts`) and used by the work share report. Client-side errors (a bad request, payment, expired prices) are not counted as host errors.

### Host client fixes (new)

- **Retry on `ErrInactiveConn`.** A connection idle for about an hour is closed by the multiplexer but used to stay in the cache; the next call failed and was recorded as a host failure, so Smart Placement skipped a healthy host for 5–7 minutes. Now the call is retried once on a new connection (only if no bytes were sent in the attempt), and the failure is not recorded if the retry succeeded.
- **Quiet contract formation log.** The lines "hosts outside the allowlist" and "filtered new contract candidates" are written at Info when the numbers change and no more often than once an hour, otherwise at Debug.

### Smart Autopilot host score groups (G2–G5)

Scores are computed and shown in all modes. They feed Smart Overall and, in `Smart Active`, drive the portfolio decisions described below. In `Original` and `Smart Shadow` they do not affect host selection, contract formation, pauses or migrations.

Scores are computed for the *evaluation set*: pool hosts that pass the price and location filters, plus hosts with an active contract. Other hosts have no score and are shown as `—`. Scores are recomputed every 10 minutes and immediately after the relevant price, location or host list settings change.

#### Performance (G2)

- Real upload and download measurements from s3d.
- A 10-day observation window; if a host has no samples in the window, its last score is kept.
- Scored relative to the 95th percentile of the evaluation set.
- Measurements arrive through `POST /hosts/speed`.
- The Hosts page shows the score and the raw Mbit/s values.

#### Resources (G3)

Seven signals are used:

- data owned by the renter;
- free space;
- other people's data;
- financial exposure;
- approximate recovery compensation;
- remaining contract term;
- share of lost sectors.

The signals are combined into three blocks and linearly normalized over the range of the evaluation set.

#### Economics (G4)

Economics is calculated from a configurable usage profile (storage, upload, download).

The host's storage price is **corrected by the collateral**: the collateral multiple score (the ratio of the collateral to the storage price, compared with the pool; the reference is the 95th percentile) from 1 to 10 leaves the price as it is at 10 and makes storage ten times as expensive at 1. The final Economics score is the score of the profile cost at the corrected prices; the collateral score is shown separately.

The cost scale is anchored to the **10th percentile** of the corrected total cost among pool hosts: a host whose price is not above this anchor gets 10. The calculation includes pool hosts and hosts that already have contracts.

Defaults: **2 TB storage / 2 TB upload / 0.5 TB download per month, `k = 9`**. The upload cost includes temporary storage of the sectors for 3 days.

#### Risk (G5)

Risk evaluates price instability from history. Price changes between consecutive scans become events after a set threshold; for ingress the threshold is higher, and collateral events are based on decreases. Price movement is compared with the opposite movement of SC/USD.

The default window is 14 days and can be configured on Scoring (`0` means 14, maximum 120). A host with 0 or 1 price points gets a score of 10.

### Smart Overall

- A weighted sum of Performance, Resources, Economics and Risk (weights 25/25/25/25, total 100) and the replacement threshold (15 %, applies to Economics) are configured on the Scoring page.
- A host with fewer than 10 speed samples gets a temporary Performance score for this calculation (the median of hosts that have contracts and enough samples).
- `GET /hosts` returns `smartOverall` and `performanceImputed`; the Hosts table has a Smart Overall column.

### Portfolio plan and execution

The decision engine was ported from `renterd`.

- The plan is computed in `Smart Shadow` and `Smart Active` and logged; `GET /smart/portfolio-plan` returns the last plan and the execution results.
- **ADD** forms new contracts in a canonical order (Economics, Risk on an exact tie) instead of a random one; the placement radius remains a grouping.
- **REPLACE** (no more than once every 4 hours, one replacement per window): the worst contract is the one with status Good and the lowest **Economics** (on a tie, the lower Smart Overall, then the public key). A candidate qualifies if its Economics is **strictly greater** than `worst Economics × (1 + threshold/100)`, its Smart Overall is not lower than the worst's, it is not `TooClose` and it belongs to the pool of allowed candidates; the largest `(Economics + Smart Overall) / 2` is chosen. The replacement is carried out when the migration is economically justified (a contract without data counts as justified). The new contract is formed first; only then is the old host released.
- If a replacement is not yet justified, the contract gets an **economic pause** (one at a time, with an alert), which is lifted when nothing else qualifies or Smart leaves Active (see "Contract pauses").
- Only one replacement is released at a time; the release is considered complete when none of our sectors remain on the host.

### Candidate pause after a failed formation

This is not a contract pause but a temporary exclusion of a host from the candidates for new contracts.

- After a failure **caused by the host**, the host is excluded from add and replacement candidates for **24 hours**; each next consecutive failure doubles the period, up to **7 days**. A success resets the record.
- Not counted as a host fault: insufficient wallet funds, client-side RPC errors, cancellation, skipped due to conditions, state races.
- The pause is kept in memory: restarting indexd clears it. It works only in `Smart Active`. Log line: `smart candidate cooldown`.
- Candidates are selected by **the same conditions as execution**: host usability, not on a Smart pause, not blocked, allowed by the allowlist, protocol version not below the minimum, remaining space of at least 10 GB, prices within tolerance.

### Upload placement and migrations

- In `Smart Active`, new sectors and indexd migrator targets go to the hosts with the highest Smart Overall, strictly in that order (random on a tie); a failing host is skipped, a host without a score goes last. Reads are unchanged.
- `GET /hosts` for applications contains `smartOverall` and `smartActive`; old clients ignore them.
- In `Smart Shadow` the previous order applies and the Smart order is logged. `s3d` writes its own `smart-autopilot.log`.

### Host spending accounting

- The `host_spending_daily` table tracks money by category: contract, storage, upload, download, audit, deletion; account funding is shown separately as "Funded".
- Sources: the server, the migrator and `s3d` (it reports the volume of every successful transfer through `POST /hosts/usage`; the cost is estimated from the host's prices and marked with `~`).
- `GET /host/{hostkey}/spending` returns totals (all time and the last 30 days). The history starts with the first run of this build.

### Scoring history

The host's Storage, Ingress, Egress and Collateral prices from successful scans, scan results and the SC/USD rate are stored every 30 minutes for 120 days and cleaned up daily.

### Hosts and Contracts interface

- **Hosts.** One view switch `Candidates | Usable + contracts | All`; by default hosts with a contract or on a Smart pause are shown even if they are not usable.
- **Hosts.** Usable shows a single check mark (failed checks in the tooltip); price icons turn yellow when the price is within the limit but above the contract formation limit; a sortable **Accepted** column shows hosts with an active good contract, and next to the check mark a yellow envelope marks hosts that accept signals.
- **Hosts.** A **View** menu, collapsible Performance, Economics, Resources and Risk groups (short labels Perf, Econ, Reso, Risk), sorting by scores and parameters; column widths follow the content, and the page renders noticeably faster.
- **Host card.** In the "Pricing & Limits" section **Max Collateral** shows the host's collateral limit per contract (indexd fits the contract size to it), and the **Collateral** row shows the collateral price per TB per month.
- **Contracts.** The columns **My Data**, **Σ spent**, **Σ storage**, **Σ ingress**, **Σ egress** and **Σ collateral**; a View menu; a panel with the data on the host, spending by category and source, transfers, scores and the pause reason; the contract of a paused host shows **Pause**.
- **Scoring.** The **Signals to hosts** block with the **Send signals automatically** switch.
- Muted labels and the world map are brighter, and log lines that used to repeat every two minutes are written only when the state changes.

### Placement policies

- A configurable price tolerance for new contracts: 80–95%.
- A configurable minimum distance between hosts; `0` disables the restriction.
- A placement radius relative to a reference point: an IP address or coordinates.
- The `prefer` mode first selects candidates inside the radius, then the nearest outside it.
- The `strict` mode allows only candidates inside the radius.

### Allowlist and blocking

- A soft allowlist directs new contracts and new data to the allowed hosts when it is set.
- Existing contracts outside the allowlist stay readable; no new data is placed on them, and removal happens gradually as the contracts expire.
- The reasons for manual and automatic blocking are stored separately.

### Redundancy policy and API

- A custom erasure coding policy is set as a K/N scheme and passed to clients through `GET /account`.
- The current model can relax the standard threshold but cannot make it stricter.
- `GET /hosts` returns `performance`, `economics`, `resources`, `risk`, `smartOverall` and `hasContract` for hosts of the evaluation set; its `involved` parameter selects usable hosts plus hosts with a contract or a pause. It also returns `acceptsSignals` for every host.
- `POST /hosts/speed` and `POST /hosts/usage` accept measurements and transfer summaries from a compatible client.
- `GET /smart/portfolio-plan` and `GET /host/{hostkey}/spending`.
- `GET/PUT /settings/signals` and `GET/PUT /settings/work-share-report`.
- The Economics profile, the Risk window, the Smart Overall weights and the replacement threshold are available through `/settings/smart`.

### Candidate preview

`GET /hosts/candidates` and the `Candidates` view show candidates for new contracts under the current settings, including ranks, distances and exclusion reasons. The preview uses the same selection mechanism and the same slot rules as real contract formation.

### Data transfer and limits

- Configurable upload and download limits for migration.
- Client transfer limits set through quotas and accounts are passed through `GET /account`; a compatible client, in particular `s3d Smart Autopilot`, must honor them.
- The build version is shown in the logs, `indexd version`, `GET /api/state` and the web interface.

## Known limitations

- The work share report and the host messages are designed for small pools; on a pool of a few hosts the results are noisy.
- The admin API listens on all addresses by default. Set a strong administrator password and restrict access with a firewall.
- Any funding RPC error (for example, a hostd restart) puts the host on an account funding pause (not a contract pause) of **up to 2 hours**; do not restart hostd or indexd during a large upload. The upstream funding metrics consider accounts "ready" once funded on 30 hosts and are not suitable as an "account is empty" alert for a pool of 5 hosts.
- The database schema changed (28 → 32): take a dump before updating; rollback only by restoring the dump.
- The Blockchain node page does not show connected peers, as in the official indexd (there is no endpoint that lists peers).

## Further development

Under discussion.

## Upstream

- [SiaFoundation/indexd](https://github.com/SiaFoundation/indexd)

## License

The project is based on [SiaFoundation/indexd](https://github.com/SiaFoundation/indexd) and continues the Smart Autopilot concept originally researched and implemented in `renterd`.

The project is distributed under the MIT license. The Sia Foundation copyright notice and license terms are preserved in [LICENSE](LICENSE).

This is an independent fork, not an official Sia Foundation release.
