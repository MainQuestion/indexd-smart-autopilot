# indexd Smart Autopilot v0.2.0-alpha.1

Second alpha release of the indexd fork: Smart Autopilot host scoring across four groups (Performance, Resources, Economics, and Risk), scoring history, grouped Hosts columns, and the View menu.

| Parameter | Value |
|---|---|
| Build version | `v0.6.1-7838380a` |
| Upstream base | indexd v0.6.1 + fix #1135 |
| Fork schema version | 13 (upstream schema 28) |
| Web UI | `feat/indexd-risk-score` |
| Companion | s3d Smart Autopilot v0.2.0-alpha.1 (`v0.2.1-ba6f8c9`) |

## What's new

**Smart Autopilot host scoring (Shadow).** Scores from 1 to 10 are **display-only**: they do **not** affect host selection, contract formation, pauses, or migrations. Scores are calculated for the *evaluation set*: pool hosts that pass price and location filters plus hosts with an active contract. Other hosts have no score and show `—`. Recalculation runs every 10 minutes and immediately when price, location, or host-list settings change.

- **Performance (G2).** Based on real s3d transfers: upload and download, 10-day window, scored relative to the pool's 95th percentile. s3d sends measurements once per minute through `POST /hosts/speed`. The result and raw Mbps are shown on Hosts.
- **Economics (G4).** Cost based on the usage profile (TB of storage, upload, and download per month) plus collateral scoring (collateral-to-storage-price ratio and absolute collateral); the final score is the minimum of the cost and collateral scores. Upload cost includes temporary sector storage for 3 days. The profile and curve steepness `k` are configured on Scoring (default 2 / 2 / 0.5 TB, `k = 9`).
- **Resources (G3).** Seven parameters in three blocks: data volumes (our data, free space, foreign data), money and time (exposure, recovery-cost estimate, remaining contract term), and lost-sector share. Normalized linearly over the evaluation-set range.
- **Risk (G5).** Price instability: each host price change between consecutive scans (5% threshold, 50% for ingress, collateral decreases only) is counted as an event when it is not explained by the inverse SC/USD movement; more events over the window produce a lower score. Default window is 14 days, configurable on Scoring (`0` means 14, maximum 120). A host with 0 or 1 price point receives score 10.

**Scoring history.** Host prices (storage, ingress, egress, collateral) from each successful scan, scan results, and SC/USD exchange rate every 30 minutes are retained for 120 days and cleaned daily. History starts accumulating from the first run of this build.

**Hosts page.**
- **View** menu: show and hide columns by section.
- Collapsible **Performance**, **Economics**, **Resources**, and **Risk** column groups: collapsed shows one aggregate group column, expanded shows the full group; groups are collapsed by default and state is stored in the browser.
- Sorting by scores and individual group parameters (Order by and And sort); hosts without a score are placed last.

**Availability pause.** A host with low availability (uptime) enters the same pause mechanism as Smart Gouging Pause (`Host grace period`).

**API.** `GET /hosts` returns `performance`, `economics`, `resources`, and `risk` fields (absent for hosts outside the evaluation set) and new sort keys; new settings are available through `/settings/smart` (Economics profile, Risk window); `POST /hosts/speed` accepts client transfer measurements. Fork migrations 8–13 are included; see `FORK.md` for details.

## UI changes

- Storage, Ingress, Egress, and Collateral price columns are now part of **Economics**, while Remaining Storage and Total Storage are part of **Resources**. When a group is collapsed, detailed columns are hidden and can be shown with the group control next to the View menu.
- These columns were moved in the View menu from General into their respective groups.

## Fixes

- **Currency in the Hosts table.** Monetary values (prices, collateral, Exposure, Compensation) in the table and host panel are recalculated when the exchange rate arrives after the host list or when the currency setting changes. Previously, they could remain in SC until the host list itself refreshed.

## Upgrade

1. Stop s3d and indexd.
2. Create a database dump: `pg_dump -U postgres -h 127.0.0.1 -Fc -f indexd-before-alpha2.dump indexd` (in PowerShell, use `-f` rather than `>` because redirection corrupts the binary dump).
3. Replace the binaries and start indexd. The log should contain `fork migration complete {"forkTarget": 13, ...}`, then start s3d.
4. When upgrading from alpha.1, migrations 8–13 are applied in one run. New settings receive default values; host-selection behavior does not change.
5. A new s3d is required for speed scoring: with old s3d, other scores still work but Performance remains 1 because there are no measurements. New s3d with upstream indexd receives `404` from `/hosts/speed` and stops sending measurements.

The fork schema **is not rolled back**. Restore the database from the dump if necessary.

## Known issues and notes

- **Alpha.** Tested on one Windows node.
- **Scores do not affect decisions.** This is Shadow mode: scores are for viewing, while host and contract selection remains unchanged.
- **Risk during the first days.** Price history starts with this build, so hosts initially have 0–1 points and score 10. Meaningful Risk scores appear after 1–2 weeks. Hosts are scanned approximately once per day, so a 14-day window contains no more than about 14 points and short-lived intra-day price changes are not visible.
- **Download speed** mainly reflects host response time: the SDK reads part of a sector (about 100 KiB per shard), so download Mbps compares hosts but is not full link capacity. Upload is measured using complete sectors.
- **Speed measurements.** Only this fork's s3d sends them and only successful transfers are included, so slow hosts may score slightly optimistically. `/hosts/speed` has no rate limit; protection includes an application signature, packet size up to 1000, and plausibility checks. Measurements are client-supplied and not spoof-proof.
- **Resources normalization** is sensitive to outliers: one host with a very large volume can compress the scores of the others.
- **Economics.** A host with zero total price after scanning receives the best cost score, as in renterd.
- The version link in the host profile points to the upstream release page, which has no corresponding fork release.
- Client transfer limits are enforced only by s3d Smart Autopilot; other clients ignore them.
- Known local test failures unrelated to the fork: `TestAppConnect` (timezone), `TestAuthConnectRateLimit` and `TestScanTimeout` (timing under load and `-race`), `TestMigrationConsistency` in a Windows checkout with CRLF (passes with LF).

## License

indexd and s3d are distributed under the MIT license (Copyright (c) 2025 The Sia Foundation); license texts are included in the release (`LICENSE-indexd.txt`, `LICENSE-s3d.txt`). Binaries contain additional dependencies under their respective licenses. The GeoLite2-City.mmdb database (MaxMind) is not included in the release: download it separately and configure its path.
