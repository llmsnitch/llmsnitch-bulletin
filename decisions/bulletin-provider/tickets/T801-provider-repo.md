# T801 — Provider repo contents

`labels: wayfinder:task`
`parent: ../map.md`
`blocked by: —`
`blocks: T802`
`status: DONE (2026-09-19)`

## Question

Assemble, locally (a scratch directory outside this repo, e.g.
`~/Repos/llmsnitch-bulletin`), everything the public provider repo needs,
and prove it builds by running the workflow's exact commands by hand once.
Nothing is pushed (credentials boundary, map Notes). Four files:

1. `distill_bulletin.py` — **verbatim** copy of `tools/distill_bulletin.py`
   (`cmp` must report identical). The workflow passes `--workdir work
   --out dist`, so its `~/.llmsnitch` default and the `0700/0600` chmods
   are never exercised on the runner and need no edit.
2. `.github/workflows/build.yml`:
   - `on: schedule: - cron: "17 14 * * *"` and `workflow_dispatch:`.
   - `permissions: contents: write`; `env: GH_TOKEN: ${{ github.token }}`.
   - `runs-on: ubuntu-latest` (python3, pip, gh, git preinstalled — no
     setup-python, no uv).
   - Steps: `actions/checkout` **pinned to the commit SHA** of its current
     major tag (look it up at execution:
     `curl -fsSL https://api.github.com/repos/actions/checkout/git/ref/tags/<tag>`);
     `pip install packaging`; `python3 distill_bulletin.py --workdir work
     --out dist`;
     `gh release view bulletin >/dev/null 2>&1 || gh release create bulletin --title "bulletin (rolling)" --notes "Rolling PyPI bulletin, schema v1. See README."`;
     `gh release upload bulletin dist/pypi.json.gz dist/pypi.json.gz.sha256 --clobber`;
     keepalive (map D03): `date -u +%FT%TZ > LAST_BUILD && git config user.name
     "bulletin-bot" && git config user.email "bulletin-bot@users.noreply.github.com"
     && git commit -qam "build $(cat LAST_BUILD)" && git push` with a
     `# ponytail: commit exists only to reset GitHub's 60-day cron disable`
     comment above it. `LAST_BUILD` is committed once in the initial
     contents so the `-a` commit has a tracked file to change.
3. `README.md` — what the file is, the stable URL
   `https://github.com/llmsnitch/<repo>/releases/download/bulletin/pypi.json.gz`
   (+ `.sha256`), a pointer to the spec (`docs/bulletin-spec.md` in the
   CLI repo, v1), the attribution block (research note "Licensing": OSV,
   GitHub Advisory Database CC-BY 4.0, PyPA advisory-database CC-BY 4.0,
   CISA KEV CC0, FIRST EPSS attribution), and "not affiliated with
   OSV/CISA/FIRST".
4. `LICENSE` — MIT for the script.

Hand run (from the scratch dir): `uv run distill_bulletin.py --workdir
work --out dist` (PEP 723 pulls `packaging`; equivalent to the runner's
pip line) → `shasum -a 256 dist/pypi.json.gz` equals the sidecar; then
from the CLI repo, `python3 -c "from llmsnitch import bulletin;
print(bulletin.load('<scratch>/dist/pypi.json.gz')[1])"` prints `None`.
Record entry count, gz size, and wall time in the resolution.

Out: any edit to the distiller, any extra action (setup-python, uv,
attestation), release pruning, a `Makefile`.

## Resolution

Assembled in `~/Repos/llmsnitch-bulletin` (no git init, no remote — the
commit identity belongs to the `llmsnitch` account, not this machine):

- `distill_bulletin.py` — `cmp` identical to `tools/distill_bulletin.py`.
- `.github/workflows/build.yml` — as specified; `actions/checkout` pinned
  to `3d3c42e5aac5ba805825da76410c181273ba90b1` (= `v7` = `v7.0.1`, the
  current major, looked up 2026-09-19 via the git/ref API). Parses (ruby
  YAML: 1 job, 6 steps, cron `17 14 * * *`).
- `README.md` — URL pair under the default repo name `llmsnitch/bulletin`
  (T802 renames if the user picks otherwise), spec pointer, attribution
  block, non-affiliation line. `LICENSE` — MIT (copy of the CLI repo's).
- `LAST_BUILD` seeded (`2026-09-19T21:54:58Z`) so the keepalive `-a`
  commit has a tracked file. `.gitignore` (`work/`, `dist/`) added so a
  `git add .` at T802 cannot push the 40 MB feed workdir — fifth file,
  two lines.

Hand run `uv run distill_bulletin.py --workdir work --out dist`:

| entries | gzipped | raw | wall |
| --- | --- | --- | --- |
| 19,091 (malicious 11,734, kev 21, epss 7,013) | 1,163,541 B | 13,074,288 B | 19.9 s |

`shasum -a 256 dist/pypi.json.gz` = sidecar =
`4e3252dc869e3eab305728d964e55ac4b33a3f6507bf9e37bf82a1bfb6ac2fc6`.
From the CLI repo `bulletin.load('<scratch>/dist/pypi.json.gz')[1]` → `None`;
`age_days` ≈ 0. `dist/` is kept in the scratch dir for T803's `file://`
tests; `work/` trashed.

Runner-side note: `pip install packaging` on `ubuntu-latest` may need
`--break-system-packages` if the runner's Python is Debian-externally-managed;
T802 sees this on the first run and fixes the one line if so.
