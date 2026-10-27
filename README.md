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

`GITHUB_DISPATCH_TOKEN` expires. The Worker's daily token check warns in the Discord channel
before it does, or once GitHub refuses it (see [SPEC Transcode](docs/SPEC.md#transcode)).
The admin then:

1. In GitHub, regenerate the fine-grained token (Settings → Developer settings → Personal
   access tokens → Fine-grained tokens) with the repository access, permission and expiry
   SPEC Transcode names.
2. Run `npx wrangler secret put GITHUB_DISPATCH_TOKEN` and paste the new token at the prompt.
   This updates the deployed Worker (prototype and production are the same Worker); local dev
   keeps its own copy in `.dev.vars`.
   Never paste it anywhere else: not in the repo, an issue, a workflow log or Discord.
3. Tell members whose Waffles failed with "its access token may have expired" to press Retry.

The next daily check stays quiet once the new token is in place.

If members see "The daily GitHub token check has failed for 3 days" on the This-week screen,
or the channel gets that message, read the `token check` lines in the Worker's logs
(Cloudflare dashboard → the Worker → Logs) and fix what they name; the banner goes at the next
passing check (see [SPEC Transcode](docs/SPEC.md#transcode)).
