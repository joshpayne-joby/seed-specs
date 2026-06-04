## Project Status

:large_green_circle: On Track

**Phase:** Setup

**Last updated:** [today]

---

## [Project Display Name]

**Project ID:** [project_id]

**Owner:** [owner name] — [role]

**Site:** [primary_site]

---

## Phase Tracker

|Phase|Status|Notes|
|---|---|---|
|Setup|:large_green_circle: Active|Canvases created, files generated|
|[Phase 2]|:white_circle: Not started|—|

:large_green_circle: Active | :large_yellow_circle: At risk | :red_circle: Blocked | :white_circle: Not started | :checkered_flag: Complete

---

## Canvas Registry

> Machine-readable canvas index. The seed Routine and Control Tower read this to find the project's canvases.

- **UID:** `P-XXXX`
- **Project ID:** `[project_id]`
- **Display Name:** [Project Display Name]
- **Claude Canvas:** _pending — backfilled at Stage 7_
- **Human Canvas:** `[human_canvas_id]` — [Open]([human_canvas_url])
- **Channel:** `[project_channel_id or "none"]` — ![](#[project_channel_id])
- **Claude Project:** —

---

## Key Resources

|Resource|Link|
|---|---|
|Project Drive folder|[url]|
<!-- Phase 1/2: leave [url] as the placeholder text (Workshop substitutes "—" if not yet set up). Phase 3 backfills with the real Drive folder URL via Stage 11. -->


---

## Team

|Name|Role|Slack|Claude Access|
|---|---|---|---|
|[Owner]|[Role]|@[slack_id]|Yes|
|[Contributor]|[Role]|@[slack_id]|[Yes/No]|

---

## Decision Authority

|Decision Type|Who Approves|
|---|---|
|Scope changes|[Owner]|
|Safety or compliance|[From constraints]|

---

## Onboarding — New Collaborators

1. Get Claude access to this Project — ask the project owner
2. Connect Slack in Claude for Desktop: Settings → Connectors → Slack
3. Open the canvas links in the Canvas Registry section above — access is granted automatically when you're added to this project's Slack channel. If a canvas returns "access_denied," tell the project owner so they can re-confirm channel-share is set on that canvas (per `contracts/canvas-access.md`). *Tip: if your team has a Slack user group (e.g., one that maps to all members of your AES/AMFG/etc. group), adding it to the project channel is faster than inviting members one-by-one — every group member auto-inherits canvas access via channel-share.*
<!-- Phase 1 rendering (no channel): replace step 3 with: "3. Open the canvas links in the Canvas Registry section above — the project owner shares each canvas with you directly via the Slack Share dialog. If a canvas returns 'access_denied,' ping the project owner so they can re-add you. (When this project graduates to Phase 2 with a project channel, channel-share replaces per-user share for durable inheritance.)" -->
4. Get added to the project's shared [Google Drive / SharePoint] folder — ask the project owner
<!-- Phase 1/2 rendering (no Drive folder yet): omit step 4 entirely. Renumber step 5 to step 4. The shared folder lands at Phase 3 when the project is registered in the seed Project Registry. -->
5. Run sessions from Claude for Desktop, not the browser

Claude orients new collaborators automatically at their first session.
