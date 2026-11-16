# indexd Smart Autopilot

English | [Українська](README_UA.md) | [Русский](README_RU.md)

**indexd Smart Autopilot** is an experimental evolution of the Sia indexer, continuing the Smart Autopilot work originally developed in `renterd`.

In this architecture, `indexd` provides placement policies, host selection, admission rules, host scores, and part of contract lifecycle management, while `s3d` receives these policies and applies them during data transfers. `indexd` can also send hosts short encrypted signals on the blockchain (a pause reason, the host's own scores).

The goal is not to replace standard mechanisms with an arbitrary algorithm, but to add a configurable intelligent layer for host-portfolio management and data placement.

> **Status: release candidate.** Tested on one Windows node. Always back up the database before upgrading and read the release notes: in `Smart Active` mode the autopilot executes its plan. Sending signals to hosts and the host work share report are **off by default**. Settings and database schema may still change before the final release.

## Relation to renterd Smart Autopilot

The project continues the same development direction and preserves the core principles established in `renterd`.

- **A profile instead of one magical Score.** A host should be evaluated across separate dimensions so strength in one area does not hide weakness in another.
- **Seven scoring groups:** reliability, performance, resources, economics, risk/collateral, technical compatibility, and network independence.
- **Measurements separate from policy.** Actual host characteristics should be separated from user-defined thresholds, weights, and decision rules.
- **Host-portfolio management.** What matters is not only an individual host's rating, but also its effect on the reliability and independence of the overall placement.
- **Safe lifecycle management.** Pausing, replacement, and migration should preserve data availability and avoid unnecessary data movement.
- **Transparency and configurability.** Decisions should have understandable reasons while users retain control over policies.
- **Gradual adoption.** New decision mechanisms should be validated separately from standard behavior before becoming active.

The seven-group model and full portfolio management remain architectural goals. This release candidate implements the working core: groups **G2–G5**, their aggregate **Smart Overall**, and a portfolio engine ported from `renterd`, all controlled by one **Autopilot Mode** switch (`Original`, `Smart Shadow`, `Smart Active`). Groups G1, G6 and G7 are still future work.

## Current version

- Release: **v0.4.0-rc.1**
- Build version: `v0.4.0-rc.1` (commit `6c9d7593`)
- Base: `indexd v0.6.1` with [#1135](https://github.com/SiaFoundation/indexd/pull/1135)
- Fork migrations: up to **#25** (upstream schema: `28`)
- Web UI: `feat/indexd-ui-signals`
- Companions: **s3d v0.3.1-rc.1** (transfer summaries), **hostd-extended v0.2.0-alpha.1** (receives signals)
- Previous release: v0.3.0-rc.1

This is an independent fork and is not an official Sia Foundation release.

## Implemented in this release candidate

### Autopilot Mode

One switch controls the whole Smart layer.

- `Original`: the standard indexd behaviour. This is the default.
- `Smart Shadow`: Original keeps running; Smart Autopilot only computes and logs its decisions to `smart-autopilot.log`.
- `Smart Active`: Smart Autopilot manages the contract portfolio and places new uploads by Smart Overall.

API values are `original`, `shadow` and `active` (`/settings/smart`). Pauses that have already started are served in every mode.

### Signals to hosts (new)

`indexd` can tell a host what the host cannot see on its own. A signal is a short encrypted record sent **on the Sia blockchain**; it triggers no automatic action on the host side and is only shown to the operator.

- **`0x10` — pause.** Sent when a Smart pause **begins** (in `Smart Active` only), with the reason: prices above the allowed limits, unavailability, insufficient collateral, an economic reason.
- **`0x20` — scores.** The host's own four scores (Performance, Economics, Resources, Risk) and the number of days in the pool. The message carries **no data about other hosts or the pool** (no share, rank or host count).
- **Only to consenting hosts.** The host operator publishes a `signals/v0` attestation on the blockchain (done by hostd-extended); revoking it or a block reorg withdraws the consent. A host without consent is sent nothing.
- **Delivery.** Every 10 minutes the queue is checked; events younger than 1 hour, one per host and signal type, go out in **one v2 transaction** with a single `ArbitraryData` field (`SIG0` plus fixed 88-byte records). The only cost is the transaction fee (about 0.011 SC for one record on mainnet).
- **Protection.** Records are encrypted and authenticated (ed25519 → X25519, HKDF-SHA256, XChaCha20-Poly1305) and addressed by a per-batch rotating tag, so an observer cannot tell which hosts received records or what they contain. The host accepts at most 12 signals per hour per renter and only with a strictly increasing sequence number.
- **Control.** One switch **Send signals automatically** in the **Scoring** window enables both signals (**off by default**) and shows how many hosts accepted and the last send. `POST /settings/signals/test-share-message` sends a `0x20` test record to one consenting host (administrator only; intended for testing in the release candidate and may be removed in the final release).
- **Where to see results.** The `host-signals.log` journal in the data directory (one `SENT` / `FAILED` line per send with the full host key and transaction id, height, sequence number and content; rotation 10 MB × 3), an Info line in the main log, a yellow envelope next to the check mark in the **Accepted** column of the Hosts table, and `GET /settings/signals`.

Code `0x02` is reserved and unused.

### Host work share report (new)

A service report that **affects nothing** (not Smart Overall, the queue, pauses or contracts): it decides which hosts should get an informational `0x20` signal.

- The metric is our data on a host per day in the pool relative to the median of scored hosts; time in the pool is counted from the root of the contract renewal chain.
- Candidates: the bottom tenth of scored hosts (rounded up, at least one) with a share below 60% of the median; repeated no more than once every 10 days, after a random delay of up to 6 hours.
- Only eligible hosts are scored (usable, not paused, not blocked, good contract, at least 2 days). Skipped when fewer than three hosts are scored.
- Files in the data directory: `host-work-share-current.json` and `host-work-share.jsonl` (rotation 20 MB × 5). Settings: `GET/PUT /settings/work-share-report`. **Off by default.**

### Transfer events from clients (new)

Clients (for example, the `s3d` fork) send per-host summaries of transfer attempts: attempts, lost races, timeouts, connection failures and **host error responses** (`host_error`), stored by hour (`host_usage_hourly`, `usage_batch_hosts`) and used by the work share report. Client-side errors (bad request, payment, expired prices) are not counted as host errors.

### Host client fixes (new)

- **Retry on `ErrInactiveConn`.** A connection idle for about an hour is closed by the multiplexer but used to stay in the cache; the next call failed and was recorded as a host failure, so Smart Placement skipped a healthy host for 5–7 minutes. The call is now retried once on a fresh connection (only if no byte was sent in the attempt) and no failure is recorded if the retry succeeds.
- **Quiet contract formation log.** The "hosts outside the allowlist" and "filtered new contract candidates" lines are written at Info when the numbers change and at most once an hour, otherwise at Debug.

### Smart Autopilot host scoring groups (G2-G5)

Scores are calculated and displayed in every mode. They feed Smart Overall and, in `Smart Active`, drive the portfolio decisions described below. In `Original` and `Smart Shadow` they do not affect host selection, contract formation, pauses, or migrations.

Scores are calculated for the *evaluation set*: pool hosts that pass price and location filters plus hosts with an active contract. Other hosts have no score and are shown as `—`. Scores are recalculated every 10 minutes and immediately when relevant price, location, or host-list settings change.

#### Performance (G2)

- Real upload and download measurements from s3d.
- 10-day observation window; when a host has no samples in the window, its last score is kept.
- Scoring relative to the 95th percentile of the evaluation set.
- Measurements arrive through `POST /hosts/speed`.
- The Hosts page shows the score and raw Mbps values.

#### Resources (G3)

Seven signals are used:

- renter-owned data;
- free space;
- foreign data;
- financial exposure;
- estimated recovery compensation;
- remaining contract term;
- lost-sector share.

The signals are combined into three blocks and normalized linearly over the evaluation-set range.

#### Economics (G4)

Economics is calculated from a configurable usage profile covering storage, upload, and download, together with collateral evaluation.

Collateral is evaluated both relative to storage price and by absolute collateral amount. The final Economics score takes the weaker of the cost score and collateral score.

Defaults are **2 TB storage / 2 TB upload / 0.5 TB download per month, `k = 9`**. Upload cost includes temporary sector storage for 3 days.

#### Risk (G5)

Risk evaluates price instability from history. Price changes between consecutive scans become events after the configured threshold; ingress uses a higher threshold and collateral events are based on decreases. Price movement is compared against the inverse SC/USD movement.

The default window is 14 days and can be configured on Scoring (`0` means 14, maximum 120). A host with 0 or 1 price point receives score 10.

### Smart Overall

- The weighted sum of Performance, Resources, Economics and Risk (weights 25/25/25/25, sum 100) and the replacement threshold (15 %) are configured on the Scoring page.
- A host with fewer than 10 speed samples gets a provisional Performance score for this calculation (the median of the hosts that have contracts and enough samples).
- `GET /hosts` returns `smartOverall` and `performanceImputed`; the Hosts table has a Smart Overall column.

### Portfolio plan and execution

The decision engine is ported from `renterd`.

- The plan is computed in `Smart Shadow` and `Smart Active` and logged; `GET /smart/portfolio-plan` returns the last plan and the execution outcomes.
- **ADD** forms new contracts in a canonical order (Economics, then Risk on an exact tie) instead of a random one; the placement radius stays a grouping.
- **REPLACE** (at most once per 4 hours, one per plan): the contract with the lowest Smart Overall is replaced when a candidate beats its Economics by the threshold and migrating is economical (a contract without data counts as justified). The new contract is formed first; only then the old host is drained.
- If no replacement is justified yet, the contract gets an **economic pause** (one at a time, with an alert), lifted when nothing qualifies any more or Smart leaves Active.
- Only one replacement drains at a time; a drain counts as finished when the host holds none of our sectors.

### Contract formation rules

- Contracts of hosts in a Smart pause **occupy slots**: Good + Paused, except a pause that already has a replacement.
- A paused host is replaced only when Good is below N+2, where N is the number of shards of the redundancy scheme: never at N+2 or more, immediately below N, after the pause timeout in between; at most one replacement per pass, in `Smart Active` only.
- This stops the system from forming extra contracts when several hosts go into a gouging pause at once. The candidate preview uses the same rules.

### Upload and migration placement

- In `Smart Active`, new sectors and the targets of the indexd migrator go to the hosts with the highest Smart Overall, strictly in that order (ties random); a failing host is skipped, a host without a score goes last. Reads are unchanged.
- `GET /hosts` for applications carries `smartOverall` and `smartActive`; older clients ignore them.
- In `Smart Shadow` the previous order is used and the Smart order is logged. `s3d` writes its own `smart-autopilot.log`.

### Spending accounting per host

- A table `host_spending_daily` records money by category: contract, storage, upload, download, verification, deletion; account funding is shown separately as "Funded".
- Sources: the server, the migrator and `s3d` (it reports the volume of every successful transfer through `POST /hosts/usage`; the cost is estimated from the host prices and marked `~`).
- `GET /host/{hostkey}/spending` returns the totals (all time and 30 days). History starts at the first run of this build.

### Scoring history

Host Storage, Ingress, Egress and Collateral prices from successful scans, scan results, and the SC/USD rate every 30 minutes are retained for 120 days and cleaned daily.

### Hosts and Contracts UI

- **Hosts.** One view switch `Candidates | Usable + contracts | All`; by default hosts with a contract or on a Smart pause are shown even when they are not usable.
- **Hosts.** Usable shows one check mark (failed checks in a tooltip); price icons turn yellow when a price is within the limit but above the contract formation limit; a sortable **Accepted** column shows hosts with an active good contract, with a yellow envelope next to the check mark for hosts that accept signals.
- **Hosts.** **View** menu, collapsible Performance, Economics, Resources and Risk groups (short labels Perf, Econ, Reso, Risk), sorting by scores and parameters; column widths follow the content and the page renders much faster.
- **Contracts.** Columns **My Data**, **Σ spent**, **Σ storage**, **Σ ingress**, **Σ egress** and **Σ collateral**; a View menu; a panel with the data on the host, the spending by category and source, the transfers, the scores and the pause reason; a contract of a paused host shows **Pause**.
- **Scoring.** The **Signals to hosts** block with the **Send signals automatically** switch.
- Muted labels and the world map are brighter, and the log lines that used to repeat every two minutes are written only when the state changes.

### Placement policies

- Configurable price allowance for new contracts: 80-95%.
- Configurable minimum distance between hosts; `0` disables the restriction.
- Placement radius relative to a reference point — IP address or coordinates.
- `prefer` mode selects candidates inside the radius first, then the nearest candidates outside it.
- `strict` mode allows only candidates inside the radius.

### Allowlist and blocking

- A soft allowlist directs new contracts and new data to allowed hosts when configured.
- Existing contracts outside the allowlist remain readable; new data is not placed on them and removal happens gradually as contracts expire.
- Manual and automatic block reasons are stored separately.

### Smart Gouging Pause

A price-policy pause mechanism can keep data on a host instead of immediately starting migration solely because of a price change.

- Controlled by Autopilot Mode (`Original`, `Smart Shadow`, `Smart Active`).
- `Keep Gouging Contract Until Replacement` and `Allow Existing Data on Overpriced Contracts`.
- Configurable pause timeout.
- Collateral-to-price checks and Reed-Solomon-aware safety recovery.
- `smart-autopilot.log`, alerts, the Scoring page, and the `Paused` column.

`Original` and `Smart Shadow` preserve standard decision logic; `Smart Shadow` additionally records decisions for analysis.

### Redundancy policy and API

- User erasure-coding policy is defined as a K/N scheme and provided to clients through `GET /account`.
- The current model can relax the standard threshold but cannot make it stricter.
- `GET /hosts` returns `performance`, `economics`, `resources`, `risk`, `smartOverall` and `hasContract` for hosts in the evaluation set; its `involved` parameter selects usable hosts plus hosts with a contract or a pause. It also returns `acceptsSignals` for each host.
- `POST /hosts/speed` and `POST /hosts/usage` accept measurements and transfer summaries from a compatible client.
- `GET /smart/portfolio-plan` and `GET /host/{hostkey}/spending`.
- `GET/PUT /settings/signals` and `GET/PUT /settings/work-share-report`.
- Economics profile, Risk window, Smart Overall weights and the replacement threshold are available through `/settings/smart`.

### Candidate preview

`GET /hosts/candidates` and the `Candidates` view show candidates for new contracts under the current settings, including ranks, distances, and exclusion reasons. The preview uses the same selection mechanism and the same slot rules as actual contract formation.

### Availability pause

A host with low availability (uptime) enters the same pause mechanism as Smart Gouging Pause, using the `Host grace period`.

### Data transfer and limits

- Configurable upload and download limits for migration.
- Client transfer limits configured through quotas and accounts are provided by `GET /account`; a compatible client, in particular `s3d Smart Autopilot`, must enforce them.
- The build version is shown in logs, `indexd version`, `GET /api/state`, and the web interface.

## Known limitations

- The work share report and host messages are designed for small pools; on a pool of a few hosts the results are noisy.
- The admin API listens on all addresses by default. Set a strong administrator password and restrict access with a firewall.
- The Blockchain node page does not list connected peers, as in the official indexd (there is no peers endpoint).

## Future development

Under discussion.

## Upstream

- [SiaFoundation/indexd](https://github.com/SiaFoundation/indexd)
- [Fix #1135](https://github.com/SiaFoundation/indexd/pull/1135)
- [Issue #1138](https://github.com/SiaFoundation/indexd/issues/1138)

## License

This project is based on [SiaFoundation/indexd](https://github.com/SiaFoundation/indexd) and continues the Smart Autopilot concept originally researched and implemented in `renterd`.

The project is distributed under the MIT license. The Sia Foundation copyright notice and license terms are preserved in [LICENSE](LICENSE).

This is an independent fork and is not an official Sia Foundation release.
