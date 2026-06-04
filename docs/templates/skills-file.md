# [Project Name] — Skills and Capabilities
# Last updated: [Date]

## Foundation Skills

| Skill | Status | When to Use |
|---|---|---|
| seed session start | Active | Every session — automatic |
| Canvas write-back | Active | End of session — automatic |
| Canvas safety fallback | Active | If write-back fails — paste from chat |
| Session summaries | Active | End of session — "Generate a Slack summary" |
| Session mirror (manual) | Active | End of session — markdown export to Drive/SharePoint |
| Session mirror (automated) | Opt-in | End of session — #seed-updates → Google Doc via Apps Script |

---

## Slack Connector

| Skill | Status | Notes |
|---|---|---|
| Read Canvas | Active | Fetches all seed canvases at session start |
| Write Canvas | Active | Full replace on Claude Canvas at session end |
| Create Canvas | Active | Setup and child canvas creation |
| Channel catch-up | Active | Reads project channel since last session (skipped if `none`) |

---

## Connected Tools

| Tool | Status | Notes |
|---|---|---|
| Slack | Active | Canvas read/write, channel catch-up, session summaries |
| [Google Drive / SharePoint] | Active | Project file storage and retrieval |
| Gmail | Not yet | Draft and send project updates |
| Google Calendar | Not yet | Flag milestones as calendar events |

---

## Skills Added This Project

| Date | Skill | Who | Notes |
|---|---|---|---|
| [today] | seed project activated | [Owner] | Three-canvas setup, channel catch-up, breadcrumbs |
