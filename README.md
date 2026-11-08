# homeworkPlanner

**A calm homework planner that tells you what to do next, helps you start, and helps you get unstuck.**

It pulls your assignments, grades and missing work from **Canvas** and **Skyward** into one list. It picks the one thing to do next and explains why. Then it helps you actually begin. It's built to work well for students with ADHD, autism, or both, and for everyone else too.

> **Status:** In design. The v1 spec is written; code hasn't started yet.
> Full design: [`docs/superpowers/specs/2026-11-08-homework-planner-v1-design.md`](docs/superpowers/specs/2026-11-08-homework-planner-v1-design.md)

---

## Why

Homework usually goes wrong in four ways:

1. **You forget it exists.** It's spread across Canvas, Skyward, and what the teacher said in class.
2. **You don't know what's first.** A pile of work makes it hard to pick.
3. **You can't get started.** The whole task looms at once.
4. **You get stuck.** One step eats the whole evening.

Most planners are just lists. This one is built around **the next small step**, and it aims for *less friction*, not *more pressure*.

## Features (v1)

### Know: everything in one place
- Syncs **Canvas** (assignments, due dates, missing flags, grades) and **Skyward** (gradebook, missing work, category weights).
- Combines duplicates carefully. When it isn't sure two items are the same, it keeps both.
- Lets you **add tasks by hand** for things teachers only say out loud.
- **"What changed"** log, so nothing shifts behind your back.

### Next: what to do first, and why
- **Next up** card: one task, the reason it's first, and its first step.
- Ranking is based on urgency, how much the task affects your grade, which classes need attention, missing work you can still turn in, and quick wins. **Every task shows its reasons in plain English.**
- **Plan my session:** tap 30m / 1h / 2h / 3h+ and get a plan that fits.
- **Catch-up:** missing work, sorted by grade points recovered per minute.

### Start: getting going
- **AI starter steps** (Claude) break an assignment into 3–7 small steps. You approve or edit them before they're saved.
- **Focus mode:** one step at a time, with a gentle visual timer you choose during setup.
- **First-5 rule:** "5 minutes, no need to finish." Starting counts.

### Unstuck: when a step stalls
- A quiet **"Stuck?"** check when a step runs well past its estimate.
- Options: make it smaller · get a hint (explains the idea, **never the answer**) · "is it good enough?" · skip and come back · a 1-minute ACT moment.
- **"I'm frustrated"** button: name the feeling, use your coping move, pause, then choose what's next.

### ACT practice
Practice based on Acceptance and Commitment Therapy, done **when calm** so the skills are ready when you need them:
- 2-minute calm drill
- Your own values
- Noticing thoughts ("I'm having the thought that…")
- Practice scripts

There's also a weekly **stuck-minutes** trend. Progress is shown as less friction, not as speed.

> This supports work with a therapist or counselor. It isn't therapy, doesn't diagnose, and makes no clinical claims.

### Parent view
- A parent login can see progress.
- Anything shared is clearly labeled **"Shared with parent"** on the student's screens. Nothing is hidden.

## Design principles

The research behind these is summarized in the spec.

- **One thing at a time.** The full list is always one tap away, never forced on you.
- **Calm by default.** Muted colors and dark mode. Respects reduce-motion. No confetti, sounds, streaks, or points.
- **No guilt.** Missing work is "can still turn in," not a wall of red. Stopping saves your spot.
- **Predictable.** The layout never rearranges itself, and the app tells you when the plan changes.
- **Plain language.** Exact dates ("Mon Nov 9, 8:00 AM"), never "soon."
- **No push notifications in v1.** Reminders come later, through Google Calendar, on your terms.
- **You stay in control.** The AI only suggests and you approve. You pick your timer, step size and settings.

## Roadmap

| Step | What | Status |
|---|---|---|
| **v1a** | Homework core: sync, ranking, Next up, starter steps, Focus mode | Designed |
| **v1b** | Unstuck toolkit + frustration reset | In design |
| **v1c** | ACT practice tab | In design |
| **v1d** | Parent view | In design |
| 2 | Google Calendar work blocks + reminders | Planned |
| 3 | Projects and extracurriculars (robotics, side projects) | Planned |
| 4 | Daily check-in + growth goals (pick 2–3 areas, weekly plan and review) | Planned |

## Tech stack

- **App:** React + TypeScript (Vite), installable as a phone app (PWA)
- **Server:** Node + TypeScript ([Hono](https://hono.dev)), SQLite via [Drizzle](https://orm.drizzle.team)
- **Sync:** Canvas REST API (personal access token); Skyward by reading its web pages (no official student API)
- **AI:** Claude API (`@anthropic-ai/sdk`) with structured JSON output
- **Hosting:** one Docker container on [Fly.io](https://fly.io)
- **Tests:** Vitest + Playwright

```
web/      React app (phone-first, installable)
server/   API, sync jobs, ranking, AI, encrypted credential vault
shared/   Types and schemas used by both
docs/     Design specs and plans
```

## Privacy and security

- Your Canvas token and Skyward password live **only on the server**, encrypted (AES-256-GCM). They're never sent to the browser or committed to this repo.
- Claude only sees **assignment text**. It never sees grades, logins, frustration notes, or ACT entries.
- Personal goals, reflections and logs stay in the app's private database. They never go into the code, test data, or logs.
- `.env` is gitignored. **This repo is public**, so never commit secrets.

## Getting started

*Coming once v1a is built.* You'll need:

- Node.js 22+
- A Canvas access token (Canvas → Account → Settings → **+ New Access Token**). Student tokens expire within 120 days.
- Skyward login
- An Anthropic API key (account holder must be 18+)
- A Fly.io account (for deploying)

## How this is built

This is a Hack Club project. The design and planning were done together with Claude (Anthropic). **All of the application code is written by hand by the student.**
