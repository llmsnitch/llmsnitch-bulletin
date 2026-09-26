# T804 — Activate

`labels: wayfinder:task`
`parent: ../map.md`
`blocked by: T803, T805 (T802 parked — D07)`
`blocks: —`
`status: TODO`

## Question

Merge T803 to `main`, `pipx install --force ~/llmsnitch-cli`, then
observe. Done criteria (close the map when all hold):

1. ~~Two consecutive scheduled GitHub builds~~ (D07: local provider) —
   instead, one patrol run built the bulletin unattended:
   `~/Repos/llmsnitch-bulletin/dist/pypi.json.gz` mtime matches a 09:30
   patrol and `patrol.out` shows no `[WARN] bulletin build` line.
2. One patrol run refreshed the cache on its own: `depaudit-state.json`'s
   `bulletin_age_days` drops from ≥ 7 to < 1 with no manual command
   (`~/Library/Logs/llmsnitch/patrol.out` shows no `[WARN] bulletin
   refresh` line for that run).
3. The digest health line reports no `bulletin stale`.
4. `tools/distill_bulletin.py` deleted from this repo (`trash`), committed
   with a message pointing at this map. Dep-audit T506/T508 stay as
   written (closed history); this map's D05 is the record.

## Resolution

—
