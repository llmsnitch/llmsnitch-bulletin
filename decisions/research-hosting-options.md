# Bulletin provider: where to build and publish it (charting note)

> Lives in `docs/research/` beside `dep-audit-advisory-sources.md` (T501).
> All URLs fetched with `curl` on **2026-09-19**. Effort:
> `wayfinder/bulletin-provider/map.md`. Contract the provider must meet:
> `docs/bulletin-spec.md` (v1). Egress model: `docs/adr/0001-bulletin-egress.md`.

## TL;DR

**Recommendation: a public GitHub repo under the `llmsnitch` account,
one scheduled GitHub Actions workflow that runs the existing distiller
daily and replaces `pypi.json.gz` + `pypi.json.gz.sha256` on one rolling
GitHub Release. Zero servers, zero cost, one YAML file.**

The stable client URL is
`https://github.com/llmsnitch/<repo>/releases/download/bulletin/pypi.json.gz`
(sidecar at the same URL + `.sha256`, exactly the spec's layout).

## What already exists

- `tools/distill_bulletin.py` (545 lines, PEP 723, one dep `packaging`,
  curl-only egress) already produces a spec-conformant full-PyPI bulletin:
  19,033 entries, 1.16 MB gzipped, sidecar written (T506). The provider is
  this script plus a scheduler plus a URL. Nothing else is missing.
- The client already reads the cache, verifies the sidecar, stamps
  `bulletin_age_days`, and the digest flags `bulletin stale` past 14 days
  (`fs_coil/digest.py`). What the client lacks is the fetch itself: T508
  closed with "documented manual re-run", cache is 7.4 days old today.

## Upstream feeds (re-measured 2026-09-19)

| feed | URL | size | last-modified | cadence |
| --- | --- | --- | --- | --- |
| OSV PyPI | `storage.googleapis.com/osv-vulnerabilities/PyPI/all.zip` | 34.4 MB | 2026-09-19 15:34 UTC | continuous |
| CISA KEV | `cisa.gov/sites/default/files/feeds/known_exploited_vulnerabilities.json` | 1.7 MB | 2026-09-18 19:00 UTC | event-driven |
| EPSS | `epss.empiricalsecurity.com/epss_scores-current.csv.gz` → 302 → `epss_scores-<date>.csv.gz` | 2.7 MB | 2026-09-19 13:30 UTC | daily 13:30 UTC |

All unauthenticated HTTPS; `curl -fsSL --retry 3` handles the EPSS
redirect. A build is ~40 MB down plus ~110 PyPI JSON lookups.

## Options considered

| option | stable URL | cost | moving parts | verdict |
| --- | --- | --- | --- | --- |
| **A. Actions cron → one rolling GitHub Release, assets `--clobber`ed** | `releases/download/bulletin/<asset>` (302 to CDN; `curl -L`) | $0 (Actions free on public repos) | 1 workflow file | **pick** |
| B. Actions cron → GitHub Pages (orphan `gh-pages`, force-pushed) | `llmsnitch.github.io/<repo>/pypi.json.gz` | $0 | workflow + Pages deploy + orphan-branch dance | works; more steps, 100 GB/month soft bandwidth cap, 1 GB site cap |
| C. Actions cron → commit artifact to `main` | raw.githubusercontent.com | $0 | 1 workflow | 1.1 MB/day into git history (~400 MB/yr); no |
| D. Cloudflare Workers cron + R2, or AWS Lambda/S3/CloudFront | custom | ~$0 but an account, IAM, TLS, deploy tooling | many | nothing gained for one 1 MB file; user's AWS guardrails make it the heaviest path |
| E. VPS cron + nginx | custom | $/month + patching | many | no |
| F. No provider: LaunchAgent runs the distiller weekly on this Mac | none (local cache only) | $0 | 1 plist | correct fallback for a single machine; does not meet "published" |
| G. Client queries OSV API directly | n/a | $0 | none | rejected by ADR 0001 (network code in package) and T503 (intake list never leaves the machine) |

### Why A over B

Same cost and same trigger; A has no bandwidth cap ("no limit on the total
size of a release, nor bandwidth usage", GitHub docs *About releases*)
and needs no Pages configuration or branch gymnastics. `gh release upload
--clobber` is one command. Pages only wins if a directory listing or a
custom domain is wanted, which nothing needs.

### Why one rolling release, not one per build

Per-build releases (tag per day) were the first draft; the cold read
called them speculative. They buy a bulletin history nobody has asked
for, cost a same-day-rerun tag collision, and the URL is no more stable.
A rolling release with `--clobber` has none of that. If a
bulletin-to-bulletin diff is ever wanted (waiver re-raise debugging), the
client keeping one `.prev` pair is the smaller change (map fog).

## GitHub facts that shape the workflow (docs fetched 2026-09-19)

- **Actions minutes are free** for public repos on standard hosted runners
  (*Billing → GitHub Actions*). Private repos get a monthly quota, but a
  private repo cannot serve unauthenticated release downloads, so the repo
  must be public anyway.
- **`schedule` can be delayed or dropped at high load**, especially at the
  top of the hour; schedule off-hour (*Events that trigger workflows*).
  Pick a minute after EPSS publishes at 13:30 UTC, e.g. `17 14 * * *`.
- **Scheduled workflows are disabled after 60 days without repository
  activity** in a public repo (same page). Whether release-asset uploads
  alone count as "activity" is **unverified**; a commit unambiguously
  does, so the workflow commits a one-line `LAST_BUILD` stamp per run
  (map D03; ~4 KB of history per year). If it trips anyway, the consumer
  sees it (`bulletin stale` in the digest after 14 days) and
  `workflow_dispatch` re-arms it. Keep both triggers.
- **Release limits**: 1000 assets per release, 2 GiB per file, no
  bandwidth limit (*About releases*). Two 1 MB assets per day is nothing.
- **Artifact attestations** (Sigstore, `actions/attest-build-provenance`)
  are free on public repos and write to the public transparency log
  (*Artifact attestations*). This is the v2 signing path the spec leaves
  open. Verification needs `gh attestation verify` or cosign, neither
  stdlib, so the client side stays a v2 item (a `gh` subprocess when
  present is the ADR 0001-shaped answer). Emitting the attestation costs
  one workflow step and can land at build time with no client change.

## Licensing the published artifact

GHSA and PYSEC records are **CC-BY 4.0**; KEV is CC0; EPSS is free with
attribution requested (T501). The bulletin is a derivative work, so the
publishing repo's README carries the attribution block (OSV/GitHub
Advisory Database/PyPA/CISA/FIRST). `meta.sources` already stamps
provenance per file. No schema change.

## Threat notes for the recommendation

- Authenticity in v1 rides on TLS to `github.com` and on the `llmsnitch`
  account. Account compromise yields a hostile bulletin, which the client
  already treats as hostile input (`bulletin.load`/`index` never raise,
  set-membership only); worst case is false or missing findings, never
  code execution. Enable 2FA on the account; attestations close the gap
  in v2.
- Fetch race: bulletin and sidecar come from two requests; an asset
  replaced between them yields a hash mismatch, which the spec already
  handles (discard, keep last good, retry next refresh). Fetch order does
  not change the window; nothing to optimize.
- Pin the one third-party action (`actions/checkout`) by commit SHA; the
  workflow needs only `contents: write`.

## Account state

`llmsnitch` (GitHub user, created 2026-08-13) has **0 public repos** as of
today; its credentials are not on this machine (only
`llmsnitch` is logged in, memory `llmsnitch-github-ownership`).
Creating the repo and enabling the workflow is a user action from wherever
those credentials live. The CLI repo stays local-only.
