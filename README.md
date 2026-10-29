# brown-symphony-waffles

Waffles: a private web app where four friends upload weekly phone videos to Cloudflare R2,
get a near-original-quality copy every device can play, and see a link posted in Discord.
It replaces the WhatsApp group, which exists only because Discord caps free uploads at 20 MB.

## Status

Scaffolded 2026-10-26. Direction: [`plans/ROADMAP.md`](plans/ROADMAP.md). Next work:
[`plans/META-PLAN.md`](plans/META-PLAN.md).

## Getting started

_Not set up yet._ <!-- prerequisites, install, run -->

## Tests

_Not set up yet._

## Operations

At launch the admin points `DISCORD_WEBHOOK_URL` at the real channel; the steps are under
**Webhook swap** in [ROADMAP § Sequencing rationale](plans/ROADMAP.md#sequencing-rationale).

In every rotation below, a new value goes only where its steps put it: the Worker's secrets,
the repository secrets, and `.dev.vars` where a step says so (where credentials live is in
[CLAUDE.md § Project rules](CLAUDE.md#project-rules)). A copy for another member's
`.dev.vars` goes in person or through a password manager. Never paste a value anywhere else:
not in the repo, an issue, a workflow log or Discord.

### Rotating the GitHub dispatch token

`GITHUB_DISPATCH_TOKEN` expires. The Worker's daily token expiry check warns in the Discord
channel before it does, or once GitHub refuses it (see
[SPEC Transcode](docs/SPEC.md#transcode)). The admin then:

1. In GitHub, regenerate the fine-grained token (Settings → Developer settings → Personal
   access tokens → Fine-grained tokens) with the repository access, permission and expiry
   SPEC Transcode names.
2. Run `npx wrangler secret put GITHUB_DISPATCH_TOKEN` and paste the new token at the prompt.
   This updates the deployed Worker (prototype and production are the same Worker).
   Then replace the token in `.dev.vars` too, since local dev keeps its own copy of it, and
   get it to every other member with a local copy.
3. Tell members whose Waffles failed with "its access token may have expired" to press Retry.

The next token expiry check stays quiet once the new token is in place.

When an upload fails with "its access token may have expired" but no alert came, read the
`token check` lines in the Worker's logs (Cloudflare dashboard → the Worker → Logs) and fix
what they name (see [SPEC Transcode](docs/SPEC.md#transcode)).

### Rotating the R2 keys

`R2_ACCESS_KEY_ID` and `R2_SECRET_ACCESS_KEY` come from the one R2 API token that the deployed
Worker, the workflow and every `.dev.vars` share (see
[SPEC Environments](docs/SPEC.md#environments)), so a leaked copy, including a laptop's
`.dev.vars`, is a reason to rotate at once. A leaked `.dev.vars` also exposes the dispatch
token (rotate it with the steps above), `DISCORD_CLIENT_SECRET` and the test channel's
`DISCORD_WEBHOOK_URL`: reset those in the Discord developer portal and the test channel's
webhook settings, then replace them in every `.dev.vars` and with `npx wrangler secret put`.
After launch the Worker holds the real channel's webhook instead (see the pointer above), so
put only the new client secret there and leave its webhook alone unless that leaked too.
The admin:

1. In the Cloudflare dashboard (R2 → Manage API tokens), creates a new token with the
   permission and buckets [SPEC Environments](docs/SPEC.md#environments) names.
2. Runs `npx wrangler secret put R2_ACCESS_KEY_ID` and `npx wrangler secret put
   R2_SECRET_ACCESS_KEY`, pasting the new values at the prompts.
3. Replaces both repository secrets of the same names (GitHub → the repository → Settings →
   Secrets and variables → Actions).
4. Replaces both values in `.dev.vars`, and gets the new values to every other member with a
   local copy.
5. Once no Waffle is `processing`, deletes the old token; a run that started before step 3
   still uses the old keys, so deleting it earlier fails that run's uploads. URLs presigned
   with it stop working: an upload in progress needs Resume or Try again, and an open video a
   page reload. If the old token leaked, the admin deletes it right after step 4 instead, has
   the uploaders of any failed Waffles press Retry, and checks the `waffles` bucket's
   originals against D1 for anything deleted (R2 keeps no old versions).

The members' read-only NAS tokens (see
[SPEC Follow-on: NAS backup](docs/SPEC.md#follow-on-nas-backup)) are separate and are not
affected.

### Rotating the callback secret

The production `TRANSCODE_CALLBACK_SECRET` lives in two places, the Worker's secrets and the
repository secrets, and both must hold the same value. `.dev.vars` holds a separate dev-only
value (see [SPEC Environments](docs/SPEC.md#environments)), which can be rotated on its own
by replacing it there. To rotate the production value, the admin:

1. Waits until no Waffle is `processing`, and asks members not to upload or press Retry until
   step 4 is done. If the production secret leaked, skips the wait: anyone holding it can
   forge a `ready` or `failed` callback.
2. Generates a new random value, for example with `openssl rand -hex 32`.
3. Runs `npx wrangler secret put TRANSCODE_CALLBACK_SECRET` and pastes it at the prompt.
4. Right away, replaces the `TRANSCODE_CALLBACK_SECRET` repository secret (GitHub → the
   repository → Settings → Secrets and variables → Actions) with the same value.

A run reads the repository secret when it starts, so any run that started before step 4 and
reports after step 3 signs with the old value and is rejected: its log ends with
`callback: 401 <waffle id>`. Its Waffle ends `failed` under the **Stuck jobs** rule in
[SPEC Transcode](docs/SPEC.md#transcode), and its uploader presses Retry. If a run started
after step 4 still ends with that line, the two copies differ; repeat steps 2 to 4.

### Transcodes that did not report back

A Waffle that a run never reported back for fails with "The transcode did not report back"
(the **Stuck jobs** rule in [SPEC Transcode](docs/SPEC.md#transcode)). The usual causes, such
as a callback secret mismatch or an unreachable callback URL, show in the repository's Actions
tab, and a rejected callback also in the Worker log. So the admin:

1. Once, turns on GitHub's notifications for failed workflow runs (GitHub → Settings →
   Notifications → System → Actions → "Only notify for failed workflows"). GitHub notifies
   the account that started the run, which for a Worker dispatch is the owner of
   `GITHUB_DISPATCH_TOKEN`, so that account must be the admin's. This covers callback and
   step failures; a run that timed out may show as cancelled, and one that never started
   sends nothing.
2. When a member says a Retry failed again, searches the Actions tab for the Waffle's ID (the
   last part of its `/w/{id}` link; each run's title is `transcode <id> (<env>)`), opens the
   latest failed or cancelled run and reads its last line. **Failure output** and **Callback**
   in [SPEC Transcode](docs/SPEC.md#transcode) say what each line means. The admin fixes
   what the line names, for example:
   - `callback: 401 <id>`: make the two copies of the secret match ([Rotating the callback
     secret](#rotating-the-callback-secret)), or fix a runner clock more than 5 minutes off.
   - `callback: retries exhausted ...`: check `CALLBACK_BASE_URL` and the Worker log. If that
     log shows `callback rejected: no secret`, set the Worker's secret with step 3 of
     [Rotating the callback secret](#rotating-the-callback-secret), using the repository
     secret's value.
   - `callback: 404 <id>` for a Waffle that still exists: point `CALLBACK_BASE_URL` at the
     deployed Worker's origin.
   - A failed secret check before the download (**Dispatch target** in SPEC Transcode): set
     the repository secret it names.
   - No run names the ID: the run never started; check
     [GitHub Status](https://www.githubstatus.com) for Actions.

   The uploader then presses Retry.
