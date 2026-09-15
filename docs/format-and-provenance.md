# Inspect a point and its provenance

A plotted point is a row in one published SQLite snapshot. Its integer `id` is
local to that collation; adding/removing earlier records can change it. Preserve
the snapshot revision and `run_id`, together with the complete point tags, when
reporting or comparing a measurement. See the
[reading guide](https://spatial-bench.org/guide) for the explorer and
[local inspection commands](publication.md#collate-locally) for SQLite.

## Trace from the explorer to the source

1. Record the explorer's point number, run ID, tags and snapshot revision. The
   manifest is available at
   [the site's data pointer](https://spatial-bench.org/data/latest.json); it can
   advance after you loaded the page, so match it to the loaded snapshot.
2. In that database, join `points.run_id` to `runs.id`. Read `points.sha`,
   `version`, `pinned_ref`, `language` and `runs.machine_hash`; use `point_tags`
   for all extension tags. The `sha` in a point is the **library artifact**
   identity. The manifest's `sha` is the **results repository** revision.
3. Check out that results revision and find the JSON whose `run.run_id` matches.
   Compare the full tags: a run can contain many points with the same library
   and tree size. The JSON retains more information than SQLite.
4. Inspect `run.subjects[impl]`, `run.source`, `run.toolchain`, `run.context`,
   `run.machine` and `machines/<machine_hash>.toml`. Find the source repository
   and driver through the bencher manifest used for that run. If its revision
   was not retained, report that gap rather than substituting today's manifest.

For example, the following are **selected fields from a real stored run**, not
an example to submit. Fields omitted here remain in the
[original JSON at results 895a701](https://github.com/spatial-bench/spatial-bench-results/blob/895a701c0dc89dfcfb60864ad30e67cf1c2d04e5/datasets/2026-09/20260913T132432Z-anxrmnkpfa-qxmvv-criterion-01M2DFHA.json):

```json
{
  "schema_version": 1,
  "run": {
    "run_id": "01M2DFHAPS3CVQ78Z37ZNCMZPT",
    "started_at": "2026-09-13T13:24:32Z",
    "runner": "criterion",
    "machine_hash": "anxrmnkpfa-qxmvv",
    "source": {
      "git_sha": null,
      "git_dirty": false,
      "crate_version": "0.1.0"
    },
    "subjects": {
      "kdtree": {
        "version": "0.8.1",
        "pinned_ref": "v0.8.1",
        "sha": "c175108ba77175b614d0a35362ff1a67b53a9515",
        "language": "rust"
      }
    }
  }
}
```

The document also contains a `points` array. This run's first point has `axis=f32`, `dims=3`, `query=exact_nn`, `k=1`,
`tree_size=65536` and `query_count=1000`. Its actual metric/statistic excerpt is:

```json
{
  "metrics": {
    "latency_ns": {
      "point": 709.376172009245,
      "lower": 705.7888032459904,
      "upper": 712.8833289675215,
      "unit": "ns/query"
    }
  },
  "stats": {
    "median_ns": 714.7331078125,
    "samples": 30,
    "ci": 0.95
  }
}
```

The median is outside the mean's confidence interval in this record. That is not a contradictory interval: the quantities differ. See the
[statistical definitions](https://spatial-bench.org/methodology).

The corresponding
[machine record](https://github.com/spatial-bench/spatial-bench-results/blob/895a701c0dc89dfcfb60864ad30e67cf1c2d04e5/machines/anxrmnkpfa-qxmvv.toml)
contains this excerpt:

```toml
cpu_model = "AMD Ryzen 5 8500GE w/ Radeon 740M Graphics"
degraded = true

[board]
name = "334A"
vendor = "LENOVO"

[mem]
speed_mts = 5200
total_bytes = 15849508864
```

`hash_components` names intended hardware inputs; `hash_observed` lists the ones
available. Here `degraded=true` records incomplete observations even though the
run names a fingerprint file. The machine TOML is the extracted hardware block,
not the privileged fingerprint document with its checksum. Kernel, OS and
frequency policy belong to the run context because they can change on one
machine.

## What the record cannot establish

The engine's schema and
[current run assembly](https://github.com/spatial-bench/spatial-bench-core/blob/2104bfe8b2ba04c31ee484e9a9aee4f0c5df0f79/crates/spatial-bench-core/src/run.rs)
have material provenance limits:

- `run.source.git_sha` is currently absent; `git_dirty` records subject-path
  overrides, not a Git status audit. The engine package version alone does not
  identify the exact engine or driver implementation.
- Seeds, measurement budgets, full bencher revisions and resolved dependency
  lockfiles are not persisted. Keep the invocation and build artifacts with the
  review evidence. Standard defaults should not be assumed for an undocumented
  custom run.
- `run.toolchain` is a structured object, but some fields are placeholders:
  `host` is an architecture, `target_cpu` is absent and features are empty.
  C++ compiler/Python interpreter and full environment versions are not recorded.
- The source digest for a PyPI subject identifies the downloaded wheel or source
  distribution that was installed; it is not a Git commit and does not pin all
  transitive dependencies. Emitted `sha256:<hex>` provenance is valid; an
  optional manifest digest currently needs unprefixed hex because the generic
  manifest loader rejects the prefix.
- Fingerprint validation tests host continuity against recorded observations.
  Its checksum uses a public constant key and is not a signature or independent
  attestation. An `UNKNOWN` suffix identifies the unfingerprinted run path;
  submission does not currently reject it.
- The run holds summary estimates, not raw per-query observations or the full
  Criterion sample directory. A `criterion_id` is a tracing hint for those local
  files, not the identity of a workload or a URL to retained samples.

[Schema](https://github.com/spatial-bench/spatial-bench-core/blob/2104bfe8b2ba04c31ee484e9a9aee4f0c5df0f79/crates/spatial-bench-core/src/schema.rs),
[fingerprint implementation](https://github.com/spatial-bench/spatial-bench-core/blob/2104bfe8b2ba04c31ee484e9a9aee4f0c5df0f79/crates/spatial-bench-core/src/fingerprint.rs),
[artifact preparation](https://github.com/spatial-bench/spatial-bench-core/blob/2104bfe8b2ba04c31ee484e9a9aee4f0c5df0f79/crates/spatial-bench-core/src/exec.rs),
[manifest validation](https://github.com/spatial-bench/spatial-bench-core/blob/2104bfe8b2ba04c31ee484e9a9aee4f0c5df0f79/crates/spatial-bench-core/src/catalog_load.rs).

## SQLite is a projection

`publish` builds four tables: `runs`, `machines`, `points`, and `point_tags`.
Core identity tags become columns; `defaults_or_tuned` becomes `points.config`.
Remaining tags become text values in `point_tags`. Subject version/pin/SHA/language
are joined into each point from its run header. It does not pool repeated runs.

The current collator reads `run.toolchain` as a string even though real documents
store an object, so `runs.toolchain` is null for the worked example. It also reads
flat machine fields; nested `mem` and `board` details and run-level context do not
populate those SQLite columns. It omits the full engine source record, selectors,
Criterion provenance and open metrics such as perf counters. Use JSON/TOML for
these facts. These are current projection limits, not evidence that the benchmark
had no compiler, memory or counter observations.

Missing statistics are null/absent, not zero. The explorer has fallbacks for some
missing data; inspect the source before interpreting a missing value or filter
option. A successful collation only establishes that input could be parsed and
inserted: the current command does not independently enforce the submit gate,
validate all schema versions, prove answer correctness or verify fingerprint
claims. It deletes an existing output database before rebuilding and can leave a
partial database after failure. Use a fresh output directory.
[Collator implementation](https://github.com/spatial-bench/spatial-bench-core/blob/2104bfe8b2ba04c31ee484e9a9aee4f0c5df0f79/crates/spatial-bench-cli/src/main.rs).

## Snapshot manifest

The production workflow creates this shape, with values derived from the
selected results commit and generated artifact:

| Field | Meaning |
| --- | --- |
| `db` | `benchmarks-<first 12 hex characters of results commit>.sqlite.gz` |
| `sha` | The same 12-character results revision; not the engine or library SHA |
| `sha256` | Lowercase SHA-256 of the **uncompressed** SQLite bytes |
| `bytes` | Length of the uploaded **compressed** gzip file |
| `generated_at` | UTC publication timestamp |

The engine command's intermediate `latest.json` differs: it includes run/point/
machine counts and the output filename, but no integrity hash or compressed byte
count. The workflow replaces it with the production shape above. The
[local example](publication.md#make-a-browser-compatible-local-snapshot) performs
that conversion and prints real generated values; do not invent a digest or hash
the compressed file. The browser decompresses gzip when its magic bytes are
present, then hashes the SQLite bytes. A supplied mismatching hash is rejected;
an absent/empty hash currently skips that comparison.
[Publisher](https://github.com/spatial-bench/spatial-bench-results/blob/895a701c0dc89dfcfb60864ad30e67cf1c2d04e5/.github/workflows/publish.yml),
[browser loader](https://github.com/spatial-bench/spatial-bench-web/blob/755159672d2601f6eb5a40a16e9b0fb9c0e96b8a/src/engine/dataset.ts).
