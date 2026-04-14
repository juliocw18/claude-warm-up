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
  - [1. Fork or Clone the Repository](#1-fork-or-clone-the-repository)
  - [2. First-Time Onboarding: Create a Cloud Environment](#2-first-time-onboarding-create-a-cloud-environment)
  - [3. Connect Your GitHub Account](#3-connect-your-github-account)
  - [4. Create the Routine](#4-create-the-routine)
  - [5. Configure the Trigger Schedule](#5-configure-the-trigger-schedule)
  - [6. Review Connectors and Permissions](#6-review-connectors-and-permissions)
  - [7. Create and Activate](#7-create-and-activate)
  - [8. Test with Run Now](#8-test-with-run-now)
- [The Warm-Up Prompt](#the-warm-up-prompt)
- [Holiday Calendar](#holiday-calendar)
- [Configuration Reference](#configuration-reference)
- [Model Selection](#model-selection)
- [Customization Guide](#customization-guide)
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
| Model | Haiku 4.5 | Cheapest model, minimizes token consumption |
| Repository | GitHub (public) | Required by Cloud Scheduled Tasks platform |

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
- **GitHub connected to Claude** — Your GitHub account must be linked to your Claude account (see [Step 3](#3-connect-your-github-account))

---

## Setup

### 1. Fork or Clone the Repository

If you want to use this warm-up scheduler for yourself, fork this repository to your own GitHub account, or clone and push to a new repo:

```bash
git clone https://github.com/juliocw18/claude-warm-up.git
cd claude-warm-up
```

If creating your own repo from scratch:

```bash
mkdir claude-warm-up && cd claude-warm-up
git init
git config user.email "your-email@example.com"
git config user.name "Your Name"
echo "# Claude Warm-Up Scheduler" > README.md
git add README.md
git commit -m "Initial commit"
gh repo create claude-warm-up --public --source=. --push
```

### 2. First-Time Onboarding: Create a Cloud Environment

If this is your first time using Cloud Scheduled Tasks, Claude will redirect you to an onboarding page at `claude.ai/code/onboarding` before you can create routines.

1. Open [claude.ai/code/scheduled](https://claude.ai/code/scheduled) in your browser
2. If redirected to the onboarding page, you will see a **"Create your first cloud environment"** form
3. Configure the cloud environment:

   | Field | Value | Why |
   |-------|-------|-----|
   | **Name** | `Default` | Keep the default name |
   | **Network access** | `None` | The warm-up task needs zero internet access — it only processes a prompt. Select "None" for maximum security |

4. Click **"Create & finish"**

This creates the cloud environment where your scheduled tasks will run. You only need to do this once — all future routines will use this environment by default.

> **Note:** If your routines need internet access for other use cases (e.g., downloading packages, calling APIs), choose "Trusted" (recommended by Anthropic) or "Full" instead. For this warm-up task specifically, "None" is sufficient.

### 3. Connect Your GitHub Account

Cloud Scheduled Tasks require access to a GitHub repository. You need to connect your GitHub account to Claude:

1. In the Claude Code web interface ([claude.ai/code](https://claude.ai/code)), look for the **"Connect to GitHub"** button at the bottom of the screen
2. Click it and authorize Claude to access your GitHub account
3. Grant access to the `claude-warm-up` repository (or all repositories if you prefer)

**Private repositories:** If your repository is private, you must also install the Claude GitHub App:

1. During the routine creation (Step 4), when you try to select a repository, private repos will show **"No repositories found"**
2. Click the **"Install the Claude GitHub app"** link that appears in the repository dropdown
3. Follow the GitHub authorization flow to grant Claude access to your private repo
4. Return to the routine creation page — your private repo should now appear in the dropdown

> **Tip:** Making the repository public avoids this extra step entirely. Since the repo only contains a README and documentation (no sensitive code or credentials), public visibility is perfectly safe.

### 4. Create the Routine

Claude Code calls scheduled tasks **"Routines"** in the web interface.

1. Open [claude.ai/code/scheduled](https://claude.ai/code/scheduled) in your browser
2. Click **"Routines"** in the left sidebar (below "New session")
3. Click **"New routine"** (or navigate directly to `claude.ai/code/routines/new`)
4. Fill in the routine form:

   **Name:**
   ```
   claude-warm-up
   ```

   **Prompt (the large text box):** Copy and paste the full warm-up prompt below exactly as written:

   ```
   Check today's date. If today matches any Panama holiday in this list
   [01-01, 01-09, 05-01, 11-03, 11-04, 11-05, 11-10, 11-28, 12-08, 12-20, 12-25],
   respond only with "Holiday. Going back to sleep." and stop.

   Also, if today is Monday and yesterday (Sunday) was a holiday from that list,
   respond only with "Holiday (Sunday observance). Going back to sleep." and stop.

   Otherwise: Explain in exactly 3 sentences why an AI waking up before sunrise
   to warm a token bucket is the most dystopian-yet-mundane thing happening right now.
   ```

   **Model:** Change the model selector (bottom-right of the prompt box) to **Haiku 4.5**. See [Model Selection](#model-selection) for why.

   **Repository:** Click **"Select a repository"** and choose `claude-warm-up` from the dropdown.

   **Environment:** Leave as **Default** (bottom-right corner of the form).

### 5. Configure the Trigger Schedule

1. Under **"Select a trigger"**, click **"Schedule"**
2. The trigger editor will open. Select the **"Custom"** tab for precise control
3. Enter the following cron expression:

   ```
   55 9 * * 1-5
   ```

   The editor will display: **"At 09:55 AM, Monday through Friday (times in UTC)"**

4. Click **"Save"**

**Understanding the cron expression:**

| Field | Value | Meaning |
|-------|-------|---------|
| `55` | Minute | At the 55th minute |
| `9` | Hour | Of the 9th hour (UTC) |
| `*` | Day of month | Every day of the month |
| `*` | Month | Every month |
| `1-5` | Day of week | Monday through Friday |

9:55 AM UTC = 4:55 AM GMT-5 (Panama Time). Since Panama does not observe daylight saving time, this mapping is permanent — no seasonal drift.

> **Note:** The web UI displays "EST" as the timezone label, but the cron expression uses UTC internally. This is confirmed by the trigger editor which states **"times in UTC"**. The EST label is a display convenience only.

> **Note:** Anthropic staggers routine runs by a few minutes to spread server load. Your task may fire at 4:55-5:00 AM rather than exactly 4:55 AM. This is expected and does not affect the warm-up strategy.

### 6. Review Connectors and Permissions

1. Scroll down to the **"Connectors"** section
2. The form states: *"All connected integrations are included by default. Remove any you don't need for this task."*
3. **Remove all connectors** — this task needs no external integrations (no Slack, no Linear, no Google Drive, etc.)
4. If no connectors are listed, leave the section empty

### 7. Create and Activate

1. Review all settings one final time:

   | Setting | Expected Value |
   |---------|---------------|
   | Name | `claude-warm-up` |
   | Prompt | The warm-up prompt with holiday list |
   | Model | Haiku 4.5 |
   | Repository | `claude-warm-up` |
   | Environment | Default |
   | Trigger | `55 9 * * 1-5` (UTC) / Weekdays at 4:55 AM |
   | Connectors | None |

2. Click **"Create"**
3. You will be taken to the routine detail page showing:
   - **Status:** Active (green toggle and badge)
   - **Next run:** The next weekday at 4:55 AM
   - **Repositories:** Your linked repo
   - **Instructions:** The warm-up prompt
   - **Repeats:** Weekday schedule

### 8. Test with Run Now

Before waiting for the first scheduled run, validate the routine immediately:

1. On the routine detail page, click the **"Run now"** button (top-right corner)
2. Wait for the run to complete — it should take under 30 seconds
3. A new entry will appear under **"Runs"** at the bottom of the page with a green checkmark and the label **"MANUAL"**
4. Click on the run entry to see Claude's response
5. **Expected response on a regular workday:** Claude checks the date, confirms it is not a holiday, and responds with 3 sarcastic sentences about warming a token bucket
6. **Expected response on a holiday:** Claude responds with "Holiday. Going back to sleep." and stops

If the run succeeded with the expected response, your warm-up scheduler is fully operational. The next automatic run will fire at 4:55 AM on the next weekday.

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

- **Holiday check first** — If today is a holiday, Claude stops immediately with minimal token usage (~20 tokens).
- **Sunday observance rule** — Handles the Panama labor law where holidays falling on Sunday are observed on Monday. Claude dynamically checks if today is Monday and yesterday was a holiday — no pre-calculated dates needed.
- **3-sentence constraint** — Forces Claude to think (starts the token window) without wasting quota (~100-150 tokens).
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
| Model | Haiku 4.5 | Cheapest model, sufficient for a trivial prompt |
| Cloud Environment | Default (network: None) | No internet access needed |
| MCP Connectors | None | No external integrations required |
| Notifications | None | Check run history manually |

To modify the schedule or prompt, click the edit icon (pencil) on the routine detail page.

---

## Model Selection

**Recommended: Haiku 4.5** (the cheapest and fastest Claude model).

The warm-up task is trivially simple — check a date against a list and write 3 sentences. There is no benefit to using a more expensive model:

| Model | Token Cost | Speed | Recommended? |
|-------|-----------|-------|-------------|
| **Haiku 4.5** | Lowest | Fastest | Yes — best choice for warm-ups |
| Sonnet 4.6 | Medium | Medium | Overkill for this task |
| Opus 4.6 | Highest | Slowest | Save Opus for actual coding work |

Using Haiku for the warm-up means:

- **Lower token consumption** — The warm-up takes a smaller bite out of your daily quota
- **Faster execution** — The task completes in seconds rather than waiting for a larger model
- **Same window activation** — The rolling token window starts regardless of which model processes the prompt
- **Preserve Opus/Sonnet capacity** — Reserve your higher-tier model quota for the real coding sessions during the workday

You can change the model at any time by editing the routine in the web UI (click the pencil icon on the routine detail page, then change the model selector in the prompt box).

---

## Customization Guide

This warm-up scheduler is designed for Panama Time (GMT-5) with Panama holidays, but you can adapt it for any timezone and country:

### Change the timezone

Update the cron expression hour to match your desired UTC offset:

| Local Time (4:55 AM) | UTC Offset | Cron Expression |
|----------------------|-----------|-----------------|
| GMT-5 (Panama) | +5 hours | `55 9 * * 1-5` |
| GMT-6 (Mexico City) | +6 hours | `55 10 * * 1-5` |
| GMT-4 (Dominican Republic) | +4 hours | `55 8 * * 1-5` |
| GMT-3 (Argentina) | +3 hours | `55 7 * * 1-5` |
| GMT+1 (Spain) | -1 hour | `55 3 * * 1-5` |
| GMT+8 (Singapore) | -8 hours | `55 1 * * 1-5` |

> **Warning:** If your timezone observes daylight saving time, the effective local time will shift by one hour during DST periods. Consider using two routines (one for standard time, one for DST) or accept the one-hour drift.

### Change the holiday list

Replace the dates in the prompt with your country's fixed public holidays in MM-DD format. Remove the Sunday observance rule if your country does not observe it.

### Change the warm-up time

If your workday starts at a different time, adjust accordingly. The goal is to fire the warm-up roughly 3-5 hours before you sit down to work, so the token window resets 0-2 hours into your workday.

| Work Start | Suggested Warm-Up | Cron (UTC, for GMT-5) |
|-----------|-------------------|----------------------|
| 7:00 AM | 3:55 AM | `55 8 * * 1-5` |
| 8:00 AM | 4:55 AM | `55 9 * * 1-5` |
| 9:00 AM | 5:55 AM | `55 10 * * 1-5` |
| 10:00 AM | 6:55 AM | `55 11 * * 1-5` |

---

## Verification

After the first scheduled run:

1. Go to [claude.ai/code/scheduled](https://claude.ai/code/scheduled)
2. Click **"Routines"** in the left sidebar
3. Click on the `claude-warm-up` routine
4. Check the **"Runs"** section at the bottom of the page
5. The run should show a green checkmark with the execution timestamp
6. Click on the run entry to view Claude's response
7. Confirm the response is either a 3-sentence sarcastic warm-up or a holiday skip message

---

## Troubleshooting

### Task doesn't appear in the dashboard
Ensure the `claude-warm-up` repository is accessible from your GitHub account linked to Claude. Check that Cloud Scheduled Tasks are enabled for your subscription. If you skipped the GitHub connection step, go to [claude.ai/code](https://claude.ai/code) and click **"Connect to GitHub"** at the bottom of the screen.

### Repository not found when creating the routine
If the repository dropdown shows **"No repositories found"**, your GitHub account is either not connected or the repo is private and requires the Claude GitHub App. Click the **"Install the Claude GitHub app"** link in the dropdown, or make the repository public.

### Task runs but doesn't start the token window
Verify the response contains actual reasoning (the 3-sentence sarcastic response), not just the holiday skip message. A skip message consumes minimal tokens and may not trigger the window.

### Wrong timezone
Panama Time is GMT-5 with no daylight saving adjustments. If the task fires at the wrong time, verify the cron expression in the trigger editor. The correct expression is `55 9 * * 1-5` (UTC). The trigger editor should display **"At 09:55 AM, Monday through Friday (times in UTC)"**.

### Task skips on a regular workday
Check if today's MM-DD accidentally matches one of the 11 holiday dates. Also check if today is Monday following a Sunday holiday (Sunday observance rule).

### Onboarding page appears instead of routines
If you are redirected to `claude.ai/code/onboarding`, you need to create a cloud environment first. See [Step 2](#2-first-time-onboarding-create-a-cloud-environment).

### Run shows error or fails
Check the run detail page for error messages. Common causes:
- Repository access revoked — re-authorize via GitHub settings
- Cloud environment misconfigured — check the environment settings in the routine
- Anthropic service disruption — retry later or check [status.anthropic.com](https://status.anthropic.com)

---

## Development Status

### Completed
- [x] Design specification with holiday handling strategy
- [x] Implementation plan with step-by-step setup instructions
- [x] GitHub repository for Cloud Scheduled Task integration
- [x] Warm-up prompt with embedded holiday list and Sunday observance rule
- [x] Cloud Scheduled Task (routine) creation and activation
- [x] Manual test run verification

---

## License

MIT
