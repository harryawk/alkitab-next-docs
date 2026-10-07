# GitHub Actions release limits: do we need to move off?

Researched 2026-10-08 from primary sources (docs.github.com, expo.dev). Question: will `.github/workflows/release.yml` (manual `workflow_dispatch`; web + Android APK on ubuntu, plus a planned unsigned iOS IPA on macos) get blocked by free-tier limits?

Note: the `release.yml` on branch `feat/release-workflow` at research time has `check`, `web`, `android`, `release` jobs only. No macOS job was present. The macOS numbers below are for the planned iOS job.

## Verdict

**No need to move.**
- If the repo is **public** (it is `harryawk/AlkitabNext`; verify visibility), standard runners, including macOS, are free and unlimited. Nothing to exhaust.
- If it were **private**, the Linux jobs are cheap, but a ~30 min macOS build costs about 10x Linux per minute, so the free 2,000 minutes cover only ~6 full releases per month. A few releases per month still fits. Dropping iOS or building it only on demand would make it fit comfortably.

## Findings

### Free tier and public repos
- Free plan includes 2,000 minutes/month, 500 MB artifact storage, 10 GB cache. Pro/Team: 3,000 min, 1 GB / 2 GB. [About billing for GitHub Actions](https://docs.github.com/en/billing/concepts/product-billing/github-actions) (the figures were read via `/en/billing/managing-billing-for-your-products/managing-billing-for-github-actions/about-billing-for-github-actions`).
- "The use of standard GitHub-hosted runners is free: In public repositories." Same page.
- "Larger runners are always charged for, even when used by public repositories." Same page. `macos-latest` is a standard runner, so it is free in public repos.
- Private-repo workflows "use your GitHub account's allotment of free minutes, and are then charged at the per minute rates"; public standard runners are "free and unlimited". [GitHub-hosted runners reference](https://docs.github.com/en/actions/reference/runners/github-hosted-runners)

### macOS runner
- `macos-latest`, `macos-15`, `macos-26` are arm64 (M1), 3 vCPU, 7 GB RAM; `macos-15-intel` and `macos-26-intel` are Intel, 4 vCPU, 14 GB. Same specs for public and private repos. [GitHub-hosted runners reference](https://docs.github.com/en/actions/reference/runners/github-hosted-runners)
- Price: macOS 3/4-core (M1 or Intel) $0.062/min; Linux 2-core $0.006/min; Windows 2-core $0.010/min. [Runner pricing](https://docs.github.com/en/billing/reference/actions-minute-multipliers) (URL as fetched; the page now lists rates, not multipliers).
- Multiplier: the current docs no longer list multipliers, only per-minute rates. The old 10x macOS / 2x Windows rule is retired from the docs and its current applicability is unclear ([github/docs#45138](https://github.com/github/docs/issues/45138)). The price ratio $0.062 / $0.006 is about 10.3x, which matches the old 10x. The arithmetic below uses 10x.

### Artifacts, release assets, storage
- Artifact and log retention default is 90 days. Public repos can set 1-90 days, private repos 1-400 days. [Configuring retention](https://docs.github.com/en/organizations/managing-organization-settings/configuring-the-retention-period-for-github-actions-artifacts-and-logs-in-your-organization)
- Artifact storage beyond the plan quota is $0.25/GB-month; cache beyond 10 GB is $0.07/GB-month. Deleting artifacts lowers current storage but storage already accrued in the cycle stays on the bill. [About billing](https://docs.github.com/en/billing/concepts/product-billing/github-actions)
- Release assets: each file under 2 GiB, up to 1000 assets per release, no total-size or bandwidth limit. [About releases](https://docs.github.com/en/repositories/releasing-projects-on-github/about-releases)
- Consequence for a private repo: the 500 MB Free artifact quota is tight (web zip + APK + IPA per run may total 100-300 MB, retained 90 days). Final binaries live on the Release, which is not counted against Actions storage. Set `retention-days: 1` on `upload-artifact` if private.

### Time and concurrency
- Job: up to 6 hours on GitHub-hosted runners. Workflow run: 35 days. [Actions limits](https://docs.github.com/en/actions/reference/limits)
- Free plan: 20 concurrent jobs, 5 concurrent macOS jobs. Matrix: 256 jobs per run. Same page.
- The workflow's `concurrency: group: release` serialises runs, which is well within the limits.

### Arithmetic (private-repo case only)
Per release, estimating job times (my estimates, not measured): check 3 + web 4 + android 12 + release 1 = 20 Linux min; iOS 30 macOS min.
- Minute-equivalents at 10x: 20 + 30 x 10 = 320 per release. 2,000 / 320 = about 6 releases/month.
- In dollars: 20 x 0.006 + 30 x 0.062 = $0.12 + $1.86 = about $1.98 per release. Free allowance of 2,000 Linux-min is worth about $12, which also gives about 6 releases.
- Without iOS: 20 min per release means about 100 releases/month. Not a constraint.
- Failed or retried runs and the `check` job on each dispatch add to this. Five or more reruns in a month would hit the cap in the private case.
- Beyond the quota the account is billed per minute (or blocked if no payment method or a spending limit of $0 is set; not verified).

### Alternatives (short)
- **EAS Build free plan**: 15 Android and 15 iOS builds/month, low-priority queue with waits of 90+ minutes. [expo.dev/pricing](https://expo.dev/pricing). No overage on Free; builds stop until the 1st of next month. [EAS billing FAQ](https://docs.expo.dev/billing/faq/)
- **iOS without a paid Apple account**: simulator builds need no Apple Developer account. [Simulator builds](https://docs.expo.dev/build-reference/simulators/) The docs fetched did not confirm that a device-installable build can be made without the paid Apple program; the unsigned-IPA route in the workflow is outside these docs.
- **Local builds**: free and unlimited, but need your own Mac (Xcode) and Android toolchain; no CI reproducibility. (General knowledge, no source cited.)

## Could not verify
- Whether the repo is actually public (check `gh repo view --json visibility`).
- A current, official macOS minute multiplier. The docs dropped it; 10x is inferred from price ratio and the issue above.
- That `macos-latest` currently resolves to macOS 26. The docs table lists arm64/M1 for it, but the exact version mapping was a summary inference.
- Behavior at quota exhaustion for a private repo (blocked vs billed) depends on the billing/spending-limit settings; not read from the docs.
- The EAS plans page gave no numbers and the EAS pricing numbers came from `expo.dev/pricing`, which may be marketing copy that differs from the docs; EAS paid Apple account requirement for device builds was not confirmed.
- The job durations in the arithmetic are estimates; no run history was inspected.
