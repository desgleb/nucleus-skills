# nucleus-skills / Daily Workflow Skills

Three Claude Code skills for structured daily work sessions: morning planning, mid-day GTD triage, and evening close-out with Wiki updates.

## Skills

| Skill | Trigger | Purpose |
|---|---|---|
| `good-morning` | `/good-morning` | Opens the day: reads mail + calendar, builds or updates the weekly plan in Eisenhower quadrants |
| `daily-digest` | `/daily-digest` | Mid-day triage: mail delta + weekly plan + Tracker statuses → GTD "what to do right now" |
| `happy-evening` | `/happy-evening` | Closes the day: summarizes what was done vs planned, updates project Wiki pages, shows tomorrow's slice |

The three skills share a single state file (`skill-state.json`) and form a daily pipeline:

```
/good-morning  →  /daily-digest (×N)  →  /happy-evening
    ↓                    ↓                      ↓
creates/updates      read-only GTD         updates Wiki
weekly Wiki plan     recommendations       project pages
```

## Dependencies

These skills are built for the [gd-yandex-tracker-mcp](https://github.com/desgleb/gd-yandex-tracker-mcp) MCP server. They use the following MCP tools:

**Mail (IMAP)**
- `get_mail_summary` — inbox delta with folder filtering
- `get_mail` — full message content by UID
- `search_mail` — search by subject/sender

**Calendar (CalDAV)**
- `list_events` — weekly calendar

**Tracker**
- `get_issue`, `manage_comments` — task details and discussion
- `search_entity`, `get_entity` — project entities with checklists

**Wiki**
- `get_wiki_page`, `create_wiki_page`, `update_wiki_page` — weekly plans
- `update_wiki_section` — section-level updates for project pages
- `list_wiki_subpages` — navigation

## State File

All three skills share `~/.claude/projects/<project-slug>/skill-state.json`:

```json
{
  "last_mail_read": "2026-05-26T09:00:00+03:00",
  "last_good_morning": "2026-05-26T08:30:00+03:00",
  "last_digest": "2026-05-26T12:00:00+03:00",
  "last_happy_evening": "2026-05-25T18:30:00+03:00"
}
```

- `last_mail_read` controls the mail delta window — updated by every skill after processing mail
- On first run (no state file) `last_mail_read` defaults to last Friday

## Weekly Plan Format

`good-morning` creates/updates a Wiki page per ISO week at a configurable slug. The page structure:

- **Q1** — Important & urgent (do first)
- **Q2** — Important, not urgent (schedule)
- **Q3** — Not important, urgent (delegate / quick close)
- **Q4** — Not important, not urgent (defer / drop)
- **Frogs** — One per weekday, tasks that keep getting postponed
- **Elephants** — Large Q2 items, decomposed into daily chunks
- **Weekly calendar** — Meetings linked to projects
- **Mail context** — Open threads requiring a response

## Eisenhower Classification Rules

- **Project = one quadrant.** All tasks for the same client/project get the highest priority in the combination.
- **Group by client, not technology.** Two clients using the same product are two different projects.
- **Overdue + rejections → top of Q1.** Separate "Requires immediate reaction" block.
- **@mention in Tracker = FYI.** Action required only if the comment body contains a direct request.
- **Role priority:** Assignee (1) > Author (2) > Watcher (3).

## GTD Rules (daily-digest)

- **Next action = physical action.** Not "work on proposal" but "open template, fill section 3 based on Akimov's data".
- **Waiting is a valid state.** If the ball is on someone else's side — no action needed, just track.
- **2-minute rule:** if it takes under 2 minutes, it's always "do now" regardless of quadrant.

## Adapting for Your Setup

1. Replace the Wiki slug constants (`users/g.deshin/...`) with your own Wiki user path.
2. Update the `lead` filter in `search_entity` calls to your Tracker login.
3. Point `skill-state.json` to your Claude Code project path.
4. Adjust mail folder names if your IMAP setup differs from `["INBOX", "Outbox", "Sent", "tracker"]`.

## Install

Copy each skill directory into your Claude Code project:

```
.claude/skills/
  good-morning/SKILL.md
  daily-digest/SKILL.md
  happy-evening/SKILL.md
```

Claude Code picks up skills automatically from `.claude/skills/`.
