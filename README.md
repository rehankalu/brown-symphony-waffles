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

### Rotating the GitHub dispatch token

`GITHUB_DISPATCH_TOKEN` expires. The Worker's daily token expiry check warns in the Discord channel
before it does, or once GitHub refuses it (see [SPEC Transcode](docs/SPEC.md#transcode)).
The admin then:

1. In GitHub, regenerate the fine-grained token (Settings → Developer settings → Personal
   access tokens → Fine-grained tokens) with the repository access, permission and expiry
   SPEC Transcode names.
2. Run `npx wrangler secret put GITHUB_DISPATCH_TOKEN` and paste the new token at the prompt.
   This updates the deployed Worker (prototype and production are the same Worker).
   Then replace the token in `.dev.vars` too, since local dev keeps its own copy of it, and
   get it to every other member with a local copy in person or through a password manager.
   Never paste it anywhere else: not in the repo, an issue, a workflow log or Discord.
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
token (rotate it with the steps above), `DISCORD_CLIENT_SECRET` and `DISCORD_WEBHOOK_URL`:
reset those in the Discord developer portal and the channel's webhook settings, then replace
them with `npx wrangler secret put` and in every `.dev.vars`. The admin:

1. In the Cloudflare dashboard (R2 → Manage API tokens), creates a new token with the
   permission and buckets [SPEC Environments](docs/SPEC.md#environments) names.
2. Runs `npx wrangler secret put R2_ACCESS_KEY_ID` and `npx wrangler secret put
   R2_SECRET_ACCESS_KEY`, pasting the new values at the prompts.
3. Replaces both repository secrets of the same names (GitHub → the repository → Settings →
   Secrets and variables → Actions).
4. Replaces both values in `.dev.vars`, and gets the new values to every other member with a
   local copy in person or through a password manager, never through Discord.
5. Once no Waffle is `processing`, deletes the old token. A run that started before step 3
   still holds the old keys, so deleting it earlier fails that run's uploads. Deleting it also
   ends every URL the Worker presigned with it: an upload in progress fails until the member
   presses Resume or Try again, and an open video until the page is reloaded, since both then
   get freshly signed URLs. If the old token leaked, delete it right after step 4 without
   waiting, have the uploaders of any failed Waffles press Retry, and check the `waffles`
   bucket's originals against D1 for anything deleted (R2 keeps no old versions).

Never paste the keys anywhere else: not in the repo, an issue, a workflow log or Discord. The
members' read-only NAS tokens (see [SPEC Follow-on: NAS backup](docs/SPEC.md#follow-on-nas-backup))
are separate and are not affected.

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

Never paste it anywhere else: not in the repo, an issue, a workflow log or Discord.

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
   last part of its `/w/{id}` link), since each run's title is `transcode <id> (<env>)`, opens
   the latest failed or cancelled one, sees which step failed, and fixes the cause. A
   `callback:` line names a callback failure (**Callback** in [SPEC
   Transcode](docs/SPEC.md#transcode)). `callback: 401 <id>` is a rejected signature, usually a
   secret mismatch (fix: [Rotating the callback secret](#rotating-the-callback-secret)) or a
   runner clock more than 5 minutes off; `callback rejected: 401` lines in the Worker log
   support it, though any unsigned request writes them too. `callback: retries exhausted ...`
   means the Worker was unreachable or kept failing, so check `CALLBACK_BASE_URL` and the
   Worker log. `callback: 404 <id>` for a deleted Waffle is the Worker's `no_waffle` 404, after
   which the run deleted its playback copy and poster. For a Waffle that still exists it is
   a 404 without that marker, so `CALLBACK_BASE_URL` points at the wrong Worker or host, and
   the run deleted nothing. If no run names that ID, the run never
   started; check [GitHub Status](https://www.githubstatus.com) for Actions. The uploader then
   presses Retry.
