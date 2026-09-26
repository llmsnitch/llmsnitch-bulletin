# T802 — First publish

`labels: wayfinder:task, needs-user`
`parent: ../map.md`
`blocked by: T801, user (llmsnitch account credentials)`
`blocks: T803 (default URL constant only), T804`
`status: PARKED (2026-09-19, D07 — local provider; revive if a second machine ever needs the bulletin)`

## Question

The user creates the public repo under the `llmsnitch` account (2FA on),
pushes T801's contents, and triggers `workflow_dispatch` once. Done when:

- the run is green; release `bulletin` exists with both assets; a
  `build …` commit touched `LAST_BUILD`;
- `curl -fsSL -o /tmp/b.gz https://github.com/llmsnitch/<repo>/releases/download/bulletin/pypi.json.gz`
  and the `.sha256` sibling both resolve (302 → CDN) and
  `shasum -a 256 /tmp/b.gz` equals the sidecar;
- from the CLI repo, `bulletin.load('/tmp/b.gz')` returns `(doc, None)`
  with today's `issued_at`.

Resolution records the final repo name and URL (T803's `BULLETIN_URL`
constant depends on it) and the run's wall time.

## Resolution

—
