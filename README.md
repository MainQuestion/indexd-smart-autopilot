# indexd Smart Autopilot

English| [Українська](README_UA.md) | [Русский](README_RU.md)

**indexd Smart Autopilot** is an experimental evolution of the Sia indexer, continuing the Smart Autopilot work originally developed in `renterd`.

In this architecture, `indexd` provides placement policies, host selection, admission rules, and part of contract lifecycle management, while `s3d` receives these policies and applies them during data transfers.

The goal is not to replace standard mechanisms with an arbitrary algorithm, but to add a configurable intelligent layer for host-portfolio management and data placement.

> **Status: experimental alpha.** Tested on one Windows node. Always back up the database before upgrading. Settings and database schema may change between alpha releases.

## Relation to renterd Smart Autopilot

The project continues the same development direction and preserves the core principles established in `renterd`.

- **A profile instead of one magical Score.** A host should be evaluated across separate dimensions so strength in one area does not hide weakness in another.
- **Seven scoring groups:** reliability, performance, resources, economics, risk/collateral, technical compatibility, and network independence.
- **Measurements separate from policy.** Actual host characteristics should be separated from user-defined thresholds, weights, and decision rules.
- **Host-portfolio management.** What matters is not only an individual host's rating, but also its effect on the reliability and independence of the overall placement.
- **Safe lifecycle management.** Pausing, replacement, and migration should preserve data availability and avoid unnecessary data movement.
- **Transparency and configurability.** Decisions should have understandable reasons while users retain control over policies.
- **Gradual adoption.** New decision mechanisms should be validated separately from standard behavior before becoming active.

The seven-group model and full portfolio management remain architectural goals. This alpha implements the first working part of the model: **G2–G5** are available in display-only (Shadow) mode and do not yet affect decisions.

## Current version

- Release: **v0.2.0-alpha.1**
- Build version: `v0.6.1-7838380a`
- Base: `indexd v0.6.1` with [#1135](https://github.com/SiaFoundation/indexd/pull/1135)
- Fork schema version: `13` (upstream schema: `28`)
- Web UI: `feat/indexd-risk-score`
- Companion: **s3d Smart Autopilot v0.2.0-alpha.1**, build `v0.2.1-ba6f8c9`

This is an independent fork and is not an official Sia Foundation release.

## Implemented in this alpha

### Smart Autopilot host scoring groups

In this version, host scores operate in **Shadow** mode: they are calculated and displayed, but **do not affect host selection, contract formation, pauses, or migrations**.

Scores are calculated for the *evaluation set*: pool hosts that pass price and location filters plus hosts with an active contract. Other hosts have no score and are shown as `—`. Scores are recalculated every 10 minutes and immediately when relevant price, location, or host-list settings change.

#### Performance (G2)

- Real upload and download measurements from s3d.
- 10-day observation window.
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

### Scoring history

Host Storage, Ingress, Egress and Collateral prices from successful scans, scan results, and the SC/USD rate every 30 minutes are retained for 120 days and cleaned daily. History starts accumulating from the first run of this build.

### Hosts UI

- **View** menu for showing and hiding columns by section.
- Collapsible **Performance**, **Economics**, **Resources**, and **Risk** groups.
- Collapsed groups show one aggregate column; expanded groups show their individual parameters.
- Groups are collapsed by default and their state is stored in the browser.
- Sorting by scores and individual group parameters; hosts without scores are placed last.
- Storage, Ingress, Egress and Collateral belong to Economics; Remaining Storage and Total Storage belong to Resources.

### Placement policies

- Configurable price allowance for new contracts: 80–95%.
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

- `Original`, `Shadow`, and `Active` modes.
- `Keep Gouging Contract Until Replacement` and `Allow Existing Data on Overpriced Contracts`.
- Configurable pause timeout.
- Collateral-to-price checks and Reed–Solomon-aware safety recovery.
- `smart-autopilot.log`, alerts, the Scoring page, and the `Paused` column.

`Original` and `Shadow` preserve standard decision logic; `Shadow` additionally records decisions for analysis. Review the release notes before using `Active`.

### Redundancy policy and API

- User erasure-coding policy is defined as a K/N scheme.
- The policy is provided to clients through `GET /account`.
- The current model can relax the standard threshold but cannot make it stricter.
- `GET /hosts` returns `performance`, `economics`, `resources`, and `risk` for hosts in the evaluation set.
- `POST /hosts/speed` accepts speed measurements from a compatible client.
- Economics profile and Risk window settings are available through `/settings/smart`.

### Candidate preview

`GET /hosts/candidates` and the `Candidates` view show candidates for new contracts under the current settings, including ranks, distances, and exclusion reasons. The preview uses the same selection mechanism as actual contract formation.

### Availability pause

A host with low availability (uptime) enters the same pause mechanism as Smart Gouging Pause, using the `Host grace period`.

### Data transfer and limits

- Configurable upload and download limits for migration.
- Client transfer limits configured through quotas and accounts are provided by `GET /account`; a compatible client, in particular `s3d Smart Autopilot`, must enforce them.
- The build version is shown in logs, `indexd version`, `GET /api/state`, and the web interface.

## Future development

The following parts of the seven-group architecture are not yet implemented as a working system:

- G1 **Reliability** as a separate score;
- G6 **Technical Compatibility** as a separate score;
- G7 **Network Diversity** as a separate score;
- configurable aggregate `Smart Overall`;
- using host profiles and current portfolio composition when selecting new hosts and replacing existing ones;
- limiting the number of shards per location instead of relying only on minimum distance;
- additional non-price pause reasons;
- configurable collateral reserve when forming contracts;
- interruptible migrations, deferred recovery, and migration priorities based on slab state;
- re-binding lost sectors to a returning host where possible instead of always re-uploading them.

## Upstream

- [SiaFoundation/indexd](https://github.com/SiaFoundation/indexd)
- [Fix #1135](https://github.com/SiaFoundation/indexd/pull/1135)
- [Issue #1138](https://github.com/SiaFoundation/indexd/issues/1138)

## License

This project is based on [SiaFoundation/indexd](https://github.com/SiaFoundation/indexd) and continues the Smart Autopilot concept originally researched and implemented in `renterd`.

The project is distributed under the MIT license. The Sia Foundation copyright notice and license terms are preserved in [LICENSE](LICENSE).

This is an independent fork and is not an official Sia Foundation release.
