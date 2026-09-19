# Inspect a measurement

When you report a plotted point, save the snapshot revision, the run ID and the
complete case tags. The explorer's integer point number will not do on its own: it
belongs to one SQLite snapshot, and it can change when records are added or removed.

## Locate the run

The website's [data pointer](https://spatial-bench.org/data/latest.json) names the
snapshot it serves and the results-repository revision, `sha`, it was built from.
Match that against the snapshot loaded in your browser, because the pointer can
advance while a page is open. Inside the database, join `points.run_id` to
`runs.id`, then find the JSON whose `run.run_id` matches. The
[local database example](publication.md#build-a-local-database) performs exactly
that join. Note that a point's `sha` identifies the library artifact, not the
results revision, so the two are independent.

A concrete example makes this easier to follow. Start from this repository root and
inspect the recorded pykdtree run:

```sh
python3 - <<'PY'
import json, pathlib
path = pathlib.Path('datasets/2026-09/20260914T093104Z-anxrmnkpfa-qxmvv-criterion-01M2FM95.json')
doc = json.loads(path.read_text())
print(json.dumps(doc['run'], indent=2))
print(json.dumps(doc['points'][0], indent=2))
PY
```

## Read the workload and estimates

The document has a `schema_version`, a `run` header and a `points` array. Its first
point uses 65,536 uniformly distributed three-dimensional `f32` points, 1,000
single-query probes, `k=1` and Euclidean distance. The relevant fields look like
this:

```json
{
  "metrics": {
    "latency_ns": {
      "point": 5125.321333333334,
      "lower": 4768.615447848511,
      "upper": 5482.027218818157,
      "unit": "ns/query"
    }
  },
  "stats": {"samples": 30, "ci": 0.95, "median_ns": 4860.157}
}
```

The chart plots the median when one is available, but the confidence bounds above
belong to the mean, and throughput is the reciprocal of the mean as well. A missing
field simply means the value was not recorded. The
[methodology](https://spatial-bench.org/methodology) explains how the Rust and exec
estimators differ.

## Read the source and machine records

This run header identifies `01M2FM95CY25TZ83AAV6S66ASZ` and pykdtree `1.4.3`. Its
subject `sha` is
`sha256:0726995df7f62bee5beabc867ba86ffab96cc38c4cf59dd92cb92eab64c51b91`,
the digest of the distribution the engine downloaded and installed; Git-based
subjects place a commit ID in the same field. `pinned_ref` records the release that
was requested. To find the source repository and the adapter, go to the bencher
manifest the run used and keep its revision as supporting evidence.

`machine_hash` points at
[machines/anxrmnkpfa-qxmvv.toml](../machines/anxrmnkpfa-qxmvv.toml), which contains:

```toml
cpu_model = "AMD Ryzen 5 8500GE w/ Radeon 740M Graphics"
degraded = true

[mem]
speed_mts = 5200
total_bytes = 15849508864
```

`hash_observed` lists which fingerprint components were available, and `degraded`
records that the observation was incomplete. Run context tracks the OS, kernel and
CPU settings separately, since those can change on one machine. Treat the full hash
as the continuity key, and read an `UNKNOWN` suffix as the unfingerprinted path.
Fingerprint checksums use a public constant key, so they detect accidental change
rather than attesting to the operator.

Reproduction needs more than the JSON. Supplement it with the invocation, the engine
and bencher revisions, and the build and runtime records. The current schema omits
the seed, the budget, resolved dependencies and C++/Python versions. Engine
`source.git_sha` is empty, and `git_dirty` marks subject-path overrides without
actually inspecting Git cleanliness. Some toolchain fields are placeholders too:
`host` is an architecture, `target_cpu` is absent, and `features` is empty. Raw
timing samples are not stored in the document at all.

## Submission checks

The submit command requires schema version 1, `source.git_dirty=false` and a
nonempty `sha` for each subject present. It skips unreadable or malformed JSON
without complaint. Missing or empty subject maps pass the current check, and neither
fingerprints nor `UNKNOWN` hashes are examined. It does not validate answers or
establish any of the provenance above, so a successful command is not a reviewed
result; it still needs the review described in
[CONTRIBUTING.md](../CONTRIBUTING.md).

Local subject-path measurements are useful during development, but they lack a
pinned subject revision and are marked dirty. Resolve the missing provenance and
rerun before submitting; editing the flags afterwards does not make the experiment
reproducible.

## Follow the published representation

The database contains the `runs`, `machines`, `points` and `point_tags` tables. Core
tags become columns, `defaults_or_tuned` becomes `config`, and extension tags become
text rows. Subject version and source identity are copied onto every point.

The JSON and TOML remain the complete record, because the projection is lossy. The
collator currently expects a string toolchain and flat machine fields, which leaves
structured toolchain data and nested memory or board values out of SQL. It also
omits selectors, full engine source metadata, Criterion provenance and perf's open
metrics. None of this indicates missing hardware or missing measurements; it is
simply what the database does not carry.

The [publication guide](publication.md#inspect-and-package-the-snapshot) shows a
manifest generated from real records. Its `db` names a gzip-compressed SQLite file,
and its `sha256` hashes the **uncompressed SQLite bytes**, while `bytes` measures the
compressed file. The browser decompresses as needed and compares a supplied hash
before opening SQLite; an absent or empty hash currently skips that comparison.

## Reference revisions

The example above comes from results `490451b`. Field definitions and behaviour are
pinned to engine `2104bfe`:
[schema](https://github.com/spatial-bench/spatial-bench-core/blob/2104bfe8b2ba04c31ee484e9a9aee4f0c5df0f79/crates/spatial-bench-core/src/schema.rs),
[run assembly](https://github.com/spatial-bench/spatial-bench-core/blob/2104bfe8b2ba04c31ee484e9a9aee4f0c5df0f79/crates/spatial-bench-core/src/run.rs),
[fingerprint checks](https://github.com/spatial-bench/spatial-bench-core/blob/2104bfe8b2ba04c31ee484e9a9aee4f0c5df0f79/crates/spatial-bench-core/src/fingerprint.rs)
and [submission/collation](https://github.com/spatial-bench/spatial-bench-core/blob/2104bfe8b2ba04c31ee484e9a9aee4f0c5df0f79/crates/spatial-bench-cli/src/main.rs).