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
5. **Show a participation heatmap: 52 or 53 weeks by 4 members (one column per
   [Waffle week](../CLAUDE.md#project-rules) in the selected year).** *Acceptance:* Phase 6's
   exit criteria.
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
- R2 buckets with CORS and the lifecycle rule, and the R2 API token, with the Worker secrets
  `R2_ACCESS_KEY_ID` and `R2_SECRET_ACCESS_KEY` from it and `R2_ACCOUNT_ID` (see
  [SPEC Environments](../docs/SPEC.md#environments)). Each later phase sets the secrets and
  `[vars]` it first needs, in the Worker and in `.dev.vars` as that section describes.

**Exit criteria.** The deployed app returns a page through the custom Worker entry, and
`db:migrate` runs locally and remotely. Read back from R2 (dashboard or Wrangler), each
bucket's CORS rule allows `PUT` and the `content-type` header from its expected origins (the
workers.dev origin on `waffles`, the local `APP_ORIGIN` on `waffles-dev`) and exposes `ETag`,
and its lifecycle rule aborts multipart uploads left incomplete for 7 days
(rules: [SPEC Upload](../docs/SPEC.md#upload) and
[Storage layout](../docs/SPEC.md#storage-layout-r2-bucket-waffles-private)).

**Why here.** Every later phase deploys to this Worker and reads this schema and these buckets.

### Phase 1: Sign-in · planned

**Goal.** Only the four members can get in. **Advances capabilities:** 6.

- Discord OAuth, guild and `members` checks, sessions, the four members seeded.
- The cross-site Origin and JSON checks from [SPEC Auth](../docs/SPEC.md#auth), as a server
  hook that every later POST and DELETE route passes through.
- The Discord application, with the Worker secrets `DISCORD_CLIENT_ID`,
  `DISCORD_CLIENT_SECRET` and `DISCORD_GUILD_ID`.
- The `APP_ORIGIN` var and both Discord redirect URIs, as in
  [SPEC Environments](../docs/SPEC.md#environments).

**Exit criteria.** All four can sign in from their phones, and a Discord account outside the
server cannot. Once in the server, that account (the admin's second account, joined for the check
and removed after; no seeded `members` row is touched) still has no row in `members`, so it sees
the "not a member" page and gets no session (rule: [SPEC Auth](../docs/SPEC.md#auth)). A member
signed in before a redeploy of the Worker is still signed in after it. Opening `/auth/callback`
with a made-up `code` and `state`, without starting sign-in first, shows the "Sign-in failed"
page, not "not a member", sets no session cookie and writes `auth callback: state mismatch` to
the Worker log; tests cover the other callback failures. A `POST /auth/logout` with a valid
session cookie and an `Origin` of another site gets 403 `bad_origin` and the session still works;
each later route's cross-site cases follow with the phase that builds it (Phases 2 to 5).

**Why here.** Every `/api` route but the transcode callback requires a session.

### Phase 2: Upload · planned

**Goal.** A member gets a full-size Waffle from their phone into R2 intact. **Advances capabilities:** 1.

- Multipart upload to R2 with presigned parts, progress, retry, resume, abort, the
  failed-upload messages and the abandoned-upload sweep, as in
  [SPEC Upload](../docs/SPEC.md#upload).
- The `TRANSCODE_ENV` and `GROUP_TZ` vars, as in
  [SPEC Environments](../docs/SPEC.md#environments): the Worker's first R2 calls need
  `TRANSCODE_ENV`, and `week_start` first needs `GROUP_TZ`.
- `GET /api/waffles` and `GET /api/waffles/:id` as in
  [SPEC API routes](../docs/SPEC.md#api-routes-all-under-the-app-origin), without the lazy
  check from **Stuck jobs** in [SPEC Transcode](../docs/SPEC.md#transcode), which Phase 3
  adds; tested through the API. Phase 4 builds its screens on them.
- The Upload screen up to its `processing` poll, and the owner's Unfinished uploads on the
  This-week screen, as in [SPEC UI](../docs/SPEC.md#ui). The Upload screen's `ready` link
  and `failed` Retry come with Phase 4, as does the rest of the This-week screen.
- `complete` stops at `processing`: it sets `status` and `processing_at` and dispatches
  nothing until Phase 3 adds the dispatch, and its dispatch tests land there. Once Phase 3
  deploys, Phase 2's Waffles turn `failed` under **Stuck jobs** in
  [SPEC Transcode](../docs/SPEC.md#transcode), and each owner's Retry recovers them.

**Exit criteria.** Each member uploads a real 10-minute Waffle from their own phone on mobile
data and Wi-Fi, and the object in R2 is byte-identical to the original (size and SHA-256).
On a phone, an upload whose connection is cut mid-way shows under Unfinished uploads on its
owner's This-week screen only and finishes with Resume, one whose tab is reloaded finishes
when the same file is picked again, and Cancel upload removes the row and the multipart
upload (an S3 `ListMultipartUploads` call, with the R2 keys read from environment variables
and pasted nowhere, no longer lists it); one file close to 5 GiB also passes the size and
SHA-256 check. After `complete`, the Upload screen shows `processing` and polls, and with
the network cut for 3 polls it shows its connection note. Both GET routes return 401
without a session. Under `wrangler dev`, an `uploading` row whose `uploaded_at` is set back
8 days is gone after a `GET /api/waffles`, with its multipart upload aborted, and one set
back just under 7 days stays. These pass their cases in
[SPEC Tests worth writing](../docs/SPEC.md#build-plan) in Phase 2: the 409 `cancelled`
re-read, the upload checks, the cross-site checks on the upload routes, the abandoned-upload
sweep, `uploading` Waffles, the Upload screen's polling (its 404 and 401 included) and the
`Content-Disposition` name. Phase 2's test Waffles stay until Phase 4's Delete removes them.

**Why here.** Large uploads from phones are the riskiest part, so they are proven first.

### Phase 3: Transcode · planned

**Goal.** Every uploaded Waffle gets a playback copy every device can play. **Advances capabilities:** 2.

- GitHub workflow, probe, remux-or-encode, HDR tone-map, poster, callback.
- The fine-grained PAT, created as in [SPEC Transcode](../docs/SPEC.md#transcode); the
  Worker secrets `GITHUB_DISPATCH_TOKEN`, `TRANSCODE_CALLBACK_SECRET` and
  `DISCORD_WEBHOOK_URL` (the test channel's webhook); the `GITHUB_REPO` and
  `GITHUB_WORKFLOW` vars (the latter matching the workflow's file name); and the repository
  secrets the workflow needs, `R2_ACCOUNT_ID` among them, as listed in
  [SPEC Environments](../docs/SPEC.md#environments).
- Once the workflow-file read in the exit criteria passes, merge the workflow file to `main`
  before any Phase 3 dispatch check, and each tuning change before dispatching it (rule:
  **Dispatch target** in [SPEC Transcode](../docs/SPEC.md#transcode)).
- Record each test clip's original size, playback size and encode time; tune CRF, preset and
  the 1.5× cap threshold from them.
- `complete` and Retry start the workflow with `workflow_dispatch`; the dispatch token, the
  daily token expiry check and rejected dispatches are as in
  [SPEC Transcode](../docs/SPEC.md#transcode).
  *Why `workflow_dispatch`:* it needs only the Actions: write permission SPEC names, while
  `repository_dispatch` needs Contents: write, which on a public repo lets a leaked
  token change code the workflow runs and expose its secrets, the R2 keys among them.
- The token expiry check's `scheduled` handler, added to Phase 0's custom Worker entry as in
  **Worker entry** in [SPEC Architecture](../docs/SPEC.md#architecture).
- A minimal webhook poster for the token alerts, ahead of Phase 5: one post to
  `DISCORD_WEBHOOK_URL` with the `allowed_mentions` rule from
  [SPEC Discord notification](../docs/SPEC.md#discord-notification). Phase 5 builds the
  Waffle posts on it.
- The owner-only `POST /api/waffles/:id/retry` route, tested through the API; Phase 4 adds
  its buttons.
- The lazy check from **Stuck jobs** in [SPEC Transcode](../docs/SPEC.md#transcode), added
  to Phase 2's `GET /api/waffles` and `GET /api/waffles/:id` and tested through the API.
- The workflow follows the rules of **Untrusted inputs**, **Workflow permissions** and
  **Untrusted file** in [SPEC Transcode](../docs/SPEC.md#transcode), its input check among
  them.

**Exit criteria.** An iPhone HDR clip, an iPhone SDR clip and an Android clip, each played
from the presigned playback URL that `GET /api/waffles/:id` returns, play correctly in
Safari on iOS, Chrome on Android and Chrome or Edge on Windows, and a 4K clip over the
threshold comes out at 1080p. An H.264 SDR clip with no audio stream takes the remux path
and its playback copy has no audio track, and an Android clip that ffprobe reports as
`yuvj420p` takes the remux path and its playback copy keeps the original's `pix_fmt` and
`color_range` (rule: the remux path in [SPEC Transcode](../docs/SPEC.md#transcode)). A
10-bit H.264 clip and an H.264 clip with MP3 audio take the re-encode path, and a clip at
1080p or below whose re-encoded copy exceeds 1.5× the original (for example an HEVC noise
clip encoded at a low bitrate) is encoded again 2 CRF higher at its own resolution (rule: the
size cap in [SPEC Transcode](../docs/SPEC.md#transcode)). A
corrupt file ends `failed` with `probe:` or `encode:` and the first non-empty line of the
tool's stderr as its stored error, while the run's log shows only the fixed line naming the
Waffle ID and category (rule: **Failure output** in
[SPEC Transcode](../docs/SPEC.md#transcode)). The no-audio and corrupt clips are made with
ffmpeg (for example with `-an`, and by truncating a file), as are the 10-bit, MP3 and noise
clips, and no test clip is committed. An `.m4v` clip made with ffmpeg reaches `ready`,
while a Matroska file and an `ffconcat version 1.0` file naming another file, each renamed
to `clip.mp4`, end `failed` with a `probe:` error and start no ffmpeg (rule: **Untrusted
file** in [SPEC Transcode](../docs/SPEC.md#transcode)).
With an invalid dispatch token, a completed upload ends `failed` with the token message from
**Rejected dispatch** in [SPEC Transcode](../docs/SPEC.md#transcode), the token appears in
no stored error, API response or Worker log, and the original stays in R2. After a rejected dispatch, another
member's Retry is refused while the Waffle is still `failed`, leaving it `failed` with
nothing dispatched, and then the owner's Retry takes the Waffle to `ready`. A Waffle whose
`processing_at` is backdated past 150 minutes reads as `failed` with the **Stuck jobs**
message from [SPEC Transcode](../docs/SPEC.md#transcode) in both GET routes, and a `ready`
Waffle with the same `processing_at` stays `ready`. The Retry route passes its cross-site
cases in [SPEC Tests worth writing](../docs/SPEC.md#build-plan).
With fixture GitHub responses and a dummy token, the token expiry check sends the GitHub
request headers from [SPEC Transcode](../docs/SPEC.md#transcode) and posts one alert for a token
expiring 13 days after the run and one each for a 401, a 403 and a 404, and posts nothing
for a token expiring 15 days after the run, one expiring in a far-future year such as 2099,
a missing expiry header, an empty one, an unparseable one, a 5xx or a network error; the
missing and empty headers, the unparseable one, the 5xx and the network error each write
their fixed `token check:` line to the Worker log (`token check: no expiry header` for the
missing and empty ones), and a failed alert post writes its `token check alert failed:` line
and is not retried. The 13-day, 15-day and 401 runs write no `token check` line. No
`token check` line or alert holds the token, the webhook URL or a fixture response body
(rules: **Token expiry check** in [SPEC Transcode](../docs/SPEC.md#transcode)).
Under `wrangler dev --test-scheduled`, a request to `/__scheduled` runs the check through the
`scheduled` handler of **Worker entry** in [SPEC Architecture](../docs/SPEC.md#architecture),
and with an invalid token it posts the invalid-token alert, while the same Worker still serves
a page and an `/api` route. The deployed Worker's triggers (dashboard or Wrangler) list the
`0 14 * * *` Cron Trigger.
With the valid token, a deployed upload's `complete` gets GitHub's 204: a
`transcode <waffle id> (prod)` run appears in the Actions list and the Waffle stays
`processing` with no `error`. The admin then sends the token expiry check's request by hand
from a local shell with `curl -sI` and the same four headers, the token read from an
environment variable, the output filtered to the `github-authentication-token-expiration`
line, which must be present; the token and the other headers are pasted nowhere (rule: the
GitHub request headers in [SPEC Transcode](../docs/SPEC.md#transcode)).
A hand dispatch with `env=prod` and a `waffle_id` that is not a ULID and holds shell
metacharacters (for example `$(id)`) fails before downloading anything, the input check's
output is only the fixed rejection message, and no callback arrives; a hand dispatch
with `env=other` starts no run. A hand dispatch with `env=dev` and the ID of a Waffle
uploaded under `wrangler dev` writes its playback copy to `waffles-dev` and sends no
callback. While no Waffle is `processing`, with the `CALLBACK_BASE_URL` secret deleted (so
it expands empty), and then set to an `http://` URL, a hand dispatch with `env=prod` and a
well-formed ULID that matches no Waffle fails before downloading anything, its log shows
only the fixed message, and no callback arrives; the secret is then set back to its
`https://` value and one upload's callback arrives. With the `TRANSCODE_CALLBACK_SECRET`
repository secret deleted, the same hand dispatch also fails before downloading anything
with only the fixed message and sends no callback; the secret is then set back (rule:
**Dispatch target** in [SPEC Transcode](../docs/SPEC.md#transcode)). With the
`TRANSCODE_CALLBACK_SECRET` repository secret set to a wrong value, an upload's run log ends
with `callback: 401 <waffle id>`, the Worker log shows `callback rejected: 401` with no Waffle ID,
and the Waffle stays `processing`; the secret is then set back (rule: **Callback** in
[SPEC Transcode](../docs/SPEC.md#transcode)). The admin finds that run by its Waffle ID with
README [Transcodes that did not report back](../README.md#transcodes-that-did-not-report-back).
A callback signed by hand with the dev-only `TRANSCODE_CALLBACK_SECRET` (read from an
environment variable and pasted nowhere), a current timestamp and a well-formed ULID that
matches no Waffle gets 401, not the `no_waffle` 404, from the deployed Worker, which logs
`callback rejected: 401`. Before launch, while no Waffle is `processing` and members are not
uploading, the admin runs each rotation runbook in README
[Operations](../README.md#operations) once end to end (dispatch token, R2 keys, callback
secret), and an upload after each reaches `ready`. A read of the workflow file before each
merge finds no inline interpolation of an input or secret, no shell tracing, top-level
`permissions: contents: read` with no job-level `permissions:`, every `uses:` pinned to a
full commit SHA, and `-protocol_whitelist file -f mov` with no `-enable_drefs` on every
ffprobe and ffmpeg command that reads `original` (rules: **Untrusted inputs**, **Workflow
permissions** and **Untrusted file** in [SPEC Transcode](../docs/SPEC.md#transcode)).

**Why here.** Transcode time and size are the second-biggest risk and need real uploads to test.

### Phase 4: Browse and play · planned

**Goal.** Members watch and manage Waffles in the app. **Advances capabilities:** 4.

- The rest of the This-week screen (around Phase 2's Unfinished uploads), the Archive and
  Waffle screens and the player; delete; the Retry button on the Upload and Waffle screens,
  and the Upload screen's `ready` link.
- The PWA is installable only, as scoped in [SPEC UI](../docs/SPEC.md#ui). *Why:* nothing
  works offline anyway, and a caching service worker risks a stale shell after a
  deploy and intercepted R2 range requests.

**Exit criteria.** Seeking works mid-video on a phone, and deleting a Waffle removes the row
and all three R2 objects. Added to the home screen from Safari on iOS and Chrome on Android,
the app opens standalone with its name and icon, and
`navigator.serviceWorker.getRegistrations()` returns an empty list.
An upload stopped mid-way does not fill its owner's avatar on the This-week screen.
Two Waffles left `failed` by a rejected dispatch, as in Phase 3's invalid-token check, show
Retry to their owner, one on the Upload screen (kept open) and one on the Waffle screen, and
once the valid token is set back, each Retry takes its Waffle to `ready`; another member's
Waffle screen for a `failed` Waffle shows no Retry. Delete passes its cross-site cases and,
from **Delete and abort** in [SPEC Tests worth writing](../docs/SPEC.md#build-plan), a
failure at each of its R2 calls followed by a repeat that finishes.

**Why here.** It needs ready playback copies from Phase 3 and the read routes from Phases 2
and 3.

### Phase 5: Discord · planned

**Goal.** The group hears about each Waffle where they already talk. **Advances capabilities:** 3.

- Webhook on `ready`, built on Phase 3's webhook poster, message deleted with the Waffle,
  and failed posts handled as in
  [SPEC Discord notification](../docs/SPEC.md#discord-notification).

**Exit criteria.** One message appears per ready Waffle, in a test channel first, with a
working link, and the message is deleted with the Waffle. In the test channel, which only
the admin can see, a Waffle captioned `@everyone` posts and the admin gets no mention
notification for it. A webhook revoked by deleting it in Discord leaves the Waffle `ready`
with Post to Discord again. While it is still revoked, deleting a Waffle posted before the
revoke removes its row and R2 objects, its message stays in the channel, and the Worker logs
its `delete: discord message delete failed` line (rules:
[SPEC Discord notification](../docs/SPEC.md#discord-notification)). Once
`DISCORD_WEBHOOK_URL` holds a new test-channel webhook, Post to Discord again posts one
message and clears `discord_error`. A duplicate callback posts once.
`POST /api/waffles/:id/discord` passes its cross-site cases in
[SPEC Tests worth writing](../docs/SPEC.md#build-plan).

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

<!-- Phase-level dependencies, launch and the domain cutover only. Feature-level chains and couplings live in META-PLAN. -->

`0 → 1 → 2 → 3 → {4, 5} → 6`. The riskiest parts (large phone uploads, then transcode time
and size) come first. 4 before 5 is a preference (the Discord link lands on the Waffle
screen), not a hard dependency: 5 needs only Phase 3's `ready` state. Launch follows Phase 6:
the webhook swap below, two Wednesdays alongside WhatsApp, then retiring the WhatsApp group.

**Webhook swap.** First delete the prototype's test Waffles, since the archive starts on
launch day and the test webhook can still delete their messages; a `ready` test Waffle left
unposted would otherwise post to the group through Post to Discord again. Then create a
webhook in the real channel and run `npx wrangler secret put DISCORD_WEBHOOK_URL`, pasting
its URL at the prompt; this updates the deployed Worker. The real webhook never goes in a
`.dev.vars`, which keeps the test channel's, or anywhere else: not in the repo, an issue, a
workflow log or Discord. The token alerts post through the same secret, so they move to the
real channel with it (rule: [SPEC Transcode](../docs/SPEC.md#transcode)). A Waffle kept from
before the swap keeps its message in the test channel, which the new webhook cannot delete,
so deleting it logs the expected `delete: discord message delete failed` line and still
deletes it (rule: [SPEC Discord notification](../docs/SPEC.md#discord-notification)).
**Swap check:** a test Waffle uploaded after the swap posts in the real channel, and
deleting it removes that message.

The [domain cutover](../docs/SPEC.md#domain-cutover-after-the-prototype-is-proven) is not a
phase; it runs once the prototype is proven, since only the custom domain can carry the
callback's zone rate limit. **Exit check:** the rule reads back from the zone's rate-limiting
rules (dashboard or API) as SPEC states it, and a burst of 11 unsigned POSTs to
`/api/internal/transcode-callback` within 10 seconds gets a 429 for the 11th, with no Worker
log line for it (rule: **Callback rate limit** in
[SPEC Environments](../docs/SPEC.md#environments)). **Cutover check:** after step 8, a member
signs in on the new origin, an upload started after step 6 reaches `ready` with its callback
sent to the new origin (Workers Logs shows the callback request on the new hostname), and
the Discord post's link opens that Waffle, which is then deleted; after step 9, the
workers.dev origin no longer serves the app and the `waffles` CORS rule, read back, lists
only the new origin.

## Maintaining this file

- **Who updates it:** the `/run-feature` fold-back that merges a phase's last remaining
  feature flips that phase's status, refreshes the Current-state rows it changed, and collapses
  the finished phase to one line, in the same PR. Vision changes are made deliberately by
  the owner, never as a side effect of a feature session.
- **History policy:** this file is forward-looking. Pivots, superseded designs and the
  story of how the plan changed live in git history and decision records, not here.
