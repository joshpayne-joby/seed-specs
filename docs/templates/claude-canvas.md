## seed Configuration

```text
uid: P-XXXX
project_id: [DOMAIN]-[SHORTNAME]
project_display_name: [Project name]
project_original_name: [Name at kickoff]
seed_mirror_filename_prefix: seed-[project_id]
file_ecosystem: [google / microsoft / mixed]
primary_site: [site code]
team_sites: [comma separated]
```

### Canvas Registry (locked at creation)

```text
tasks_canvas_id: [id]
tasks_canvas_url: [url]
human_canvas_id: [id]
human_canvas_url: [url]
hub_canvas_id: [id]
hub_canvas_url: [url]
```

### Communication

```text
project_channel_id: [channel ID or "none"]
project_channel_type: [channel / group_dm / none]
```

### Photo Breadcrumb Config

```text
photo_staging_dm_id: [DM channel ID or blank]
photo_project_keywords: [project name, nicknames]
photos_drive_folder_url: [URL or blank]
```

### Digest Config

```text
digest_enabled: false
digest_channel_id: [defaults to project_channel_id — override here if different]
digest_escalation_sessions: 5
```

### Pending Digest Items

|Tag|Type|Directed At|Question|Task|Opened|Escalations|Last Digest TS|
|---|---|---|---|---|---|---|---|
|—|—|—|—|—|—|—|—|

### Resolved Digest Items

|Tag|Type|Directed At|Question|Task|Opened|Resolved|Resolution|Response|
|---|---|---|---|---|---|---|---|---|
|—|—|—|—|—|—|—|—|—|

### Mirror Config

```text
seed_mirror_drive_folder_url: [URL or blank]
seed_mirror_script_url: [Apps Script URL or blank — automated only]
last_session_date: [today]
pending_photo_transfer: false
pending_photo_count: 0
```

### Code Integration (optional — leave blank if no code repo)

```text
repo_path:
repo_remote:
repo_platform:
default_branch:
dev_command:
dev_url:
github_channel_id:
branching_strategy:
architecture_canvas_id:
architecture_canvas_url:
```

---

## Status Codes

:white_circle: Not started
:large_blue_circle: In progress
:white_check_mark: Done
:red_circle: Blocked

## Rules

* Claude fetches this Canvas at the start of every session automatically.
* Write-back is always full replace — never a targeted section update.
* If write-back fails, full content appears in chat — paste manually.
* Log blockers the same day you hit them.

---

## [Owner Name] — [Role]

### [Team / Location / Context]

|#|Task|Status|Blocks|
|---|---|---|---|
|1|[First task from conversation]|:white_circle:|—|

---

## Shared / Unblocked Now

|#|Task|Owner|Status|
|---|---|---|---|
|—|None yet|—|:white_circle:|

---

## Blockers Log

|#|Blocked Task|Waiting On|Since|
|---|---|---|---|
|—|None yet|—|—|

---

## Done

|#|Task|Completed|
|---|---|---|
|—|None yet|—|

---

## Session Log

|Date|Contributor|Changes Made|
|---|---|---|
|[today]|[Owner name]|seed project created — initial setup|
