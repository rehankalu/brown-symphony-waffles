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
- [ ] **Make the dispatch ref configurable or document merge-to-iterate** — `docs/SPEC.md` (~line 77) — Add a `GITHUB_REF` var defaulting to `main`, or note that Phase 3 tuning merges each workflow change to `main` first.
- [ ] **Fix the domain-cutover steps** — `docs/SPEC.md` (~line 322) — Drop step 7 ("Point the webhook at the real Waffles channel") or mark it "if not already done at launch". Note that old workers.dev links work only while that route stays enabled, and that members sign in again on the new origin.
- [ ] **Log a failed cleanup after a 404** — `docs/SPEC.md` (~line 99) — Cleanup errors are non-fatal, but the job prints a fixed line `cleanup failed for <Waffle ID>`.
- [ ] **Fill the remaining test and exit-criterion gaps** — `docs/SPEC.md` (Tests worth writing), `plans/ROADMAP.md` (Phases 3 and 5) — Add:
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
- [ ] **Trim the duplicated and over-long prose** — `CLAUDE.md:48`, `docs/SPEC.md` (~lines 42, 73, 88, 101, 133, 351, Overview), `plans/ROADMAP.md` (~lines 102, 167) — Make each Project rule's other copies links to CLAUDE.md or SPEC. Delete the stale diagram comment. Link the SPEC Overview's goals and non-goals to ROADMAP and keep the Decisions table. Validate the env value in one place, not both. Shorten Racing runs, and point Tests worth writing at the rules instead of re-listing them. Cut ROADMAP's External dependencies down to links, and drop the repeated "Actions: write". Decide whether the token-expiry Cron and the Discord claim lock are worth their cost against a calendar reminder and the once-only guarded ready UPDATE.
- [ ] **Fix the small reference and naming nits** — `docs/SPEC.md:21`, `plans/ROADMAP.md:168`, `plans/META-PLAN.md:35`, `CLAUDE.md` (Where things live table) — Say "in the Follow-on: NAS backup section". Use "token expiry check" everywhere. Set Verification to "pending — set by the first feature with tests". Add "and operate" to the README row.

6b: 0 misses (every line Review 1's diff touched in SPEC and ROADMAP has changed since eb7b1bd, so no Review 2 finding sits on unchanged Review 1 lines)
