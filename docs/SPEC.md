# Brown Symphony Waffles — MVP Spec

Oct 13, 2026 · Rehan Milarachi

## Overview

Waffles is a private web app where four friends upload weekly phone videos to Cloudflare R2, get a near-original-quality copy every device can play, and see a link posted in Discord. It replaces the WhatsApp group, which exists today only because Discord caps free uploads at 20 MB.

**Goals (MVP)**

- Upload a Waffle from a phone browser, up to about 2 GB and 15 minutes, without an app install.
- Keep the original file and produce one playback copy: H.264/AAC MP4 at the original resolution and frame rate, visually close to the source, capped at 1080p when that copy would be much larger than the original.
- Post a message to the Discord Waffles channel when a Waffle is ready to watch.
- Browse and play every Waffle, filtered by member and week.
- Show a participation heatmap: 52 weeks by 4 members for the selected year.
- Restrict access to members of the Discord server.

**Non-goals (MVP)**

- Importing historical Waffles from WhatsApp. The archive starts on launch day.
- NAS backup. Documented as a follow-on in the last section.
- Comments, reactions or editing in the app. Discussion stays in Discord.
- Adaptive streaming (HLS/DASH), multiple renditions, or a native mobile app.

**Decisions already made**

| Area | Decision |
| --- | --- |
| Storage | Cloudflare R2, private bucket |
| Hosting | Cloudflare Workers (app + API), D1 for metadata |
| Playback format | H.264 High + AAC in MP4, SDR; original resolution, capped at 1080p when the copy would exceed 1.5× the original's size |
| Notification | Discord channel webhook (bot is a later option) |
| Auth | Discord OAuth, limited to the friends' server |
| Domain | `*.workers.dev` during prototype; `waffles.brownsymphony.app` after it is proven |
| Archive import | None |
| NAS backup | Follow-on, not MVP |

## Architecture

One Cloudflare Worker serves the app and API, R2 holds every video, and a GitHub Actions job does the transcoding; video bytes never pass through the Worker.

<!-- The original doc embedded an architecture diagram (6 components) here; it was not
included in the pasted text. The component table below carries the same information. -->

The phone uploads directly to R2 with presigned URLs. The Worker records metadata in D1, starts the transcode job, receives its callback, and posts to Discord once the playback copy is ready.

| Component | Role |
| --- | --- |
| Phone browser (PWA) | Sign in, upload, browse, play, heatmap |
| Cloudflare Worker (SvelteKit) | UI, API, OAuth, presigned URLs, webhook calls, daily token check |
| D1 | Members, Waffles, sessions |
| R2 bucket `waffles` | Originals, playback MP4s, posters |
| GitHub Actions | ffprobe and ffmpeg: remux or re-encode, poster frame |
| Discord webhook | One message per ready Waffle, plus the admin's token expiry alert |

## Video pipeline

Every Waffle keeps its untouched original and gets one playback MP4 (H.264 High, AAC, SDR, original resolution and frame rate) that plays on iOS, Android, Windows, macOS and every current browser.

### Upload

1. The browser asks the Worker to start an upload. The Worker creates the `waffles` row (`status = uploading`) and an R2 multipart upload, and returns the upload ID and object key.
2. The browser requests presigned part URLs in batches and PUTs 50 MiB parts straight to R2, 3 to 4 at a time, retrying each part up to 3 times.
3. Uploaded part numbers and ETags are kept in `localStorage`, so a dropped connection resumes instead of restarting.
4. The browser calls `complete`. The Worker finishes the multipart upload, sets `status = processing` and `processing_at`, and dispatches the transcode job.

The video never passes through the Worker. R2 needs a CORS rule allowing `PUT` from the app origin and exposing the `ETag` header, or the browser cannot read part ETags.

### Transcode

Runs as a GitHub Actions workflow, started by a `workflow_dispatch` event the Worker sends with the Waffle ID as an input. The Worker authenticates with a fine-grained PAT scoped to this repository with only the Actions: write permission, created with a one-year expiry.

**Dispatch target.** The Worker sends `POST /repos/{GITHUB_REPO}/actions/workflows/{GITHUB_WORKFLOW}/dispatches` with `ref: main` and two inputs: `waffle_id` and `env` (`prod` or `dev`, from the `TRANSCODE_ENV` var; see [Environments](#environments)). The workflow runs from `main`, so it must be merged there before the first dispatch. The Worker refuses to dispatch unless `TRANSCODE_ENV` is exactly `prod` or `dev`, and the workflow declares `env` as a `choice` input, so no other value starts a run. The workflow maps `env` to its bucket (`prod` → `waffles`, `dev` → `waffles-dev`). A `prod` run posts to `CALLBACK_BASE_URL` (a repository secret, so GitHub masks it in the public logs) followed by `/api/internal/transcode-callback`, so no URL appears in a run's public inputs; it fails before downloading anything if that secret is empty or does not start with `https://`. GitHub cannot reach `wrangler dev` on localhost, so a `dev` run writes to `waffles-dev` and always skips the callback. A local Waffle then finishes only through a signed callback sent by hand, or ends `failed` under the stuck-jobs rule below.

**Untrusted inputs.** The repository is public, so the workflow treats both inputs as untrusted, even though only the Worker's token or a collaborator with write access can dispatch it. No step interpolates `${{ inputs.* }}` or `${{ secrets.* }}` inline in a `run:` script or a script-like action input (such as `actions/github-script`'s `script:`); each value reaches the script only through the step's `env:` block (for example `WAFFLE_ID: ${{ inputs.waffle_id }}`) and is quoted where the script uses it (`"$WAFFLE_ID"`). No step runs with `set -x`. The first step checks that `WAFFLE_ID` matches the ULID pattern `^[0-7][0-9A-HJKMNP-TV-Z]{25}$` (the `waffles.id` format in the [Data model](#data-model)) and, as a backstop to the `choice` input, that `RUN_ENV` (from `inputs.env`) is exactly `prod` or `dev`. If either does not match, the step prints a fixed message that never includes the rejected value and fails the run before anything is downloaded; no callback is sent, not even the failure one in step 6. Such a run can only come from a hand-made dispatch, so its Waffle, if any, ends `failed` under the stuck-jobs rule below.

Steps:

1. Download the original from R2 with the S3 API.
2. `ffprobe` it: codec, resolution, frame rate, rotation, duration, colour transfer, `creation_time`.
3. Choose a path:
   - Already H.264 + AAC, SDR, MP4 (common on Android): remux only, `-c copy -movflags +faststart`. No quality loss.
   - Otherwise (HEVC `.mov` from iPhones, HDR, other codecs): re-encode with the settings below.
4. Extract a poster frame at about 1 second as a JPEG (1280 px wide).
5. Upload the playback MP4 and poster to R2.
6. POST the result to the Worker's callback (see **Callback** below): `ready` with the playback results, or `failed` with the error text for any failure after the input check (see **Failure output** below).

**Failure output.** Raw tool output can carry file paths, metadata or URLs, so it never reaches the public log (see the logging rule at the end of this section). The job works on fixed local filenames (`original`, `playback.mp4`, `poster.jpg`), runs `ffprobe` with `-hide_banner -loglevel error` and `ffmpeg` with `-hide_banner -nostdin -nostats -loglevel error`, and sends every tool's stderr in steps 1 to 5 to a file in `$RUNNER_TEMP`. Each step has a category: `download` (step 1), `probe` (2), `encode` (3, including the tone-map and the size-cap re-encode), `poster` (4) and `upload` (5). When a step fails, the log shows only a fixed line naming the Waffle ID and that category. The callback's error text is `<category>: <detail>`, the job's counterpart of the rejected-dispatch format below:
- `download` and `upload` use a fixed detail ("could not read the original from storage", "could not save the playback copy to storage"), since storage-client errors echo endpoint URLs, bucket names and keys.
- `probe`, `encode` and `poster` use the first non-empty line of the tool's stderr file (with `-loglevel error` the last line is usually a generic "Conversion failed!"), with each byte outside printable ASCII replaced by a space, or "no error output" if the file is empty.
- The job truncates the whole text to 300 characters, and the Worker truncates `error` to 300 characters again before storing it, so a hand-signed callback can't store more.

**Callback.** The job POSTs a JSON body to `/api/internal/transcode-callback`. It sends two headers: `X-Waffles-Timestamp` (Unix seconds) and `X-Waffles-Signature`, the hex HMAC-SHA256 of `<timestamp>.<raw body>` keyed with `TRANSCODE_CALLBACK_SECRET`. The body holds `waffle_id`, `status` (`ready` or `failed`), and then either the error text or the values the run measured (`original_codec`, `is_hdr`, `playback_bytes`, `duration_s`, `width`, `height`, `recorded_at` if present). The Worker sets `playback_key` and `poster_key` itself from the [Storage layout](#storage-layout-r2-bucket-waffles-private) and never reads keys from the body. The Worker checks the signature before parsing the body, compares it in constant time, and rejects a timestamp more than 5 minutes from its own clock. Its responses:
- **401**, empty body: a missing or wrong signature, or a stale timestamp. Nothing is read or changed.
- **400**: a signed body that is malformed: `waffle_id` is not a ULID, `status` is not `ready` or `failed`, a `ready` body lacks a measured value other than `recorded_at` or has one of the wrong type, or a `failed` body has no error text.
- **404**: no Waffle has that ID, for example because it was deleted while its run was in progress.
- **200** in every other case, whether or not the result changed the Waffle. A `ready` result applies in one guarded `UPDATE … SET status = 'ready', ready_at = now, error = NULL, … WHERE id = ? AND status IN ('processing', 'failed')`. Only a request that changed the row takes the Discord claim (see [Discord notification](#discord-notification)), so a duplicate `ready` is a no-op. A `ready` that arrives after the stuck-jobs rule or a rejected dispatch set `failed` still recovers the Waffle, because the copy it describes exists. A `failed` result applies only `WHERE id = ? AND status = 'processing'`, so it never replaces `ready`.

The job gives each callback request a 20-second timeout and signs every attempt again with the current timestamp, so a retry never fails the 5-minute check. On a 5xx, a 429 or a network error, it retries the callback 3 times, waiting 30, 60 and 120 seconds, and then fails the run. Any other non-2xx response (401, 400, 404 or anything else) is final and fails the run at once. On a 404, the job first deletes the playback copy and poster it uploaded in step 5, if any, so a Waffle deleted during its run leaves nothing behind in R2. A failed run shows as failed in the Actions tab, with a fixed message that names only the Waffle ID and, for a step failure, its category (see **Failure output**). If its Waffle is still `processing`, it stays that way until the stuck-jobs rule sets it `failed`.

**Racing runs.** Two runs can report for one Waffle: a stuck run that finishes after a Retry, or a dispatch that timed out but did start a run. Both encode the same untouched original to the same R2 keys, so either `ready` describes a valid copy, and the guards in **Callback** decide which result counts in D1. One consequence: a `failed` from one run can be followed by the other run's `ready`, which recovers the Waffle. Accepted limitation: the run that loses still uploads its copy and poster over the winner's. Its `playback_bytes` and dimensions then go unrecorded and may differ slightly from the row's, and someone watching at that moment can see a glitch. This is rare, both copies come from the same original, and a reload plays normally.

**Stuck jobs.** The workflow job sets `timeout-minutes: 120`. A Waffle still `processing` 150 minutes after its last dispatch (120 minutes of run time plus 30 minutes of queue slack, which assumes a hosted runner picks the job up within 30 minutes) is presumed to have a run that will never report back, so it is set to `failed` with the error "transcode did not report back". The check runs lazily, as one guarded statement (`UPDATE … SET status = 'failed', error = … WHERE status = 'processing' AND processing_at < now − 150 min`), so it can never overwrite a callback's `ready`. `GET /api/waffles` and `GET /api/waffles/:id` apply it before responding. The retry route applies it first, then requires `failed`, then sets `status = processing` and `processing_at = now` in one `UPDATE … WHERE status = 'failed'`, so a stuck Waffle can be retried and a double tap dispatches once. A Waffle that is not `failed`, including one a late `ready` already recovered (see **Callback**), gets 409 and nothing is dispatched. This rule needs no scheduled job; the Worker's one scheduled task, the token expiry check below, never touches Waffles. If Phase 3 encode times approach 120 minutes, raise both values together.

**Rejected dispatch.** A dispatch, from `complete` or Retry, fails when GitHub answers anything but 2xx, or on a network error or timeout. The Waffle is then set to `failed` in one guarded `UPDATE … WHERE id = ? AND status = 'processing'`, so it never overwrites a callback's `ready`. `error` is stored as `<status code or network>: <fixed message>`; GitHub's response body and headers are never stored or logged. The admin is the member who holds the Worker and GitHub secrets.
- A 401, 403 or 404 usually means the token has expired or lost access: "GitHub refused the transcode request; its access token may have expired or the setup changed. Your upload is safe. Tell the admin; the uploader can press Retry once it is fixed."
- Any other status code, a network error or a timeout reads: "GitHub could not start the transcode. Your upload is safe. The uploader can press Retry later; if it keeps failing, tell the admin."
- A timeout is ambiguous: GitHub may have started the run anyway, so a Retry can start a second run. This is accepted, since both runs encode the same untouched original (see **Racing runs** above).

**Token expiry check.** The PAT (`GITHUB_DISPATCH_TOKEN`) expires, so the Worker checks it daily instead of waiting for a member's upload to fail. A Cron Trigger (`[triggers] crons = ["0 14 * * *"]`, 14:00 UTC) sends `GET /repos/{GITHUB_REPO}/actions/workflows` with the token; it reads the workflow list and starts no run.
- A 401, 403 or 404 means the token is invalid or has lost access.
- On a 2xx, the `github-authentication-token-expiration` response header gives the expiry as a UTC timestamp (for example `2026-11-01 00:00:00 UTC`). An expiry within 14 days counts as expiring. A missing header (a token with no expiry) never alerts; an unparseable one is treated like any other failure below.
- An invalid or expiring token posts a fixed plain-text message, with no @mention, to the channel webhook (`DISCORD_WEBHOOK_URL`) once per daily run until the token is renewed. Expiring: "Admin: the GitHub token that starts transcodes expires on `YYYY-MM-DD`. Renew it before uploads fail; see the README's token rotation steps." Invalid: "Admin: GitHub refused the token that starts transcodes, so uploads will fail. Renew it; see the README's token rotation steps."
- Any other status code, a network error or an unparseable header alerts nothing and is checked again the next day. The alert post makes one attempt, with no retry and nothing stored, so a failed post is simply repeated the next day. Like the dispatch, the check never stores or logs GitHub's response body or headers, or the webhook URL; only the expiry date appears in the message.
- Accepted: a rare rate-limit 403 raises a false "refused" alert that the next day's check clears, and a dead webhook silences the alert until a member reports a failed upload.

Re-encode settings, the starting point to tune in Phase 3:

```bash
ffmpeg -hide_banner -nostdin -nostats -loglevel error -i original \
  -c:v libx264 -preset slow -crf 18 -profile:v high -pix_fmt yuv420p \
  -fps_mode passthrough \
  -c:a aac -b:a 192k \
  -map_metadata 0 -movflags +faststart \
  playback.mp4
```

- **CRF 18** is close to visually lossless for H.264. Raise it to 20 if files are too large; each +6 roughly halves the size.
- **Resolution is kept.** 4K stays 4K. ffmpeg applies the phone's rotation flag automatically.
- **HDR is tone-mapped to SDR.** Recent iPhones record HLG HDR by default, which looks washed out on many SDR screens. When `ffprobe` reports `color_transfer` of `arib-std-b67` (HLG) or `smpte2084` (PQ), add a tone-map filter to BT.709, for example `zscale=t=linear:npl=100,format=gbrpf32le,zscale=p=bt709,tonemap=hable:desat=0,zscale=t=bt709:m=bt709:r=tv,format=yuv420p`. The original keeps the HDR.
- **Expect the playback copy to be larger than an iPhone original.** H.264 needs more bits than HEVC for the same quality, so a 4K Waffle may land at 1.5 to 2 times the original's size. **Size cap:** if the re-encoded copy is more than 1.5× the original's size, the job encodes it again with the shorter side capped at 1080 px (`-vf "scale='if(gt(iw,ih),-2,1080)':'if(gt(iw,ih),1080,-2)'"`, which handles portrait and landscape) and keeps that version. A source already at 1080p or below is never downscaled; if its copy exceeds 1.5×, raise CRF by 2 instead.

The repository is public, so Actions minutes are free. Workflow logs are public too: log only Waffle IDs, never names, captions or URLs, and keep every credential in repository secrets. Videos and member data live only in R2 and D1, never in the repo. About 4 Waffles a week is a light load, but a slow-preset 4K encode on a hosted runner can take a long time. Measure in Phase 3; if it is too slow, use `-preset medium` or move the job to Cloudflare Containers.

### Storage layout (R2 bucket `waffles`, private)

| Object | Key |
| --- | --- |
| Original | `originals/{waffleId}/source.{ext}` |
| Playback copy | `playback/{waffleId}/video.mp4` |
| Poster | `playback/{waffleId}/poster.jpg` |

Playback uses presigned GET URLs from the Worker with a 6-hour expiry. R2 supports range requests on them, so seeking works. Abort any multipart upload left incomplete for more than 7 days with an R2 lifecycle rule.

## Data model

Three D1 tables hold everything; the heatmap is a query over `waffles`, not stored data.

```sql
CREATE TABLE members (
  discord_id    TEXT PRIMARY KEY,      -- Discord user ID (snowflake)
  display_name  TEXT NOT NULL,
  avatar_url    TEXT,
  sort_order    INTEGER NOT NULL,      -- fixed row order on the heatmap
  active        INTEGER NOT NULL DEFAULT 1,
  created_at    TEXT NOT NULL
);

CREATE TABLE waffles (
  id                 TEXT PRIMARY KEY,  -- ULID made by the Worker: 26 uppercase Crockford base32 chars
  member_id          TEXT NOT NULL REFERENCES members(discord_id),
  week_start         TEXT NOT NULL,     -- 'YYYY-MM-DD', the Wednesday that opens the Waffle week
  status             TEXT NOT NULL,     -- uploading | processing | ready | failed
  processing_at      TEXT,              -- UTC ISO-8601, set at each dispatch (complete, retry)
  caption            TEXT,
  original_key       TEXT NOT NULL,
  original_bytes     INTEGER,
  original_codec     TEXT,
  is_hdr             INTEGER,
  playback_key       TEXT,
  playback_bytes     INTEGER,
  poster_key         TEXT,
  duration_s         REAL,
  width              INTEGER,
  height             INTEGER,
  recorded_at        TEXT,              -- from the file's creation_time, if present
  uploaded_at        TEXT NOT NULL,
  ready_at           TEXT,
  discord_message_id TEXT,
  discord_posting_at TEXT,              -- UTC ISO-8601, set while a Discord post is in flight
  discord_error      TEXT,              -- why the last Discord post failed; NULL once a post succeeds
  error              TEXT               -- why the Waffle failed; shown with Retry
);

CREATE INDEX waffles_week   ON waffles(week_start);
CREATE INDEX waffles_member ON waffles(member_id, week_start);

CREATE TABLE sessions (
  id          TEXT PRIMARY KEY,         -- random 256-bit token, stored hashed
  member_id   TEXT NOT NULL REFERENCES members(discord_id),
  expires_at  TEXT NOT NULL
);
```

### Waffle week

- A Waffle week runs **Wednesday 00:00 to Tuesday 23:59, America/New_York** (a config value, `GROUP_TZ`). One member is on Central time; the cutoff is informal, so the whole group uses ET.
- `week_start` is the date of that Wednesday, computed from `uploaded_at` when the upload starts.
- The upload form defaults to the current week and offers the previous week, for a Waffle filmed Tuesday night and posted Thursday.
- A member may post more than one Waffle in a week. The heatmap counts the member as present; the archive shows them all.
- A heatmap **year** is every week whose Wednesday falls in that calendar year: 52 weeks, 53 in some years.

## Auth, API and Discord

Sign-in is Discord OAuth; a person gets in only if they belong to the friends' Discord server and appear in `members`.

### Auth

- OAuth2 authorization-code flow with scopes `identify guilds`. On callback, fetch `/users/@me` and `/users/@me/guilds`, and require `DISCORD_GUILD_ID` in the list.
- Then require the user's ID in `members` (seeded by hand: 4 rows). Anyone else sees a "not a member" page.
- Sessions: an opaque random token in an `HttpOnly; Secure; SameSite=Lax` cookie, stored hashed in `sessions`, 30-day expiry renewed on use.
- The OAuth `state` parameter is a signed, short-lived value checked on callback.

### API routes (all under the app origin)

| Method | Path | Purpose |
| --- | --- | --- |
| GET | `/auth/login` | Redirect to Discord |
| GET | `/auth/callback` | Finish OAuth, create session |
| POST | `/auth/logout` | Delete session |
| POST | `/api/uploads` | Create Waffle row and R2 multipart upload; body: file name, size, type, caption, `week_start` |
| POST | `/api/uploads/:id/parts` | Presigned PUT URLs for a list of part numbers |
| POST | `/api/uploads/:id/complete` | Complete multipart upload with part ETags, dispatch transcode |
| POST | `/api/uploads/:id/abort` | Abort upload, delete row |
| GET | `/api/waffles` | List; filters `member`, `year`, `week_start`; newest first |
| GET | `/api/waffles/:id` | Metadata plus presigned playback, poster and original URLs |
| DELETE | `/api/waffles/:id` | Owner only: delete row and all three R2 objects, and its Discord message if there is one |
| POST | `/api/waffles/:id/retry` | Owner only: re-dispatch a failed transcode, including one that did not report back |
| POST | `/api/waffles/:id/discord` | Owner only: post a `ready` Waffle that has no Discord message yet |
| GET | `/api/heatmap?year=YYYY` | Week starts × members with counts |
| POST | `/api/internal/transcode-callback` | From the GitHub job; HMAC-signed; sets ready or failed (see [Transcode](#transcode)) |

Only the uploader may abort, delete, retry or re-post their own Waffle. Any member may download any Waffle's original; its presigned URL sets `Content-Disposition: attachment` with a readable name such as `2026-10-14_alex.mov`. Every `/api` route except the callback requires a session.

### Discord notification

- A channel webhook (URL kept as a Worker secret) posts once, when a Waffle reaches `ready`, never at upload start. The same webhook also carries the admin's alert from the [token expiry check](#transcode); the rules below cover Waffle posts only.
- Message: "🧇 **Alex** posted a Waffle for the week of Oct 14 · 6:42" plus the caption and the app link `/w/{id}`. Post with `?wait=true` so Discord returns the message, and store its ID in `discord_message_id`.
- Post only when `discord_message_id IS NULL`, and claim the post first with one guarded statement (`UPDATE … SET discord_posting_at = now WHERE status = 'ready' AND discord_message_id IS NULL AND (discord_posting_at IS NULL OR discord_posting_at < now − 2 min)`); only the request that changed the row posts. So a duplicate callback or a second tap on "Post to Discord again" can't post twice, and a claim left by a Worker that died mid-post expires after 2 minutes. The post's result clears `discord_posting_at`.
- The callback posts after it responds (`ctx.waitUntil`), so a slow Discord never delays the GitHub job.
- A failed post never changes `ready`: the Waffle stays watchable. On a 429, wait the `Retry-After` time and try again, up to 3 attempts; a `Retry-After` over 10 seconds counts as a failure. Any other non-2xx response (a revoked webhook URL returns 401 or 404), a network error or a third 429 sets `discord_error` to the status code (or `network`) and a fixed message; the webhook URL and the response body are never stored. A successful post clears `discord_error`.
- While a `ready` Waffle has no `discord_message_id` and no live claim, the owner's Waffle screen shows "Not posted to Discord" (with the `discord_error` text, if any) and a **Post to Discord again** action. Its route, `POST /api/waffles/:id/discord`, takes the same claim and runs the same post; it returns 409 when the Waffle is not `ready`, is already posted or is being posted.
- Deleting a Waffle deletes its Discord message when `discord_message_id` is set. When it is NULL (never posted, or every post failed), there is nothing to delete and the Waffle is deleted without calling Discord. A failed message delete (including a 404 for a message already removed in Discord) never blocks deleting the Waffle.
- Accepted limitation: a post that Discord accepted but whose response was lost (a network error or timeout), or that finishes after its Waffle was deleted, leaves a message the app has no ID for. A re-post then makes a second message, and the orphan is removed by hand in Discord.
- The link sits behind sign-in, so Discord shows a plain link, not a preview. Adding a poster image to the embed would need a public or long-lived URL; leave it out of the MVP.
- A bot is only worth adding later for slash commands (for example `/waffle status` to list who has not posted this week).

## UI

The app is a mobile-first PWA with five screens, installable from the home screen (a manifest and icons) with no offline caching; most uploads happen from a phone, so every screen must work one-handed at 375 px wide.

| Screen | Path | Contents |
| --- | --- | --- |
| This week | `/` | Who has posted this Waffle week (4 avatars, filled or empty), the latest Waffles, a large Upload button |
| Upload | `/upload` | File picker (`accept="video/*"`), caption, week selector (this week / last week), progress bar with MB sent and time left |
| Waffle | `/w/:id` | Player, member, week, caption, duration; Download original; Delete and Retry for the owner, and Post to Discord again while it has no Discord message |
| Archive | `/archive` | Grid of poster cards grouped by week, newest first; filter by member and year |
| Participation | `/participation` | The heatmap for a selected year |

**Upload screen behaviour**

- Warn that the screen must stay on while uploading, and request a Screen Wake Lock where the browser supports it.
- Show `processing` after the upload completes, then update to `ready` (poll every 10 s) with a link to the Waffle.
- Resume an interrupted upload when the same file is picked again.

**Player**

- `<video controls playsinline preload="metadata" poster=…>` pointing at the presigned playback URL.
- While `processing`, show the poster if present and a "getting it ready" note; if `failed`, show the error and Retry.

**Participation heatmap**

- One row per member (in `sort_order`) plus a top "Group" row; one column per Waffle week, Wednesdays across the year, months labelled above.
- Member cells: empty or filled. Group cells: 0 to 4 shades for how many members posted.
- The current week is outlined; future weeks are blank and not clickable.
- Hover or tap shows the week and who posted; clicking a filled cell opens that Waffle (or the week in the archive if there are several).
- Below the grid: each member's count for the year and current streak of consecutive weeks. The streak counts back from last week, plus the current week once the member has posted in it; an unposted current week never breaks it, so a streak breaks only when a week ends with no Waffle from that member.
- Built as a plain CSS grid from `/api/heatmap`; no chart library needed.

## Cost, environments and domain

Storage is the only real cost: about $2 a month by the end of year one, growing by roughly that much each year because both the original and the playback copy are kept.

### Cost estimate

Assumes 208 Waffles a year (4 members × 52 weeks), averaging 5 minutes of 1080p from the phone.

| Item | Estimate |
| --- | --- |
| Originals | about 60 GB/year (about 60 MB per minute in HEVC) |
| Playback copies | about 60 GB/year (H.264 at CRF 18; measure in Phase 3) |
| R2 storage | first 10 GB free, then $0.015 per GB-month: about $1.65/month at 120 GB, about $3.45/month at 240 GB |
| R2 egress | free |
| Workers, D1 | free plan is enough for 4 people |
| GitHub Actions | free (public repo) |
| Domain | `brownsymphony.app`, a yearly registrar fee; check the current price before buying |

Waffles shot in 4K roughly triple the originals figure; the 1080p cap keeps their playback copies from growing as fast.

### Environments

- **Local:** `wrangler dev` with a local D1 and a separate `waffles-dev` R2 bucket. It cannot receive transcode callbacks (see [Transcode](#transcode)).
- **Prototype:** deployed to `waffles.<account>.workers.dev`, with the real bucket and webhook pointed at a test Discord channel until launch.
- **Production:** the same Worker on the custom domain after cutover.

Secrets (set with `wrangler secret put`): `DISCORD_CLIENT_ID`, `DISCORD_CLIENT_SECRET`, `DISCORD_GUILD_ID`, `DISCORD_WEBHOOK_URL`, `R2_ACCESS_KEY_ID`, `R2_SECRET_ACCESS_KEY`, `R2_ACCOUNT_ID`, `GITHUB_DISPATCH_TOKEN` (the fine-grained PAT described in [Transcode](#transcode), rotated as the README describes), `TRANSCODE_CALLBACK_SECRET`, `SESSION_SECRET`. The GitHub workflow needs the R2 keys, the callback secret and `CALLBACK_BASE_URL` (the deployed Worker's origin, `https://waffles.<account>.workers.dev` until the domain cutover) as repository secrets.

Vars (non-secret, in the Wrangler config's `[vars]`): `GITHUB_REPO` (this repository's `owner/name`), `GITHUB_WORKFLOW` (the transcode workflow's file name, for example `transcode.yml`), `TRANSCODE_ENV` and `GROUP_TZ`. `[vars]` sets `TRANSCODE_ENV = "prod"` for the deployed Worker. Locally, `.dev.vars` overrides it with `dev`; that file also holds the local `GITHUB_DISPATCH_TOKEN`, so a checkout without it cannot dispatch at all.

### Domain cutover (after the prototype is proven)

1. Buy `brownsymphony.app` and use Cloudflare for its DNS.
2. Add a custom domain to the Worker. Serve it at `waffles.brownsymphony.app`, leaving the apex free.
3. Add the new redirect URI in the Discord developer portal, and set it as the app's OAuth callback.
4. Add the new origin to the R2 bucket's CORS rule.
5. Set the `CALLBACK_BASE_URL` repository secret to `https://waffles.brownsymphony.app`, or transcode callbacks keep going to the workers.dev origin. Keep the workers.dev route enabled until no Waffle is `processing`, because a run that started before the change still reports to it.
6. Point the webhook at the real Waffles channel.
7. `.app` domains only load over HTTPS; Cloudflare provides the certificate, so nothing else is needed.

## Build plan

Build in seven phases, each ending in something the four of you can try; the riskiest parts (large uploads from phones, transcode time and size) come first.

**Stack:** TypeScript throughout · SvelteKit with `@sveltejs/adapter-cloudflare` (UI and API in one Worker) · D1 with Drizzle ORM and its migrations · `aws4fetch` for R2 presigning in the Worker · a GitHub Actions workflow on `ubuntu-latest` with ffmpeg · Vitest for unit tests.

Start the Claude Code session by giving it this doc and asking it to write a `CLAUDE.md` that records the stack, the folder layout, the commands (`dev`, `test`, `deploy`, `db:migrate`) and the Waffle-week rule.

1. **Phase 0 — Scaffold.** Repo, SvelteKit app deployed to `workers.dev`, D1 database with the schema and migrations, R2 buckets with CORS and the lifecycle rule, secrets set.
   - Done when: the deployed app returns a page, and `db:migrate` runs locally and remotely.
2. **Phase 1 — Sign-in.** Discord OAuth, guild and `members` checks, sessions, the four members seeded.
   - Done when: all four can sign in from their phones, and a Discord account outside the server cannot.
3. **Phase 2 — Upload.** Multipart upload to R2 with presigned parts, progress, retry, resume, abort.
   - Done when: each member uploads a real 10-minute Waffle from their own phone on mobile data and Wi-Fi, and the object in R2 is byte-identical to the original (compare size and SHA-256).
4. **Phase 3 — Transcode.** GitHub workflow, probe, remux-or-encode, HDR tone-map, poster, callback, and the owner-only retry route (tested through the API; Phase 4 adds its button).
   - Done when: an iPhone HDR clip, an iPhone SDR clip and an Android clip each play correctly in Safari on iOS, Chrome on Android and Chrome or Edge on Windows.
   - Done when (rejected dispatch): with an invalid `GITHUB_DISPATCH_TOKEN`, `complete` ends the Waffle `failed` with the token message from **Rejected dispatch** in [Transcode](#transcode), the token appears in no stored error, API response or Worker log, and the original stays in R2.
   - Done when (token check): with fixture GitHub responses, a token expiring within 14 days and a 401 each post one alert, and a missing header or a 5xx posts nothing.
   - Done when (Retry, through the API): after a rejected dispatch, another member's Retry is refused by the owner check while the Waffle is still `failed`, leaving it `failed` with nothing dispatched; then the owner's Retry takes it to `ready`.
   - Record for each test clip: original size, playback size, encode time. Adjust CRF, preset and the 1.5× cap threshold from these, and confirm a 4K clip over the threshold comes out at 1080p.
5. **Phase 4 — Browse and play.** This-week, Archive and Waffle screens; delete and the Retry button.
   - Done when: seeking works mid-video on a phone, and deleting a Waffle removes the row and all three R2 objects; added to the home screen from Safari on iOS and Chrome on Android, the app opens standalone with its name and icon, and `navigator.serviceWorker.getRegistrations()` returns an empty list.
6. **Phase 5 — Discord.** Webhook on `ready`, message deleted with the Waffle, failed-post recording and Post to Discord again.
   - Done when: one message appears per ready Waffle, in a test channel first, with a working link; a revoked webhook leaves the Waffle `ready` with Post to Discord again, and a duplicate callback posts once.
7. **Phase 6 — Participation.** `/api/heatmap` and the heatmap screen.
   - Done when: the grid matches the archive for seeded test data, including a week with two Waffles from one member, a Waffle filed to the previous week, and Waffles uploaded at Tuesday 23:59:59 and Wednesday 00:00:00 ET, in both an EST and an EDT week, landing in consecutive weeks (seeds get `week_start` from `uploaded_at` through the Waffle-week function); a member who has not yet posted in the current week keeps their streak, posting in it adds one, and a past week with no Waffle from them breaks it.

**Launch:** point the webhook at the real channel, use it for two Wednesdays alongside WhatsApp, then retire the WhatsApp group. Domain cutover can follow at any point after.

**Tests worth writing:** the Waffle-week function (Tuesday 23:59 vs Wednesday 00:00 ET, and the daylight-saving changes), the callback (a `ready` from `processing` sets `ready`, clears `error` and claims the Discord post once; a `failed` from `processing` stores the error, truncated to 300 characters; a `failed` with no error text gets 400; a tampered body, a bad signature or a timestamp over 5 minutes old gets 401 and changes nothing; a malformed body gets 400; an unknown ID gets 404, and the job then deletes the playback copy and poster it uploaded; a duplicate `ready` changes nothing; a `ready` after the 150-minute rule or a rejected dispatch set `failed` recovers the Waffle, and a Retry after it gets 409; a `failed` after `ready` changes nothing), the owner-only rules, the stuck-job check (149 vs 150 minutes after `processing_at`, and a Retry after it), a rejected dispatch from `complete` and from Retry (a 401 stores `401:` and the token message; a 500 or a network error stores the generic message; neither stores anything from GitHub's response), the Discord post (two concurrent callbacks post once; a 429 retries after `Retry-After`; a failed post leaves `ready` and sets `discord_error`; a re-post clears it; a Delete with no message ID, or a 404 on the message delete, still deletes the Waffle), and the heatmap query.

## Follow-on: NAS backup

After launch, each member's NAS pulls a nightly copy of the bucket, so the archive survives an R2 account problem or an accidental delete.

- Create a **read-only** R2 API token per member, scoped to the `waffles` bucket.
- Each NAS runs a nightly scheduled job: `rclone copy r2:waffles /volume/waffles --fast-list` (copy, not sync, so a delete in R2 never removes a backup). Synology, Unraid and TrueNAS can all run rclone in Docker or a scheduled task.
- Back up the database too: a nightly GitHub Actions job runs `wrangler d1 export` and uploads the dump to `backups/d1/YYYY-MM-DD.sql` in the bucket, which the NAS copy picks up. Without it, the videos lose their member and week.
- Optionally copy originals only, to halve NAS storage; playback copies can be regenerated.
- Document the restore steps: create a bucket, upload from a NAS, import the latest SQL dump into D1.

## Resolved questions

| Question | Decision |
| --- | --- |
| Domain | Subdomain `waffles.brownsymphony.app` |
| Time zone | ET (America/New_York), Wednesday to Tuesday; one member is in CT and the cutoff is informal |
| GitHub repo | Public; no video or member data in the repo or its logs |
| 4K playback | Keep original resolution; cap at 1080p when the copy exceeds 1.5× the original |
| Original downloads | Any member can download any Waffle's original |
