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
- [ ] **Define the dispatch target and per-environment inputs** — `docs/SPEC.md` (Transcode, config, environments/domain cutover) — Add non-secret Worker vars: `GITHUB_REPO`, the workflow file name, and dispatch `ref` = `main`. Say how the job gets the bucket and the callback base URL (inputs, or derived values). Note that local dev can't receive callbacks, and add the callback URL to the domain-cutover checklist.
- [ ] **Harden workflow input handling** — `docs/SPEC.md` (Transcode) — The workflow validates the Waffle ID against a strict pattern and passes it through `env:`, never inline `${{ inputs.* }}` in `run:`.
- [ ] **Specify callback semantics** — `docs/SPEC.md` (~line 192) — Cover the responses for an unknown ID, an already-final Waffle (idempotent handling) and a bad HMAC. The job retries the callback a few times and then fails the run visibly.
- [ ] **Keep raw tool errors out of public logs** — `docs/SPEC.md` (Transcode) — Log only the Waffle ID and an error category, use `-loglevel error` with fixed local filenames, and post a sanitized, truncated error to the callback.
- [ ] **Correct the `repository_dispatch` rationale** — `plans/ROADMAP.md:162` — Replace "rewrite the workflow" with "change code the workflow runs and expose its secrets", since workflow files need the separate Workflows permission.
- [ ] **Extend acceptance and exit criteria for the new rules** — `plans/ROADMAP.md` (capability 5, Phases 2, 3 and 6) — Add the streak break case and the Wed 00:00 ET boundary to capability 5, and mirror the streak case in Phase 6's exit criteria. In Phase 2, an invalid token ends the Waffle `failed` with visible text that never contains the token. In Phase 3, the owner's Retry reaches `ready` and another member's Retry is refused. Also check that the PWA installs on iOS and Android with no service worker registered.
- [ ] **Remove duplicated facts** — `plans/ROADMAP.md` (Phases 2 and 3), `docs/SPEC.md` (streak sentence) — Keep each *Why:* in ROADMAP and link to `docs/SPEC.md#transcode` and `#ui` for the PAT scope, the failure behaviour and the PWA scope. Shorten the streak sentence so it states the rule once.
- [ ] **Sync the SPEC Build plan Stack line** — `docs/SPEC.md:276` — Add npm and `workflow_dispatch`, or point the line at CLAUDE.md's Stack as the home for those facts.
