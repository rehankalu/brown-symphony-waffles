# Brown Symphony Waffles — MVP Spec

Oct 13, 2026 · Rehan Milarachi

## Overview

Waffles is a private web app where four friends upload weekly phone videos to Cloudflare R2, get a near-original-quality copy every device can play, and see a link posted in Discord. It replaces the WhatsApp group, which exists today only because Discord caps free uploads at 20 MB.

**Goals and non-goals.** The MVP's goals are ROADMAP's [Capabilities](../plans/ROADMAP.md#capabilities), and its non-goals are ROADMAP's [Out of scope](../plans/ROADMAP.md#out-of-scope).

**Decisions already made**

| Area | Decision |
| --- | --- |
| Storage | Cloudflare R2, private bucket |
| Hosting | Cloudflare Workers (app + API), D1 for metadata |
| Playback format | H.264 + AAC in MP4 (video only when the source has no audio), SDR (High profile when re-encoded); original resolution, capped at 1080p when the copy would exceed 1.5× the original's size |
| Notification | Discord channel webhook (bot is a later option) |
| Auth | Discord OAuth, limited to the friends' server |
| Domain | `*.workers.dev` during prototype; `waffles.brownsymphony.app` after it is proven |
| Archive import | None |
| NAS backup | Follow-on, not MVP |

## Architecture

One Cloudflare Worker serves the app and API, R2 holds every video, and a GitHub Actions job does the transcoding; video bytes never pass through the Worker (a rule in [CLAUDE.md § Project rules](../CLAUDE.md#project-rules)).

The phone uploads directly to R2 with presigned URLs. The Worker records metadata in D1, starts the transcode job, receives its callback, and posts to Discord once the playback copy is ready.

| Component | Role |
| --- | --- |
| Phone browser (PWA) | Sign in, upload, browse, play, heatmap |
| Cloudflare Worker (SvelteKit) | UI, API, OAuth, presigned URLs, webhook calls, daily token expiry check |
| D1 | Members, Waffles, sessions |
| R2 bucket `waffles` | Originals, playback MP4s, posters |
| GitHub Actions | ffprobe and ffmpeg: remux or re-encode, poster frame |
| Discord webhook | One message per ready Waffle, plus the admin's token expiry alert |

**Worker entry.** `@sveltejs/adapter-cloudflare` builds a Worker that exports only a `fetch` handler (`.svelte-kit/cloudflare/_worker.js`), so a Cron Trigger would have nothing to call. A small custom entry, `src/worker.ts`, imports that output and exports an object with a `fetch` that passes `request`, `env` and `ctx` to the adapter's `fetch` unchanged, plus a `scheduled` handler that runs the token expiry check (see [Transcode](#transcode)). The Wrangler config's `main` points at `src/worker.ts` and keeps the adapter's assets block (`directory = ".svelte-kit/cloudflare"`, `binding = "ASSETS"`), so pages, static files and bindings behave as without the wrapper. Its `[build] command = "npm run build"` (the `build` script is `vite build`) makes `wrangler dev` and `wrangler deploy` build the adapter output before Wrangler bundles the entry. `vite dev` never runs the entry, so the scheduled handler runs locally only under `wrangler dev`.

## Video pipeline

Every Waffle keeps its untouched original and gets one playback MP4 (H.264, AAC, SDR, original resolution and frame rate) that plays on iOS, Android, Windows, macOS and every current browser.

### Upload

1. The browser asks the Worker to start an upload. The Worker creates an R2 multipart upload, then the `waffles` row (`status = uploading`, with the multipart upload's ID in `upload_id`), and returns the upload ID and object key. If the row insert fails, the route returns a 5xx and the member starts again; the R2 lifecycle rule aborts the multipart upload left behind.
2. The browser requests presigned part URLs in batches and PUTs 50 MiB parts straight to R2, 3 to 4 at a time, retrying each part up to 3 times. Part URLs expire (see [Storage layout](#storage-layout-r2-bucket-waffles-private)), so the browser requests each batch as it needs it, and each retry first gets a fresh URL from `/parts`.
3. Uploaded part numbers and ETags are kept in `localStorage`, so a dropped connection resumes instead of restarting. Part URLs are not kept: every resume gets fresh ones from `/parts` for the parts not yet sent.
4. The browser calls `complete`. The Worker finishes the multipart upload, checks the original's size against `original_bytes`, sets `status = processing` and `processing_at`, and dispatches the transcode job.

**Upload checks.** `POST /api/uploads` takes `ext` from the text after the file name's last `.`, lowercased, and accepts only `mov`, `mp4` or `m4v`; any other name, including one with no `.` or nothing after it, gets 400 `bad_ext` "Waffles must be .mov, .mp4 or .m4v files." and creates nothing. The original's key uses that `ext`, never the raw file name. The declared size must be a positive integer of at most 5 GiB (5,368,709,120 bytes), enough for a 10-minute 4K60 HEVC iPhone clip: any other value gets 400 `bad_size` "The file size is not valid.", a larger size gets 400 `too_large` "Waffles can be up to 5 GiB.", and neither creates anything. The Worker stores it in `original_bytes`, and `complete` checks the finished original against it (see **Abort or Delete** in [API routes](#api-routes-all-under-the-app-origin)). `/parts` and `complete` check, in this order: a missing row (cancelled or swept) gets 409 `cancelled` "This upload was cancelled.", which the browser handles as in [Failed uploads](#upload); a row another member uploaded gets 404 `not_found` "No such upload." whatever its status, as `GET /api/waffles/:id` gives other members for an `uploading` Waffle; the uploader's row that has left `uploading` gets 409 `finished` "This upload has finished.". The part numbers in a `/parts` request, and in a `complete` part list, must be unique integers from 1 to the part count, `ceil(original_bytes / 50 MiB)` (at most 103, well within R2's 10,000-part limit), and a `complete` list must be in ascending order; anything else gets 400 `bad_parts` "The part list is not valid.". None of these rejections makes an R2 call.

**Failed uploads.** In the first two cases below the row stays `uploading`, so nothing is lost and the member can carry on:
- When a part's retries run out, the Upload screen shows "The upload stopped. Keep this screen open and press Resume to continue where it left off, or Cancel upload." Resume continues from the parts saved in `localStorage` (step 3); Cancel upload calls the abort route.
- When `complete` fails with a 5xx or a network error, it shows "Could not finish the upload. Press Try again; if it keeps failing, cancel the upload and upload it later." Try again calls `complete` again with the same part ETags, which is safe after a `complete` that finished in R2 but not in D1 (see **Abort or Delete** in [API routes](#api-routes-all-under-the-app-origin)).
- While a `complete` for an upload is pending, the Upload screen disables its button, so one screen sends one `complete` at a time; a sibling `complete` comes only from Try again after a request that timed out on the browser but may still be running in the Worker.
- When `complete` returns 409 `cancelled`, the browser clears that upload's `localStorage` entry and reads the Waffle again (`GET /api/waffles/:id`) before saying anything, because a sibling `complete` (the request behind a timed-out Try again) may still be finishing the same upload. A 404 means it really was cancelled: the browser asks the member to upload the file again from the start. A Waffle that has left `uploading` is handled as 409 `finished` below. While the Waffle is still `uploading`, the browser shows "Finishing the upload…" and reads it again every 10 s for up to a minute, so it never offers Cancel upload while a sibling may still set `processing`. If it is still `uploading` after that minute, the screen shows "Could not finish the upload. Cancel the upload and upload the file again from the start.", with only Cancel upload.
- When `complete` returns 409 `finished`, an earlier `complete` already finished it (a double tap, or a response lost before Try again), so the browser clears that upload's `localStorage` entry and shows `processing` as on success.
- When R2 rejects the part list (for example ETags that no longer match), `complete` returns 400 `parts_rejected` "The uploaded parts could not be put together. Upload the file again from the start.", never R2's status or response body, and the Upload screen shows "Could not finish the upload. Cancel the upload and upload the file again from the start." with only Cancel upload. A 400 `bad_parts` or a 404 `not_found` from `/parts` or `complete` (a browser bug or a damaged `localStorage` entry) gets the same screen. The row stays `uploading` until the member cancels it or the sweep removes it.
- When `POST /api/uploads` returns 400 `bad_ext`, `bad_size` or `too_large`, the Upload screen shows its message and starts nothing.
- When `complete` returns 400 `size_mismatch` "The uploaded file is not the size this upload started with. Upload the file again from the start.", the finished original did not match `original_bytes` and the Worker has removed the upload (see **Abort or Delete**), so the browser clears that upload's `localStorage` entry and shows that message, asking for a fresh upload.

**Abandoned uploads.** An upload that is never finished or cancelled stays on its owner's This-week screen under Unfinished uploads (see [UI](#ui)) for 7 days. After that a lazy sweep removes it, matching the R2 lifecycle rule in [Storage layout](#storage-layout-r2-bucket-waffles-private). `GET /api/waffles` and `POST /api/uploads` start the sweep after they respond (`ctx.waitUntil`). For each `uploading` row whose `uploaded_at` is more than 7 days old, it runs the abort path from **Abort or Delete**: it aborts the multipart upload with the row's `upload_id` (`NoSuchUpload` counts as success, as when the lifecycle rule has already aborted it), then runs abort's guarded `DELETE … WHERE id = ? AND status = 'uploading'`. A row whose R2 abort fails stays for the next sweep, and the sweep writes one fixed line to the Worker log for it, `upload sweep: abort failed <waffle id>`, naming only the Waffle ID. When the sweep's `DELETE` removes the row, it deletes the original as abort does, writing `upload sweep: original delete failed <waffle id>` if that fails; when the `DELETE` changes no row, it leaves R2 alone. A `complete` racing the sweep loses or wins exactly as it does against an abort.

R2 needs a CORS rule allowing `PUT` from the app origin and exposing the `ETag` header, or the browser cannot read part ETags.

### Transcode

Runs as a GitHub Actions workflow, started by a `workflow_dispatch` event the Worker sends with the Waffle ID as an input. The Worker authenticates with a fine-grained PAT scoped to this repository with only the Actions: write permission, created with a one-year expiry. The workflow sets `run-name: transcode ${{ inputs.waffle_id }} (${{ inputs.env }})`, so each run in the Actions list shows the Waffle ID and env it was started for. A run title is display text, never a shell script, so this is the one place an input is interpolated directly; a hand-made dispatch's bad value shows there as it does in a step's `env:` header (see **Untrusted inputs**).

**Dispatch target.** The Worker sends `POST /repos/{GITHUB_REPO}/actions/workflows/{GITHUB_WORKFLOW}/dispatches` with `ref: main` and two inputs: `waffle_id` and `env` (`prod` or `dev`, from the `TRANSCODE_ENV` var; see [Environments](#environments)). The ref is fixed, with no var to change it, so a workflow change, Phase 3's tuning included, reaches `prod` and `dev` runs only once merged to `main`, where it has passed the required CI check (see **Untrusted inputs**). Hand dispatches also run from `main` only, since a run from another branch skips that check while holding the repository secrets. The Worker and the workflow both map `env` to a bucket (`prod` → `waffles`, `dev` → `waffles-dev`), so the Worker's R2 calls and the run use the same one. Each side checks the value for its own calls: the Worker refuses to dispatch or make any R2 call unless `TRANSCODE_ENV` is exactly `prod` or `dev`, and the workflow, which hand dispatches also reach, declares `env` as a `choice` input, backed by the validation script in **Untrusted inputs**. A `prod` run posts to `CALLBACK_BASE_URL` (a repository secret, so GitHub masks it in the public logs) followed by `/api/internal/transcode-callback`, so no URL appears in a run's public inputs; it fails before downloading anything if that secret is empty or does not start with `https://`. GitHub cannot reach `wrangler dev` on localhost, so a `dev` run writes to `waffles-dev` and always skips the callback. A local Waffle then finishes only through a signed callback sent by hand, or ends `failed` under the stuck-jobs rule below.

**Untrusted inputs.** The repository is public, so the workflow treats both inputs as untrusted, even though only the Worker's token or a collaborator with write access can dispatch it. No step interpolates `${{ inputs.* }}` or `${{ secrets.* }}` inline in a `run:` script or a script-like action input (such as `actions/github-script`'s `script:`); each value reaches the script only through the step's `env:` block (for example `WAFFLE_ID: ${{ inputs.waffle_id }}`) and is quoted where the script uses it (`"$WAFFLE_ID"`). No step turns on shell tracing (`set -x`). A CI check, run on every push and pull request with no path filters, parses every workflow file and composite action and fails if any step's `run:` or `script:` value holds a `${{ }}` expression that references `inputs`, `github.event.inputs` or `secrets`, or if anything turns on shell tracing (a `set` flag group containing `x`, `set -o xtrace`, or `-x` in a `shell:` value). Its patterns live in a committed script, not in a workflow file, because GitHub evaluates `${{ }}` anywhere in a workflow file. The check is a required status check on `main` (through branch protection or a ruleset, with no bypass), so a workflow change that fails it cannot land on the branch every dispatch runs from; this guards against mistakes, not against a collaborator with write access who also edits the check. The first step after `actions/checkout` (which takes no inputs) runs a committed validation script, which checks that `WAFFLE_ID` matches the ULID pattern `^[0-7][0-9A-HJKMNP-TV-Z]{25}$` (the `waffles.id` format in the [Data model](#data-model)) and, as a backstop to the `choice` input, that `RUN_ENV` (from `inputs.env`) is exactly `prod` or `dev`. If either does not match, the script prints a fixed message that never includes the rejected value and exits non-zero, failing the run before anything is downloaded; no callback is sent, not even the failure one in step 6. CI unit-tests the script with a bad `RUN_ENV` (GitHub rejects one before a run starts, so only these tests reach that check), a `WAFFLE_ID` that is not a ULID, and one that holds shell metacharacters (for example `$(id)`), asserting for each a non-zero exit, the fixed message, and stdout and stderr that do not contain the value. GitHub prints a step's `env:` block in that step's log header, so a hand-supplied value does appear there; this is accepted, since only the Worker or a collaborator with write access can dispatch. Such a run can only come from a hand-made dispatch, so its Waffle, if any, ends `failed` under the stuck-jobs rule below.

Steps:

1. Download the original from R2 with the S3 API.
2. `ffprobe` it: video codec, audio codec (if there is an audio stream), H.264 profile, pixel format, resolution, frame rate, rotation, duration, colour transfer, `creation_time`.
3. Choose a path:
   - A source whose first video stream is H.264 with a `profile` of `Constrained Baseline`, `Main` or `High` and a `pix_fmt` of `yuv420p` or `yuvj420p` (8-bit 4:2:0; `yuvj420p` is the full-range form many Android cameras record), that has no audio stream or whose first audio stream is AAC, and whose `color_transfer` is neither of the HDR values in **HDR is tone-mapped to SDR** below (common on Android): remux only, `-map 0:v:0 -map '0:a:0?' -c copy -movflags +faststart` (the `?` makes the audio map optional). The video codec, `profile` and `pix_fmt` must match exactly, and a missing one re-encodes; an audio stream whose codec is missing or not AAC re-encodes; a missing `color_transfer` counts as SDR. The container is not checked: ffprobe reports `.mov`, `.mp4` and `.m4v` as the same format, and `-c copy` rewraps their H.264 and AAC into `playback.mp4` unchanged. Stream copy keeps the H.264 stream's range flag, so a full-range source plays with the right levels. No quality loss.
   - Everything else (HEVC `.mov` from iPhones, HDR, 10-bit or 4:2:2 H.264 such as High 10 or High 4:2:2, other video or audio codecs): re-encode with the settings below, since most phones cannot play those H.264 variants.
4. Extract a poster frame at about 1 second as a JPEG (1280 px wide).
5. Upload the playback MP4 and poster to R2.
6. POST the result to the Worker's callback (see **Callback** below): `ready` with the playback results, or `failed` with the error text for any failure after the input check (see **Failure output** below).

**Failure output.** Raw tool output can carry file paths, metadata or URLs, so it never reaches the public log (see the public-repo rule in [CLAUDE.md § Project rules](../CLAUDE.md#project-rules)). The job works on fixed local filenames (`original`, `playback.mp4`, `poster.jpg`), runs `ffprobe` with `-hide_banner -loglevel error` and `ffmpeg` with `-hide_banner -nostdin -nostats -loglevel error`, and sends every tool's stderr in steps 1 to 5 to a file in `$RUNNER_TEMP`. Each step has a category: `download` (step 1), `probe` (2), `encode` (3, including the tone-map and the size-cap re-encode), `poster` (4) and `upload` (5). When a step fails, the log shows only a fixed line naming the Waffle ID and that category. The callback's error text is `<category>: <detail>`, the job's counterpart of the rejected-dispatch format below:
- `download` and `upload` use a fixed detail ("could not read the original from storage", "could not save the playback copy to storage"), since storage-client errors echo endpoint URLs, bucket names and keys.
- `probe`, `encode` and `poster` use the first non-empty line of the tool's stderr file (with `-loglevel error` the last line is usually a generic "Conversion failed!"), with each byte outside printable ASCII replaced by a space, or "no error output" if the file is empty.
- The job truncates the whole text to 300 characters, and the Worker truncates `error` to 300 characters again before storing it, so a hand-signed callback can't store more.

**Callback.** The job POSTs a JSON body to `/api/internal/transcode-callback`. It sends two headers: `X-Waffles-Timestamp` (Unix seconds) and `X-Waffles-Signature`, the hex HMAC-SHA256 of `<timestamp>.<raw body>` keyed with `TRANSCODE_CALLBACK_SECRET`. The body holds `waffle_id`, `status` (`ready` or `failed`), and then either the error text or the values the run measured (`original_codec`, `is_hdr`, `playback_bytes`, `duration_s`, `width`, `height`, `recorded_at` if present). The Worker sets `playback_key` and `poster_key` itself from the [Storage layout](#storage-layout-r2-bucket-waffles-private) and never reads keys from the body. The Worker checks the signature before parsing the body, compares it in constant time, and rejects a timestamp more than 5 minutes from its own clock. Its responses:
- **401**, empty body: a missing or wrong signature, or a stale timestamp. Nothing is read or changed; the Worker writes only the fixed line `callback rejected: 401` to its log.
- **400**: a signed body that is malformed: `waffle_id` is not a ULID, `status` is not `ready` or `failed`, a `ready` body lacks a measured value other than `recorded_at` or has one of the wrong type, or a `failed` body has no error text. The Worker writes only the fixed line `callback rejected: 400` to its log.
- **404**, with the fixed body `{"error":"no_waffle"}`: no Waffle has that ID, for example because it was deleted while its run was in progress. The body marks the 404 as the Worker's, so the job can tell it from a 404 sent by some other host.
- **200** in every other case, whether or not the result changed the Waffle. A `ready` result applies in one guarded `UPDATE … SET status = 'ready', ready_at = now, error = NULL, … WHERE id = ? AND status IN ('processing', 'failed')`. Only a request that changed the row takes the Discord claim (see [Discord notification](#discord-notification)), so a duplicate `ready` is a no-op. A `ready` that arrives after the stuck-jobs rule or a rejected dispatch set `failed` still recovers the Waffle, because the copy it describes exists. A `failed` result applies in one guarded `UPDATE … SET status = 'failed', error = ? WHERE id = ? AND status IN ('processing', 'failed')`, so it never replaces `ready` and changes no other column. On a Waffle already `failed` it changes only `error`, so a run's real cause replaces the stuck-jobs or rejected-dispatch text.

These `callback rejected:` lines hold nothing from the request (no body, header or Waffle ID), so a secret mismatch, which fails every transcode, leaves a Worker-side trace; any unsigned request to the public route writes the same 401 line, so the line alone is not tied to a run.

The job sends each callback without following redirects, gives each request a 20-second timeout (a timeout counts as a network error) and signs every attempt again with the current timestamp, so a retry never fails the 5-minute check. On a 5xx, a 429 or a network error, it retries the callback 3 times, waiting 30, 60 and 120 seconds, and then fails the run. Any other non-2xx response (401, 400, 404 or anything else) is final and fails the run at once. On a 404 whose body is exactly `{"error":"no_waffle"}`, the job first deletes the playback copy and poster it uploaded in step 5, if any, so a Waffle deleted during its run leaves nothing behind in R2, except in the window noted under [Discord notification](#discord-notification). Any other 404 (for example from a wrong host behind `CALLBACK_BASE_URL`) deletes nothing. A failed cleanup delete is not retried and does not change the outcome: the job still tries the other object, writes the fixed line `cleanup failed <waffle id>` to the log, naming only the Waffle ID and never a key, bucket or storage-client error, and fails the run as for any 404. A failed callback ends the run with one fixed line naming its category and the Waffle ID: `callback: <status code> <waffle id>` for a final non-2xx (for example `callback: 401 <waffle id>`, after any `no_waffle` cleanup), or `callback: retries exhausted <last status code or network> <waffle id>` once the retries run out. A failed run shows as failed in the Actions tab with only its fixed line (see **Failure output**, or the `callback:` line above). If its Waffle is still `processing`, it stays that way until the stuck-jobs rule sets it `failed`.

**Racing runs.** Two runs can report for one Waffle: a stuck run that finishes after a Retry, or a dispatch that timed out but did start a run. Both encode the same untouched original to the same R2 keys, so either `ready` describes a valid copy, and the guards in **Callback** decide what D1 records: a `failed` from one run can be followed by the other's `ready`, which recovers the Waffle, but nothing undoes `ready` or posts to Discord twice. Two limitations are accepted, since both copies come from the same original and a Retry or the newer run's `ready` fixes the Waffle:
- The losing run still uploads its copy and poster over the winner's, so the row's `playback_bytes` and dimensions may differ slightly from the file, and someone watching at that moment can see a glitch that a reload clears.
- A callback carries no per-dispatch nonce, so a signed callback can be replayed within its 5-minute window, and a superseded run's result can land after a Retry or after a newer dispatch was rejected. A stale `failed` can then show the Waffle `failed`, with an out-of-date error, while a newer run is still going; of two `failed` callbacks, the last to arrive sets the error text.

**Stuck jobs.** The workflow job sets `timeout-minutes: 120`. A Waffle still `processing` 150 minutes after its last dispatch (120 minutes of run time plus 30 minutes of queue slack, which assumes a hosted runner picks the job up within 30 minutes) is presumed to have a run that will never report back, so it is set to `failed` with the error "The transcode did not report back. Your upload is safe. The uploader can press Retry; if it fails again, tell the admin to check the workflow run." The admin (see **Rejected dispatch**) finds the run with README [Transcodes that did not report back](../README.md#transcodes-that-did-not-report-back). The check runs lazily, as one guarded statement (`UPDATE … SET status = 'failed', error = … WHERE status = 'processing' AND processing_at < now − 150 min`), so it can never overwrite a callback's `ready`. `GET /api/waffles` and `GET /api/waffles/:id` apply it before responding. The retry route applies it first, then requires `failed`, then sets `status = processing` and `processing_at = now` in one `UPDATE … WHERE status = 'failed'`, so a stuck Waffle can be retried and a double tap dispatches once. A Waffle that is not `failed`, including one a late `ready` already recovered (see **Callback**), gets 409 and nothing is dispatched. This rule needs no scheduled job; the Worker's one scheduled task, the token expiry check below, never touches Waffles. If Phase 3 encode times approach 120 minutes, raise both values together.

**Rejected dispatch.** A dispatch, from `complete` or Retry, fails when GitHub answers anything but 2xx, or on a network error or timeout. The Waffle is then set to `failed` in one guarded `UPDATE … WHERE id = ? AND status = 'processing'`, so it never overwrites a callback's `ready`. `error` is stored as `<status code or network>: <fixed message>`; GitHub's response body and headers are never stored or logged. The admin is the member who holds the Worker and GitHub secrets.
- A 401, 403 or 404 usually means the token has expired or lost access: "GitHub refused the transcode request; its access token may have expired or the setup changed. Your upload is safe. Tell the admin; the uploader can press Retry once it is fixed."
- Any other status code, a network error or a timeout reads: "GitHub could not start the transcode. Your upload is safe. The uploader can press Retry later; if it keeps failing, tell the admin."
- A timeout is ambiguous: GitHub may have started the run anyway, so a Retry can start a second run. This is accepted, since both runs encode the same untouched original (see **Racing runs** above).

**Token expiry check.** The PAT (`GITHUB_DISPATCH_TOKEN`) expires, so the Worker checks it daily instead of waiting for a member's upload to fail. A Cron Trigger (`[triggers] crons = ["0 14 * * *"]`, 14:00 UTC) calls the Worker's `scheduled` handler (see **Worker entry** in [Architecture](#architecture)), which sends `GET /repos/{GITHUB_REPO}/actions/workflows` with the token; it reads the workflow list and starts no run.
- A 401, 403 or 404 means the token is invalid or has lost access.
- On a 2xx, the `github-authentication-token-expiration` response header gives the expiry as a UTC timestamp (for example `2026-11-01 00:00:00 UTC`). An expiry within 14 days counts as expiring. A missing header (a token with no expiry) never alerts; an unparseable one is treated like any other failure below.
- An invalid or expiring token posts a fixed plain-text message, with no @mention, to the channel webhook (`DISCORD_WEBHOOK_URL`) once per daily run until the token is renewed. Expiring: "Admin: the GitHub token that starts transcodes expires on `YYYY-MM-DD`. Renew it before uploads fail; see the README's token rotation steps." Invalid: "Admin: GitHub refused the token that starts transcodes, so uploads will fail. Renew it; see the README's token rotation steps."
- Any other status code, a network error or an unparseable header alerts nothing and is checked again the next day. The alert post makes one attempt, with no retry, so a failed post is repeated the next day.
- A failed run writes one fixed line to the Worker log: `token check: <status code, network or unparseable>` when GitHub's answer alerts nothing, `token check: error` when the run throws (see **Unexpected errors** below), and `token check alert failed: <status code or network>` for each post that fails. A run that gets a 2xx with a missing or parseable header, or a 401, 403 or 404, and posts any alert it owes writes no line. The Wrangler config turns on Workers Logs (`[observability] enabled = true`), so these lines are kept for the admin to read after the run. Like the dispatch, the check never stores or logs GitHub's or Discord's response body or headers, or the webhook URL; only the expiry date appears in the message.
- **Unexpected errors.** The `scheduled` handler catches any exception the check throws, writes the fixed line `token check: error` to the Worker log, with no response body, headers or token, and returns, so the next day's run tries again.
- The check keeps no state: no D1 row, failure count or banner. Accepted: a check or alert path that keeps failing shows only in its log lines, which Workers Logs keeps for a limited time; the check repeats daily, so recent lines exist. README [Rotating the GitHub dispatch token](../README.md#rotating-the-github-dispatch-token) says when to read them.
- Accepted: a rare rate-limit 403 raises a false "refused" alert that the next day's check clears.

Re-encode settings, the starting point to tune in Phase 3:

```bash
ffmpeg -hide_banner -nostdin -nostats -loglevel error -i original \
  -map 0:v:0 -map '0:a:0?' \
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

The repository is public, so Actions minutes are free and the workflow logs are public too; the job follows the public-repo rule in [CLAUDE.md § Project rules](../CLAUDE.md#project-rules). About 4 Waffles a week is a light load, but a slow-preset 4K encode on a hosted runner can take a long time. Measure in Phase 3; if it is too slow, use `-preset medium` or move the job to Cloudflare Containers.

### Storage layout (R2 bucket `waffles`, private)

| Object | Key |
| --- | --- |
| Original | `originals/{waffleId}/source.{ext}` |
| Playback copy | `playback/{waffleId}/video.mp4` |
| Poster | `playback/{waffleId}/poster.jpg` |

The Worker presigns every R2 URL the browser uses: part PUT URLs expire after 1 hour, and the playback, poster and original-download GET URLs after 6 hours. R2 supports range requests on the GET URLs, so seeking works. Abort any multipart upload left incomplete for more than 7 days with an R2 lifecycle rule; the Worker's sweep of [abandoned uploads](#upload) removes the matching `uploading` rows.

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
  upload_id          TEXT NOT NULL,     -- R2 multipart upload ID, set by POST /api/uploads; read by abort and the sweep
  caption            TEXT,
  original_key       TEXT NOT NULL,
  original_bytes     INTEGER NOT NULL,  -- declared size from POST /api/uploads; sets the part count
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

The Waffle week rule (Wednesday 00:00 to Tuesday 23:59 in `GROUP_TZ`, how `week_start` is set, and the heatmap **year**) is in [CLAUDE.md § Project rules](../CLAUDE.md#project-rules). Beyond it:

- One member is on Central time; the cutoff is informal, so the whole group uses ET.
- The upload form defaults to the current week; the previous-week option is for a Waffle filmed Tuesday night and posted Thursday.
- A member may post more than one Waffle in a week. The heatmap counts the member as present; the archive shows them all.

## Auth, API and Discord

Sign-in is Discord OAuth; a person gets in only if they belong to the friends' Discord server and are an active member in `members`.

### Auth

- OAuth2 authorization-code flow with scopes `identify guilds`. On callback, fetch `/users/@me` and `/users/@me/guilds`, and require `DISCORD_GUILD_ID` in the list.
- Then require the user's ID in `members` with `active = 1` (seeded by hand: 4 rows). Anyone else sees a "not a member" page. Removing a member sets their `active` to 0; their Waffles stay.
- Sessions: an opaque random token in an `HttpOnly; Secure; SameSite=Lax` cookie, stored hashed in `sessions`, 30-day expiry renewed on use.
- Every session lookup joins `members` and accepts the session only while the member's `active = 1`, so a removal takes effect at the member's next request. A session whose member is removed is treated as signed out: the Worker deletes its `sessions` row, so setting `active` back to 1 does not revive it, the response clears the cookie, a page redirects to sign-in and an `/api` route returns 401. Every presigned URL already issued to the member stays valid until it expires (see [Storage layout](#storage-layout-r2-bucket-waffles-private)); part URLs let them finish PUTting parts, but not call `/parts` or `complete`, so the sweep removes that upload.
- The OAuth `state` parameter is a short-lived random value bound to the browser, so it needs no signature: `/auth/login` also sets it in an `HttpOnly; Secure; SameSite=Lax` cookie that expires in 10 minutes, and `/auth/callback` requires the query's `state` to match that cookie, then clears it. A callback without the cookie, or with a mismatch, creates no session.

### API routes (all under the app origin)

**Error codes.** Every 400, 404 and 409 from the upload routes (`/api/uploads` and its `/parts`, `complete` and `abort`), and every 400 from `GET /api/heatmap`, has a JSON body `{ "code": "<code>", "message": "<fixed message>" }`, where `code` is a stable snake_case name. The browser chooses what to do from `code`, never from the message text or the status alone. Each code and its message are given where the rejection is defined: **Upload checks** and **Failed uploads** in [Upload](#upload), **Abort or Delete** below, and the `/api/heatmap` row of the table.

| Method | Path | Purpose |
| --- | --- | --- |
| GET | `/auth/login` | Redirect to Discord |
| GET | `/auth/callback` | Finish OAuth, create session |
| POST | `/auth/logout` | Delete session |
| POST | `/api/uploads` | Create R2 multipart upload and Waffle row, storing `upload_id`; body: file name, size, type, caption, `week_start`; starts the [abandoned-upload sweep](#upload) |
| POST | `/api/uploads/:id/parts` | Owner only: presigned PUT URLs for a list of part numbers (see **Upload checks** in [Upload](#upload)) |
| POST | `/api/uploads/:id/complete` | Owner only: complete multipart upload with part ETags, dispatch transcode (see **Upload checks** in [Upload](#upload)) |
| POST | `/api/uploads/:id/abort` | Owner only: abort upload, delete row (see **Abort or Delete**) |
| GET | `/api/waffles` | List; filters `member`, `year`, `week_start`; newest first; leaves out `uploading` Waffles; starts the [abandoned-upload sweep](#upload) |
| GET | `/api/waffles/:id` | Metadata plus presigned playback, poster and original URLs; 404 for an `uploading` Waffle unless the caller is its owner, who gets metadata only, with no URLs |
| DELETE | `/api/waffles/:id` | Owner only: delete row and all three R2 objects, and its Discord message if there is one (see **Abort or Delete**) |
| POST | `/api/waffles/:id/retry` | Owner only: re-dispatch a failed transcode, including one that did not report back |
| POST | `/api/waffles/:id/discord` | Owner only: post a `ready` Waffle that has no Discord message yet |
| GET | `/api/heatmap?year=YYYY` | Week starts × members with counts, plus each member's streak for that year (current or best, per [UI](#ui)); `uploading` Waffles are not counted; a missing or non-integer `year`, or one after the current heatmap year (the year that holds the current Waffle week; see [Waffle week](#waffle-week)), gets 400 `bad_year` "No heatmap for that year." |
| POST | `/api/internal/transcode-callback` | From the GitHub job; HMAC-signed; sets ready or failed (see [Transcode](#transcode)) |

**Delete and abort order.** R2 goes first, so a failure leaves the row in place to try again rather than a deleted row with orphaned files nobody knows about. Delete removes the three keys from the [Storage layout](#storage-layout-r2-bucket-waffles-private) whether or not the row records them (a run in progress may already have uploaded a playback copy), in the order poster, playback copy, original, so a partial failure always keeps the untouched original. Abort aborts that Waffle's R2 multipart upload, using the row's `upload_id`. A missing object or upload counts as success (a 204 or 404 from a delete, a 404 `NoSuchUpload` from an abort), so repeating the action after a failure is safe. Only when every R2 call succeeds does the Worker delete the Discord message (Delete only, as in [Discord notification](#discord-notification)) and then the row. Abort is the one exception: its only R2 call before the row `DELETE` is the multipart abort, and it deletes the original after the row is gone, logging rather than failing if that delete fails (see **Abort or Delete**). If any R2 call before the row delete fails, the row and its Discord message stay, any files already removed stay removed, and the route returns 502 with only the fixed message "Could not delete the video files; try again, or tell the admin.", never R2's status or response body.

**Abort or Delete.** Abort works only while `status = uploading` and Delete only once the Waffle has left `uploading`; otherwise abort returns 409 `cannot_cancel` "This upload has finished; delete the Waffle instead." and Delete returns 409 with "This upload is still in progress; cancel the upload instead.", and neither touches R2. On `cannot_cancel` the browser clears that upload's `localStorage` entry and shows the message. An abort on another member's row gets 404 `not_found` "No such upload." at any status, as `/parts` and `complete` do, before any R2 call. The 409s and the 400 that `complete` returns below carry the codes given for the same messages in **Upload checks** and **Failed uploads** (`cancelled`, `finished`, `size_mismatch`). `complete` likewise requires `status = uploading`, with the 409s in **Upload checks** (see [Upload](#upload)). Abort ends with one guarded `DELETE … WHERE id = ? AND status = 'uploading'`, and `complete`, after it finishes the multipart upload, sets `status = processing` in one guarded `UPDATE … WHERE id = ? AND status = 'uploading'`. When the two race, whichever of these statements runs first wins and the other route returns 409. When abort's `DELETE` removes the row, abort then deletes the row's `original_key` (read before the `DELETE`), since an earlier `complete` may have finished the multipart upload without recording it. This is safe: once the row is gone, no `complete` can set `processing`. If that delete fails, abort still succeeds (its row is gone) and writes one fixed line to the Worker log, `abort: original delete failed <waffle id>`, naming only the Waffle ID. An abort whose `DELETE` changes no row reads the row again. If the row has left `uploading`, abort lost to `complete`: it has done nothing but find no upload to abort (`NoSuchUpload`, counted as success), so the Waffle goes on to `processing`, and abort returns its 409 message above. If the row is gone (a double Cancel or a Cancel after the sweep), the upload is already cancelled and abort returns success; so does an abort that finds no row to begin with. A `complete` whose R2 call fails with `NoSuchUpload` first checks with a HEAD whether the original object exists. If it does, an earlier `complete` finished the multipart upload but never recorded it (its Worker failed before the `UPDATE`, or its response was lost and the member pressed Try again), so this `complete` carries on normally: it runs the guarded `UPDATE` and dispatches. A failed HEAD fails `complete` like any other 5xx (see [Upload](#upload)), leaving the row `uploading`. Before its `UPDATE`, every `complete` that has an original, whether it finished the multipart upload itself or found the original by HEAD, checks the original's size against `original_bytes` (a HEAD after finishing the upload; the size from the HEAD that found it). R2 bounds only the part count and each part's size, not the total, so this is what holds an upload to its declared size. A failed HEAD fails `complete` like any other 5xx. A 404 from the HEAD after finishing means an abort or the sweep has already removed the row and deleted the original, so `complete` returns 409 "This upload was cancelled." and touches nothing. If the sizes differ, `complete` never sets `processing`: it runs abort's guarded `DELETE … WHERE id = ? AND status = 'uploading'` and, when that removes the row, deletes the original (if that delete fails, it still returns the 400 and writes `complete: original delete failed <waffle id>`), safe for the same reason as abort's delete. When that `DELETE` changes no row, an abort, the sweep or a sibling `complete` (which finds the same mismatch) has removed the row and deletes the original itself, so this `complete` leaves R2 alone. Either way it returns 400 with only the fixed message "The uploaded file is not the size this upload started with. Upload the file again from the start.", never the sizes. A `complete` whose `UPDATE` changes no row reads the row again: if the row is gone, abort or the sweep won, and it returns 409 "This upload was cancelled."; if it has any other status, another `complete` won, and it returns 409 "This upload has finished." without touching R2. When the row is gone, a `complete` that finished the multipart upload itself deletes that original (if that delete fails, it still returns the 409 and writes `complete: original delete failed <waffle id>`), safe for the same reason. When its `UPDATE` loses that way, a `complete` that found the original by HEAD did not finish it and leaves R2 alone; the abort or sweep that removed the row deletes it. A `complete` whose `NoSuchUpload` finds no original has no upload left to finish (abort's R2 call or the R2 lifecycle rule ended it, or a sibling `complete` is still finishing it in R2 and has not yet made the original visible), so it reads the row again: if the row is gone or still `uploading` it returns 409 "This upload was cancelled."; if the row has any other status, it returns 409 "This upload has finished.". It deletes nothing, neither the row nor the original: a sibling `complete` may be about to finish the upload and set `processing`, and only that sibling, or an abort or the sweep that wins the row, may decide the original's fate. A dead `uploading` row left this way is harmless; it waits under Unfinished uploads for Cancel upload or the sweep. This holds for a retried `complete` racing an abort too. In short, a `complete` deletes R2 objects only in two cases, the original it finished itself after its own `UPDATE` lost to a row already gone, and an original of the wrong size after its own guarded `DELETE` removed the row, and that `DELETE` is the only row it ever deletes. A Waffle never returns to `uploading`, so Delete cannot race `complete`. A Worker that fails between finishing the multipart upload and its row update leaves the row `uploading` with its original in R2: a retried `complete` recovers it, and a cancel or the sweep deletes the row and then the original. Accepted limitation: an original whose delete fails after its row is gone (logged by abort, the sweep or `complete`), or whose Worker stops before deleting it, is left with no row and removed by hand.

Only the uploader may send parts for, complete, abort, delete, retry or re-post their own Waffle. Any member may download any Waffle's original; its presigned URL sets `Content-Disposition: attachment` with a readable name such as `2026-10-14_alex.mov`, built from `week_start`, the uploader's `display_name` and the allowlisted `ext`. The Worker strips CR, LF and other control characters, quotes (`"`, `'`), `/` and `\` from the display name, and uses `member` if nothing is left. The header is `attachment; filename="<name>"; filename*=UTF-8''<percent-encoded name>`, where the quoted `filename` replaces each non-ASCII character with `_`. Every `/api` route except the callback requires a session.

### Discord notification

- A channel webhook (URL kept as a Worker secret) posts once, when a Waffle reaches `ready`, never at upload start. The same webhook also carries the admin's alerts from the [token expiry check](#transcode); the rules below cover Waffle posts only.
- Message: "🧇 **Alex** posted a Waffle for the week of Oct 14 · 6:42" plus the caption and the app link `/w/{id}`. Post with `?wait=true` so Discord returns the message, and store its ID in `discord_message_id`.
- Post only when `discord_message_id IS NULL`, and claim the post first with one guarded statement (`UPDATE … SET discord_posting_at = now WHERE status = 'ready' AND discord_message_id IS NULL AND (discord_posting_at IS NULL OR discord_posting_at < now − 2 min)`); only the request that changed the row posts. So a duplicate callback or a second tap on "Post to Discord again" can't post twice, and a claim left by a Worker that died mid-post expires after 2 minutes. The post's result clears `discord_posting_at`.
- The callback posts after it responds (`ctx.waitUntil`), so a slow Discord never delays the GitHub job.
- A failed post never changes `ready`: the Waffle stays watchable. On a 429, wait the `Retry-After` time and try again, up to 3 attempts; a `Retry-After` over 10 seconds counts as a failure. Any other non-2xx response (a revoked webhook URL returns 401 or 404), a network error or a third 429 sets `discord_error` to the status code (or `network`) and a fixed message; the webhook URL and the response body are never stored. A successful post clears `discord_error`.
- While a `ready` Waffle has no `discord_message_id` and no live claim, the owner's Waffle screen shows "Not posted to Discord" (with the `discord_error` text, if any) and a **Post to Discord again** action. Its route, `POST /api/waffles/:id/discord`, takes the same claim and runs the same post; it returns 409 when the Waffle is not `ready`, is already posted or is being posted.
- Deleting a Waffle deletes its Discord message when `discord_message_id` is set. When it is NULL (never posted, or every post failed), there is nothing to delete and the Waffle is deleted without calling Discord. A failed message delete (including a 404 for a message already removed in Discord) never blocks deleting the Waffle.
- Accepted limitation: a post that Discord accepted but whose response was lost (a network error or timeout), or that finishes after its Waffle was deleted, leaves a message the app has no ID for. A re-post then makes a second message, and the orphan is removed by hand in Discord.
- Accepted limitation: a transcode callback that lands between Delete's R2 deletes and its row delete gets 200, so the job's new playback copy and poster (and any Discord post the callback claims) outlive the Waffle and are removed by hand.
- The link sits behind sign-in, so Discord shows a plain link, not a preview. Adding a poster image to the embed would need a public or long-lived URL; leave it out of the MVP.
- A bot is only worth adding later for slash commands (for example `/waffle status` to list who has not posted this week).

## UI

The app is a mobile-first PWA with five screens, installable from the home screen (a manifest and icons) with no offline caching; most uploads happen from a phone, so every screen must work one-handed at 375 px wide.

| Screen | Path | Contents |
| --- | --- | --- |
| This week | `/` | Who has posted this Waffle week (4 avatars, filled or empty), the latest Waffles, a large Upload button; the member's own Unfinished uploads |
| Upload | `/upload` | File picker (`accept="video/*"`), caption, week selector (this week / last week), progress bar with MB sent and time left |
| Waffle | `/w/:id` | Player, member, week, caption, duration; Download original; Delete and Retry for the owner, and Post to Discord again while it has no Discord message |
| Archive | `/archive` | Grid of poster cards grouped by week, newest first; filter by member and year |
| Participation | `/participation` | The heatmap for a selected year |

**Unfinished uploads**

- `uploading` Waffles appear only here, on their owner's This-week screen (the page's server load reads the member's own `uploading` rows); listings, the archive, the This-week "who has posted" avatars and the heatmap leave them out, and the [abandoned-upload sweep](#upload) removes them.
- Each shows its caption and week with **Resume** and **Cancel upload**. Resume opens the Upload screen to pick the same file again and continues from the parts saved in `localStorage`, so it is offered only on the device that started the upload. Cancel upload calls `POST /api/uploads/:id/abort`.

**Upload screen behaviour**

- Warn that the screen must stay on while uploading, and request a Screen Wake Lock where the browser supports it.
- Show `processing` after the upload completes, then update to `ready` (poll every 10 s) with a link to the Waffle.
- Before starting, check the picked file's size: a file over 5 GiB shows "Waffles can be up to 5 GiB." and starts nothing (the Worker repeats the check in **Upload checks** in [Upload](#upload)).
- Resume an interrupted upload when the same file is picked again.
- On a failed part or `complete`, show the message and actions from **Failed uploads** in [Upload](#upload).

**Player**

- `<video controls playsinline preload="metadata" poster=…>` pointing at the presigned playback URL.
- While `processing`, show the poster if present and a "getting it ready" note; if `failed`, show the error and Retry.

**Participation heatmap**

- One row per member (in `sort_order`) plus a top "Group" row; one column per Waffle week, Wednesdays across the year, months labelled above.
- Member cells: empty or filled. Group cells: 0 to 4 shades for how many members posted.
- The year selector offers only years up to the current heatmap year (see [Waffle week](#waffle-week)), never a later one.
- The current week is outlined; future weeks are blank and not clickable.
- Hover or tap shows the week and who posted; clicking a filled cell opens that Waffle (or the week in the archive if there are several).
- Below the grid: each member's count for the year and a streak. A streak is a run of consecutive weeks with a Waffle from them, and it carries across calendar years: a week in late December and the next week in early January continue one run.
  - Current year (the heatmap year that holds the current Waffle week; see [Waffle week](#waffle-week)): **Current streak**, the run counting back from last week, plus the current week once they have posted in it.
  - Past year: **Best YYYY streak** (for example "Best 2025 streak") replaces it: the longest run they reached in any week of that year, counting the weeks a run carried in from earlier years.
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

Secrets (set with `wrangler secret put`): `DISCORD_CLIENT_ID`, `DISCORD_CLIENT_SECRET`, `DISCORD_GUILD_ID`, `DISCORD_WEBHOOK_URL`, `R2_ACCESS_KEY_ID`, `R2_SECRET_ACCESS_KEY`, `R2_ACCOUNT_ID`, `GITHUB_DISPATCH_TOKEN` (the fine-grained PAT described in [Transcode](#transcode), rotated as the README describes), `TRANSCODE_CALLBACK_SECRET`. One R2 API token, with Object Read & Write permission applied to only the `waffles` and `waffles-dev` buckets, covers both, so the same R2 keys serve the deployed Worker, local dev and the workflow. Sharing them means a leaked local copy can read and delete production originals; this is accepted for four members, and the README's [rotation steps](../README.md#rotating-the-r2-keys) replace the token everywhere it lives. The README also has the [callback secret's rotation steps](../README.md#rotating-the-callback-secret). The GitHub workflow needs the R2 keys, the callback secret and `CALLBACK_BASE_URL` (the deployed Worker's origin, `https://waffles.<account>.workers.dev` until the domain cutover) as repository secrets.

Vars (non-secret, in the Wrangler config's `[vars]`): `GITHUB_REPO` (this repository's `owner/name`), `GITHUB_WORKFLOW` (the transcode workflow's file name, for example `transcode.yml`), `TRANSCODE_ENV` (which also picks the Worker's bucket; see the dispatch target in [Transcode](#transcode)), `GROUP_TZ` and `APP_ORIGIN` (the app's public origin, with no trailing slash). The Worker builds the OAuth `redirect_uri` (`APP_ORIGIN` + `/auth/callback`) and the Discord post's `/w/{id}` link from `APP_ORIGIN`. `[vars]` sets `TRANSCODE_ENV = "prod"` and `APP_ORIGIN = "https://waffles.<account>.workers.dev"` for the deployed Worker.

Local dev reads `.dev.vars` (gitignored), which overrides `TRANSCODE_ENV` with `dev` and `APP_ORIGIN` with the local dev server's origin (for example `http://localhost:8787` under `wrangler dev`). It also holds a local value for every secret above, so a checkout without the file cannot dispatch at all. Its `DISCORD_WEBHOOK_URL` points at the test channel. Its `TRANSCODE_CALLBACK_SECRET` is a dev-only value, never the production one, so the deployed Worker rejects a callback signed by hand for local dev, and a leaked laptop copy of the file does not expose the production callback secret (the R2 keys, Discord app credentials and dispatch token in it are the shared production ones). Sessions need no secret: they are random tokens stored hashed in each environment's own D1 (see [Auth](#auth)), so a session created locally does not exist in production.

Two settings outside the repo make local sign-in and upload work: the Discord app lists the local `APP_ORIGIN` + `/auth/callback` as a second redirect URI, and the `waffles-dev` bucket's CORS rule (see [Upload](#upload)) allows the local `APP_ORIGIN`. Both must match it exactly, port included.

**Callback rate limit.** A Cloudflare WAF rate-limiting rule on the `brownsymphony.app` zone matches requests whose path is `/api/internal/transcode-callback` and blocks an IP that sends more than 10 in 10 seconds, for 10 seconds, with a 429 (the Free plan allows one such rule, counted per IP). The rule runs before the Worker, so a blocked request writes no Worker log line; this keeps an unsigned flood from evicting real log lines or using up the Workers Logs quota. Legitimate traffic never comes close: each run sends one callback, and at most 3 retries spaced 30 s or more apart, and four members' runs at once are at most 4 requests from one IP. A real callback that did trip it gets a 429, which the job retries (see **Callback** in [Transcode](#transcode)). Zone rules cannot cover a workers.dev route, so the rule is added with the custom domain (step 2 of the [Domain cutover](#domain-cutover-after-the-prototype-is-proven)), and the callback route stays unlimited on workers.dev until step 9 disables that route.

### Domain cutover (after the prototype is proven)

1. Buy `brownsymphony.app` and use Cloudflare for its DNS.
2. Add a custom domain to the Worker. Serve it at `waffles.brownsymphony.app`, leaving the apex free. Add the zone's [callback rate limit](#environments) rule in the same step.
3. Add `https://waffles.brownsymphony.app/auth/callback` as a redirect URI in the Discord developer portal.
4. Add the new origin to the `waffles` bucket's CORS rule.
5. Set the `APP_ORIGIN` var to `https://waffles.brownsymphony.app` and redeploy, so sign-in redirects and Discord links use the new origin.
6. Set the `CALLBACK_BASE_URL` repository secret to `https://waffles.brownsymphony.app`, or transcode callbacks keep going to the workers.dev origin. A run that started before the change still reports to the workers.dev origin, so step 9 waits for it.
7. Remove the workers.dev `/auth/callback` redirect URI from the Discord developer portal, keeping the local one; sign-in stops using it once `APP_ORIGIN` changes.
8. Members sign in again on the new origin, because session cookies belong to the origin that set them, and finish or cancel any unfinished upload on workers.dev first, because Resume reads part ETags from that origin's `localStorage`. Links to the workers.dev origin, such as those in earlier Discord posts, stop working once step 9 disables that route; until then they open only for a member still signed in there, and a sign-in started there fails, because its callback lands on the new origin.
9. Once no Waffle is `processing`, so no transcode run still reports to the workers.dev origin, and no member has an unfinished upload there, disable the Worker's workers.dev route and remove the workers.dev origin from the `waffles` bucket's CORS rule.
10. `.app` domains only load over HTTPS; Cloudflare provides the certificate, so nothing else is needed.

## Build plan

Build in seven phases, Phase 0 to Phase 6. Each phase's goal, scope and exit criteria live in [ROADMAP § Phases](../plans/ROADMAP.md#phases).

**Stack:** see [`CLAUDE.md` § Stack](../CLAUDE.md#stack).

Start the Claude Code session by giving it this doc and `CLAUDE.md`, which already records the stack and the Waffle-week rule, and ask it to fill in CLAUDE.md's folder layout and commands (`dev`, `test`, `deploy`, `db:migrate`) as Phase 0 creates them.

**Launch:** point the webhook at the real channel, use it for two Wednesdays alongside WhatsApp, then retire the WhatsApp group. Domain cutover can follow at any point after.

**Tests worth writing.** Each area below tests every outcome its rules state; the list names those rules and adds only the cases they do not spell out (boundaries, examples and races).
- The Waffle-week function: the cases in the Waffle week rule in [CLAUDE.md § Project rules](../CLAUDE.md#project-rules).
- The callback (rules: **Callback** and **Failure output** in [Transcode](#transcode)): a table-driven test runs each result, `ready` and `failed`, against each current status, `uploading`, `processing`, `ready`, `failed` and no row, and checks the response and the row; each 400 and 401 case changes nothing, a tampered body included, and writes only its `callback rejected: 400` or `callback rejected: 401` line, with no Waffle ID; the no-row case returns 404 with exactly `{"error":"no_waffle"}`; an error text over 300 characters is stored truncated; a Retry after a `ready` that recovered a `failed` Waffle gets 409.
- The job's callback sender, with fake timers (rules: the retry, cleanup and log rules after **Callback**): a 503, a 429 or a network error and then a 200 sends the second attempt with a new timestamp and signature, which the Worker accepts; four such failures in a row fail the run with `callback: retries exhausted <last status code or network> <waffle id>`; a 401, 400 or 404 is not retried and fails the run with `callback: <status code> <waffle id>`; a 404 with the body `{"error":"no_waffle"}` deletes the playback copy and poster, and a 404 with any other body deletes nothing; every failure, a failed cleanup delete after a `no_waffle` 404 included, logs only its fixed line naming the Waffle ID, never a key, bucket or URL.
- The owner-only rules (the paragraph after **Abort or Delete** in [API routes](#api-routes-all-under-the-app-origin)).
- The stuck-job check (rules: **Stuck jobs**): 149 vs 150 minutes after `processing_at`, and a Retry after it, after which the next read still shows `processing`.
- A rejected dispatch, from `complete` and from Retry (rules: **Rejected dispatch**): a 401 stores `401:` and the token message, a 500 or a network error stores the generic message, and neither stores anything from GitHub's response.
- The dispatch target (rules: **Dispatch target**): with `TRANSCODE_ENV` set to `prod` or `dev`, `complete` and Retry send its path, `ref` and inputs; with it missing, empty or any other value, such as `Prod`, no route dispatches, makes an R2 call or presigns a URL; with `dev` every R2 call goes to `waffles-dev`.
- The Discord post (rules: [Discord notification](#discord-notification)): two concurrent callbacks post once; a claim 119 s old blocks a second post and one 121 s old is taken over; a `Retry-After` of 11 s or a third 429 is a failed post; `discord_error` never holds the webhook URL or response body; each 409 of `POST /api/waffles/:id/discord`.
- Delete and abort (rules: **Delete and abort order** and **Abort or Delete**): a failure at each R2 call, then a repeat that finishes; an abort racing `complete`, which ends with either no row and no original or a `processing` Waffle, and the loser gets 409; a double-tap `complete`, which dispatches once and keeps the original.
- Failed and retried uploads (rules: **Failed uploads** in [Upload](#upload) and **Abort or Delete**): each screen and action in **Failed uploads**, including a 409 `cancelled` whose re-reads show `uploading` and then `processing`; and each `complete` path in **Abort or Delete**: a retried `complete` after one that finished the multipart upload but not its `UPDATE`; a `NoSuchUpload` with and without the original, including while a sibling `complete` is still finishing the multipart upload; an `UPDATE` that loses after the original was found by HEAD; a finished original larger or smaller than `original_bytes`, found directly or by HEAD, whose `DELETE` wins or finds the row gone; an abort racing a retried `complete`; and each failing cleanup delete, which logs and keeps its response.
- The abandoned-upload sweep (rules: **Abandoned uploads** in [Upload](#upload)): an `uploading` row just over 7 days old is removed and one just under is kept; a sweep whose `DELETE` loses to `complete` leaves the `processing` Waffle's original in place.
- `uploading` Waffles (rules: **Unfinished uploads** in [UI](#ui) and the [API routes](#api-routes-all-under-the-app-origin) table): left out of every listing, avatar and count those rules name, shown under Unfinished uploads only to their owner, and 404 from `GET /api/waffles/:id` for another member.
- The upload checks (rules: **Upload checks** in [Upload](#upload)): `clip.MOV` and `x.exe.mp4` are accepted, and `x.mp4.exe`, `clip` and `clip.` are not; a size of 0 or a non-integer is rejected; exactly 5 GiB is accepted with 103 parts, and 5 GiB + 1 byte is rejected; a part number of 0, above the part count or repeated, or a `complete` list out of order, is rejected with no R2 call; every rejection there, in **Failed uploads** and in **Abort or Delete** carries its `code`, and the browser acts on the code, not the message; a part retry, or a Resume after the part URLs have expired, gets fresh URLs from `/parts` and finishes the upload.
- The OAuth `state` (rules: [Auth](#auth)): a callback without the cookie, with an expired cookie or with a different `state` creates no session.
- Removed members (rules: [Auth](#auth)): each effect of setting `active` to 0, including that setting it back to 1 does not let the old cookie back in and that signing in again shows "not a member" while `active` is 0.
- The `Content-Disposition` name (rules: the paragraph after **Abort or Delete**): a display name holding CR, LF, quotes or slashes, a non-ASCII name, and a name left empty.
- The heatmap query, and the heatmap year (rules: the `/api/heatmap` row of [API routes](#api-routes-all-under-the-app-origin)): on a day in early January before the year's first Wednesday, the current heatmap year is still the previous one, so the new calendar year gets 400 `bad_year`.
- The token expiry check, beyond Phase 3's exit criteria (rules: **Unexpected errors** in [Transcode](#transcode)): a run that throws before any post writes only `token check: error`, posts nothing and returns without rethrowing.
- The CI check and the validation script's tests from **Untrusted inputs**, and the hand-dispatch checks in Phase 3's exit criteria in [ROADMAP § Phases](../plans/ROADMAP.md#phases).

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
