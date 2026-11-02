# Push Review: docs/spec

## Review 1
Generated: 2026-10-26 11:03
Comparison: origin/docs/spec b0e089f0a6fa27f629a846f7c4fc11df2bfe1442..eb7b1bd767e5bd610b1762e16a99dc7592ce0a2d
Verdict: **BLOCKED**

### Results by Reviewer

#### 1. Safety & Security — PASS
The fine-grained PAT choice (one repo, Actions: write only, and the reasoning for avoiding Contents: write) is sound. Two minor hardening notes:
- (minor) `docs/SPEC.md:71`: the Waffle ID arrives as a `workflow_dispatch` input, but the spec doesn't say the workflow treats it as untrusted. Interpolating `${{ inputs.* }}` inline in a `run:` step allows script injection in a public repo. The dispatch `ref` is also unspecified.
- (minor) `docs/SPEC.md:71`: the raw GitHub error text from a rejected dispatch is stored on the Waffle and shown to members, so it may expose internals.

#### 2. Correctness — PASS
Facts, cross-references (`docs/SPEC.md#ui` resolves), the streak rule, week boundaries, and the `workflow_dispatch` and PWA claims are correct. One minor overstatement:
- (minor) `plans/ROADMAP.md:162`: changing `.github/workflows` files also needs the separate Workflows permission, so "a leaked token could rewrite the workflow" overstates the risk. Reword it as "could change code the workflow runs and expose its secrets".

#### 3. Simplicity & Conciseness — PASS
Nothing is over-engineered. A few facts are duplicated:
- (minor) `plans/ROADMAP.md`: Phase 2 repeats the PAT scope and the failure behaviour that SPEC Transcode now holds.
- (minor) `plans/ROADMAP.md`: Phase 3 repeats the PWA scope that SPEC UI holds.
- (minor) `docs/SPEC.md`: the streak sentence states its rule twice.
- (minor) `plans/ROADMAP.md`: the npm decision is also in CLAUDE.md Stack. This is acceptable as a decision record.

#### 4. Test Coverage — PASS
None of the new rules has a regression check yet; all gaps are minor:
- (minor) `plans/ROADMAP.md:53`: the capability 5 acceptance example covers only the case where the streak is kept. It doesn't cover the break case or the Tue 23:59 / Wed 00:00 ET boundary.
- (minor) The rejected dispatch → `failed` rule has no exit criterion in Phase 2.
- (minor) Retry re-dispatch, the recovery path named for GitHub outages, has no exit criterion in Phase 3.
- (minor) PWA installability has no check: install it on iOS and Android and confirm no service worker is registered.

#### 5. Completeness & Cleanup — PASS
All decision markers and the accepted-risk placeholder are resolved, and no stale `repository_dispatch` references remain.
- (minor) `docs/SPEC.md:261`: the secrets list names `GITHUB_DISPATCH_TOKEN` without saying it is the PAT, that it expires, or that it needs rotating.

#### 6. Consistency & Style — PASS
Wrapping, the bold-claim / *Why:* / *Trigger:* format, backticked identifiers and status names are consistent.
- (minor) `plans/ROADMAP.md:53`: capability 5's acceptance example now includes the streak case, but Phase 6's exit criteria (lines ~203-204) don't.
- (minor) The PAT and PWA scopes are stated in both ROADMAP and SPEC. This is a light overlap, not a contradiction.
- (minor) `plans/ROADMAP.md:105`: "a Waffle still reaches `ready` with only its channel post failing" has no home in SPEC.

#### 7. Integration Risk — FAIL
- (major) `plans/ROADMAP.md:105`: the GitHub-outage claim says Waffles wait in `processing` and "Retry re-dispatches them". SPEC allows Retry only for `failed`, and nothing moves a Waffle out of `processing` when a dispatch is accepted but its run never calls back. The claim has no backing in the spec.
- (major) `docs/SPEC.md:71`: `workflow_dispatch` needs a repo, a workflow file and a `ref`, and none of them is in the Worker config. The spec also never says how the job learns the per-environment bucket (`waffles` or `waffles-dev`) or the callback URL (workers.dev or the custom domain after cutover). A wrong ref or callback URL would strand Waffles in `processing`.
- (minor) `plans/ROADMAP.md:164`: the PAT expires, but no expiry length, rotation owner or reminder is documented.
- (minor) `plans/ROADMAP.md:103`: SPEC never says a failed Discord webhook is non-fatal.
- (minor) `docs/SPEC.md:276`: the Build plan's Stack line doesn't mention npm or `workflow_dispatch`, so it no longer matches CLAUDE.md.

#### 8. Error Handling & Silent Failures — FAIL
- (critical) `docs/SPEC.md:71`: a dispatch that GitHub accepts but whose run never calls back leaves the Waffle in `processing` forever. Possible causes are a disabled workflow, a stuck runner, a killed job, an HMAC rejection, or the Worker being down at callback time. The spec has no timeout, no sweep and no Retry for this case.
- (critical) `docs/SPEC.md:80/198`: what happens when the Discord webhook fails after `ready` is unspecified. That covers a non-2xx response, a 429 and a revoked webhook URL. The spec can't tell "post failed" from "not yet posted", nothing surfaces the failure, and a duplicate callback could post twice.
- (major) `docs/SPEC.md:71`: the stored error for a rejected dispatch isn't actionable. It should map 401/403/404 to a fixed message: the token may have expired, the upload is safe, tell the admin.
- (major) `docs/SPEC.md:261`: nothing detects PAT expiry before it causes a member's upload to fail.
- (minor) `docs/SPEC.md:80`: ffmpeg/ffprobe error text can leak filenames and metadata into the public Actions logs and the callback.
- (minor) `docs/SPEC.md:192`: the callback's semantics are unstated: unknown ID, a duplicate or late callback, a bad HMAC, and whether the job fails visibly.

### To-Do: Required Changes

- [x] **Add a stuck-`processing` rule** (decided: lazy on read, 120/150 min; dd-261026-047) — `docs/SPEC.md` (Transcode, Retry/permissions, UI), `plans/ROADMAP.md:105` — Set `timeout-minutes` on the workflow job. A Waffle in `processing` longer than N minutes becomes `failed` with "transcode did not report back", applied either by a Cron Trigger or lazily when the Waffle is read, so Retry covers it. Then reword the ROADMAP's GitHub-outage sentence to match.
- [x] **Specify Discord webhook failure handling** (atomic `discord_posting_at` claim, 10 s Retry-After cap, orphan message accepted) — `docs/SPEC.md` (Discord notification, Transcode step 6, Delete), `plans/ROADMAP.md:103` — A failed post never changes `ready`. It is recorded on the Waffle and shown to the owner with a "Post to Discord again" action. On a 429, honour Retry-After. Post only when `discord_message_id IS NULL`, so a duplicate callback can't double-post. State how Delete behaves when there is no message ID.
- [x] **Make the rejected-dispatch error actionable and sanitized** (`<code or network>: <fixed message>`, guarded update, admin defined) — `docs/SPEC.md:71` — Store the status code plus a fixed message, not raw response bodies. For 401/403/404 the message says the GitHub token may have expired, the upload is safe, and to tell the admin and retry once it is renewed.
- [x] **Add PAT expiry detection** (decided: daily Worker Cron Trigger alerting via the channel webhook; dd-261026-049) — `docs/SPEC.md` (secrets/config, ~line 261) — Describe `GITHUB_DISPATCH_TOKEN` as the fine-grained PAT (Actions: write, this repo only) with its expiry date. Add a scheduled check that alerts when the token is invalid or within about 14 days of expiry, and a rotation step for the README or runbook.
- [x] **Define the dispatch target and per-environment inputs** (decided: one `env` input mapped to the bucket, callback base URL from the `CALLBACK_BASE_URL` repository variable; dd-261026-050; later stored as a repository secret so logs mask it) — `docs/SPEC.md` (Transcode, config, environments/domain cutover) — Add non-secret Worker vars: `GITHUB_REPO`, the workflow file name, and dispatch `ref` = `main`. Say how the job gets the bucket and the callback base URL (inputs, or derived values). Note that local dev can't receive callbacks, and add the callback URL to the domain-cutover checklist.
- [x] **Harden workflow input handling** (covers `waffle_id`, `env` and every `${{ secrets.* }}`; ULID check with a fixed failure message and no callback) — `docs/SPEC.md` (Transcode) — The workflow validates the Waffle ID against a strict pattern and passes it through `env:`, never inline `${{ inputs.* }}` in `run:`.
- [x] **Specify callback semantics** (decided: a `ready` may replace `failed`, a `failed` applies only while `processing`; dd-261026-052) — `docs/SPEC.md` (~line 192) — Cover the responses for an unknown ID, an already-final Waffle (idempotent handling) and a bad HMAC. The job retries the callback a few times and then fails the run visibly.
- [x] **Keep raw tool errors out of public logs** (`<category>: <detail>`, fixed detail for storage steps, first stderr line for ffprobe/ffmpeg, 300-char cap on both sides) — `docs/SPEC.md` (Transcode) — Log only the Waffle ID and an error category, use `-loglevel error` with fixed local filenames, and post a sanitized, truncated error to the callback.
- [x] **Correct the `repository_dispatch` rationale** — `plans/ROADMAP.md:162` — Replace "rewrite the workflow" with "change code the workflow runs and expose its secrets", since workflow files need the separate Workflows permission.
- [x] **Extend acceptance and exit criteria for the new rules** (decided: rejected-dispatch and Retry criteria in Phase 3, which now builds the retry route; "or logged" added to Rejected dispatch) — `plans/ROADMAP.md` (capability 5, Phases 2, 3 and 6) — Add the streak break case and the Wed 00:00 ET boundary to capability 5, and mirror the streak case in Phase 6's exit criteria. In Phase 2, an invalid token ends the Waffle `failed` with visible text that never contains the token. In Phase 3, the owner's Retry reaches `ready` and another member's Retry is refused. Also check that the PWA installs on iOS and Android with no service worker registered.
- [x] **Remove duplicated facts** — `plans/ROADMAP.md` (Phases 2 and 3), `docs/SPEC.md` (streak sentence) — Keep each *Why:* in ROADMAP and link to `docs/SPEC.md#transcode` and `#ui` for the PAT scope, the failure behaviour and the PWA scope. Shorten the streak sentence so it states the rule once.
- [x] **Sync the SPEC Build plan Stack line** (pointed at CLAUDE.md § Stack; the next sentence now says CLAUDE.md already records the stack) — `docs/SPEC.md:324` — Add npm and `workflow_dispatch`, or point the line at CLAUDE.md's Stack as the home for those facts.

## Review 2
Generated: 2026-10-26 17:12
Comparison: origin/docs/spec b0e089f0a6fa27f629a846f7c4fc11df2bfe1442..70a9e411f1997c701701c4114c9d8386f4eea0d9
Verdict: **BLOCKED**

### Results by Reviewer

#### 1. Safety & Security — PASS
Review 1's security items hold: inputs and secrets go through `env:`, the ULID and env checks are in place, errors are sanitized, the PAT is narrowly scoped, and the HMAC uses a timestamp and a constant-time compare. Remaining gaps are minor:
- (minor) `docs/SPEC.md:62`: the upload boundary isn't specified. The spec doesn't give an `ext` allowlist, a size or part-count cap, or an owner and `uploading` check on `/parts` and `/complete`, and it doesn't say to sanitize the caption and name in `Content-Disposition`.
- (minor) `docs/SPEC.md:212`: the OAuth `state` isn't bound to the browser, so login CSRF is possible. A removed member also keeps a 30-day sliding session.
- (minor) `docs/SPEC.md:308`: the R2 token's scope isn't stated, and `TRANSCODE_CALLBACK_SECRET` and the R2 keys have no rotation steps.
- (minor) `docs/SPEC.md:93`: a signed callback can be replayed within 5 minutes. A replayed `failed` could fail a Waffle that was retried in that window.

#### 2. Correctness — PASS
Review 1's resolutions are correct and consistent across SPEC, ROADMAP, META-PLAN, README and CLAUDE.md. Remaining gaps are minor:
- (minor) `docs/SPEC.md:82`: the remux-only test doesn't check H.264 profile or `pix_fmt`, so a 10-bit H.264 source would be remuxed and wouldn't play on most phones.
- (minor) `docs/SPEC.md:278`: "current year" is ambiguous before the first Wednesday of January, when the current week belongs to the previous year's heatmap.
- (minor) `docs/SPEC.md:110`: `adapter-cloudflare` emits a Worker with a fetch handler only, so the Cron `scheduled` handler has no stated home.
- (minor) `docs/SPEC.md:101`: a stale run's `failed` can land while a Retry's run is still in progress, which invites a redundant second Retry.

#### 3. Simplicity & Conciseness — FAIL
- (major) `docs/SPEC.md:330`: the Build plan restates each ROADMAP phase goal and exit criterion, and capabilities 1, 2 and 5 restate them a third time. The copies have already drifted: the EST/EDT week, the token-check criterion and `getRegistrations()` appear only in SPEC.
- (minor) `CLAUDE.md:48`: the Project rules are also stated in SPEC and in ROADMAP Principles. The logging sentence at SPEC:133 is a further copy.
- (minor) `docs/SPEC.md:73`: the workflow and the Worker both validate the env value. Pick one place, and shorten the rationale prose.
- (minor) `docs/SPEC.md:88`: the failure-output scheme is elaborate. It has category-specific fixed details and truncation on both sides.
- (minor) `docs/SPEC.md:101`: the Racing runs and Stuck jobs prose is long, and Tests worth writing (SPEC:351) re-lists the callback cases.
- (minor) `docs/SPEC.md:110`: the token-expiry Cron is a whole subsystem for a once-a-year renewal. A calendar reminder may be enough.
- (minor) `docs/SPEC.md:239`: the Discord claim lock (`discord_posting_at` with a 2-minute expiry) may be more than this needs, since the guarded ready UPDATE already posts once.
- (minor) `plans/ROADMAP.md:102`: External dependencies and the Phase 3/4 bullets re-explain behaviour that SPEC defines.
- (minor) `docs/SPEC.md:42`: there's a stale HTML comment about the diagram, and the Overview goals and non-goals repeat ROADMAP.

#### 4. Test Coverage — FAIL
- (major) `docs/SPEC.md:75`: no exit criterion or test covers the workflow's untrusted-input rules. These are the ULID and env rejection, the fixed message that never echoes the rejected value, sending no callback, keeping `${{ inputs.* }}` and `${{ secrets.* }}` out of `run:`, and not using `set -x`. A script-injection regression would pass every check.
- (minor) `docs/SPEC.md:73`: nothing tests the Worker's refusal when `TRANSCODE_ENV` is invalid, or the dispatch's path, ref and inputs.
- (minor) `docs/SPEC.md:73`: nothing checks that a dev run sends no callback, or that a prod run with an empty or `http://` `CALLBACK_BASE_URL` fails before it downloads anything.
- (minor) `docs/SPEC.md:114`: the token-check edges are untested. They are the 14- vs 15-day boundary, an unparseable header, 403 and 404, a far-future expiry, and a failed alert post.
- (minor) `docs/SPEC.md:239`: the Discord claim expiry (119 s vs 121 s), clearing the claim after a post, and the `/discord` 409 cases are untested.
- (minor) `plans/ROADMAP.md:207`: the revoked-webhook criterion never runs the "Post to Discord again" recovery, and deleting a Waffle that was never posted isn't in any criterion.
- (minor) `docs/SPEC.md:103`: nothing tests that Retry resets `processing_at`, or that both GET routes apply the stuck check.
- (minor) `docs/SPEC.md:99`: no drill covers the job-side callback behaviour, either cleanup on a 404 or a fresh timestamp on each retry.
- (minor) `docs/SPEC.md:105`: no check covers the job-side `<category>: <detail>` format, raw tool output staying out of the logs, or the no-secret-in-logs rule on the token-check and Discord paths.
- (minor) `docs/SPEC.md:279`: the cross-year streak cases are untested. They are a past year's best that includes carried-in weeks, a run capped at the selected year, and a 53-week year.
- (minor) `docs/SPEC.md:97`: the callback tests don't cover the full status × result table (for example failed→failed and ready→ready).

#### 5. Completeness & Cleanup — PASS
Every ticked Review 1 To-Do is in the docs as its note describes, and all anchors resolve.
- (minor) `docs/SPEC.md:21`: the text says the follow-on is "in the last section", but the last section is now Resolved questions.
- (minor) `docs/SPEC.md:41`: a leftover HTML comment about the import remains.
- (minor) `plans/META-PLAN.md:35`: Verification says `npm test`, but no test runner exists yet. Per the file's own convention it should say "pending".
- (minor) `README.md:11`: Getting started is still a placeholder (acceptable until Phase 0).

#### 6. Consistency & Style — PASS
Names of secrets, statuses, routes and columns agree across the documents.
- (minor) `docs/SPEC.md:21`: the same "last section" reference as in reviewer 5.
- (minor) `plans/ROADMAP.md:168`: the daily check has three different names.
- (minor) `plans/ROADMAP.md:175`: ROADMAP's Phase 3 exit criteria leave out the token-check criterion that SPEC's Phase 3 has.
- (minor) `plans/ROADMAP.md:219`: the streak and week-boundary criterion has three homes.
- (minor) `plans/ROADMAP.md:167`: "needs only Actions: write" is repeated after the link to SPEC.
- (minor) `README.md:20`: CLAUDE.md's table entry for README doesn't mention the new Operations runbook.

#### 7. Integration Risk — FAIL
- (major) `docs/SPEC.md:312`: no var names the Worker's own R2 bucket (`waffles` or `waffles-dev`) or the app's public origin. The bucket is needed for presigning, and the origin for the OAuth `redirect_uri`, the Discord `/w/{id}` link, CORS and the domain cutover. Local dev therefore has no stated way to target `waffles-dev`.
- (major) `plans/ROADMAP.md:151`: Phase 3's exit criteria (playing clips, seeing status after a rejected dispatch or a Retry, the stuck check) need `GET /api/waffles` and `/api/waffles/:id` or a player, and only Phase 4 builds those.
- (minor) `docs/SPEC.md:114`: the Cron handler needs a custom Worker entry around the SvelteKit adapter.
- (minor) `docs/SPEC.md:77`: with the dispatch ref fixed to `main`, every change to the workflow during Phase 3 tuning needs a merge first.
- (minor) `docs/SPEC.md:322`: cutover step 6 duplicates the Launch step, and the cutover doesn't mention the old workers.dev links or that members must sign in again.
- (minor) `docs/SPEC.md:66`: the multipart `uploadId` has no column, and stale `uploading` rows are never swept.
- (minor) `docs/SPEC.md:77`: the spec doesn't say whether the R2 secrets cover both buckets, or list what goes in `.dev.vars` for local dev.

#### 8. Error Handling & Silent Failures — FAIL
Review 1's resolutions hold, and the 429 and 404-message-delete paths are handled well. New gaps:
- (major) `docs/SPEC.md:114`: the token check can stop working without anyone noticing. A 5xx, a network error, an unparseable header or a stopped cron logs nothing and is retried the next day, and a failed alert post leaves no trace.
- (major) `docs/SPEC.md:99`: systemic callback failures are visible only in the public Actions tab. These are a secret mismatch, an empty `CALLBACK_BASE_URL` and exhausted retries. The Waffle ends with "transcode did not report back", which names no cause or next step, and Retry fails the same way.
- (major) `docs/SPEC.md:225`: Delete and abort don't specify the order of the R2 and D1 deletes or what happens when an R2 delete fails. Videos can be orphaned without anyone knowing, or a row can survive with its files gone.
- (major) `docs/SPEC.md:63`: the spec doesn't say what the member sees, or what state the row is left in, when part retries are exhausted or `complete` fails. Rows abandoned in `uploading` are never swept.
- (minor) `docs/SPEC.md:99`: a failed cleanup after a 404 isn't logged.
- (minor) `docs/SPEC.md:97`: a real `failed` cause that arrives after the stuck rule has already fired is thrown away.

### To-Do: Required Changes

- [x] **Make ROADMAP the single home of phase goals and exit criteria** — `docs/SPEC.md` (Build plan, ~line 330), `plans/ROADMAP.md` (capabilities 1, 2 and 5; Phase 3 exit criteria; Phase 6 exit criteria at ~line 219) — Reduce SPEC's Build plan to a pointer to ROADMAP Phases plus the Launch sentence. First move the criteria that only SPEC has into ROADMAP: the EST and EDT week, the token-check outcome, the `getRegistrations()` check and "posting in it adds one". Point the capability acceptance examples at their owning phase's exit criteria instead of restating them.
- [x] **Add Worker vars for its R2 bucket and the app origin** — `docs/SPEC.md` (Vars/secrets ~line 312, domain cutover ~line 322) — Add, for example, `R2_BUCKET` and `APP_ORIGIN` and say `.dev.vars` overrides them locally (`waffles-dev`, `http://localhost:5173`). Add `APP_ORIGIN` to the cutover checklist. Note the localhost CORS origin on `waffles-dev` and the localhost Discord redirect URI. Say whether the R2 secrets cover both buckets, and list what goes in `.dev.vars` for local dev.
- [x] **Fix the Phase 3 dependency on Phase 4 read routes** — `plans/ROADMAP.md` (~line 151, Phases 3 and 4) — Either move `GET /api/waffles` and `GET /api/waffles/:id` (with the lazy stuck-job check) into Phase 3, or state that Phase 3 is verified with a presigned URL and a D1 query, leaving the routes in Phase 4. (assumed: dd-policy, dd-261026-066)
- [x] **Add an exit criterion for the workflow's untrusted-input rules** — `plans/ROADMAP.md` (Phase 3 exit criteria), `docs/SPEC.md` (Tests worth writing) — Hand-dispatch with a non-ULID `waffle_id` containing shell metacharacters, and with `env=other`. Assert the run fails before downloading, the log holds only the fixed message (never the value), and no callback arrives. Add a CI check that fails on `${{ inputs.` or `${{ secrets.` inside `run:` or `script:`.
- [x] **Make the token check fail loudly** — `docs/SPEC.md` (Token expiry check ~line 110-115) — Each outcome that doesn't alert (status code, `network`, `unparseable`) writes a fixed, body-free line to the Worker log. Persist the consecutive-failure count and the last alert result, and after 3 days surface it where the admin will see it. Add exit criteria for an unparseable header and a failed alert post. (decided: dd-261026-069)
- [x] **Make the stuck-job error actionable** — `docs/SPEC.md` (Stuck jobs ~line 103), `README.md` (Operations) — Reword the error to say the transcode did not report back, the upload is safe, press Retry, and if it fails again tell the admin to check the workflow run. Add a README step for the admin to turn on GitHub's failed-workflow notifications.
- [x] **Specify the delete order and R2 failure handling for Delete and abort** — `docs/SPEC.md` (DELETE `/api/waffles/:id` ~line 225, `POST /api/uploads/:id/abort`) — Delete the R2 objects first and treat 404 as success. Delete the row only when every R2 delete succeeds; otherwise return "Could not delete the video files; try again, or tell the admin" and keep the row. Add a test with a failing R2 delete.
- [x] **Specify the outcomes of a failed upload, and sweep stale `uploading` rows** — `docs/SPEC.md` (Upload ~line 63-66, data model) — State the member-facing message and the recovery (resume or abort) when part retries are exhausted and when `complete` fails. Store `upload_id` on the row, or say the Worker uses the client's ID only together with that Waffle's key. Say whether `uploading` rows appear in listings and the heatmap, and add a lazy sweep for rows older than 7 days to match the lifecycle rule. (decided: dd-261027-010..014)
- [x] **Validate the upload boundary and harden sessions** — `docs/SPEC.md` (~lines 62, 139, 212, 233) — Take `ext` from an allowlist (mov, mp4, m4v), cap the size and part count, and require the uploader and `status = uploading` on `/parts` and `/complete`. Strip CR, LF, quotes and path characters from the `Content-Disposition` name. Bind the OAuth `state` to a short-lived HttpOnly cookie. Either re-check `members` on each request or delete a removed member's sessions. (decided: dd-261027-022, dd-261027-023)
- [x] **State the R2 token scope and add secret rotation steps** — `docs/SPEC.md` (~line 308), `README.md` (Operations) — Limit the R2 token to Object Read & Write on the two buckets. Add rotation steps for `TRANSCODE_CALLBACK_SECRET` and the R2 keys.
- [x] **Decide on callback replay and stale-run results** — `docs/SPEC.md` (~lines 93, 97, 101) — Either include a per-dispatch nonce or `processing_at` in the signed body and have the Worker match it, or document replay and superseded-run results as an accepted limitation. Let a `failed` callback replace the error text while the status is already `failed`, so the real cause replaces "did not report back". (decided: dd-261027-031)
- [x] **Tighten the remux condition** — `docs/SPEC.md` (~line 82) — Remux only an 8-bit `yuv420p` H.264 source in Constrained Baseline, Main or High profile with AAC audio; re-encode everything else.
- [x] **Define "current year" for streaks** — `docs/SPEC.md` (~line 278, UI streak bullets) — Define it as the year whose heatmap holds the current Waffle week, meaning the year of that week's Wednesday.
- [x] **Give the Cron handler a home** (Phase 3; **Worker entry** in SPEC Architecture, `[build] command`, `--test-scheduled` exit criterion) — `docs/SPEC.md` (~line 110), `plans/ROADMAP.md` (Phase 0 or 3) — Say the Worker entry wraps the SvelteKit adapter output so it exports both `fetch` and `scheduled`, and add a build step for it.
- [x] **Make the dispatch ref configurable or document merge-to-iterate** — `docs/SPEC.md` (~line 77) — Add a `GITHUB_REF` var defaulting to `main`, or note that Phase 3 tuning merges each workflow change to `main` first. (decided: dd-261027-038)
- [x] **Fix the domain-cutover steps** — `docs/SPEC.md` (~line 322) — Drop step 7 ("Point the webhook at the real Waffles channel") or mark it "if not already done at launch". Note that old workers.dev links work only while that route stays enabled, and that members sign in again on the new origin. (webhook step dropped as Launch already does it; added removing the workers.dev redirect URI and signing in again)
- [x] **Log a failed cleanup after a 404** (fixed line `cleanup failed <waffle id>`, matching `original delete failed <waffle id>`) — `docs/SPEC.md` (~line 99) — Cleanup errors are non-fatal, but the job prints a fixed line `cleanup failed for <Waffle ID>`.
- [x] **Fill the remaining test and exit-criterion gaps** (both GET routes' stuck check, the unparseable header, the failed alert post, the 404 cleanup and deleting a never-posted Waffle were already covered; added the no-audio and `yuvj420p` remux clips from item 12) — `docs/SPEC.md` (Tests worth writing), `plans/ROADMAP.md` (Phases 3 and 5) — Add:
  - an invalid `TRANSCODE_ENV` dispatches nothing, and the dispatch carries the right path, ref and inputs;
  - a dev run sends no callback, and a prod run with an empty or `http://` `CALLBACK_BASE_URL` fails before downloading;
  - the token-check edges: 14 vs 15 days, an unparseable header, 403 and 404, a far-future expiry, and a failed alert post;
  - the Discord claim at 119 s vs 121 s, the claim cleared after a post, and the `/discord` 409 cases;
  - the revoked-webhook recovery: restore the webhook, re-post, see one message and `discord_error` cleared; and deleting a never-posted Waffle;
  - Retry resets `processing_at`, and both GET routes apply the stuck check;
  - the job-side callback behaviour: a 404 removes the playback copy and poster, and a 503 followed by a 200 gets a fresh timestamp;
  - a corrupt file leaves only the `<category>: <detail>` line in the logs and the stored error; no secret appears in the token-check or Discord logs;
  - the cross-year streak seeds: a December–February run and a 53-week year;
  - a table-driven test over current status × callback result.
- [x] **Trim the duplicated and over-long prose** — `CLAUDE.md:48`, `docs/SPEC.md` (~lines 42, 73, 88, 101, 133, 351, Overview), `plans/ROADMAP.md` (~lines 102, 167) — Make each Project rule's other copies links to CLAUDE.md or SPEC. Delete the stale diagram comment. Link the SPEC Overview's goals and non-goals to ROADMAP and keep the Decisions table. Validate the env value in one place, not both. Shorten Racing runs, and point Tests worth writing at the rules instead of re-listing them. Cut ROADMAP's External dependencies down to links, and drop the repeated "Actions: write". Decide whether the token-expiry Cron and the Discord claim lock are worth their cost against a calendar reminder and the once-only guarded ready UPDATE. *(2026-10-27: everything but this last sentence applied; the env value stays checked in both places on purpose, since the Worker guards its own R2 calls and the workflow guards hand dispatches.)* (decided: dd-261027-059 — keep both)
- [x] **Fix the small reference and naming nits** (the NAS reference was already gone; log-line literals, the fixed three-day message and `token_check` keep their names) — `docs/SPEC.md:21`, `plans/ROADMAP.md:168`, `plans/META-PLAN.md:35`, `CLAUDE.md` (Where things live table) — Say "in the Follow-on: NAS backup section". Use "token expiry check" everywhere. Set Verification to "pending — set by the first feature with tests". Add "and operate" to the README row.

6b: 0 misses (every line Review 1's diff touched in SPEC and ROADMAP has changed since eb7b1bd, so no Review 2 finding sits on unchanged Review 1 lines)

## Review 3
Generated: 2026-10-28 21:40
Comparison: origin/docs/spec b0e089f0a6fa27f629a846f7c4fc11df2bfe1442..dd8f2d62f4b57974e0f68b54646b6f3ccbd6587a
Verdict: **BLOCKED**

Mode: full review (first non-docs path: `docs/SPEC.md`); Type Design not launched (no code type definitions).

### Results by Reviewer

#### 1. Safety & Security — PASS
The HMAC callback, presigned-URL scoping, workflow-injection controls and auth all hold up. One minor gap:
- (minor) `docs/SPEC.md:261`: the Discord Waffle post carries the member's caption without `allowed_mentions: {parse: []}`, so an `@everyone` in a caption pings the whole server.

#### 2. Correctness — PASS
No contradictions or API misuse. Cross-references resolve, and the part counts, week and streak rules, dispatch and ffmpeg usage are correct.
- (minor) `docs/SPEC.md:104`: the stuck-jobs prose says "150 minutes after", but the statement uses a strict `<` and the test checks 149 vs 150, so a Waffle at exactly 150 minutes stays `processing`.
- (minor) `plans/ROADMAP.md:44`: capability 5 says "52 weeks", but a heatmap year has 52 or 53.
- (minor) `docs/SPEC.md:286`: the banner says "failed for 3 days" even on day 5, because it shows whenever `consecutive_failures >= 3`.

#### 3. Simplicity & Conciseness — FAIL
- (major) `docs/SPEC.md:117`: the token expiry check carries a second monitoring layer, made up of `token_check`, the failure counter, the three-day post, the catch-upsert accounting and the This-week banner, plus their tests. That is a lot for four users.
- (major) `docs/SPEC.md:254`: the "Abort or Delete" paragraph runs about 900 words and lists every interleaving of complete, abort, the sweep and sibling completes. "Failed uploads" (lines 54-62) mirrors it.
- (major) `docs/SPEC.md:74`: the workflow-lint CI check, its required status check or ruleset, and the committed validation script with its own tests are heavy hardening for a ULID input and a two-value choice input.
- (minor) `docs/SPEC.md:98`: the callback retry ladder and the Racing runs accepted limitations restate each other.
- (minor) `docs/SPEC.md:262`: the extras around the Discord claim lock (the Retry-After 10 s threshold and the 119/121 s test) could go, while the lock itself stays.
- (minor) `docs/SPEC.md:39`: the Worker entry paragraph hand-describes Wrangler config keys that will drift.
- (minor) `docs/SPEC.md:375`: Tests worth writing re-enumerates rules and exit criteria, against CLAUDE.md's rule that each fact has one home.
- (minor) `docs/SPEC.md:344`: the Environments section repeats the rotation steps and the R2-key trade-off, and has a long paragraph on `.dev.vars`.

#### 4. Test Coverage — PASS
Most specified behaviours have matching tests or exit criteria. All gaps are minor:
- (minor) `plans/ROADMAP.md:174`: no criterion checks that the deployed Cron is registered, or runs the check once against a real PAT's expiry header.
- (minor) `README.md:20`: none of the rotation runbooks or the stuck-transcode runbook has a drill.
- (minor) `docs/SPEC.md:352`: the domain cutover has no verification.
- (minor) `docs/SPEC.md:74`: the CI check's patterns have no must-fail or must-pass fixture workflows.
- (minor) `plans/ROADMAP.md:174`: there are no negative remux clips (10-bit H.264, non-AAC audio) and no CRF+2 case.
- (minor) `docs/SPEC.md:375`: there is no test for a Delete with a NULL `discord_message_id`, or for a Discord message delete that fails.
- (minor) `docs/SPEC.md:375`: several sweep and upload failure branches have no test entry.
- (minor) `plans/ROADMAP.md:142`: Phase 2 has no resume, reload, cancel or near-5 GiB on-device check.
- (minor) `plans/ROADMAP.md:237`: there are no banner cases for 0, 2, a missing row or a reload.
- (minor) `docs/SPEC.md:375`: the callback's 299 s vs 301 s boundary, a dev-secret callback rejected in production, session expiry and renewal, and Retry's double tap are all untested.
- (minor) `plans/ROADMAP.md:107`: Phase 0 doesn't read back the CORS and lifecycle rules.

#### 5. Completeness & Cleanup — PASS
There are no stale pending items, broken anchors or committed temp files, and all review items are ticked.
- (minor) `README.md:98`: one line in the stuck-transcode runbook's step 2 runs past the wrap width.

#### 6. Consistency & Style — PASS
Env vars, secrets, columns, log literals and the "token expiry check" name are consistent across all docs.
- (minor) `README.md:40`: "or the channel gets that message" reads as though the banner and the Discord text were identical.

#### 7. Integration Risk — PASS
- (minor) `docs/SPEC.md:72`: the GitHub calls don't state the `User-Agent`, `Accept` and `X-GitHub-Api-Version` headers. Without `User-Agent`, GitHub returns 403, which this design reads as a refused token.
- (minor) `plans/ROADMAP.md:116`: no phase's scope covers creating the repository secrets or the PAT, and Phase 0's "secrets set" names secrets that don't exist yet.
- (minor) `plans/ROADMAP.md:134`: Phase 2's `complete` would dispatch before a workflow exists (Phase 3), and Phase 3's token alerts use the webhook poster that Phase 5 builds.
- (minor) `docs/SPEC.md:66`: the R2 CORS rule needs `AllowedHeaders` (content-type), and the spec doesn't say whether Worker-side R2 calls use the S3 API or a binding.
- (minor) `docs/SPEC.md:261`: the Discord post lacks `allowed_mentions` (same as reviewer 1).
- (minor) `docs/SPEC.md:39`: `src/worker.ts` imports a build output, so typecheck and Vitest fail on a fresh checkout.

#### 8. Error Handling & Silent Failures — FAIL
Most failure paths are handled well.
- (major) `docs/SPEC.md:98`: a callback failure has no category, so a 401, a 400, a 404 and exhausted retries all look the same in the Actions tab. The Worker also logs nothing for a rejected callback, so a secret mismatch fails every transcode with no Worker-side trace.
- (minor) `docs/SPEC.md:252`: Delete's 502 writes no Worker log line naming the failed R2 object.
- (minor) `docs/SPEC.md:266`: a failed Discord message delete is neither logged nor listed as an accepted limitation.
- (minor) `docs/SPEC.md:58`: a D1 error in the sweep is neither caught nor logged.
- (minor) `docs/SPEC.md:113`: a missing expiry header silently disables the check.
- (minor) `docs/SPEC.md:296`: the Upload screen's 10 s poll has no failed-poll behaviour, and nothing says what it shows when the Waffle turns `failed`.

### To-Do: Required Changes

- [x] **Make callback failures distinguishable and log rejected callbacks** — `docs/SPEC.md` (callback ~line 98, Tests worth writing), `README.md` (stuck-transcode runbook, callback-secret rotation) — Give the job's failure line a fixed callback category and the HTTP status (for example `callback: 401 <waffle id>`, `callback: retries exhausted <waffle id>`). Have the Worker write one fixed line per rejected callback (`callback rejected: 401` or `: 400`, with no body, headers or ID taken from an unverified request). Point the runbook and the rotation steps at those lines.
- [x] **Decide whether to trim the token expiry check's monitoring layer** — `docs/SPEC.md` (~lines 111-120, UI banner ~line 286, Data model `token_check`), `plans/ROADMAP.md` (Phases 3 and 4 exit criteria), `README.md:40` — Either keep the daily check with only its alert and log lines, dropping `token_check`, the counter, the three-day post, the catch-upsert accounting and the banner along with their tests, or keep them as decided (dd-261026-069, dd-261027-059). Record the decision. (decided: dd-261028-011, trimmed)
- [x] **Condense the Abort or Delete race paragraph** — `docs/SPEC.md` (~line 254, Failed uploads ~lines 54-62) — State the invariants in a few lines: guarded UPDATE or DELETE on status, R2 before the row, a missing object counts as success, leftover originals are removed by hand. Drop the case-by-case branches, or decide to keep them and record why. (decided: dd-261028-012)
- [x] **Decide whether to simplify the workflow-injection CI check** — `docs/SPEC.md` (~line 74), `plans/ROADMAP.md` (Phase 3 exit criteria) — Either keep only the simple rules (inputs only through `env:`, no `set -x`, a ULID check at the start, no raw tool output in logs) and drop the custom lint with its required check and ruleset, or keep it as Review 2 asked and add must-fail and must-pass fixture workflows (`secrets.*` in `run:`, `actions/github-script` `script:`, `set -o xtrace`, and a compliant `env:` mapping). (decided: dd-261028-013)
- [x] **Suppress mentions on every webhook post** — `docs/SPEC.md` (~line 261, Tests worth writing) — Send `allowed_mentions: {parse: []}` on every Discord webhook post, and test that a caption holding `@everyone` pings nobody.
- [x] **State the GitHub API request headers** — `docs/SPEC.md` (~lines 72, 111) — Every GitHub call sends `User-Agent`, `Accept: application/vnd.github+json` and `X-GitHub-Api-Version`. Add a Phase 3 exit check that a real dispatch with a valid token returns 204 and that the real 2xx response carries the expiry header.
- [x] **Log a missing token expiry header** — `docs/SPEC.md` (~line 113) — Write `token check: no expiry header` to the Worker log on a 2xx with no header, without alerting.
- [x] **Fix the R2 CORS rule and name the Worker's R2 access method** — `docs/SPEC.md` (~line 66, Environments) — Add `AllowedHeaders: content-type`. State that every Worker-side R2 call (multipart create, abort, HEAD, delete, presign) goes through the S3 API with the bucket chosen by `TRANSCODE_ENV`.
- [x] **Schedule secret and PAT setup and the Phase 2 dispatch gap** — `plans/ROADMAP.md` (~lines 116, 134, Phase 3 scope) — Have Phase 0 name the Worker secrets it sets, put the repository secrets and the fine-grained PAT creation in Phase 3's scope, say Phase 2's `complete` stops at `processing` until Phase 3 adds dispatch, and say Phase 3's token alerts use a minimal webhook poster ahead of Phase 5.
- [x] **Keep the Worker entry out of typecheck and tests** (declaration shim `src/worker-adapter.d.ts`; Vitest never imports the entry; Wrangler keys trimmed) (assumed: dd-policy, dd-261028-015) — `docs/SPEC.md` (~line 39) — Say `src/worker.ts` is typed with a declaration shim (or excluded from svelte-check) and that Vitest never imports it. Cut the hand-described Wrangler keys down to what Phase 0 needs.
- [x] **Fix the stuck-jobs boundary** — `docs/SPEC.md` (~line 104, Tests worth writing) — Say "more than 150 minutes" and test 149 vs 151, or use `<=`.
- [x] **Say 52 or 53 weeks in capability 5** ("Waffle week" links to CLAUDE.md § Project rules, which assigns a week to the year of its Wednesday) — `plans/ROADMAP.md:44` — Write "52 or 53 weeks by 4 members (one column per Waffle week in the selected year)".
- [x] **Drop the day count from the token banner** (moot: banner removed by item 2, dd-261028-011) — `docs/SPEC.md:286`, `README.md:40`, `plans/ROADMAP.md` (Phase 4) — Word the banner without "3 days", or accept the wording. In README, say "this banner or the matching channel message".
- [x] **Log the remaining silent failure paths** (logged, not an accepted limitation; also logs abort's 502 as `abort: r2 failed <waffle id>`, and the sweep catches any error, D1 included) — `docs/SPEC.md` (~lines 252, 266, 58) — Write `delete: r2 failed <waffle id> <poster|playback|original>` before Delete's 502. Log `delete: discord message delete failed <waffle id> <status|network>`, or list it as an accepted limitation. Wrap the sweep and log `upload sweep: error` on a D1 failure.
- [x] **Specify the Upload screen's poll failures** (3 failed polls in a row show the note; a 401 goes to sign-in and a 404 stops polling; any Retry answer returns to polling; the Player's Retry points to the Upload screen) — `docs/SPEC.md` (~line 296) — Say a failed poll keeps polling and shows "Connection problem, still trying" after a few misses, and that a `failed` status shows the Waffle's error with Retry.
- [x] **Shorten the callback retry, Discord extras and Environments prose** (applied: accepted limitations merged into one list each in Discord and the token check, Racing runs' restated guards cut, Environments links README § Operations once and its `.dev.vars` paragraph is shorter, Tests worth writing trimmed to cases; Retry-After cap and 119/121 s test kept (assumed: dd-policy, dd-261028-016)) — `docs/SPEC.md` (~lines 98, 101-102, 262, 268, 344-348, 375) — Merge the accepted-limitation bullets into one list. Consider dropping the Retry-After special case and its 119/121 s test. Link the rotation steps to README. Reduce Tests worth writing to the cases the rules don't spell out.
- [x] **Fill the remaining test and drill gaps** (banner cases skipped as moot: banner removed by item 2, dd-261028-011; cutover check placed next to the rate-limit exit check in ROADMAP's Sequencing rationale; also added the Upload screen's 3-failed-polls boundary from item 15, and a missing or non-numeric callback timestamp now reads as a 401) — `docs/SPEC.md` (Tests worth writing), `plans/ROADMAP.md` (Phases 0, 2, 3, 4, 5) — Add:
  - Phase 0: read back both buckets' CORS rules (PUT allowed, ETag exposed, expected origins) and the 7-day abort lifecycle rule;
  - Phase 2: on a phone, kill the connection mid-upload and resume, reload the tab and resume, cancel and see the row and the multipart upload gone, and upload one file near 5 GiB;
  - Phase 3: the deployed Worker lists its Cron trigger; negative remux clips (10-bit H.264, non-AAC audio) that re-encode, and a CRF+2 case;
  - the callback's 299 s vs 301 s boundary, a callback signed with the dev secret rejected in production, session 30-day expiry and renewal, and a double-tapped Retry dispatching once;
  - banner cases for 0, 2, a missing row and a reload;
  - a Delete with a NULL `discord_message_id` makes no Discord call, and a failed Discord delete doesn't block it;
  - a sweep whose R2 abort fails, `NoSuchUpload` counted as success, a failed row insert after the multipart create, and a 409 `cancelled` still `uploading` after one minute;
  - one end-to-end run of each rotation runbook, a failed run found by Waffle ID in the Actions tab, and a cutover check (sign in, upload to `ready` with the callback reaching the new origin, the Discord link opens, then disable workers.dev).
- [x] **Re-wrap the stuck-transcode runbook** (item 1 already kept every line within the wrap width; step 2's tail lines re-flowed to fill it, wording unchanged) — `README.md:98` — Re-wrap step 2 to the surrounding line width.

6b: 0 misses (the only Review 3 findings on lines unchanged since 70a9e41 and inside Review 2's diff, `docs/SPEC.md` 238 at that head (Discord message) and 112 (expiry header), sit within ±5 of lines Review 2 reported, 239 and 114)

## Review 4
Generated: 2026-10-29 09:45
Comparison: origin/docs/spec b0e089f0a6fa27f629a846f7c4fc11df2bfe1442..d778742a49ceebce86bbf5de45b492c46d3f696a
Verdict: **BLOCKED**

Mode: full review (first non-docs path: `docs/SPEC.md`); Type Design not launched (no code type definitions).

### Results by Reviewer

#### 1. Safety & Security — PASS
The untrusted-input rules, the signed callback, the masked logs and the guarded operations all hold up. Only minor hardening gaps remain:
- (minor) `docs/SPEC.md:52`, `docs/SPEC.md:235`: `POST /api/uploads` validates `ext` and size but not `caption` (no length cap) or `week_start` (no rule that it is this or last Waffle week's Wednesday). `week_start` goes into the download name and `caption` into Discord and the UI.
- (minor) `docs/SPEC.md:93`: nothing says the callback route fails closed when `TRANSCODE_CALLBACK_SECRET` is unset or empty. An empty HMAC key makes signatures forgeable.
- (minor) `docs/SPEC.md:79`: ffmpeg auto-detects the demuxer on a member-uploaded file, so a crafted playlist or concat input could read local files or fetch URLs on a runner that holds the R2 keys. The workflow's actions aren't pinned to SHAs and its `GITHUB_TOKEN` permissions aren't stated.
- (minor) `docs/SPEC.md:224`: CSRF defence rests on `SameSite=Lax` alone. State-changing routes have no Origin check.

#### 2. Correctness — PASS
Cross-document facts, anchors and boundary values (149/150 min, 300/301 s, 119/121 s, 13/15 days, 103 parts, the 53-week year) are consistent.
- (minor) `docs/SPEC.md:343`: the workflow's repository secrets don't include `R2_ACCOUNT_ID`, which the S3 endpoint needs (same as reviewer 7).
- (minor) `docs/SPEC.md:43`: the pipeline intro says "original resolution and frame rate", but elsewhere the copy is capped at 1080p when it would exceed 1.5× the original's size.
- (minor) `docs/SPEC.md:235`: no rule validates the client-supplied `week_start` (same as reviewer 1).

#### 3. Simplicity & Conciseness — FAIL
- (major) `docs/SPEC.md:351`: the Callback rate limit WAF rule adds a zone rule, a cutover step, a ROADMAP exit check and a drill to protect the log quota for a four-person app, when the HMAC check already rejects unsigned requests cheaply. Commit ce092ed cites dd-261028-009 for adding it, but SPEC records no decision.
- (minor) `README.md:35`, `README.md:70`, `README.md:95`: "Never paste it anywhere else…" appears three times and copies CLAUDE.md's rule.
- (minor) `README.md:97`: step 2 of the stuck-transcode runbook re-explains each callback failure line that SPEC already defines.
- (minor) `README.md:62`: step 5 of the R2 key rotation packs timing, URL invalidation and a leak audit into one item.
- (minor) `docs/SPEC.md:376`: Tests worth writing still restates many rules.
- (minor) `docs/SPEC.md:54`: Failed uploads gives seven failure shapes their own message and recovery, including a one-minute re-read loop for a 409 `cancelled`.

#### 4. Test Coverage — PASS
Nearly every specified behaviour has a test or an exit check. Only minor drill gaps remain:
- (minor) `plans/ROADMAP.md:140`: Phase 1 has no drill for a guild member who isn't in `members`, for a removed member, or for a session surviving a redeploy.
- (minor) `plans/ROADMAP.md:287`: Phase 4 has no exit check for the Retry button, and none for a Delete repeated after an R2 failure.
- (minor) `plans/ROADMAP.md:304`: Phase 5 has no real-channel drill for an `@everyone` caption or for a Delete after the webhook was revoked.
- (minor) `docs/SPEC.md:376`: no named table-driven test covers the streak and Best-streak calculation.

#### 5. Completeness & Cleanup — PASS
No stale references to removed features, every anchor resolves, and the review ticks match the fixes.
- (minor) `plans/META-PLAN.md:35`: Verification says "pending", which the first feature with tests must replace (intended).
- (minor) `README.md:13`: Getting started and Tests are still scaffold placeholders, which matches CLAUDE.md (intended).

#### 6. Consistency & Style — PASS
Routes, error codes, log lines, env vars and terms are consistent across the docs.
- (minor) `README.md:24`: this line is the longest in Operations. Re-flowing it is optional.

#### 7. Integration Risk — FAIL
- (major) `docs/SPEC.md:343`: the workflow signs R2 S3 requests against `https://<R2_ACCOUNT_ID>.r2.cloudflarestorage.com`, but its repository secrets list only the R2 keys, the callback secret and `CALLBACK_BASE_URL`. `R2_ACCOUNT_ID` is a Worker secret only, and ROADMAP Phase 3 defers to this list.
- (major) `plans/ROADMAP.md:145`: Phase 2 relies on `GET /api/waffles/:id` (the 409 `cancelled` re-read and the Upload screen's poll) and on `GET /api/waffles` (the trigger for the abandoned-upload sweep), but Phase 3 builds both routes.
- (minor) `plans/ROADMAP.md:132`: no phase schedules the `[vars]` block (`APP_ORIGIN`, `GITHUB_REPO`, `GITHUB_WORKFLOW`, `TRANSCODE_ENV`, `GROUP_TZ`) or registering the Discord redirect URIs, which Phase 1's sign-in needs.
- (minor) `docs/SPEC.md:374`: nothing says when to change the `DISCORD_WEBHOOK_URL` secret to the real channel at launch (leaving `.dev.vars` on the test channel), or that token alerts follow it.
- (minor) `plans/ROADMAP.md:170`: Phase 3 doesn't say the workflow file must be merged to main first, or that `GITHUB_WORKFLOW` must match its file name.

#### 8. Error Handling & Silent Failures — PASS
Every failure path has a fixed, ID-only log line or an actionable message. Catches are scoped, and retries end in a visible failure.
- (minor) `docs/SPEC.md:47`: a failed `waffles` row insert after the multipart create returns a 5xx with no fixed log line.
- (minor) `docs/SPEC.md:109`: a rejected dispatch writes no Worker log line.
- (minor) `docs/SPEC.md:93`: an unexpected exception in the callback route (a D1 failure, for example) has no fixed log line.

### To-Do: Required Changes

- [x] **Give the workflow the R2 account ID** — `docs/SPEC.md` (Environments ~line 343), `plans/ROADMAP.md` (Phase 3 secret list) — Add `R2_ACCOUNT_ID` to the repository secrets (or a repository variable) the transcode workflow needs, and name it in Phase 3's secret list.
- [x] **Fix Phase 2's dependency on Phase 3's GET routes** — `plans/ROADMAP.md` (~lines 145-160, Phase 3 ~line 194), `docs/SPEC.md` (Failed uploads ~line 57, sweep ~line 63, Upload screen poll) — Either move `GET /api/waffles` and `GET /api/waffles/:id` (without the stuck-jobs check) into Phase 2, or say that Phase 2's 409 re-read, poll and sweep trigger use only what Phase 2 builds, and that the GET-route triggers arrive in Phase 3. (assumed: dd-policy, dd-261029-019)
- [x] **Decide whether to keep the callback WAF rate limit** — `docs/SPEC.md` (~lines 351, 356), `plans/ROADMAP.md` (~lines 343-346) — Either drop the rule, its cutover step and its exit check, or keep it and cite dd-261028-009 next to it so the choice is recorded where it lives.
- [x] **Validate `week_start` and `caption` on upload start** — `docs/SPEC.md` (Upload checks ~line 52, route table ~line 235, Tests worth writing) — Accept only the current or previous Waffle week's Wednesday as `week_start` (anything else gets a 400 with a stable code), trim `caption` and cap its length well under Discord's 2000 characters, and add test cases for both.
- [x] **Fail closed on an empty callback secret** — `docs/SPEC.md` (Callback ~line 93, Tests worth writing) — Reject every callback, with a fixed log line, when `TRANSCODE_CALLBACK_SECRET` is missing or empty, have the workflow refuse to run with an empty secret, and add a test.
- [x] **Harden the ffmpeg input and the workflow's permissions** — `docs/SPEC.md` (Transcode step 2 ~line 79) — Require ffprobe's `format_name` to be in the mov/mp4 family, read the original with an explicit `-f mov` or a file-only protocol whitelist, pin third-party actions to commit SHAs, and set the job's `GITHUB_TOKEN` permissions to read-only or none.
- [x] **Add an Origin check on state-changing routes** — `docs/SPEC.md` (Auth ~line 224) — Reject a POST or DELETE to `/api` or `/auth` whose `Origin` header is present and isn't `APP_ORIGIN`, and require `application/json` on the JSON routes.
- [x] **Say the playback copy may be capped at 1080p** — `docs/SPEC.md:43` — Add the 1080p cap (when the copy would exceed 1.5× the original's size) to the pipeline intro, or link to it.
- [x] **Schedule the Worker `[vars]` and the Discord redirect URIs** — `plans/ROADMAP.md` (Phases 0, 1, 3) — Put the `[vars]` block in Phase 0 (or `APP_ORIGIN` and the redirect URI registration in Phase 1, and `GITHUB_*` and `TRANSCODE_ENV` in Phase 3), and say which phase first needs `GROUP_TZ`.
- [x] **Add the launch webhook swap** — `plans/ROADMAP.md` (Launch) or `README.md` (Operations) — Add a launch step: `wrangler secret put DISCORD_WEBHOOK_URL` with the real channel's webhook, `.dev.vars` stays on the test channel, and token alerts move with the webhook.
- [x] **Merge the workflow file before Phase 3's checks** — `plans/ROADMAP.md` (Phase 3 scope ~line 170) — Say the transcode workflow is merged to main before any Phase 3 dispatch check, and that `GITHUB_WORKFLOW` matches its file name.
- [x] **Log the three remaining unlogged failures** — `docs/SPEC.md` (~lines 47, 93, 109) — Add fixed lines: `upload start: row insert failed` before the 5xx, `dispatch rejected: <status|network> <waffle id>` on a rejected dispatch, and `callback: error <waffle id>` with a 500 on an unexpected callback-route error.
- [x] **Trim the README's repeated and restated prose** — `README.md` (~lines 35, 62-68, 70, 95, 97-123) — State the never-paste-secrets warning once (or link to CLAUDE.md § Project rules). Cut step 2 of the stuck-transcode runbook to finding the run and reading its last line, with a link to SPEC's Callback for what each line means. Shorten step 5 of the R2 key rotation.
- [x] **Consider merging the Failed uploads outcomes and the restated tests** — `docs/SPEC.md` (~lines 54-63, 376-397) — Merge the non-retryable upload failures into one "cancel and upload again" outcome, and keep in Tests worth writing only the cases the rules don't spell out. Or keep both as decided and say so. (asked: dd-261029-023)
- [x] **Fill the remaining drill gaps** — `plans/ROADMAP.md` (Phases 1, 4, 5), `docs/SPEC.md` (Tests worth writing) — Phase 1: a guild member who isn't in `members` is refused, and a session survives a redeploy. Phase 4: the Retry button reaches `ready` for its owner and is absent for others, and a Delete repeated after an injected R2 failure finishes. Phase 5: an `@everyone` caption pings nobody in the real channel, and deleting a Waffle after the webhook was revoked still deletes it. SPEC: a table-driven test of the streak and Best-streak calculation.
- [x] **Optionally re-flow README line 24** — `README.md:24` — Re-wrap it to the surrounding Operations width.

### Missed Findings

| Finding | Severity | Unit | Ledger id |
|---|---|---|---|
| `POST /api/uploads` doesn't validate `week_start` or `caption` (`docs/SPEC.md:235`) | minor | subagent-1:safety-security | git-push-20261029-001 |
| No Origin check, so CSRF rests on SameSite=Lax alone (`docs/SPEC.md:224`) | minor | subagent-1:safety-security | git-push-20261029-002 |
| A failed row insert after the multipart create has no log line (`docs/SPEC.md:47`) | minor | subagent-8:error-handling | git-push-20261029-003 |
| Step 5 of the R2 key rotation is overly branchy (`README.md:62`) | minor | subagent-3:simplicity | git-push-20261029-004 |
| The never-paste-secrets warning is repeated (`README.md:70`) | minor | subagent-3:simplicity | git-push-20261029-005 |

## Review 5
Generated: 2026-11-01 20:15
Comparison: origin/docs/spec b0e089f0a6fa27f629a846f7c4fc11df2bfe1442..6e102a60dee35e74056eb986b520dfe4ac9dd89b
Verdict: **BLOCKED**

Mode: full review (first non-docs path: `docs/SPEC.md`); Type Design not launched (no code type definitions).

### Results by Reviewer

#### 1. Safety & Security — PASS
The design is security-conscious: untrusted-input rules, a fail-closed HMAC, the ffmpeg protocol whitelist, Origin and Content-Type checks, and sanitized logs. Only minor hardening gaps remain:
- (minor) `docs/SPEC.md:349`: every `.dev.vars` copy holds the both-bucket R2 token, so a leaked laptop copy can delete production originals, which are the only copy. A dev-only token scoped to `waffles-dev` would close this.
- (minor) `docs/SPEC.md:76`: workflow secrets are plain repository secrets that any branch's run can read, and the spec never states that `workflow_dispatch` is the only trigger.
- (minor) `docs/SPEC.md:306`: no security headers are specified (CSP `frame-ancestors 'none'`, `nosniff`), and nothing says caption, display name, `error` and `discord_error` are always rendered as text, never `{@html}`.
- (minor) `docs/SPEC.md:230`: requests with no Origin header pass the cross-site check, so it is a single layer. `Sec-Fetch-Site` could be added as a second, optional layer.
- (minor) `docs/SPEC.md:227`: nothing says login mints a fresh session token and discards the Discord access token, or how the `GET /api/waffles` query filters are validated.
- (minor) `plans/ROADMAP.md:281`: the hand-curl drill would put the dispatch token in argv and shell history unless the header is passed via `-H @file` or `-K -`.

#### 2. Correctness — PASS
Anchors, boundary arithmetic and cross-document references are consistent.
- (minor) `docs/SPEC.md:87`: a poster taken at about 1 s fails the whole transcode on a clip shorter than 1 s, and a fixed 1280 px width upscales narrow portrait clips.

#### 3. Simplicity & Conciseness — FAIL
- (major) `docs/SPEC.md:39`: the token expiry check (custom Worker entry, shim, Cron, log lines, alert type) could be replaced by a calendar reminder. *Already decided to keep: dd-261027-059, dd-261028-011.*
- (major) `docs/SPEC.md:357`: the callback WAF rate limit. *Already decided to keep: dd-261028-009 (Review 4 item 3).*
- (major) `docs/SPEC.md:62`: the abandoned-upload sweep (two triggers, guarded DELETE, R2 abort, log lines, race analysis) duplicates the R2 lifecycle rule. The owner's Cancel upload, or a by-hand script, could cover stale rows.
- (major) `docs/SPEC.md:255`: the Abort or Delete invariants, together with Failed uploads and Error codes, are a failure model beyond what a four-person app needs. *Partly addressed by Review 4 item 14 (dd-261029-023).*
- (major) `docs/SPEC.md:74`: the public-repo hardening cluster could shrink to three rules. *The rules were kept by dd-261028-013, and SHA pinning plus the `format_name` check were added by Review 4 item 6.*
- (minor) `docs/SPEC.md:230`: the custom Origin hook could be dropped in favor of SvelteKit's default `csrf.checkOrigin`. *Added by Review 4 item 7 (the default check is turned off on purpose so the callback stays reachable).*
- (minor) `docs/SPEC.md:269`: the Discord claim gives exactly-once delivery. *Kept by dd-261027-059.*
- (minor) `docs/SPEC.md:108`: Racing runs lists limitations that change no decision. *dd-261027-031.*
- (minor) `README.md:52`: the rotation runbooks carry leak-response branches. *Trimmed by Review 4 item 13.*
- (minor) `docs/SPEC.md:382`: Tests worth writing still restates its rules. *Trimmed by Review 4 item 14.*

#### 4. Test Coverage — FAIL
- (major) `docs/SPEC.md:262`: no signed-out (401) check exists for any mutating `/api` route or for `GET /api/heatmap`. A route that forgets its session check would let anyone start uploads or delete Waffles.
- (major) `plans/ROADMAP.md:329`: capability 4's Archive filters, grouping and newest-first order have no exit check or test, and the `year` filter of `GET /api/waffles` (`docs/SPEC.md:245`) is undefined (calendar year or heatmap year).
- (minor) `docs/SPEC.md:389`: the dispatch-target test covers only complete and Retry, and doesn't assert the bucket name in signed requests for `prod` and `dev`.
- (minor) `plans/ROADMAP.md:144`: no check that a successful logout invalidates the old cookie (it should then 401), and the `Set-Cookie` flags (HttpOnly, Secure, SameSite=Lax) are never asserted.
- (minor) `plans/ROADMAP.md:329`: Phase 4 never checks that Download original saves a byte-identical file under its readable name, and an expired 6-hour GET URL has no defined behavior.
- (minor) `plans/ROADMAP.md:233`: Phase 3's clip checks don't compare the resolution, frame rate and duration of original and playback, don't include a portrait 4K clip with a rotation flag, and don't assert HDR to bt709 with `is_hdr = 1`.
- (minor) `plans/ROADMAP.md:233`: no real drill covers the job's `no_waffle` 404 cleanup (delete a Waffle mid-run, then list R2).
- (minor) `docs/SPEC.md:393`: the sweep's `POST /api/uploads` trigger and its runs-after-response behavior (`ctx.waitUntil`) are untested.

#### 5. Completeness & Cleanup — PASS
No stale references to removed items, all anchors resolve, and all 16 Review 4 ticks match the commit log.
- (minor) `plans/META-PLAN.md:35`, `README.md:13`: intended placeholders (Verification pending, Getting started not set up).

#### 6. Consistency & Style — PASS
- (minor) `README.md:123`, `README.md:129`: the log-line placeholder is written `<id>`, but SPEC's literals use `<waffle id>`.
- (minor) `docs/SPEC.md`: paragraphs aren't wrapped, while the other docs wrap at about 95–100 characters.

#### 7. Integration Risk — PASS
- (minor) `plans/ROADMAP.md:113`: no phase adds `[observability] enabled = true` or the `[triggers] crons` entry, though Phase 2–3 exit checks read Worker logs.
- (minor) `plans/ROADMAP.md:219`: Phase 3's callback doesn't say it takes no Discord claim and posts nothing until Phase 5.
- (minor) `docs/SPEC.md:112`: the retry UPDATE doesn't clear `error`.

#### 8. Error Handling & Silent Failures — FAIL
Most failure paths have fixed, ID-only log lines and scoped catches. Three paths remain silent:
- (major) `docs/SPEC.md:300`: a Retry request that fails before touching the row (network error or 5xx) leaves the same failed Waffle on the next poll with no signal, and the retry route has no log line.
- (major) `docs/SPEC.md:269`: in the post-callback Discord path, a D1 failure while storing `discord_message_id` or `discord_error`, or any other throw, is neither logged nor stored. The result is an orphan message and an unexplained Waffle. The Post to Discord again route has the same gap.
- (major) `docs/SPEC.md:225`: OAuth callback failures other than "not a member" have no user message and no log line: a rejected token exchange, a Discord 429/5xx or network error, a state-cookie mismatch, or a D1 session insert error. They risk being shown as "not a member".
- (minor) `docs/SPEC.md:259`: R2 HEAD and finish failures in complete, and the `size_mismatch` and `parts_rejected` rejections, have no log line.
- (minor) `docs/SPEC.md:253`: a D1 failure on the final row DELETE in delete or abort has no specified response or log line.
- (minor) `docs/SPEC.md:56`: the could-not-finish message never says "tell the admin", and a failed localStorage write silently makes an upload non-resumable.
- (minor) `docs/SPEC.md:102`: a valid signed callback that changes no row isn't logged, and the spec doesn't say whether that is deliberate.

### To-Do: Required Changes

- [x] **Specify OAuth callback failures** — `docs/SPEC.md` (Auth ~line 225, Tests worth writing), `plans/ROADMAP.md` (Phase 1 exit criteria) — A rejected token exchange, a Discord 429/5xx or network error on `/users/@me` or `/users/@me/guilds`, a missing or mismatched state cookie, and a D1 error creating the session each show a fixed "Sign-in failed; try again, or tell the admin." page, distinct from "not a member". Each writes one fixed, ID-free line (`auth callback: state mismatch`, `auth callback: discord <status code or network>`, `auth callback: error`) and never logs the code, tokens or body. Add test cases.
- [x] **Handle a failed Retry request** — `docs/SPEC.md` (Upload screen ~line 300, Player Retry, retry route ~line 112, Tests worth writing) — A Retry answered with a network error, a 5xx or any non-409 error shows a fixed "Could not retry; try again, or tell the admin." instead of silently polling. An unexpected retry-route error writes `retry: error <waffle id>`. In the same pass, add `error = NULL` to the retry UPDATE (or say `error` matters only while `status = failed`).
- [x] **Log post-callback Discord failures** — `docs/SPEC.md` (Discord ~lines 269-276, Tests worth writing) — Wrap the `ctx.waitUntil` post, and the Post to Discord again route, in a catch that writes one fixed `discord post: error <waffle id>` line with no error text. Add a test: Discord accepts the post, then the D1 update that stores `discord_message_id` fails.
- [x] **Test signed-out access on every /api route** — `docs/SPEC.md` (Tests worth writing), `plans/ROADMAP.md` (exit criteria of Phases 2, 4, 5, 6 for the routes each builds) — Add a table-driven test over every `/api` route except the callback: no session gives 401 and no state change. Include a removed member (`active = 0`) on one mutating route.
- [x] **Define and test the Archive filters** — `docs/SPEC.md` (`GET /api/waffles` ~line 245, Tests worth writing), `plans/ROADMAP.md` (Phase 4 exit criteria ~line 329) — Define `year` (the heatmap year, i.e. the year of `week_start`'s Wednesday, to match CLAUDE.md), validate `member`, `year` and `week_start` (400 with a code), and add API tests for each filter, their combinations, newest-first order and the late-December/January boundary. Add a Phase 4 check that the Archive, filtered by member and by year, shows exactly the seeded Waffles.
- [x] **Decide whether to keep the abandoned-upload sweep** (kept: dd-261101-018) — `docs/SPEC.md` (~lines 62-63, Abort or Delete, Storage layout, Tests worth writing ~line 393), `plans/ROADMAP.md` (Phase 2) — Either drop the sweep (keep the R2 lifecycle rule plus the owner's Cancel upload, with leftover rows removed by hand), or keep it and cite the decision next to it. If kept, add tests that `POST /api/uploads` triggers it and that a throwing sweep leaves the route's response unchanged.
- [x] **Close the remaining simplicity re-asks against their decisions** (all kept: dd-261101-019) — `docs/SPEC.md` (~lines 39, 74, 108, 230, 255, 269, 357, 382), `README.md:52` — Review 5's Simplicity reviewer re-raised the token expiry check, WAF rule, hardening cluster, Abort or Delete invariants, Origin hook, Discord claim, Racing runs, rotation runbooks and test list. Each is already decided (dd-261027-059, dd-261028-011, dd-261028-009, dd-261028-013, dd-261029-023, dd-261027-031) or was trimmed in Review 4. Tick this if you are keeping them; otherwise reopen the named decision.
- [ ] **Log complete's and the row-delete's remaining failures** — `docs/SPEC.md` (~lines 253, 259, Tests worth writing) — Add fixed, ID-only lines `complete: r2 failed <waffle id> <finish|head>` and `complete: rejected <waffle id> <parts_rejected|size_mismatch>`. Specify a D1 failure on the final row DELETE as a 5xx with a fixed message plus `delete: row failed <waffle id>` / `abort: row failed <waffle id>`. Optionally log `callback: no change <waffle id>` for a signed, valid callback that changes no row, or say it is deliberately unlogged (~line 102).
- [ ] **Improve the could-not-finish and resume messages** — `docs/SPEC.md` (Failed uploads ~line 56, Upload steps ~line 49) — Append "or tell the admin" to the 5xx message, and show a note when the Upload screen cannot save resume state to localStorage.
- [ ] **Widen the dispatch-target test** — `docs/SPEC.md:389` — Cover every R2-touching route (uploads, parts, abort, DELETE, GET presigns), checking that a bad `TRANSCODE_ENV` makes no R2 call, and assert the bucket name in the signed request for `prod` → `waffles` and `dev` → `waffles-dev`.
- [ ] **Test logout and the cookie attributes** — `plans/ROADMAP.md` (Phase 1 exit criteria ~line 144), `docs/SPEC.md` (Tests worth writing) — After logout, the old cookie gets 401. Assert HttpOnly, Secure and SameSite=Lax on the login `Set-Cookie`.
- [ ] **Fill Phase 3–4 media and download drills** — `plans/ROADMAP.md` (Phase 3 ~line 233, Phase 4 ~line 329), `docs/SPEC.md` (Player) — For each clip, compare ffprobe width, height, frame rate and duration of original and playback (except where the cap applies). Add a portrait 4K clip with a rotation flag, and assert the HDR clip's playback is `bt709` with `is_hdr = 1`. Add a drill that deletes a Waffle mid-run and then finds nothing under `playback/{id}/`. Add a Phase 4 check that Download original saves a byte-identical file under its readable name on iOS Safari and desktop. Define what the Waffle screen does when its 6-hour GET URLs expire (refetch `/api/waffles/:id`).
- [ ] **Fix the poster edge cases** — `docs/SPEC.md:87` — Take the poster at min(1 s, duration/2), or fall back to the first frame, and scale it down only (never above the source width).
- [ ] **Schedule Workers Logs and the Cron trigger, and fence Phase 3's callback** — `plans/ROADMAP.md` (Phase 0 ~line 113, Phase 3 ~line 219) — Put `[observability] enabled = true` in Phase 0's Wrangler-config bullet and the `[triggers] crons` entry in Phase 3. State that Phase 3's callback applies the guarded UPDATE only and takes no Discord claim until Phase 5.
- [ ] **Add the minor security hardening** — `docs/SPEC.md` (~lines 76, 227, 230, 306, 349), `plans/ROADMAP.md:281` — State that the workflow has only a `workflow_dispatch` trigger. Add CSP `frame-ancestors 'none'` and `nosniff` in the server hook, and say user and callback text is never rendered as HTML. Say login mints a fresh session token and drops the Discord access token. Consider a `waffles-dev`-only R2 token for `.dev.vars`. Have the hand-curl drill pass the token header via `-H @file` or `-K -`. `Sec-Fetch-Site` is optional.
- [ ] **Match log placeholders and decide SPEC's wrap** — `README.md:123`, `README.md:129`, `docs/SPEC.md` — Use `<waffle id>` in place of `<id>`, and either wrap SPEC prose at about 95 characters or leave it unwrapped on purpose.

### Missed Findings

| Finding | Severity | Unit | Ledger id |
|---|---|---|---|
| No signed-out 401 check on mutating `/api` routes (`docs/SPEC.md:262`) | major | subagent-4:test-coverage | not logged |
| A failed Retry request gives no signal and no log line (`docs/SPEC.md:300`) | major | subagent-8:error-handling | not logged |
| Post-callback Discord failures are neither logged nor stored (`docs/SPEC.md:269`) | major | subagent-8:error-handling | not logged |
| The abandoned-upload sweep duplicates the R2 lifecycle rule (`docs/SPEC.md:62`) | major | subagent-3:simplicity | not logged |

Not logged: the ledger is dormant and logs critical misses only. Minor misses (dispatch-target test, clip checks, sweep trigger, error-handling minors, curl drill, Integration Risk minors) were skipped for the same reason. Findings that reopen decided DDs, and findings on lines from before `b0e089f`, are not counted as misses.
