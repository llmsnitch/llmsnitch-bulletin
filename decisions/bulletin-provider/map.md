# Bulletin Provider — Wayfinder Map

`label: wayfinder:map`
`status: UNDERWAY (2026-09-19) — frontier T804 (activate; needs merge + pipx reinstall)`

## Destination

**Fresh bulletins arrive on this machine without a human running
anything.** A public provider builds a spec-v1 PyPI **Bulletin** every day
from OSV + KEV + EPSS and publishes it at one stable HTTPS URL with its
`.sha256` sidecar; the CLI pulls it weekly as a `curl` subprocess riding
the daily patrol (ADR 0001), hash-verifies, and swaps the cache
atomically. The map closes when [T804](tickets/T804-activate.md)'s done
criteria hold: two consecutive scheduled builds published, one automated
client refresh observed (`bulletin_age_days` drops without a manual run),
and `tools/distill_bulletin.py` deleted from this repo.

## Notes

- **Tracker location**: `wayfinder/bulletin-provider/` on branch
  `feature/bulletin-provider` in the worktree
  `~/.supacode/repos/llmsnitch-cli/feature/bulletin-provider` — commit
  here; merging to `main` (canonical, `~/llmsnitch-cli`) is the user's
  call. Provider *code* lives in a separate public repo under the `llmsnitch`
  GitHub account (name: user's call at T802, default `llmsnitch/bulletin`);
  this repo stays local-only and never gains a remote.
- **Grounding**: `docs/research/bulletin-provider-hosting.md` (options
  A–G, GitHub limits, licensing, threat notes), `docs/bulletin-spec.md`
  (contract), `docs/adr/0001-bulletin-egress.md` (one URL, weekly,
  pull-only, subprocess), dep-audit map T506/T508 (distiller, cache path).
- **Plan-vs-do override**: T801–T804 carry execution — the destination is
  an observed refresh, not a spec.
- **Local mode (D07, 2026-09-19)**: the provider is a directory on this
  Mac, not a GitHub repo; the daily build runs inside the weekly patrol
  refresh (~25 s, ~40 MB down). Notes above about the `llmsnitch` account
  and the public repo describe the parked remote path.
- **Credentials boundary**: the `llmsnitch` account's credentials are not
  on this machine. T801 produces the repo contents locally; creating the
  repo and pushing is the user's action. Nothing here pushes.
- **D08 (2026-09-26)**: the provider dir is its own local git repo —
  `git init` at `~/Repos/llmsnitch-bulletin` (initial commit `45107f4`),
  repo-local identity `llmsnitch <llmsnitch@users.noreply.github.com>` so
  no real identity enters history that T802 might someday publish. No
  remote. Rationale: after T804 deletes `tools/distill_bulletin.py` here,
  the provider dir holds the only copy of the distiller — it needs version
  control. T801's "no git init" stands as history; this supersedes it.
- **Repo policy** (CLI side): local-only git; `trash`, never `rm`; never
  `install.sh`; stdlib only, Python 3.9+; no network imports
  (test-enforced) — the fetch is a `curl` subprocess; files 0600 / dirs
  0700; `python3 tests/all.py` is the gate. Closed maps and tickets
  (dep-audit T506/T508) are history — never edited; this map records the
  supersession.
- **Vocabulary**: **Bulletin** (CONTEXT.md) — its definition already says
  "a provider publishes and the CLI consumes … refreshed weekly". No new
  terms.

## Decisions so far

<!-- one line per closed ticket: gist + link -->

- [T801](tickets/T801-provider-repo.md) (2026-09-19): provider contents
  assembled at `~/Repos/llmsnitch-bulletin` — verbatim distiller, workflow
  (`actions/checkout` pinned `3d3c42e5…` = v7.0.1), README/attribution,
  MIT, seeded `LAST_BUILD`, `.gitignore`; hand build 19,091 entries /
  1.16 MB gz / 19.9 s, sidecar matches, `bulletin.load` → `None`. Not
  pushed; no git init (identity boundary).
- [T805](tickets/T805-local-build.md) (2026-09-19): **D07 local provider**
  — user decision "keep it local": no public repo, no Actions. The provider
  is `~/Repos/llmsnitch-bulletin` (T801's contents); the patrol runs
  `uv run distill_bulletin.py --fresh` there and `refresh()` pulls
  `dist/` over `file://`. Supersedes D01–D04's GitHub half (rolling
  release, cron, keepalive, account 2FA); D05 (distiller ownership moves
  to the provider dir) and D06 (client shape) stand. `LLMSNITCH_BULLETIN_BUILD`
  is the second env seam. Live: 25 s, 19,092 entries. T802 parked.
- [T803](tickets/T803-client-refresh.md) (2026-09-19): `bulletin.refresh`
  (curl subprocess → temps → `load()` verifies → `os.replace`),
  `needs_refresh()` at `REFRESH_DAYS = 7`, patrol wiring before
  `cmd_depaudit`, `llmsnitch bulletin refresh`; 7 new tests, gate 211/211,
  e2e against T801's dist via `file://` in 1.7 s. `BULLETIN_URL` assumes
  repo `llmsnitch/bulletin`; T802 edits that one line if the name differs.
- Charting (2026-09-19, research note + cold-read/ponytail pass):
  **D01 host** = GitHub Actions cron → **one rolling GitHub Release**
  (tag `bulletin`, assets replaced with `gh release upload --clobber`),
  client URL `https://github.com/llmsnitch/<repo>/releases/download/bulletin/pypi.json.gz`
  (+ `.sha256`). Pages, cloud buckets, VPS, local-only rejected — table in
  the research note. Per-build snapshot releases considered and dropped:
  no stated need; add when a bulletin-to-bulletin diff is actually wanted.
  **D02 cadence** = provider builds daily at 14:17 UTC (after EPSS
  13:30 UTC, off the top of the hour). Client refreshes from the patrol
  (`scan --patrol`, scan.py's patrol-only branch, before `cmd_depaudit`)
  when the cache is missing, unloadable, or `bulletin.age_days(meta) ≥ 7`
  (payload age from `meta.issued_at`, the same clock the audit stamps);
  manual `llmsnitch depaudit` never fetches; no new LaunchAgent.
  **D03 keepalive** = the workflow commits a one-line `LAST_BUILD` stamp
  each run so the public-repo 60-day-inactivity rule can never disable the
  cron (whether release uploads alone count is unverified; a commit
  unambiguously does).
  **D04 authenticity** = TLS + account 2FA in v1; Sigstore attestation
  emitted at build time is v2 fog, client verification stays out until a
  stdlib-shaped path exists.
  **D05 distiller ownership** = the script is copied verbatim (zero
  edits) to the provider repo; this repo deletes
  `tools/distill_bulletin.py` at T804 (no copies, no `// legacy`).
  **D06 client shape** = fetch lives in `llmsnitch/bulletin.py` (the
  module already owns the cache path and verification); URL is a module
  constant with `LLMSNITCH_BULLETIN_URL` as the env seam (the repo's
  `LLMSNITCH_*` pattern — no config section, no flags, no `refresh_days`
  knob); a failed refresh writes one `[WARN]` line to the patrol output
  and the audit proceeds on the old cache — persistent failure already
  surfaces as `bulletin stale` in the digest at 14 days.

## Frontier

- [T804 — Activate](tickets/T804-activate.md) — merge, `pipx install
  --force`, observe one unattended build + refresh, delete
  `tools/distill_bulletin.py` here.
- [T802 — First publish](tickets/T802-first-publish.md) — PARKED (D07).

## Not yet specified

- **Bulletin authenticity (v2)** — `actions/attest-build-provenance` on the
  provider is one step; the client-side verify (`gh attestation verify`
  subprocess when `gh` exists, else skip and stamp) waits for a decision
  on whether a non-stdlib binary may be *optionally* used. Inherits the
  dep-audit map's fog item.
- **Bulletin history** — a rolling release keeps only the newest file. If
  debugging a waiver re-raise ever needs the previous bulletin, the lazy
  answer is the client keeping one `.prev` pair, not provider snapshots.
- **Non-PyPI ecosystems** — the workflow gains a matrix over
  `PyPI, npm, …` and a file per ecosystem; the spec already allows it. Waits
  on the client's npm intake grammar (dep-audit fog).
- **Build failure visibility** — a failed Actions run is invisible here
  until the digest says `bulletin stale` (14 days). Acceptable for v1; a
  tighter signal (client compares `meta.issued_at` age against the daily
  cadence) is a digest change, not a provider one.

## Out of scope

- **Package-scoped or per-client bulletins** — the intake list never leaves
  the machine (spec).
- **A query API** — the product is a file; anything request/response is
  ADR 0001's rejected path.
- **Publishing the CLI repo** — the provider repo is public because
  unauthenticated `curl` needs it; the CLI repo's local-only rule is
  untouched.
- **Conditional GET / ETag** — 1 MB once a week is not worth the code.
