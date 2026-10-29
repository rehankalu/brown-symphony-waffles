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

1. **Upload a Waffle from a phone browser.** Up to 5 GiB (about 10 minutes of 4K60), without an
   app install. *Acceptance:* Phase 2's exit criteria.
2. **Keep the original and produce one playback copy.** H.264/AAC MP4 at the original
   resolution and frame rate, visually close to the source, capped at 1080p when that copy
   would exceed 1.5× the original's size. *Acceptance:* Phase 3's exit criteria.
3. **Post to the Discord Waffles channel when a Waffle is ready to watch.** *Acceptance:*
   Phase 5's exit criteria.
4. **Browse and play every Waffle, filtered by member and week.** *Acceptance:* Phase 4's
   exit criteria.
5. **Show a participation heatmap: 52 weeks by 4 members for the selected year.**
   *Acceptance:* Phase 6's exit criteria.
6. **Restrict access to members of the Discord server.** *Acceptance:* Phase 1's exit
   criteria.

### Out of scope

- Importing historical Waffles from WhatsApp. The archive starts on launch day.
- NAS backup: a follow-on after launch, not MVP (see
  [SPEC Follow-on: NAS backup](../docs/SPEC.md#follow-on-nas-backup)).
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

- **The project rules.** The rules in [CLAUDE.md § Project rules](../CLAUDE.md#project-rules)
  shape every phase. *Why each:*
  - Waffle week: the cutoff is informal, and one member in CT does not justify per-member weeks.
  - Public repo: Actions minutes are free on a public repo, which makes its logs public too.
  - Video bytes skip the Worker: Worker request limits, and R2 egress is free.
  - Original untouched: the playback copy can be regenerated; the original cannot.
- **Accepted risk: R2 holds the only copy.** Until the NAS backup exists, an R2 account
  problem or a deleted Waffle loses its original for good. *Trigger:* retiring the WhatsApp
  group at launch ends the de facto backup, so the NAS follow-on starts then.
- **External dependencies.** Cloudflare (Workers, D1, R2), Discord (OAuth and the channel
  webhook) and GitHub Actions (transcoding). If Cloudflare is down, the whole app is down. If
  Discord is down, new sign-ins fail ([SPEC Auth](../docs/SPEC.md#auth)) but existing sessions
  work. How uploads, Discord posts and transcodes recover:
  [SPEC Upload](../docs/SPEC.md#upload), [SPEC Discord notification](../docs/SPEC.md#discord-notification)
  and [SPEC Transcode](../docs/SPEC.md#transcode).
- **npm is the package manager.** *Why:* it ships with Node, is what Wrangler and SvelteKit
  document, and pnpm's install-speed gains do not matter for one small app.

## Phases

<!--
Phase numbers are permanent. To insert a phase between N and N+1, use N.5; never renumber.
When a phase finishes, collapse its section to a single line (status + date + what it
delivered). Status markers: `done (YYYY-MM-DD)`, `active`, `planned`.
-->

### Phase 0: Scaffold · active

**Goal.** Stand up the empty app and every Cloudflare resource the later phases build on.
**Advances capabilities:** none directly (foundation for all).

- Repo and a SvelteKit app deployed to `workers.dev`, served through the custom Worker entry
  (`src/worker.ts`, passing `fetch` through to the adapter), the Wrangler config's `main` and
  its `[build] command`, as in **Worker entry** in [SPEC Architecture](../docs/SPEC.md#architecture).
- D1 database with the schema and migrations.
- R2 buckets with CORS and the lifecycle rule; secrets set.

**Exit criteria.** The deployed app returns a page through the custom Worker entry, and
`db:migrate` runs locally and remotely.

**Why here.** Every later phase deploys to this Worker and reads this schema and these buckets.

### Phase 1: Sign-in · planned

**Goal.** Only the four members can get in. **Advances capabilities:** 6.

- Discord OAuth, guild and `members` checks, sessions, the four members seeded.

**Exit criteria.** All four can sign in from their phones, and a Discord account outside the
server cannot.

**Why here.** Every `/api` route but the transcode callback requires a session.

### Phase 2: Upload · planned

**Goal.** A member gets a full-size Waffle from their phone into R2 intact. **Advances capabilities:** 1.

- Multipart upload to R2 with presigned parts, progress, retry, resume, abort, the
  failed-upload messages and the abandoned-upload sweep, as in
  [SPEC Upload](../docs/SPEC.md#upload).

**Exit criteria.** Each member uploads a real 10-minute Waffle from their own phone on mobile
data and Wi-Fi, and the object in R2 is byte-identical to the original (size and SHA-256).

**Why here.** Large uploads from phones are the riskiest part, so they are proven first.

### Phase 3: Transcode · planned

**Goal.** Every uploaded Waffle gets a playback copy every device can play. **Advances capabilities:** 2.

- GitHub workflow, probe, remux-or-encode, HDR tone-map, poster, callback.
- Record each test clip's original size, playback size and encode time; tune CRF, preset and
  the 1.5× cap threshold from them. Merge each tuning change to `main` before dispatching it
  (rule: **Dispatch target** in [SPEC Transcode](../docs/SPEC.md#transcode)).
- The Worker starts the workflow with `workflow_dispatch`; the dispatch token, the daily
  token expiry check and rejected dispatches are as in [SPEC Transcode](../docs/SPEC.md#transcode).
  *Why `workflow_dispatch`:* it needs only the Actions: write permission SPEC names, while
  `repository_dispatch` needs Contents: write, which on a public repo lets a leaked
  token change code the workflow runs and expose its secrets, the R2 keys among them.
- The token expiry check's `scheduled` handler, added to Phase 0's custom Worker entry as in
  **Worker entry** in [SPEC Architecture](../docs/SPEC.md#architecture).
- The owner-only `POST /api/waffles/:id/retry` route, tested through the API; Phase 4 adds
  its button.
- `GET /api/waffles` and `GET /api/waffles/:id` as in
  [SPEC API routes](../docs/SPEC.md#api-routes-all-under-the-app-origin), with the lazy check
  from **Stuck jobs** in [SPEC Transcode](../docs/SPEC.md#transcode), tested through the API;
  Phase 4 builds on them.
- The CI check and the validation script with its tests from **Untrusted inputs** in
  [SPEC Transcode](../docs/SPEC.md#transcode).
- Land the CI check first, make it a required status check on `main` as **Untrusted inputs**
  states, then merge the transcode workflow. Classic branch protection lists only checks that
  have reported in the past 7 days, so set the rule soon after the check's first run.

**Exit criteria.** An iPhone HDR clip, an iPhone SDR clip and an Android clip, each played
from the presigned playback URL that `GET /api/waffles/:id` returns, play correctly in
Safari on iOS, Chrome on Android and Chrome or Edge on Windows, and a 4K clip over the
threshold comes out at 1080p. An H.264 SDR clip with no audio stream takes the remux path
and its playback copy has no audio track, and an Android clip that ffprobe reports as
`yuvj420p` takes the remux path and its playback copy keeps the original's `pix_fmt` and
`color_range` (rule: the remux path in [SPEC Transcode](../docs/SPEC.md#transcode)). A
corrupt file ends `failed` with `probe:` or `encode:` and the first non-empty line of the
tool's stderr as its stored error, while the run's log shows only the fixed line naming the
Waffle ID and category (rule: **Failure output** in
[SPEC Transcode](../docs/SPEC.md#transcode)). The no-audio and corrupt clips are made with
ffmpeg (for example with `-an`, and by truncating a file), and no test clip is committed.
With an invalid dispatch token, a completed upload ends `failed` with the token message from
**Rejected dispatch** in [SPEC Transcode](../docs/SPEC.md#transcode), the token appears in
no stored error, API response or Worker log, and the original stays in R2. After a rejected dispatch, another
member's Retry is refused while the Waffle is still `failed`, leaving it `failed` with
nothing dispatched, and then the owner's Retry takes the Waffle to `ready`. A Waffle whose
`processing_at` is backdated past 150 minutes reads as `failed` with the **Stuck jobs**
message from [SPEC Transcode](../docs/SPEC.md#transcode) in both GET routes, and a `ready`
Waffle with the same `processing_at` stays `ready`.
With fixture GitHub responses, the token expiry check posts one alert for a token
expiring 13 days after the run and one each for a 401, a 403 and a 404, and posts nothing
for a token expiring 15 days after the run, one expiring in a far-future year such as 2099,
a missing expiry header, an unparseable one, a 5xx or a network error; the unparseable
header, the 5xx and the network error each write their fixed `token check:` line to the
Worker log, and a failed alert post writes its `token check alert failed:` line and is not
retried. The 13-day, 15-day and 401 runs write no `token check` line. No `token check` line
or alert holds the token, the webhook URL or a fixture response body (rules: **Token expiry
check** in [SPEC Transcode](../docs/SPEC.md#transcode)).
Under `wrangler dev --test-scheduled`, a request to `/__scheduled` runs the check through the
`scheduled` handler of **Worker entry** in [SPEC Architecture](../docs/SPEC.md#architecture),
and with an invalid token it posts the invalid-token alert, while the same Worker still serves
a page and an `/api` route.
A hand dispatch with `env=prod` and a `waffle_id` that is not a ULID and holds shell
metacharacters (for example `$(id)`) fails before downloading anything, the validation
script's output is only the fixed rejection message, and no callback arrives; a hand dispatch
with `env=other` starts no run. A hand dispatch with `env=dev` and the ID of a Waffle
uploaded under `wrangler dev` writes its playback copy to `waffles-dev` and sends no
callback. While no Waffle is `processing`, with the `CALLBACK_BASE_URL` secret deleted (so
it expands empty), and then set to an `http://` URL, a hand dispatch with `env=prod` and a
well-formed ULID that matches no Waffle fails before downloading anything, its log shows
only the fixed message, and no callback arrives; the secret is then set back to its
`https://` value and one upload's callback arrives (rule: **Dispatch target** in
[SPEC Transcode](../docs/SPEC.md#transcode)). With the `TRANSCODE_CALLBACK_SECRET`
repository secret set to a wrong value, an upload's run log ends with
`callback: 401 <waffle id>`, the Worker log shows `callback rejected: 401` with no Waffle ID,
and the Waffle stays `processing`; the secret is then set back (rule: **Callback** in
[SPEC Transcode](../docs/SPEC.md#transcode)). The CI check fails on a branch that adds
`${{ github.event.inputs.waffle_id }}` inside a `run: |` block, and on one that adds
`set -ex`, and GitHub refuses to merge either branch into `main`. The validation script's
tests from **Untrusted inputs** pass in CI.

**Why here.** Transcode time and size are the second-biggest risk and need real uploads to test.

### Phase 4: Browse and play · planned

**Goal.** Members watch and manage Waffles in the app. **Advances capabilities:** 4.

- This-week (with its Unfinished uploads), Archive and Waffle screens and the player; delete
  and the Retry button.
- The PWA is installable only, as scoped in [SPEC UI](../docs/SPEC.md#ui). *Why:* nothing
  works offline anyway, and a caching service worker risks a stale shell after a
  deploy and intercepted R2 range requests.

**Exit criteria.** Seeking works mid-video on a phone, and deleting a Waffle removes the row
and all three R2 objects. Added to the home screen from Safari on iOS and Chrome on Android,
the app opens standalone with its name and icon, and
`navigator.serviceWorker.getRegistrations()` returns an empty list.
An upload stopped mid-way shows under Unfinished uploads on its owner's This-week screen
only and does not fill their avatar; Resume on the same phone finishes it, and Cancel upload removes it.

**Why here.** It needs ready playback copies and the read routes from Phase 3.

### Phase 5: Discord · planned

**Goal.** The group hears about each Waffle where they already talk. **Advances capabilities:** 3.

- Webhook on `ready`, message deleted with the Waffle, and failed posts handled as in
  [SPEC Discord notification](../docs/SPEC.md#discord-notification).

**Exit criteria.** One message appears per ready Waffle, in a test channel first, with a
working link, and the message is deleted with the Waffle. A revoked webhook leaves the
Waffle `ready` with Post to Discord again; once `DISCORD_WEBHOOK_URL` holds a working
webhook again, Post to Discord again posts one message and clears `discord_error`. A
duplicate callback posts once.

**Why here.** It fires on Phase 3's `ready` and links to Phase 4's Waffle screen.

### Phase 6: Participation · planned

**Goal.** Members see who has kept up across the year. **Advances capabilities:** 5.

- `/api/heatmap` and the heatmap screen.

**Exit criteria.** The grid matches the archive for seeded test data, including a week with
two Waffles from one member, a Waffle filed to the previous week, and Waffles at Tuesday
23:59:59 and Wednesday 00:00:00 ET landing in consecutive weeks, in both an EST and an EDT
week (seeds get `week_start` from `uploaded_at` through the Waffle-week function). A member
who has not yet posted in the current week keeps their streak, posting in it adds one, a
past week with no Waffle from them breaks it, a run spanning the last week of one year and
the first week of the next counts as one streak, a run from December into February counts
its December weeks toward the Best streak of the year it ends in, a 53-week year such as
2025 shows 53 columns and a run through its 53rd week carries into 2026's first week, and a
past year shows "Best YYYY streak" instead of the current streak (rules: SPEC
[Waffle week](../docs/SPEC.md#waffle-week) and [UI](../docs/SPEC.md#ui)).

**Why here.** It is a read-only view over Waffles the earlier phases create.

## Sequencing rationale

<!-- Phase-level dependencies only. Feature-level chains and couplings live in META-PLAN. -->

`0 → 1 → 2 → 3 → {4, 5} → 6`. The riskiest parts (large phone uploads, then transcode time
and size) come first. 4 before 5 is a preference (the Discord link lands on the Waffle
screen), not a hard dependency: 5 needs only Phase 3's `ready` state. Launch follows Phase 6:
point the webhook at the real channel, run two Wednesdays alongside WhatsApp, then retire it.

The [domain cutover](../docs/SPEC.md#domain-cutover-after-the-prototype-is-proven) is not a
phase; it runs once the prototype is proven, since only the custom domain can carry the
callback's zone rate limit. **Exit check:** the rule reads back from the zone's rate-limiting
rules (dashboard or API) as SPEC states it, and a burst of 11 unsigned POSTs to
`/api/internal/transcode-callback` within 10 seconds gets a 429 for the 11th, with no Worker
log line for it (rule: **Callback rate limit** in
[SPEC Environments](../docs/SPEC.md#environments)).

## Maintaining this file

- **Who updates it:** the `/run-feature` fold-back that merges a phase's last remaining
  feature flips that phase's status, refreshes the Current-state rows it changed, and collapses
  the finished phase to one line, in the same PR. Vision changes are made deliberately by
  the owner, never as a side effect of a feature session.
- **History policy:** this file is forward-looking. Pivots, superseded designs and the
  story of how the plan changed live in git history and decision records, not here.
