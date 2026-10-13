# indexd Smart Autopilot

[English](README.md) | [Українська](README_UA.md) | [Русский](README_RU.md)

**indexd Smart Autopilot** is an experimental extension of the Sia indexer that continues the Smart Autopilot work started in `renterd`.

As the Sia ecosystem evolves, storage management increasingly benefits from separating responsibilities across components. In this architecture, `indexd` handles placement policies, host selection, admission rules, and part of the contract lifecycle management. `s3d` receives these policies and applies them when transferring data.

The project’s goal remains unchanged: rather than replacing existing mechanisms with an arbitrary algorithm, it adds a configurable intelligent layer for managing the host portfolio and data placement. The ideas and engineering experience gained while working on `renterd` are being carried into this architecture step by step.

> **Status: experimental alpha.** Tested on one Windows node. Settings and database schema may change between alpha releases.

## Relationship to renterd Smart Autopilot

This repository continues the same development direction; it does not abandon the principles established earlier.

- **A host profile instead of a single magic score.** Hosts should be evaluated across separate dimensions so that a high rating in one area does not hide a weakness in another.
- **Seven scoring groups:** reliability, performance, resources, economics, risk/collateral, technical compatibility, and network diversity.
- **Measurements separate from policy.** A host’s measured characteristics should be kept separate from user-defined thresholds, weights, and decision rules.
- **Host portfolio management.** A host’s individual rating matters, but so does its effect on the overall reliability and independence of data placement.
- **A safe lifecycle.** Pausing, replacing, and migrating contracts should protect data integrity and availability while avoiding unnecessary data movement.
- **Transparency and configurability.** Decisions should have understandable reasons, and users should retain control over policies.
- **Gradual rollout.** New decisions should be evaluated separately from existing behavior before they are actively applied.

The seven-group model and full portfolio management are architectural goals. Not all of these elements are implemented in the current alpha release.

## Current version

- Release: **v0.1.0-alpha.1**
- Build version: `v0.6.1-9fd0bf52`
- Based on: `indexd v0.6.1` with fix [#1135](https://github.com/SiaFoundation/indexd/pull/1135)
- Fork schema version: `7` (upstream schema: `28`)
- Companion component: **s3d Smart Autopilot v0.1.0-alpha.1**, build `v0.2.1-a538024`

This is an independent fork, not an official Sia Foundation release.

## Implemented in the alpha release

### Placement policies

- Configurable price allowance for new contracts: 80–95%.
- Configurable minimum distance between hosts; `0` disables this restriction.
- Placement radius relative to a reference point defined by an IP address or coordinates.
- `prefer` mode: candidates inside the radius are preferred, followed by the nearest candidates outside it.
- `strict` mode: only candidates inside the radius are allowed.

### Allowlist and blocking

- A soft allowlist directs new contracts and new data to allowed hosts when a list is configured.
- Existing contracts outside the allowlist remain readable. No new data is placed on those hosts, and they are phased out gradually as contracts expire.
- Manual and automatic blocking reasons are stored separately: automatic unblocking does not remove a manual block.

### Smart Gouging Pause

For price checks, a pause policy can keep data on a host instead of immediately triggering migration solely because a price has changed.

- `Original`, `Shadow`, and `Active` modes.
- `Keep Gouging Contract Until Replacement` and `Allow Existing Data on Overpriced Contracts` settings.
- Configurable pause timeout (336 hours in the current alpha configuration).
- Collateral-to-price checks and protective recovery using Reed–Solomon.
- `smart-autopilot.log`, alerts, the `Scoring` page, and the `Paused` column.

`Original` and `Shadow` preserve the upstream decision-making behavior; `Shadow` additionally records decisions for analysis. Review the release notes before enabling `Active`.

### Redundancy policy and API

- The user-defined erasure-coding policy is specified as a K/N scheme.
- The policy is exposed to client applications through `GET /account`.
- In the current model, a user-defined policy can relax the default threshold, but cannot make it stricter.

### Candidate preview

`GET /hosts/candidates` and the `Candidates` view in the web interface show candidates for new contracts under the current settings, including their ranks, distances, and exclusion reasons. The preview uses the same selection mechanism as actual contract formation.

### Interface and data transfer

- Multi-level sorting, sorting by location and candidate data, and a `Collateral` column.
- Settings for upload and download speed limits during migration.
- Client transfer limits based on quotas and accounts are exposed through `GET /account`; a compatible client, such as `s3d Smart Autopilot`, must enforce them.
- The build version is shown in logs, the `indexd version` command, `GET /api/state`, and the web interface.

## Future development goals

The following items describe the Smart Autopilot direction and should not be understood as features already implemented in this release:

- A complete seven-group host profile and a configurable aggregate `Smart Overall` score.
- Using host profiles and the current portfolio composition when selecting new hosts and replacing existing ones.
- Limiting the number of shards placed in one location instead of relying only on minimum distance.
- Additional pause reasons unrelated to price, such as uptime, contract duration, and a host refusing new contracts.
- A configurable collateral margin when forming contracts.
- Interruptible migrations, deferred recovery, and migration priorities based on slab state.
- Reattaching lost sectors when a host returns, where possible, instead of always uploading them again.

## Upstream

- [SiaFoundation/indexd](https://github.com/SiaFoundation/indexd)
- [Fix #1135](https://github.com/SiaFoundation/indexd/pull/1135)
- [Issue #1138](https://github.com/SiaFoundation/indexd/issues/1138)

## License

This project is based on [SiaFoundation/indexd](https://github.com/SiaFoundation/indexd) and continues the Smart Autopilot concept first researched and implemented in `renterd`.

The project is distributed under the MIT License. The Sia Foundation copyright notice and license terms are preserved in [LICENSE](LICENSE).

This is an independent fork and is not an official Sia Foundation release.
