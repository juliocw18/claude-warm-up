# Claude Warm-Up Scheduler

Cloud Scheduled Task that automatically starts a Claude session every weekday at 4:55 AM Panama Time (GMT-5), warming up the token consumption window so it resets earlier during the workday.

The task checks today's date against 11 fixed Panama public holidays and skips execution on those days. If today is a regular workday, Claude responds to a brief sarcastic prompt — just enough to start the rolling token window without wasting quota.

---

## Table of Contents

- [How It Works](#how-it-works)
- [Architecture](#architecture)
- [Project Structure](#project-structure)
- [Prerequisites](#prerequisites)
- [Setup](#setup)
  - [1. Clone the Repository](#1-clone-the-repository)
  - [2. Create the Cloud Scheduled Task](#2-create-the-cloud-scheduled-task)
- [The Warm-Up Prompt](#the-warm-up-prompt)
- [Holiday Calendar](#holiday-calendar)
- [Configuration Reference](#configuration-reference)
- [Verification](#verification)
- [Troubleshooting](#troubleshooting)
- [Development Status](#development-status)
- [License](#license)

---

## How It Works

Claude subscriptions use a rolling token window. When you start a session, the window begins counting down. By triggering a session at 4:55 AM — hours before actual work begins at 8:00 AM — the window resets earlier in the workday, giving a fresh token bucket for the bulk of active coding.

The scheduled task runs on Anthropic's cloud infrastructure:

1. **Cron trigger** — Fires at 4:55 AM Panama Time (GMT-5, no daylight saving) every weekday.
2. **Holiday check** — Claude reads today's MM-DD against a list of 11 fixed Panama public holidays. If matched, it responds with a short skip message and stops (~20 tokens consumed).
3. **Sunday observance** — If today is Monday and yesterday (Sunday) was a holiday, the task also skips. Panama observes the following Monday as a non-working day when a holiday falls on Sunday.
4. **Warm-up response** — On regular workdays, Claude responds to a sarcastic 3-sentence prompt (~100-150 tokens consumed), starting the token window.

---

## Architecture

The entire system is a single Cloud Scheduled Task. No code, no servers, no local machine dependency.

```
Anthropic Cron (4:55 AM UTC-5, weekdays)
    |
    v
Claude receives the warm-up prompt
    |
    v
Claude checks today's date against embedded holiday list
    |
    |-- Holiday? --> "Holiday. Going back to sleep." --> Session ends
    |
    +-- Workday? --> 3-sentence sarcastic response --> Session ends
                     (token window started)
```

**Key technical decisions:**

| Component | Technology | Why |
|-----------|-----------|-----|
| Scheduling | [Cloud Scheduled Tasks](https://code.claude.com/docs/en/web-scheduled-tasks) | Runs on Anthropic infrastructure, no machine required |
| Holiday logic | Embedded in prompt | Zero dependencies, no external API calls |
| Timezone | GMT-5 fixed | Panama does not observe daylight saving time |
| Repository | GitHub (private) | Required by Cloud Scheduled Tasks platform |

---

## Project Structure

```
claude-warm-up/
├── docs/
│   └── superpowers/
│       ├── specs/
│       │   └── 2026-04-14-claude-warm-up-design.md   # Design specification
│       └── plans/
│           └── 2026-04-14-claude-warm-up-plan.md      # Implementation plan
├── .gitignore
├── LICENSE
└── README.md
```

The repository itself is minimal — the Cloud Scheduled Task configuration (prompt, schedule, environment) lives on Anthropic's platform, not in the repo. The repo exists because Cloud Scheduled Tasks require a linked GitHub repository.

---

## Prerequisites

- **Claude subscription** — Any paid plan (Pro, Max, Team, Enterprise) with Cloud Scheduled Tasks access
- **GitHub account** — Repository must be hosted on GitHub for Cloud Scheduled Tasks integration

---

## Setup

### 1. Clone the Repository

```bash
git clone https://github.com/juliocw18/claude-warm-up.git
```

### 2. Create the Cloud Scheduled Task

Open [claude.ai/code/scheduled](https://claude.ai/code/scheduled) in your browser and click **"New scheduled task"**.

Configure with the following values:

| Field | Value |
|-------|-------|
| Name | `panama-warm-up` |
| Repository | `claude-warm-up` |
| Schedule | Weekdays, 4:55 AM (UTC-5 / Panama Time) |
| Cloud Environment | Default |
| MCP Connectors | Remove all |

Paste the warm-up prompt (see [The Warm-Up Prompt](#the-warm-up-prompt) section below) and click **"Create"**.

---

## The Warm-Up Prompt

```
Check today's date. If today matches any Panama holiday in this list
[01-01, 01-09, 05-01, 11-03, 11-04, 11-05, 11-10, 11-28, 12-08, 12-20, 12-25],
respond only with "Holiday. Going back to sleep." and stop.

Also, if today is Monday and yesterday (Sunday) was a holiday from that list,
respond only with "Holiday (Sunday observance). Going back to sleep." and stop.

Otherwise: Explain in exactly 3 sentences why an AI waking up before sunrise
to warm a token bucket is the most dystopian-yet-mundane thing happening right now.
```

**Design rationale:**

- **Holiday check first** — If today is a holiday, Claude stops immediately with minimal token usage.
- **3-sentence constraint** — Forces Claude to think (starts the token window) without wasting quota.
- **Sarcastic tone** — Makes the output entertaining to review while serving its functional purpose.

---

## Holiday Calendar

11 fixed-date Panama public holidays. These dates never change and require no yearly maintenance.

| Date (MM-DD) | Holiday | Spanish Name |
|--------------|---------|-------------|
| 01-01 | New Year's Day | Ano Nuevo |
| 01-09 | Martyrs' Day | Dia de los Martires |
| 05-01 | Labor Day | Dia del Trabajo |
| 11-03 | Separation Day | Separacion de Panama de Colombia |
| 11-04 | Flag Day | Dia de la Bandera |
| 11-05 | Colon Day | Dia de Colon |
| 11-10 | Uprising of Los Santos | Primer Grito de Independencia |
| 11-28 | Independence from Spain | Independencia de Panama de Espana |
| 12-08 | Mother's Day | Dia de las Madres |
| 12-20 | National Day of Mourning | Dia de Duelo Nacional |
| 12-25 | Christmas Day | Navidad |

**Note:** Movable holidays (Carnival Tuesday, Good Friday) are excluded by design to eliminate yearly maintenance. The Sunday observance rule (Monday skip when a holiday falls on Sunday) is handled dynamically by Claude in the prompt.

---

## Configuration Reference

All configuration lives in the Cloud Scheduled Task settings at [claude.ai/code/scheduled](https://claude.ai/code/scheduled):

| Setting | Value | Description |
|---------|-------|-------------|
| Schedule | `55 9 * * 1-5` (UTC) | 4:55 AM Panama Time, weekdays only |
| Cloud Environment | Default | No special setup needed |
| MCP Connectors | None | No external integrations required |
| Notifications | None | Check run history manually |

To modify the schedule or prompt, edit the task directly in the web UI.

---

## Verification

After the first scheduled run:

1. Go to [claude.ai/code/scheduled](https://claude.ai/code/scheduled)
2. Open the run history for `panama-warm-up`
3. Confirm the session executed at approximately 4:55 AM UTC-5
4. Confirm the response is either a 3-sentence warm-up or a holiday skip message

---

## Troubleshooting

### Task doesn't appear in the dashboard
Ensure the `claude-warm-up` repository is accessible from your GitHub account linked to Claude. Check that Cloud Scheduled Tasks are enabled for your subscription.

### Task runs but doesn't start the token window
Verify the response contains actual reasoning (the 3-sentence sarcastic response), not just the holiday skip message. A skip message consumes minimal tokens and may not trigger the window.

### Wrong timezone
Panama Time is GMT-5 with no daylight saving adjustments. If the task fires at the wrong time, verify the timezone setting in the scheduled task configuration. The UTC equivalent is 9:55 AM (`55 9 * * 1-5`).

### Task skips on a regular workday
Check if today's MM-DD accidentally matches one of the 11 holiday dates. Also check if today is Monday following a Sunday holiday (Sunday observance rule).

---

## Development Status

### Completed
- [x] Design specification with holiday handling strategy
- [x] Implementation plan with step-by-step setup instructions
- [x] GitHub repository for Cloud Scheduled Task integration
- [x] Warm-up prompt with embedded holiday list and Sunday observance rule

### Planned
- [ ] Cloud Scheduled Task creation and activation
- [ ] First run verification
- [ ] Optional: holiday behavior manual test

---

## License

MIT
