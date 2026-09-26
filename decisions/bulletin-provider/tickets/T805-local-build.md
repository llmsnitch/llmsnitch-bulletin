# T805 — Local build pipeline

`labels: wayfinder:task`
`parent: ../map.md`
`blocked by: T801, T803`
`blocks: T804`
`status: DONE (2026-09-19)`

## Question

User decision 2026-09-19: **keep it local** — no public repo, no GitHub
Actions (research note option F). The provider is the directory
`~/Repos/llmsnitch-bulletin` (T801's contents, unpushed); the build runs
on this Mac from the patrol, and T803's refresh pulls the result over
`file://`. Nothing about the CLI's shape changes: build + fetch are both
subprocesses, the package stays stdlib and import-clean.

## Resolution

In `llmsnitch/bulletin.py`:

- `PROVIDER = ~/Repos/llmsnitch-bulletin`; `BUILD_CMD = [uv, run,
  <PROVIDER>/distill_bulletin.py, --fresh, --workdir <PROVIDER>/work,
  --out <PROVIDER>/dist]` (`uv` via `shutil.which` or `~/.local/bin/uv`
  — launchd's PATH has neither; `--fresh` because the distiller reuses
  cached feeds otherwise, which on a persistent machine means forever);
  `BULLETIN_URL = file://<PROVIDER>/dist/pypi.json.gz`.
- `_build(out)`: runs `BUILD_CMD` (15 min timeout), `True` or one
  `[WARN] bulletin build failed: <rc N | ExceptionName>` line. Env seam
  `LLMSNITCH_BULLETIN_BUILD` = full command string; `""` skips the build
  (remote-provider mode, and every test).
- `refresh(out)` calls `_build` first; on failure returns 2 without
  fetching. `llmsnitch bulletin refresh` and the patrol therefore mean
  "build, then install" — one verb, one path.
- Tests: fake distiller (stdlib script writing a valid pair) proves
  build → file:// pull; build rc 3 and a missing binary each give 2 and
  leave the old cache; `BUILD_CMD` shape pinned. `with_tmp` and the
  bulletin test store both pin `LLMSNITCH_BULLETIN_BUILD=""`. Gate
  214/214.
- Live run with `env -i HOME PATH=/usr/bin:/bin` and a fresh
  `LLMSNITCH_DIR`: rc 0 in 25.2 s wall, cache `cmp`-identical to
  `dist/`, 19,092 entries, `issued_at 2026-09-20T00:43:41Z`.

The scratch dir keeps its workflow/README/LICENSE untouched: switching
back to a remote provider is `LLMSNITCH_BULLETIN_BUILD=""` plus an https
`BULLETIN_URL`, or a two-line constant edit. `work/` and `dist/` there
are gitignored.
