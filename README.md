# spatial-bench-results

Benchmark results for the spatial-bench engine — the source of truth and
audit trail. Everything here arrives through reviewed pull requests,
submitted by CI from actual runs.

## Layout

- `datasets/YYYY-MM/<timestamp>-<machine-hash>-<runner>-<ulid>.json` — run
  documents: a run header (provenance, machine, toolchain, subjects with
  their resolved versions) and one point per measured case.
- `machines/<machine-hash>.toml` — one hardware fingerprint per machine.
- `.github/workflows/publish.yml` — on merge to main, collates everything
  into `benchmarks.sqlite` and pushes it to Cloudflare R2
  (`benchmarks-<sha>.sqlite.zst`, immutable, plus `latest.json` — the only
  mutable pointer the front-end polls).

## Collating locally

```sh
git clone https://github.com/spatial-bench/spatial-bench-core ../spatial-bench
cargo run -p spatial-bench -- publish --results . --out benchmarks.sqlite
```

The schema: `runs`, `machines`, `points` (core identity keys as columns —
they are a closed vocabulary), and `point_tags` for each subject's own
namespace (`kiddo.stem` and friends).
