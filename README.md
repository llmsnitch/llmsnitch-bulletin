# llmsnitch bulletin

A daily-built **Bulletin** of known-vulnerable PyPI package versions, for the
`llmsnitch` CLI's offline dependency audit. One file, one stable URL, refreshed
every day by GitHub Actions. No servers, no API.

## The file

- `https://github.com/llmsnitch/bulletin/releases/download/bulletin/pypi.json.gz`
- `https://github.com/llmsnitch/bulletin/releases/download/bulletin/pypi.json.gz.sha256`

Fetch with `curl -fsSL -o pypi.json.gz <url>` (the URL 302s to GitHub's CDN).
Verify with `shasum -a 256 -c pypi.json.gz.sha256`. The release tag `bulletin`
is rolling: each build replaces both assets in place, so the URL never changes.

Format: bulletin contract **v1** (`schema_version: 1`), defined in
[`SPEC.md`](SPEC.md) (shared with the `llmsnitch` CLI, which consumes this
file). Gzipped JSON, roughly 1 MB, ~19k entries.

## How it is built

`.github/workflows/build.yml` runs `distill_bulletin.py` daily at 14:17 UTC.
The script pulls three public feeds with `curl`, distills them, and writes the
bulletin plus its `.sha256` sidecar:

- **OSV** PyPI export (`storage.googleapis.com/osv-vulnerabilities/PyPI/all.zip`)
- **CISA KEV** (Known Exploited Vulnerabilities catalog)
- **FIRST EPSS** (Exploit Prediction Scoring System daily scores)

Each run also commits a one-line `LAST_BUILD` stamp so GitHub's 60-day
inactivity rule never disables the scheduled workflow.

## Attribution

The bulletin is a derivative work of:

- **OSV** (osv.dev) advisory data, including records from the
  **GitHub Advisory Database** (CC-BY 4.0) and the
  **PyPA advisory-database** (CC-BY 4.0)
- **CISA Known Exploited Vulnerabilities catalog** (CC0)
- **FIRST EPSS** scores (free to use with attribution to FIRST.org)

Per-file provenance is stamped in the bulletin's `meta.sources`.

This project is not affiliated with or endorsed by OSV, Google, GitHub,
the PyPA, CISA, or FIRST.

## License

`distill_bulletin.py` and this repository are MIT licensed (see `LICENSE`).
The published bulletin carries the upstream licenses listed above.
