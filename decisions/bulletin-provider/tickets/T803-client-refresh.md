# T803 — Client refresh

`labels: wayfinder:task`
`parent: ../map.md`
`blocked by: T802 for the default URL constant only — build and test now against file:// URLs with a placeholder`
`blocks: T804`
`status: DONE (2026-09-19; default URL constant assumes repo name `llmsnitch/bulletin` — T802 confirms or edits one line)`

## Question

Give the CLI the fetch half of `docs/bulletin-spec.md` "Fetch, verify,
cache" as a `curl` subprocess (ADR 0001; `test_no_network_imports` stays
green). Everything lands in `llmsnitch/bulletin.py` (map D06); expect
~50 lines.

**`bulletin.refresh(out) -> int`** (0 ok / 2 failed; never raises):

- `url = os.environ.get("LLMSNITCH_BULLETIN_URL", BULLETIN_URL)`;
  `BULLETIN_URL` is the T802 URL (placeholder until then). Sidecar URL is
  `url + ".sha256"`. `file://` URLs work through curl unchanged — that is
  the test seam.
- Targets: `p = default_path()`; temps `p + ".tmp"` and `p + ".tmp.sha256"`
  in the same directory (same filesystem → atomic `os.replace`; the
  `.sha256` suffix on the temp lets `load(tmp)` find its sidecar
  unchanged). `store._mkdir_private(p.parent)` first.
- Two `subprocess.run(["curl", "-fsSL", "--retry", "3", "--max-time",
  "120", "-o", tmp, url])` calls, order irrelevant (a provider upload
  landing between them is a hash mismatch, which self-heals at the next
  patrol). `FileNotFoundError` (no curl) and non-zero return are both
  "fetch failed". `store._chmod_private(tmp)` after each download (curl
  opens the file itself, so `_open_private` cannot wrap it).
- `load(tmp)` is the whole verification (hash, gzip bound, schema {1}).
  On `(None, reason)`: `out.write(f"[WARN] bulletin refresh failed: {reason}\n")`,
  return 2. On success: `os.replace` sidecar then gz, return 0. `finally`:
  unlink both temps if still present.

**Patrol wiring** — in `scan.py`'s patrol-only branch (currently
scan.py:849–854, `if a.patrol_run:` before `depaudit.cmd_depaudit`):
`if bulletin.needs_refresh(): bulletin.refresh(out)` and ignore the
return — the audit always runs. `needs_refresh()` = cache missing, or
`load()` fails, or `age_days(meta) >= REFRESH_DAYS` (`REFRESH_DAYS = 7`,
a constant with a `# ponytail:` comment). Manual `llmsnitch depaudit`
never fetches. `bulletin.age_days` is the same clock `depaudit` stamps
into `depaudit-state.json`, so "age" means one thing.

**CLI** — `cli.py`'s if-chain (cli.py:94–131) gains
`cmd == "bulletin" and argv[1:] == ["refresh"]` → `bulletin.refresh(out)`;
anything else under `bulletin` falls through to the existing
unknown-command exit. One help line (cli.py:1–11). Update the exit-2 hint
at depaudit.py:448 from "run the weekly refresh (T508)" to
"run: llmsnitch bulletin refresh".

**Tests** — in `tests/test_depaudit_bulletin.py` (already registered in
`tests/all.py`; reuse its bulletin-writing helper and `_seams.with_tmp`):
happy path against a `file://` pair → cache swapped, `load()` ok, no temps
left; sidecar mismatch → old cache untouched, return 2, `[WARN]` in out;
`file://` path that does not exist → return 2 (curl `-f` non-zero covers
the "fetch failed" branch; no PATH scrub for the no-curl case); a
`schema_version: 2` pair → rejected before swap; `needs_refresh()` false
for a fresh cache, true for missing / corrupt / 8-day-old `issued_at`;
patrol branch: refresh returning 2 still runs the audit (call
`scan`'s patrol path with `LLMSNITCH_BULLETIN_URL` pointing at a
nonexistent `file://` and assert the dep-audit line still prints).

Out: `--force`, config keys, ETag, retries beyond curl's, any daemon, a
new module, touching fs_coil.

## Resolution

Built as specified; 66 lines added to `llmsnitch/bulletin.py`, 4 to
`scan.py`/`cli.py`, 1 in `depaudit.py`.

- `BULLETIN_URL` = `https://github.com/llmsnitch/bulletin/releases/download/bulletin/pypi.json.gz`
  (map default repo name); `LLMSNITCH_BULLETIN_URL` overrides.
  `REFRESH_DAYS = 7` with the `# ponytail:` note.
- `needs_refresh()`: cache missing / unloadable / `age_days` None or ≥ 7.
- `refresh(out)`: `_mkdir_private`, two `curl -fsSL --retry 3 --max-time
  120 -o` calls into `pypi.json.gz.tmp` + `.tmp.sha256`, `_chmod_private`
  after each, `load(tmp)` is the verification, `os.replace` sidecar then
  gz; `finally` unlinks temps; `OSError` anywhere (no curl, replace
  failure) becomes `[WARN] bulletin refresh failed: <reason>` + return 2.
  One `[WARN]` line, never raises.
- Patrol: `scan --patrol` calls `refresh` when `needs_refresh()` before
  `cmd_depaudit`; return ignored. `llmsnitch depaudit` never fetches.
- CLI: `llmsnitch bulletin refresh` (exact argv match; anything else under
  `bulletin` falls to the existing unknown-command exit 2). Help line
  added. depaudit exit-2 hint now says `run: llmsnitch bulletin refresh`.
- Tests: 6 new in `test_depaudit_bulletin.py` (happy swap + 0600, empty
  store creates 0700 dir, sidecar mismatch keeps old cache, dead `file://`
  → fetch failed, `schema_version: 2` rejected before swap,
  `needs_refresh` matrix incl. 8-day and garbage stamps), 1 in
  `test_scan_notify.py` (patrol refresh failure still runs the audit).
  `tests/_seams.with_tmp` now pins `LLMSNITCH_BULLETIN_URL` to a dead
  `file://` so no test can reach the real provider.
  `python3 tests/all.py`: 211/211 green, `test_no_network_imports` included.
- End to end against T801's `dist/` via `file://` and a fresh
  `LLMSNITCH_DIR`: `llmsnitch bulletin refresh` → rc 0 in 1.73 s wall,
  cache 0600 in 0700 dir, `needs_refresh()` False, `llmsnitch depaudit`
  → `bulletin age 0.0d … [OK]`; dead URL → one `[WARN]` line, rc 2.
