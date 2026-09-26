# Decision history

This folder is a snapshot of the design/decision trail from when this
provider was still developed inside the `llmsnitch-cli` repo, on branch
`feature/bulletin-provider` (wayfinder map `wayfinder/bulletin-provider/`,
decisions D01-D08, tickets T801-T805). Copied here as flat files —
historical reference, not a live tracker; nothing in `bulletin-provider/`
below is rewritten going forward.

`research-hosting-options.md` is the options survey (A-G) that led to
D01 (GitHub Actions cron + one rolling Release).

## Superseded (2026-09-26)

D07/D08 — the "local mode" pivot (patrol runs the distiller locally via
`~/Repos/llmsnitch-bulletin`, `bulletin.refresh()` pulls `dist/` over
`file://`, this repo gets its own git identity to protect the only copy
of the distiller) — is **superseded**. The user decided this project
forks out of `llmsnitch-cli` and continues independently instead:

- `llmsnitch-cli`'s `main` now carries only a slim client: `bulletin.py`'s
  `refresh()`/`needs_refresh()` fetch a URL via `curl`, verify, and swap
  atomically — no local-build coupling, no `PROVIDER`/`BUILD_CMD`. It
  defaults `BULLETIN_URL` back to the original D01 plan:
  `github.com/llmsnitch/bulletin/releases/download/bulletin/pypi.json.gz`
  (`LLMSNITCH_BULLETIN_URL` overrides).
- The `feature/bulletin-provider` branch and its wayfinder tracker were
  retired — this `decisions/` copy and `../SPEC.md` are what's left of
  that history.
- Practically: **D01-D06's original GitHub-hosted plan is back in play**
  — this project needs its own repo (or another real, independently
  scheduled host) and a stable published URL. `README.md`'s
  `github.com/llmsnitch/bulletin` URLs were always the target, not a stale
  aspiration to "not fix" — the local `file://` detour is what's gone.
