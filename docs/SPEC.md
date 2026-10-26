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
| Cloudflare Worker (SvelteKit) | UI, API, OAuth, presigned URLs, webhook calls |
| D1 | Members, Waffles, sessions |
| R2 bucket `waffles` | Originals, playback MP4s, posters |
| GitHub Actions | ffprobe and ffmpeg: remux or re-encode, poster frame |
| Discord webhook | One message per ready Waffle |

## Video pipeline

Every Waffle keeps its untouched original and gets one playback MP4 (H.264 High, AAC, SDR, original resolution and frame rate) that plays on iOS, Android, Windows, macOS and every current browser.

### Upload

1. The browser asks the Worker to start an upload. The Worker creates the `waffles` row (`status = uploading`) and an R2 multipart upload, and returns the upload ID and object key.
2. The browser requests presigned part URLs in batches and PUTs 50 MiB parts straight to R2, 3 to 4 at a time, retrying each part up to 3 times.
3. Uploaded part numbers and ETags are kept in `localStorage`, so a dropped connection resumes instead of restarting.
4. The browser calls `complete`. The Worker finishes the multipart upload, sets `status = processing`, and dispatches the transcode job.

The video never passes through the Worker. R2 needs a CORS rule allowing `PUT` from the app origin and exposing the `ETag` header, or the browser cannot read part ETags.

### Transcode

Runs as a GitHub Actions workflow, started by a `repository_dispatch` event the Worker sends with the Waffle ID. Steps:

1. Download the original from R2 with the S3 API.
2. `ffprobe` it: codec, resolution, frame rate, rotation, duration, colour transfer, `creation_time`.
3. Choose a path:
   - Already H.264 + AAC, SDR, MP4 (common on Android): remux only, `-c copy -movflags +faststart`. No quality loss.
   - Otherwise (HEVC `.mov` from iPhones, HDR, other codecs): re-encode with the settings below.
4. Extract a poster frame at about 1 second as a JPEG (1280 px wide).
5. Upload the playback MP4 and poster to R2.
6. POST the results to the Worker's callback, signed with a shared HMAC secret. The Worker sets `status = ready` and fires the Discord webhook. Any failure posts `status = failed` with the error text.

Re-encode settings, the starting point to tune in Phase 3:

```bash
ffmpeg -i source.mov \
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
  id                 TEXT PRIMARY KEY,  -- ULID
  member_id          TEXT NOT NULL REFERENCES members(discord_id),
  week_start         TEXT NOT NULL,     -- 'YYYY-MM-DD', the Wednesday that opens the Waffle week
  status             TEXT NOT NULL,     -- uploading | processing | ready | failed
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
  error              TEXT
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
| DELETE | `/api/waffles/:id` | Owner only: delete row and all three R2 objects |
| POST | `/api/waffles/:id/retry` | Owner only: re-dispatch a failed transcode |
| GET | `/api/heatmap?year=YYYY` | Week starts × members with counts |
| POST | `/api/internal/transcode-callback` | From the GitHub job; HMAC-signed; sets ready or failed |

Only the uploader may abort, delete or retry their own Waffle. Any member may download any Waffle's original; its presigned URL sets `Content-Disposition: attachment` with a readable name such as `2026-10-14_alex.mov`. Every `/api` route except the callback requires a session.

### Discord notification

- A channel webhook (URL kept as a Worker secret) posts once, when a Waffle reaches `ready`, never at upload start.
- Message: "🧇 **Alex** posted a Waffle for the week of Oct 14 · 6:42" plus the caption and the app link `/w/{id}`. Store the returned message ID in `discord_message_id` and delete that message if the Waffle is deleted.
- The link sits behind sign-in, so Discord shows a plain link, not a preview. Adding a poster image to the embed would need a public or long-lived URL; leave it out of the MVP.
- A bot is only worth adding later for slash commands (for example `/waffle status` to list who has not posted this week).

## UI

The app is a mobile-first PWA with five screens; most uploads happen from a phone, so every screen must work one-handed at 375 px wide.

| Screen | Path | Contents |
| --- | --- | --- |
| This week | `/` | Who has posted this Waffle week (4 avatars, filled or empty), the latest Waffles, a large Upload button |
| Upload | `/upload` | File picker (`accept="video/*"`), caption, week selector (this week / last week), progress bar with MB sent and time left |
| Waffle | `/w/:id` | Player, member, week, caption, duration; Download original; Delete and Retry for the owner |
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
- Below the grid: each member's count for the year and current streak of consecutive weeks.
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

- **Local:** `wrangler dev` with a local D1 and a separate `waffles-dev` R2 bucket.
- **Prototype:** deployed to `waffles.<account>.workers.dev`, with the real bucket and webhook pointed at a test Discord channel until launch.
- **Production:** the same Worker on the custom domain after cutover.

Secrets (set with `wrangler secret put`): `DISCORD_CLIENT_ID`, `DISCORD_CLIENT_SECRET`, `DISCORD_GUILD_ID`, `DISCORD_WEBHOOK_URL`, `R2_ACCESS_KEY_ID`, `R2_SECRET_ACCESS_KEY`, `R2_ACCOUNT_ID`, `GITHUB_DISPATCH_TOKEN`, `TRANSCODE_CALLBACK_SECRET`, `SESSION_SECRET`. The GitHub workflow needs the R2 keys and the callback secret as repository secrets.

### Domain cutover (after the prototype is proven)

1. Buy `brownsymphony.app` and use Cloudflare for its DNS.
2. Add a custom domain to the Worker. Serve it at `waffles.brownsymphony.app`, leaving the apex free.
3. Add the new redirect URI in the Discord developer portal, and set it as the app's OAuth callback.
4. Add the new origin to the R2 bucket's CORS rule.
5. Point the webhook at the real Waffles channel.
6. `.app` domains only load over HTTPS; Cloudflare provides the certificate, so nothing else is needed.

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
4. **Phase 3 — Transcode.** GitHub workflow, probe, remux-or-encode, HDR tone-map, poster, callback.
   - Done when: an iPhone HDR clip, an iPhone SDR clip and an Android clip each play correctly in Safari on iOS, Chrome on Android and Chrome or Edge on Windows.
   - Record for each test clip: original size, playback size, encode time. Adjust CRF, preset and the 1.5× cap threshold from these, and confirm a 4K clip over the threshold comes out at 1080p.
5. **Phase 4 — Browse and play.** This-week, Archive and Waffle screens; delete and retry.
   - Done when: seeking works mid-video on a phone, and deleting a Waffle removes the row and all three R2 objects.
6. **Phase 5 — Discord.** Webhook on `ready`, message deleted with the Waffle.
   - Done when: one message appears per ready Waffle, in a test channel first, with a working link.
7. **Phase 6 — Participation.** `/api/heatmap` and the heatmap screen.
   - Done when: the grid matches the archive for seeded test data, including a week with two Waffles from one member and a Waffle filed to the previous week.

**Launch:** point the webhook at the real channel, use it for two Wednesdays alongside WhatsApp, then retire the WhatsApp group. Domain cutover can follow at any point after.

**Tests worth writing:** the Waffle-week function (Tuesday 23:59 vs Wednesday 00:00 ET, and the daylight-saving changes), the HMAC callback check, the owner-only rules, and the heatmap query.

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
