# CLAUDE.md — brown-symphony-waffles

Waffles: a private web app where four friends upload weekly phone videos to Cloudflare R2,
get a near-original-quality copy every device can play, and see a link posted in Discord.

<!-- Project-specific rules only. Global rules (secrets, branch guard, sandbox, env-var
parsing) come from ~/.claude/CLAUDE.md and are not repeated here. -->

## Where things live

| Question | Document |
|---|---|
| How do I run and test it? | `README.md` |
| What rules must an agent follow here? | this file |
| Where is the project going, and why in this order? | `plans/ROADMAP.md` |
| What is the next feature, and what does the code look like now? | `plans/META-PLAN.md` |
| How is a given feature being built? | `plans/<topic>/<slug>.md` |
| What exactly should the MVP do (data model, API, pipeline, UI)? | `docs/SPEC.md` |

Each fact has one home. Link to it from elsewhere; never copy it.

## Stack

- TypeScript throughout.
- SvelteKit with `@sveltejs/adapter-cloudflare`: UI and API in one Cloudflare Worker.
- D1 for metadata, via Drizzle ORM and its migrations.
- R2 for every video (private buckets `waffles` and `waffles-dev`), presigned with `aws4fetch`.
- Transcoding in a GitHub Actions workflow on `ubuntu-latest` with ffprobe/ffmpeg, started
  by `repository_dispatch`; it reports back through an HMAC-signed callback.
- Discord OAuth for sign-in, a Discord channel webhook for notifications.
- Vitest for unit tests.

## Folder layout

<!-- Fill in when Phase 0 creates the SvelteKit app. -->

_Not created yet._

## Commands

<!-- Fill in during Phase 0. Keep them in sync with .claude/skill-config.md's test fields. -->

- `dev`: _not set up yet_ (`wrangler dev` with a local D1 and the `waffles-dev` bucket)
- `test`: _not set up yet_ (Vitest)
- `deploy`: _not set up yet_
- `db:migrate`: _not set up yet_ (local and remote)

## Project rules

- **Waffle week.** A week runs Wednesday 00:00 to Tuesday 23:59 in `GROUP_TZ`
  (`America/New_York`). `week_start` is that Wednesday's date (`YYYY-MM-DD`), computed from
  `uploaded_at` when the upload starts; the upload form may file to the previous week. A
  heatmap year is every week whose Wednesday falls in that calendar year (52 or 53). Tests
  must cover Tuesday 23:59 vs Wednesday 00:00 ET and both daylight-saving changes.
- **The repo and its Actions logs are public.** Workflow logs carry Waffle IDs only, never
  names, captions or URLs. Every credential lives in Worker secrets (`wrangler secret put`)
  or GitHub repository secrets. Videos and member data live only in R2 and D1, never in the
  repo, its fixtures or its issues.
- **Video bytes never pass through the Worker.** Browsers upload to R2 with presigned part
  URLs and play from presigned GET URLs.
- **The original is never modified.** Every Waffle keeps its untouched original; the playback
  copy is derived and can be regenerated.
- Local secrets go in `.dev.vars`, which is gitignored and must never be read.
