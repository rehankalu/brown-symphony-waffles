# META-PLAN — brown-symphony-waffles feature rollout

> **What this file is.** The execution manifest: F-IDs, order, ★FOCUS, status, the
> code-level Baseline, Standing invariants, per-feature contracts and couplings. Each
> feature runs as its own session via `/run-feature`; features enter only via
> `/new-feature`. The git repository, not conversation memory, carries state between
> sessions.
>
> **What it does not own.** End state, capabilities and phases: [`ROADMAP.md`](ROADMAP.md)
> (rows cite a phase; this file never restates the vision). Implementation steps: each
> feature's plan file. Run/test commands: `README.md`.

---

## Project facts

<!-- Per-project values the global workflow skills read instead of hardcoding. Keep the
Fact names exactly as written; use `none` for an optional fact that does not apply.
Machine-local values go in `.claude/skill-config.md`.
Every row ships with a value, never bare template text. Merge strategy, Worktree prefix,
Install command and Port caveats default to the common small-project values below; change
one only when the project differs (`merge`/`rebase`/`n/a` with no PRs; a prefix when features
run in sibling worktrees, which then usually need an Install command and Port caveats).
Verification is stack-specific, e.g. Python venv `cd backend && .venv/bin/pytest`, Node
`npm test`, both `cd backend && .venv/bin/pytest && cd .. && npm test`. Until a test runner
exists, write `pending — set by F<n>` (or `pending — set by the first feature with tests`);
/run-feature stops on a `pending` Verification exactly as on a missing one.
With `Merge strategy` `n/a` (no PRs), `/run-feature` runs its no-PR mode and also requires
`Branch`, `Merge gate` and `Push`, and reads an optional `Backup before edit` (unset means git
is the backup). Add those rows (prose instructions, followed as written); `PR link form`,
`Worktree prefix` and `Port caveats` are then unused. -->

| Fact | Value |
|---|---|
| Verification | pending — set by F1 |
| PR link form | `[#N](https://github.com/rehankalu/brown-symphony-waffles/pull/N)` |
| Merge strategy | `squash` |
| Worktree prefix | `none` |
| Install command | `none` |
| Port caveats | `none` |

---

## Where the work stands

**★FOCUS: F1 — app scaffold and Worker entry.** Phase 0 (Scaffold) is the only runnable
phase: F1 has no prerequisites; F2 (D1) and F3 (R2) each wait only on F1 and can then run in
either order.

<!-- Two or three sentences at most: what is runnable today, what is gated and on what.
Shipped work is git's record (`gh pr list --state merged`), not this file's. -->

### Shortest path to the focus feature

★F1 is runnable now (zero prerequisites).

---

## Feature list (intake record)

> Written only by `/new-feature`; `/run-feature` removes a line at fold-back.

**Next F-ID:** `F4`

- [ ] **F1** (added 2026-11-02): Phase 0 scaffold: SvelteKit app served through the custom Worker entry (`src/worker.ts`), Wrangler config with `main`, the assets block, `[build] command` and `[observability] enabled = true`, Vitest, deployed to workers.dev.
- [ ] **F2** (added 2026-11-02): Phase 0 D1: the database with the SPEC schema as Drizzle migrations, and `db:migrate` running locally and remotely.
- [ ] **F3** (added 2026-11-02): Phase 0 R2: buckets `waffles` and `waffles-dev` with their CORS and lifecycle rules, the R2 API token, and the Worker secrets `R2_ACCESS_KEY_ID`, `R2_SECRET_ACCESS_KEY` and `R2_ACCOUNT_ID`.

---

## Backlog summary (remaining work only)

| Order | Feature | Effort | Depends on | Phase |
|---|---|---|---|---|
| **1** | **★ F1 — App scaffold and Worker entry** | **M** | — | **0** |
| 2 | F2 — D1 schema and migrations | M | F1 | 0 |
| 3 | F3 — R2 buckets, CORS, lifecycle and token | S | F1 | 0 |

<!-- *Depends on* holds dependency edges in the grammar of
/home/rmila/Code/_STRONGHOLD/plans/roadmap.md §3.1 *Dependency edges and `NEXT.md`*, which is
the authority. Address: `<repo>#F<n>` or `<repo>#P<n>`. `<repo>` is the directory name under
`/home/rmila/Code` (`stronghold` for the stronghold); a bare `F<n>`/`P<n>` means this repo;
`P2.5` addresses an inserted phase. `P<n>` exists only for a repo whose ROADMAP has
`### Phase N:` headings (or, with no ROADMAP, a `STATUS.md` with `## Phase N` headings), never
for `stronghold`. Kinds: a bare ref blocks; `soft:` enables (never gates); `couples:` marks a
shared contract (never gates); `cond: <text>` is a manual gate with no address yet; `—` means
none. Parse rules: split the cell on `;` into groups, outside parentheses; a kind prefix at the
start of a group applies to every `,`-separated ref in that group (a prefix anywhere else is
not in the grammar); a `cond:` group is never split on `,`: its whole text, to the next `;`
outside parentheses or the cell end, is one gate; a `(note)` after a ref is ignored by readers
and never split (an IDEAS id goes in a note, e.g. `soft: meals4irl#F3 (X-004)`). Prefixes are
lowercase and exact, never bolded; an unbalanced `(` is reported, not guessed. A ref to a
shipped (folded-back) F-ID stays valid; whether it is done follows the §3.1 done rule. Only the
waiting feature declares an edge. `/new-feature` writes and checks refs.
A legacy `external:` or `**external:**` at the start of a `;` group reads as `cond:`
for that group until backfilled (`F3, F4; external: canary repo` = hard F3, F4 plus a manual
gate), unsplit like `cond:`; any other text not in the grammar is reported, never guessed. A
superseded row (Feature cell struck through with `~~`, *Depends on* starting `**superseded`)
holds its supersession note, not edges; readers skip it. -->

**Effort tally (remaining).** 5 pts (S=1 / M=2 / L=3 / XL=5).

---

## Status ledger

> Maintained by `/run-feature` (`pending` → `in-progress` → `in-review`; a verified-merged
> row is deleted at fold-back). Never hand-delete a row. Git wins any conflict.

| Feature | Status | Branch | PR |
|---|---|---|---|
| **F1** | pending | `feature/app-scaffold` | — |
| F2 | pending | `feature/d1-schema` | — |
| F3 | pending | `feature/r2-buckets` | — |

---

## Baseline: the codebase as it exists today (`main` at PR #1, `e218f64`)

> The Assumed starting state for every remaining feature unless its section says
> otherwise. Describe what *is*, never what changed.

**Stack.** Chosen, not yet installed: see [`CLAUDE.md` § Stack](../CLAUDE.md#stack). No
code exists yet: `main` holds only docs (`README.md`, `CLAUDE.md`, `docs/SPEC.md`,
`plans/`), `.gitignore` and `.claude/settings.json` (sandbox allowing `registry.npmjs.org`,
`api.cloudflare.com`, `dash.cloudflare.com`, `discord.com`, `github.com`, `api.github.com`).
There is no `package.json`, no Wrangler config and no `src/`. The MVP contract is
[`docs/SPEC.md`](../docs/SPEC.md); CLAUDE.md's *Folder layout* and *Commands* still read
"not set up yet".

**Domain model.** None yet.

**Interfaces.** None yet.

**Key modules / UI.** None yet.

**Tests & CI.** None yet.

**Deployment.** None yet.

### Standing invariants (must not regress)

<!-- Number once, never renumber; strike through a retired one and say where it went. -->

None yet.

### Assumptions to revisit at planning time

None yet.

---

## Per-feature session contract

Run `/run-feature <F-ID>`; it owns the contract (cold survey → plan → review → implement
→ commit → verify → PR → fold-back). With `Merge strategy` `n/a`, run the same skills by
hand: `/plan-creator`, `/plan-reviewer`, `/run-plan`, `/git-commit`, then verify.
Parallel features: `/worktree <F-ID>...`.

---

# COMPLETED-FEATURE CONTRACTS STILL IN FORCE

> Only a merged feature's contract that a **remaining** feature builds on, edits or must
> remove.

None yet.

## Deferred follow-ups (confirmed, unscheduled, no F-ID yet)

<!-- Promote an item with `/new-feature`. -->

None yet.

---

# REMAINING FEATURES

## F1 — App scaffold and Worker entry · Phase 0 · Effort M · ★FOCUS · (added 2026-11-02)

**Goal.** An empty SvelteKit app deployed to `workers.dev` through the custom Worker entry,
with Workers Logs kept and a test runner in place
([Feature list](#feature-list-intake-record) F1; ROADMAP Phase 0, first bullet).

**Rank rationale.** First: every later feature, F2 and F3 included, deploys to this Worker
and runs its tests with this runner.

**Dependencies.** None.

**Sizing.** unscored (intake estimate M); split check pending.

**Assumed starting state** = **Baseline**. Verify:
- `ls package.json wrangler.toml wrangler.jsonc src 2>&1` finds none of them.
- `grep -ni 'not set up yet' CLAUDE.md README.md` hits CLAUDE.md *Commands* and README
  *Getting started* / *Tests*.

**Expected end state** (repo-checkable):
- SvelteKit with `@sveltejs/adapter-cloudflare`, TypeScript, npm; `package.json` scripts
  include `dev`, `build` (`vite build`, never calling Wrangler), `check` (`svelte-check`),
  `test` (Vitest) and `deploy`.
- `src/worker.ts` exports an object whose `fetch` passes `request`, `env` and `ctx` to the
  adapter output unchanged and holds no other logic; `src/worker-adapter.d.ts` declares the
  build output's default export, so `npm run check` passes on a fresh checkout without a build
  (SPEC Architecture, **Worker entry**).
- The Wrangler config sets `main` to `src/worker.ts`, the adapter's assets block with
  `binding = "ASSETS"`, a `[build] command` running `npm run build`, and
  `[observability] enabled = true`.
- At least one Vitest test passes with `npm test`, and no test imports `src/worker.ts`.
- META-PLAN's `Verification` fact is set (e.g. `npm run check && npm test`).
- CLAUDE.md *Folder layout* and *Commands* (`dev`, `test`, `deploy`) describe what now
  exists (SPEC Build plan asks for this; `db:migrate` stays a placeholder until F2), and
  README *Getting started* / *Tests* replace their `_Not set up yet._` placeholders with the
  working install, run and test steps (`grep -ni 'not set up yet' README.md` finds nothing).
- Manual, recorded in the plan's verification log: the deployed app on
  `waffles.<account>.workers.dev` returns a page, and Workers Logs shows that request after it
  finishes (ROADMAP Phase 0 exit criteria, first half).

**Test-suite deltas.** Adds Vitest and its first (smoke) test.

**Open risks / decisions.** (a) Wrangler config format, `wrangler.toml` or `wrangler.jsonc`:
SPEC and ROADMAP use TOML section names; recommend `wrangler.toml`. (b) Whether
`src/worker.ts` gets a `scheduled` handler now: recommend leaving it out until Phase 3's
token expiry check, with no Cron Trigger yet. (c) Deploying needs Cloudflare auth on this
machine: the admin runs `! npx wrangler login` (or supplies an API token through the
environment); the agent never reads or stores credentials. (d) Whether the Worker is named
`waffles` (SPEC's `waffles.<account>.workers.dev`): recommend yes. (e) Vitest setup: plain
Node environment, or `@cloudflare/vitest-pool-workers`: recommend plain Node until a feature
needs bindings in tests (F2 may revisit).

**Session.** `/run-feature F1` on branch `feature/app-scaffold`; plan at
`plans/feature/app-scaffold/app-scaffold.md`.

---

## F2 — D1 schema and migrations · Phase 0 · Effort M · (added 2026-11-02)

**Goal.** The D1 database with SPEC's three tables, managed by Drizzle migrations, and a
`db:migrate` command that applies them locally and remotely
([Feature list](#feature-list-intake-record) F2; ROADMAP Phase 0, second bullet).

**Rank rationale.** Phase 1 (sessions, `members`) and every later phase read this schema.
Independent of F3; ordered first because Phase 1 needs D1 but not R2.

**Dependencies.** F1 (the Worker, Wrangler config and test runner it extends).

**Sizing.** unscored (intake estimate M); split check pending.

**Assumed starting state** = **Baseline** plus F1's Expected end state. Verify:
- `npm test` passes; the Wrangler config has no `d1_databases` entry.

**Expected end state** (repo-checkable):
- A Drizzle schema module defines `members`, `waffles` and `sessions` with the columns,
  types, nullability, defaults, references and the indexes `waffles_week` and
  `waffles_member` exactly as SPEC *Data model* gives them.
- Generated SQL migrations are committed, and the Wrangler config binds the D1 database
  (with `migrations_dir` pointing at them).
- `package.json` has `db:migrate` for local and remote (e.g. `db:migrate:local` /
  `db:migrate:remote` wrapping `wrangler d1 migrations apply`); CLAUDE.md *Commands* lists
  them.
- A test applies the migrations to a fresh database and checks the three tables and two
  indexes exist with SPEC's columns.
- Manual, recorded in the plan's verification log: `db:migrate` runs locally and against
  the remote D1 (ROADMAP Phase 0 exit criteria).

**Test-suite deltas.** Adds a migration/schema test.

**Open risks / decisions.** (a) How the schema test gets a database: Miniflare/D1 through
`@cloudflare/vitest-pool-workers`, or `better-sqlite3` in Node: recommend whichever F1's
Vitest setup makes cheapest, keeping tests free of the build output. (b) Database name and
binding: recommend database `waffles`, binding `DB`. (c) No seed rows here: the four members
are seeded in Phase 1. (d) Remote apply needs the admin's Wrangler auth, as F1 (c).

**Session.** `/run-feature F2` on branch `feature/d1-schema`; plan at
`plans/feature/d1-schema/d1-schema.md`.

---

## F3 — R2 buckets, CORS, lifecycle and token · Phase 0 · Effort S · (added 2026-11-02)

**Goal.** Private buckets `waffles` and `waffles-dev`, each with its CORS and lifecycle
rule, and the R2 API token whose keys the Worker holds as secrets
([Feature list](#feature-list-intake-record) F3; ROADMAP Phase 0, third bullet).

**Rank rationale.** Phase 2 (Upload) is the first reader; it can run in parallel with F2.
Needs F1 first because the `waffles` CORS rule names the deployed workers.dev origin and the
secrets go on the deployed Worker.

**Dependencies.** F1 (the deployed Worker and its workers.dev origin).

**Sizing.** unscored (intake estimate S); split check pending.

**Assumed starting state** = **Baseline** plus F1's Expected end state. Verify:
- The Wrangler config exists and the Worker is deployed on workers.dev.

**Expected end state** (repo-checkable):
- The CORS rule for each bucket is committed as a file and applied with Wrangler:
  `AllowedOrigins` the workers.dev origin on `waffles` and the local `APP_ORIGIN` (e.g.
  `http://localhost:8787`) on `waffles-dev`, `AllowedMethods: PUT`,
  `AllowedHeaders: content-type`, `ExposeHeaders: ETag` (SPEC Upload).
- Each bucket has a lifecycle rule aborting multipart uploads left incomplete for 7 days
  (SPEC Storage layout).
- README *Operations* says how to re-apply the CORS and lifecycle rules and read them back.
- Manual, recorded in the plan's verification log (names and readback only, never a
  value): the R2 API token exists with Object Read & Write on only the two buckets; the
  Worker secrets `R2_ACCESS_KEY_ID`, `R2_SECRET_ACCESS_KEY` and `R2_ACCOUNT_ID` are set
  (`wrangler secret list`); `.dev.vars` holds them; and a Wrangler (or dashboard) readback of
  each bucket's CORS and lifecycle rule matches the ROADMAP Phase 0 exit criteria.

**Test-suite deltas.** None expected (no Worker code reads R2 until Phase 2).

**Open risks / decisions.** (a) The token is created in the Cloudflare dashboard by the
admin, who pastes each value at a `! npx wrangler secret put` prompt and into `.dev.vars`
themselves; the agent never sees, reads or echoes a key, and never reads `.dev.vars`.
(b) Documenting the `.dev.vars` names: `.gitignore` ignores `.dev.vars.*`, so a
`.dev.vars.example` would not be committed: recommend a names-only table in README *Getting
started* instead. (c) Bucket creation and rule changes are outward-facing account changes:
confirm with the user before each `wrangler r2` write. (d) The local `APP_ORIGIN` port must
match `wrangler dev`'s exactly (SPEC Environments).

**Session.** `/run-feature F3` on branch `feature/r2-buckets`; plan at
`plans/feature/r2-buckets/r2-buckets.md`.

---

## Dependencies & couplings (remaining work)

```
Phase 0:  ★ F1 ──┬──► F2   (D1)
                 └──► F3   (R2)
```

- **Hard chains:** F1 → F2; F1 → F3.
- **Soft preferences:** F2 before F3, since Phase 1 needs D1 but not R2; they share no files
  beyond the Wrangler config and can run in either order.
- **Couplings to honor:** F2 and F3 both edit the Wrangler config and CLAUDE.md *Commands* /
  README; whichever merges second rebases on the first's edits rather than overwriting them.
  Phase 0 closes (ROADMAP status flip) when the last of F1–F3 folds back.

---

## History policy

Forward-looking: merged work is pruned by `/run-feature`'s fold-back and `/new-feature`'s
reconcile; git is the history.
