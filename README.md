# seed-specs

Lean operational specs fetched by seed framework headless agents (Claude.ai scheduled tasks) at session start.

This repo exists because Claude.ai's scheduled-task runtime only permits outbound HTTP to a small allow-list of hosts — `raw.githubusercontent.com` is on it, secret gists are not. So the operational specs that headless agents need at runtime live here, in a deliberately bare-bones public repo. 
## Contents

- `SMARTSHEET_DRIFT_WATCHER.md` — operational spec for the daily Drift Watcher routine. Read at session start by the Drift Watcher scheduled task. Sanitized — uses `{{placeholder}}` syntax for operational IDs (the scheduled-task prompt provides them via its Configuration block).

## What this repo is NOT

- The canonical home for any of these specs. Source-of-truth edits happen here; rich maintainer versions with full history live in the private repo.
- A general-purpose seed framework reference. For that, see the private repo.
- Production-grade. Each spec here is the lean operational subset of a fuller doc in the private repo.
