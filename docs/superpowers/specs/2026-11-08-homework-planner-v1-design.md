# Homework Planner, Version 1 ("Know + Next"): Design Spec

**Date:** 2026-11-08
**Status:** Approved in brainstorming. Waiting for the user to review this written spec.
**Builder:** The student types all code by hand (Hack Club project). Claude designs, explains and reviews.

---

## 1. Goal

A phone-first web app for one high-school student (Canyons School District, Utah). It:

- pulls assignments, grades and missing work from **Canvas** and **Skyward**;
- merges them into one trusted list;
- tells the student **what to do next and why**;
- helps them **start**, using AI-generated starter steps and a gentle focus mode.

It has to work well for neurodivergent students (ADHD, autism, both) and for neurotypical students.

The student reports four problems. Version 1 targets the first three:

| Problem | Version 1 answer |
|---|---|
| Forgets assignments exist | Automatic Canvas + Skyward sync, plus manual add for things only said in class |
| Doesn't know what's first | Explainable ranking, a "Next up" card, session plans, a Catch-up list |
| Can't get started | Tiny first step from Claude, Start → Focus mode, "I'm stuck: make it smaller" |
| Time gets away | Partly: visual step timer and time estimates. Fully in v2, with calendar blocks |

### Non-goals for version 1

- Google Calendar sync and reminders
- Push notifications
- Extracurriculars and side projects
- Daily check-ins and if-then planning rituals
- ACT and other practice
- Multiple users
- Reading teacher PDFs or attachments

## 2. Roadmap (each step gets its own spec → plan → build cycle)

1. **v1 Know + Next.** This document.
2. **Calendar + reminders.** A Google Calendar secondary calendar (`calendar.app.created` scope), planned work blocks, popup reminders, and plan-change notices.
3. **Projects + extracurriculars.** Robotics and side projects as projects with milestones, plus a capacity check.
4. **Daily check-in.** If-then start plans, guilt-free re-planning, weekly review.
5. **Practice.**
   - Official ACT practice tests timed by section (Enhanced ACT: English 50Q/35m, Math 45Q/50m, Reading 36Q/40m, optional Science 40Q/40m).
   - Error log tagged content / misread / timing / careless.
   - Spaced, interleaved review.
   - "Start practice" sprints on real tasks.
   - **No brain games**: there's no evidence they transfer to schoolwork.

## 3. Research summary (2026-11-08)

### Landscape

- No existing tool combines Canvas assignments with Skyward grades.
- The best-loved neurodivergent-friendly tools share these traits:
  - Visible time (Tiimo)
  - One-thing-at-a-time views
  - Adjustable task breakdown (Goblin.tools)
  - Honest capacity checks (Shovel, Sunsama)
  - Forgiveness instead of punishment
  - Near-zero setup
- What drives people away:
  - Streaks
  - Red overdue walls
  - Notification overload
  - AI reshuffling plans without asking
  - Heavy setup
  - Hidden pricing

### Evidence

- **Strongest:** organizational-skills programs (HOPS, OST). The ingredients worth copying:
  - One capture system
  - Assignments split into dated sub-steps
  - Regular check-ins
  - Rewards
- **Supported:**
  - Implementation intentions ("if-then" plans)
  - Retrieval practice and spacing, which work for ADHD students too
  - Exercise, which improves executive function
- **Autism:**
  - Predictability matters: intolerance of uncertainty is strongly linked to anxiety.
  - Low sensory load.
  - Literal language.
  - Low-pressure wording, because demand-heavy apps backfire.
- **Brain training:** working-memory and brain-game training does not transfer to academic outcomes (Melby-Lervåg & Hulme; Simons et al. 2016; FTC v. Lumosity).
- **Timers:** some ND students find them stressful, which is why the default is gentle and the student chooses at setup.

### Technical constraints

- **Canvas tokens:** student personal access tokens expire in ≤120 days (since Oct 2025). The student's district currently allows them (confirmed by screenshot).
- **Canvas and the browser:** the Canvas API doesn't allow browser apps to call it directly (no CORS), so a server is required.
- **Skyward:**
  - No student API. Scraping is fragile.
  - One session per account, so each sync logs the student out of their browser.
  - Some districts lock accounts after 3 failed logins.
  - The student signs in with username and password (no SSO), so scraping is possible.
  - The student reports Skyward data is **very different** from Canvas, so Skyward is required in v1.
- **Phone:** the student uses Android (Pixel 9 Pro XL). An installable web app (PWA) is fine there.

## 4. Architecture

```
Phone / Chromebook (installable React app)
      │ HTTPS + session cookie
      ▼
Node + TypeScript server (one Docker container on Fly.io, 1 GB volume)
  api/       REST endpoints for the app
  sync/      scheduler → canvas adapter (REST + token)
                       → skyward adapter (login + HTML scrape)
  planner/   ranking: pure functions, no I/O
  ai/        Claude starter-step suggestions
  vault/     encrypted credentials
  db/        SQLite via Drizzle ORM + better-sqlite3
```

### Repo layout

npm workspaces:

- `web/`: Vite + React + TypeScript, `vite-plugin-pwa`, TanStack Query.
- `server/`: Hono, Drizzle, better-sqlite3, cheerio, croner, `@anthropic-ai/sdk`.
- `shared/`: TypeScript types and zod schemas used by both sides.

### Unit boundaries

- **Source adapter.** Interface: `sync(credentials) → { tasks, courseGrades, categoryWeights, errors }`. Adapters fetch and normalize only. They know nothing about ranking or storage. A new source means a new adapter.
- **Merge / store.** Takes adapter output and upserts tasks, matching on `(source, externalId)`. It runs the cross-source merge, writes `task_events`, and applies the disappearance rule.
- **Planner.** Interface: `rank(tasks, grades, weights, now, settings) → RankedTask[]`. Each result carries a score and `reasons: string[]`. Pure and deterministic.
- **Session planner.** Interface: `planSession(rankedTasks, minutes) → { items, breaks, didNotFit }`.
- **AI.** Interfaces: `suggestSteps(taskContext, granularity) → Step[]` and `splitStep(step, taskContext) → Step[]`. It hides the Claude client behind an interface so tests can use a fake.
- **Vault.** Interface: `get(source)` / `set(source, secret, meta)`. The only module that decrypts secrets.

### Sync schedule

| Source | Automatic | Manual |
|---|---|---|
| Canvas | Hourly, 6:00 AM–11:00 PM | Pull to refresh |
| Skyward | 6:30 AM, 3:30 PM, 8:00 PM | Manual refresh, at most once per 10 minutes |

Skyward syncs are kept rare because each one logs the student out of Skyward in their browser.

### App authentication

- Single user. The password is set on first run and stored as an argon2 hash.
- Session: an httpOnly, Secure, SameSite=Strict cookie lasting 90 days.
- Login attempts are rate-limited.

## 5. Data model

| Table | Purpose |
|---|---|
| `courses` | One per class: name, Canvas course ID, Skyward class ID, target grade (default 90%) |
| `tasks` | Title, course, description (plain text), due date, points, category, status (`todo/started/submitted/missing/graded/done`), recoverable flag, URL, origin (`canvas/skyward/manual`), student overrides (estimate, snoozedUntil, hidden, pinned) |
| `task_sources` | Task ↔ source + external ID, latest raw payload, lastSeenAt, consecutiveMisses |
| `course_grades` | Snapshot per course per sync (percent, letter, grading period, source, capturedAt), used for trends |
| `category_weights` | Per course: category → weight |
| `steps` | Per task: order, text, estimateMin, done, origin (`ai/template/user`), approved |
| `task_events` | Change log: new, due date moved, marked missing, grade posted, plan changed. Has a seen flag |
| `sync_runs` | Source, start, end, status (`ok/partial/failed`), counts, error |
| `credentials` | Source, ciphertext, IV, auth tag, metadata (e.g. Canvas token expiry) |
| `settings` | Key-value: timer style, step size, session default, weights, AI model, daily AI cap |

### Course matching

On first run, the app auto-suggests Canvas↔Skyward pairs by fuzzy name match. The student confirms them once. Matches can be edited in Settings.

### Cross-source task merge

- Auto-merge only when **all** of these hold:
  - same matched course;
  - normalized titles are very similar (exact after normalization, or token similarity ≥ 0.8);
  - due dates are within 1 day of each other.
- Which source wins each field:
  - **Skyward:** grade, score, missing flag.
  - **Canvas:** description, due time, URL.
- When uncertain, keep the tasks separate. The student can hide a duplicate.

### Disappearance rule

A task missing from a **successful** sync has its `consecutiveMisses` incremented. At 2, it's archived (not deleted). If it reappears, the counter resets.

## 6. Sync details and failure handling

### Canvas endpoints

- `GET /api/v1/users/self/courses?enrollment_state=active&include[]=total_scores`
- `GET /api/v1/planner/items` (submission flags: missing, late, graded, excused)
- `GET /api/v1/users/self/missing_submissions`
- `GET /api/v1/courses/:id/assignment_groups` (weights)
- `GET /api/v1/users/self/enrollments` (current grades)

Pagination follows `Link` headers.

### Skyward

**First plan task:** a test against the student's real login to find out whether Canyons runs SMS 2.0 or Qmlativ. Then build the matching scraper. Reference implementations:

- `christianfurr/skyward` (SMS 2.0)
- `andrewfraley/skyward-dashboard` (Qmlativ)

Data to extract:

- class list
- current grade per class and term
- assignments with score, due date and category
- missing flags
- category weights

### Failure rules

- Adapters run independently. One failing never blocks the other.
- The last good data is always kept.
- **Sanity checks:** for example, zero classes when the last good sync had N > 0 counts as a scrape failure, not as real data.
- **Skyward login failure:** stop after **1** attempt (never auto-retry) and ask the student to re-enter the password. This avoids the 3-strike lockout.
- **Canvas 401:** show the banner "Canvas key expired — paste a new one" with steps. Also warn 7 days before the stored expiry date.
- **Freshness** is shown for each source in plain language, e.g. "Skyward: 3 hours ago."

## 7. Ranking

### Score

The score is a weighted sum of five parts. Each part also emits a plain-English reason.

| Part | Logic | Example reason |
|---|---|---|
| Urgency | Slack = hours until due − estimated hours. Rises sharply as slack → 0 | "Due tomorrow 8:00 AM · needs ~45 min" |
| Grade impact | Category weight × points ÷ (category points so far + points) | "Worth about 3% of your Chem grade" |
| Class need | Course below target, within 1% of a letter cutoff, or down ≥2% in 7 days | "Chem is 89.2%, 0.8 from an A-" |
| Recovery | Missing and still recoverable (student can mark "can't turn in anymore") | "Missing · late work still accepted" |
| Quick win | Small bonus for estimates ≤15 min. Capped so it never outranks a high-urgency task | "Quick win · ~10 min" |

- **Pinned** tasks always come first.
- **Weights** are editable under Settings → Advanced.
- **Missing inputs** get neutral defaults, e.g. an unknown category weight counts as medium impact.

### Time estimates

- Defaults by type: worksheet 30 min, reading 30 min, essay 120 min, lab 45 min, test → 60 min of studying.
- Claude refines the estimate when it writes steps.
- The student can change any estimate.

### Stability (predictability)

- The list re-sorts only after a sync or when the app opens, never while it's being viewed.
- **Next up** stays until it's finished or skipped. The exception is when another task overtakes it by a large margin *and* is due within 24 hours. That swap creates a `task_events` "plan changed" entry and an on-screen notice. Swaps are never silent.

### Views

- **Next up:** one card. "Not this one" shows 2 alternatives, never the whole list.
- **Plan my session:** time chips 30m / 1h / 2h / 3h+. Greedy fill from the ranked list, with a 5-minute break after each 45 minutes of work, plus a "didn't fit" note.
- **Catch-up:** missing work only, sorted by (estimated grade points recovered ÷ minutes).

## 8. Screens and interaction

Mockups are in `../.superpowers/brainstorm/` (outside the repo).

### Home (layouts B + C combined)

- **Top:** the Next up card, showing title, class, due date, estimate, a one-line reason, the first step, and **Start** / **Not this one** buttons.
- **Below:** "After that", the next 3 tasks.
- The "After that" header holds time chips (30m/1h/2h/3h+). Tapping one turns the list into a session plan in place. Tapping it again goes back.
- **Remember my usual:** on weekdays after 3:00 PM, Home opens in plan mode using the student's most-picked duration. It can be turned off in Settings.

**Tabs:** Home · Catch-up · All · Settings.

### Task detail

- Header: source, due date, points + category weight, editable estimate, "Open in Canvas ↗".
- Claude's suggested steps appear in a dashed "suggested" box with **Keep / Edit / Redo** buttons and a bigger↔tinier step-size slider. Nothing is saved until the student approves it.
- A note says: "Claude reads only the assignment text, never your grades or logins."

### Focus mode

- Dimmed screen, one step in large text, "Step N of M".
- Buttons: **Done with this step** · **I'm stuck: make it smaller** (calls `splitStep`) · **Stop for now** (saves progress, no guilt copy).

### Timer

- **Default:** a gentle shrinking bar per step.
- No sound or flashing.
- When time runs out, the bar fills and the text reads "over by N min, that's fine."
- Tapping the bar hides it.
- **First-time setup** shows a short preview of shrinking bar / count-up / none and the student picks one. Sprint blocks (Pomodoro, default 20 min) are available in Settings.

### All

Every open task, grouped This week / Later / No due date. Filter by class, search, a "What changed" sheet (from `task_events`), and **+ Add task**.

### Catch-up

- Summary line, e.g. "4 missing · about 1 h 20 m to recover about 6% in Spanish."
- Each item can be marked "can't turn in anymore."

### Settings

- Timer style
- Step-size default
- Remember-my-usual
- Target grade per class
- Course matching
- Credentials (status only, with re-enter)
- AI model + daily cap
- Theme (light/dark)
- Advanced weights

### First-time setup (~5 minutes)

1. App password
2. Canvas token + its expiry date
3. Skyward login
4. Confirm course matches
5. Target grades
6. Timer preview

### Calm defaults for everyone

- Muted palette, dark mode, respects `prefers-reduced-motion`.
- No confetti, sounds, streaks or points.
- Literal language with exact dates ("Mon Nov 9, 8:00 AM"), no guilt copy.
- Neutral styling for missing and past-due items (no red walls).
- Stable layout. Every automatic change appears in "What changed."
- Large tap targets.
- **No push notifications in v1.**

## 9. AI (Claude)

**When it's called:**

- Opening a task with no steps.
- **Redo**.
- **I'm stuck: make it smaller**.

Results are stored with the task, so a task typically costs one call.

**Input:**

- Title
- Instructions (Canvas HTML → plain text)
- Class name, points, due date
- Step-size setting

Never grades, credentials or personal data. No attachments in v1.

**Output:** structured outputs (`output_config.format` with a JSON schema) returning 3–7 steps, each `{ text, estimateMin }`. The server re-validates with zod.

**Prompt rule:** step 1 must be a concrete action that takes under 5 minutes.

**Model:**

- Default: `claude-opus-5-5` at `output_config.effort: "low"`. Thinking can't be disabled on Opus 5.5; low effort keeps it fast and cheap.
- Selectable in Settings: `claude-haiku-5-5` for lower cost.
- Server-side refusal fallback enabled (`fallbacks: "default"` with the matching beta header). Confirm the exact request shape against the current claude-api docs at implementation time.

**Cost:** about $0.02–0.04 per call on Opus 5.5, so ~$6–12/month at ~10 calls/day. Haiku 5.5 is about $0.30/month.

**Guardrails:**

- Anthropic console monthly spend limit.
- In-app daily cap (default 40 calls).
- On API error, refusal, or cap reached: fall back to built-in templates by assignment type (worksheet, essay, lab, reading, test prep), with a quiet note.
- No prompt caching: the prompt is below the minimum cacheable size.

**Account:** an Anthropic API account held by an adult (parent), since API accounts require 18+. The API key lives in the server environment only.

## 10. Security

- Credentials are encrypted at rest with AES-256-GCM. The key is a Fly.io secret (`VAULT_KEY`), never committed.
- `.env` is gitignored before the first commit. **The repo is public.**
- The API never returns secrets, only status (e.g. "Canvas connected · expires Mar 3").
- Logs redact tokens and passwords. Raw Skyward HTML goes to a debug table that keeps the latest 3 copies.
- HTTPS only (Fly TLS). Strict cookie settings, as in §4.

## 11. Error handling

- Every failure maps to **one calm message + one action**:
  - Skyward login failed → re-enter password
  - Canvas key expired → paste new one
  - Claude unavailable → using a template
  - Server unreachable → offline mode
- **Offline:** the PWA serves the last fetched data with "Offline · showing data from 3:30 PM."
- **`/health`** (authenticated) shows the last sync result for each source.

## 12. Testing

- **Planner:** Vitest unit tests on fixed scenarios, e.g. "missing 20-pt lab in a class at 89.4% outranks a 5-pt worksheet due Friday." Reason strings are asserted. Stability rule and session fill are tested too.
- **Adapters:** fixture tests against saved, **anonymized** Canvas JSON and Skyward HTML. Sanity checks are tested with broken fixtures.
- **Merge:** confident merge, ambiguous kept separate, two-miss archive, reappearance.
- **API:** integration tests on in-memory SQLite.
- **AI:** fake client. Covers schema validation, daily cap, template fallback. No live calls in tests.
- **UI:** Playwright smoke test at phone viewport: Home → Start → complete a step.

## 13. Open items, resolved during implementation

- **Skyward platform** (SMS 2.0 vs Qmlativ): settled by the first spike task.
- **Canvas missing flags:** confirm whether they appear for this district's external-tool assignments. If not, Skyward's missing flag covers them.
