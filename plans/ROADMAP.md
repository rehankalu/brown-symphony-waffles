# brown-symphony-waffles Roadmap — Current State to End State

> **What this file owns.** The project's *intent*: the end state it is driving toward, the
> capabilities that define "done", the coarse phases that get there and why they are
> ordered that way. It changes rarely: when the vision, a phase boundary or a phase's
> status changes.
>
> **What it does not own.** Feature IDs, feature status, branches, PRs, effort points,
> code-level facts (files, routes, schemas) and per-phase close-out detail all live in
> [`META-PLAN.md`](META-PLAN.md), the execution manifest. Implementation detail lives in
> per-feature plan files. How to run and test the project lives in `README.md`, and agent
> operating rules live in the project's `CLAUDE.md` / `AGENTS.md`. This file never copies
> from those; it links to them.
>
> **Direction of reference.** META-PLAN rows cite a phase (`Phase N`). This file never
> lists F-IDs, so adding, splitting or renumbering features never touches it. "What is
> next?" is answered by META-PLAN's ★FOCUS, not here.

## End state (the north star)

Waffles is a private, mobile-first web app for four friends in a Discord server. Each week
a member records a Waffle on their phone, uploads it straight from the phone browser to
Cloudflare R2 with no app install, and the app keeps the untouched original plus one
playback copy (H.264/AAC MP4) that plays on every phone, laptop and browser. When the copy
is ready, a link appears in the Discord Waffles channel. Members sign in with Discord,
browse and play every Waffle by member and week, download any original, and see a yearly
participation heatmap. It replaces the WhatsApp group the friends use today only because
Discord caps free uploads at 20 MB.

### Capabilities

The numbered capabilities define "done". Each is a user-observable outcome, never a
component.

1. **Upload a Waffle from a phone browser.** Up to about 2 GB and 15 minutes, without an app
   install. *Acceptance example:* each member uploads a real 10-minute Waffle from their own
   phone on mobile data and Wi-Fi, and the object in R2 is byte-identical to the original
   (same size and SHA-256).
2. **Keep the original and produce one playback copy.** H.264/AAC MP4 at the original
   resolution and frame rate, visually close to the source, capped at 1080p when that copy
   would exceed 1.5× the original's size. *Acceptance example:* an iPhone HDR clip, an iPhone
   SDR clip and an Android clip each play correctly in Safari on iOS, Chrome on Android and
   Chrome or Edge on Windows, and a 4K clip over the threshold comes out at 1080p.
3. **Post to the Discord Waffles channel when a Waffle is ready to watch.** *Acceptance
   example:* one message appears per ready Waffle, in a test channel first, with a working
   link, and the message is deleted with the Waffle.
4. **Browse and play every Waffle, filtered by member and week.** *Acceptance example:*
   seeking works mid-video on a phone, and deleting a Waffle removes the row and all three
   R2 objects.
5. **Show a participation heatmap: 52 weeks by 4 members for the selected year.**
   *Acceptance example:* the grid matches the archive for seeded test data, including a week
   with two Waffles from one member and a Waffle filed to the previous week.
   <decision: does a member's current streak count the in-progress week before they have posted in it>
6. **Restrict access to members of the Discord server.** *Acceptance example:* all four
   members can sign in from their phones, and a Discord account outside the server cannot.

### Out of scope

- Importing historical Waffles from WhatsApp. The archive starts on launch day.
- NAS backup: a documented follow-on after launch, not MVP.
- Comments, reactions or editing in the app. Discussion stays in Discord.
- Adaptive streaming (HLS/DASH), multiple renditions, or a native mobile app.

## Current state

<!-- Capability-level only: one row per capability above. Code-level detail (what files
exist, what the schema is) is META-PLAN's Baseline; do not restate it here. Update the
"as of" date whenever a row changes, which is normally at a phase-status change. -->

*As of 2026-10-26.*

| # | Capability | State | Delivered by |
|---|---|---|---|
| 1 | Upload from a phone browser | none — not started | Phase 2 |
| 2 | Original plus one playback copy | none — not started | Phase 3 |
| 3 | Discord post when ready | none — not started | Phase 5 |
| 4 | Browse and play | none — not started | Phase 4 |
| 5 | Participation heatmap | none — not started | Phase 6 |
| 6 | Members-only access | none — not started | Phase 1 |

## Principles & constraints

<!-- Durable, project-wide rules that shape *every* phase, written as policy with a short
reason. Verified code facts that must not regress are META-PLAN Standing invariants, and
command-level operating rules (exact backup commands, test commands) belong in CLAUDE.md /
README. Keep each item to one or two sentences. -->

- **Public repo, private data.** Videos and member data live only in R2 and D1; Actions logs
  carry Waffle IDs only and every credential is a secret. *Why:* the repo is public so
  Actions minutes are free, which makes its logs public too.
- **Video bytes never pass through the Worker.** Uploads and playback use presigned R2 URLs.
  *Why:* Worker request limits, and R2 egress is free.
- **The original is kept untouched.** *Why:* the playback copy can be regenerated; the
  original cannot.
- **One group time zone.** Waffle weeks run Wednesday to Tuesday in America/New_York
  (`GROUP_TZ`). *Why:* the cutoff is informal and one member in CT does not justify per-member
  weeks.
- **Accepted risk: <name>.** <The risk the project deliberately carries, and the trigger
  that makes it no longer acceptable.>
- **External dependencies.** Cloudflare (Workers, D1, R2), Discord (OAuth and the channel
  webhook) and GitHub Actions (transcoding). <what happens if each one is unavailable>
- <decision: package manager: npm or pnpm>

## Phases

<!--
Phase numbers are permanent. To insert a phase between N and N+1, use N.5; never renumber.
When a phase finishes, collapse its section to a single line (status + date + what it
delivered). Status markers: `done (YYYY-MM-DD)`, `active`, `planned`.
-->

### Phase 0: Scaffold · active

**Goal.** Stand up the empty app and every Cloudflare resource the later phases build on.
**Advances capabilities:** none directly (foundation for all).

- Repo and a SvelteKit app deployed to `workers.dev`.
- D1 database with the schema and migrations.
- R2 buckets with CORS and the lifecycle rule; secrets set.

**Exit criteria.** The deployed app returns a page, and `db:migrate` runs locally and remotely.

**Why here.** Every later phase deploys to this Worker and reads this schema and these buckets.

### Phase 1: Sign-in · planned

**Goal.** Only the four members can get in. **Advances capabilities:** 6.

- Discord OAuth, guild and `members` checks, sessions, the four members seeded.

**Exit criteria.** All four can sign in from their phones, and a Discord account outside the
server cannot.

**Why here.** Every `/api` route but the transcode callback requires a session.

### Phase 2: Upload · planned

**Goal.** A member gets a full-size Waffle from their phone into R2 intact. **Advances capabilities:** 1.

- Multipart upload to R2 with presigned parts, progress, retry, resume, abort.

**Exit criteria.** Each member uploads a real 10-minute Waffle from their own phone on mobile
data and Wi-Fi, and the object in R2 is byte-identical to the original (size and SHA-256).

**Why here.** Large uploads from phones are the riskiest part, so they are proven first.

### Phase 3: Transcode · planned

**Goal.** Every uploaded Waffle gets a playback copy every device can play. **Advances capabilities:** 2.

- GitHub workflow, probe, remux-or-encode, HDR tone-map, poster, callback.
- Record each test clip's original size, playback size and encode time; tune CRF, preset and
  the 1.5× cap threshold from them.
- <decision: credential the Worker uses for repository_dispatch: fine-grained PAT or GitHub App token>

**Exit criteria.** An iPhone HDR clip, an iPhone SDR clip and an Android clip each play
correctly in Safari on iOS, Chrome on Android and Chrome or Edge on Windows, and a 4K clip
over the threshold comes out at 1080p.

**Why here.** Transcode time and size are the second-biggest risk and need real uploads to test.

### Phase 4: Browse and play · planned

**Goal.** Members watch and manage Waffles in the app. **Advances capabilities:** 4.

- This-week, Archive and Waffle screens; delete and retry.
- <decision: PWA scope: installable only, or offline caching of the app shell>

**Exit criteria.** Seeking works mid-video on a phone, and deleting a Waffle removes the row
and all three R2 objects.

**Why here.** It needs ready playback copies from Phase 3.

### Phase 5: Discord · planned

**Goal.** The group hears about each Waffle where they already talk. **Advances capabilities:** 3.

- Webhook on `ready`, message deleted with the Waffle.

**Exit criteria.** One message appears per ready Waffle, in a test channel first, with a
working link.

**Why here.** It fires on Phase 3's `ready` and links to Phase 4's Waffle screen.

### Phase 6: Participation · planned

**Goal.** Members see who has kept up across the year. **Advances capabilities:** 5.

- `/api/heatmap` and the heatmap screen.

**Exit criteria.** The grid matches the archive for seeded test data, including a week with
two Waffles from one member and a Waffle filed to the previous week.

**Why here.** It is a read-only view over Waffles the earlier phases create.

## Sequencing rationale

<!-- Phase-level dependencies only. Feature-level chains and couplings live in META-PLAN. -->

`0 → 1 → 2 → 3 → {4, 5} → 6`. The riskiest parts (large phone uploads, then transcode time
and size) come first. 4 before 5 is a preference (the Discord link lands on the Waffle
screen), not a hard dependency: 5 needs only Phase 3's `ready` state. Launch follows Phase 6:
point the webhook at the real channel, run two Wednesdays alongside WhatsApp, then retire it.

## Maintaining this file

- **Who updates it:** the `/run-feature` fold-back that merges a phase's last remaining
  feature flips that phase's status, refreshes the Current-state rows it changed, and collapses
  the finished phase to one line, in the same PR. Vision changes are made deliberately by
  the owner, never as a side effect of a feature session.
- **History policy:** this file is forward-looking. Pivots, superseded designs and the
  story of how the plan changed live in git history and decision records, not here.
