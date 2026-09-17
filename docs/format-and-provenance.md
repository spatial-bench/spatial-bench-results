# Inspect a measurement

Save the snapshot revision, run ID and complete case tags when reporting a point.
The explorer's integer point number belongs to one SQLite snapshot and can change
when records are added or removed.

## Locate the run

The website's [data pointer](https://spatial-bench.org/data/latest.json) names the
snapshot and its results-repository revision, `sha`. Match it to the snapshot
loaded in your browser, since the pointer can advance. In that database, join
`points.run_id` to `runs.id`, then find the JSON whose `run.run_id` matches.
The [local database example](publication.md#build-a-local-database) performs this
join. A point's `sha` identifies its library artifact, independently of the
pointer's results revision.

For a concrete example, inspect the recorded
[pykdtree run](https://github.com/spatial-bench/spatial-bench-results/blob/490451bb996d646eff2fe73e7bb75bc1bce1973e/datasets/2026-09/20260914T093104Z-anxrmnkpfa-qxmvv-criterion-01M2FM95.json).
From this repository root:

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

The document has `schema_version`, a `run` header and a `points` array. Its first
point uses 65,536 uniform three-dimensional `f32` points and 1,000 single-query
probes, requesting `k=1` with Euclidean distance. Selected fields from that point:

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

The chart uses the median for latency when available; the confidence bounds here
refer to the mean. Throughput is the reciprocal of the mean. Missing fields mean
unrecorded values. The [methodology](https://spatial-bench.org/methodology)
defines the different Rust and exec estimators.

## Read the source and machine records

The run header identifies `01M2FM95CY25TZ83AAV6S66ASZ` and pykdtree `1.4.3`.
Its subject `sha` is
`sha256:0726995df7f62bee5beabc867ba86ffab96cc38c4cf59dd92cb92eab64c51b91`,
the digest of the downloaded distribution installed by the engine. Git-based
subjects put a commit ID in that field. `pinned_ref` records the requested release.
Find the source repository and adapter through the bencher manifest used for the
run, retaining its revision as supporting evidence.

`machine_hash` links to
[machines/anxrmnkpfa-qxmvv.toml](../machines/anxrmnkpfa-qxmvv.toml), which includes:

```toml
cpu_model = "AMD Ryzen 5 8500GE w/ Radeon 740M Graphics"
degraded = true

[mem]
speed_mts = 5200
total_bytes = 15849508864
```

`hash_observed` lists available fingerprint components; `degraded` records
incomplete observations. Run context separately records the OS, kernel and CPU
settings because those can change on one machine. Use the full hash as the
continuity key. An `UNKNOWN` suffix identifies the unfingerprinted path.
Fingerprint checksums use a public constant key and provide accidental-change
detection, not independent attestation.

For reproduction, supplement the JSON with the invocation, engine/bencher
revisions and build/runtime records. The current schema omits seed, budget,
resolved dependencies and C++/Python versions. Engine `source.git_sha` is empty;
`git_dirty` marks subject-path overrides, without checking actual Git cleanliness.
Some toolchain fields are placeholders: `host` is an architecture, `target_cpu`
is absent and `features` is empty. Raw timing samples are not in the document.

## Submission checks

The submit command requires schema version 1, `source.git_dirty=false` and a
nonempty `sha` for each subject present. It skips unreadable or malformed JSON.
Missing/empty subject maps pass the current check, and fingerprints or `UNKNOWN`
hashes are not checked. It does not validate answers or establish the omitted
provenance. A successful command therefore still needs the review described in
[CONTRIBUTING.md](../CONTRIBUTING.md).

Local subject-path measurements can be useful for development, but lack a pinned
subject revision and are marked dirty. Resolve missing provenance and rerun for
submission; editing flags does not make the experiment reproducible.

## Follow the published representation

The database contains `runs`, `machines`, `points` and `point_tags`. Core tags
become columns; `defaults_or_tuned` becomes `config`, and extension tags become
text rows. Subject version and source identity are copied onto each point.

Use the JSON/TOML for the complete record. The collator currently expects a string
toolchain and flat machine fields, leaving structured toolchain and nested
memory/board values absent in SQL. It also omits selectors, full engine source
metadata, Criterion provenance and perf's open metrics. These omissions describe
the database projection, rather than missing hardware or measurements.

The [publication guide](publication.md#inspect-and-package-the-snapshot) shows a
manifest generated from real records. Its `db` names a gzip-compressed SQLite
file; `sha256` hashes the **uncompressed SQLite bytes**, while `bytes` measures the
compressed file. The browser decompresses as needed and compares a supplied
hash before opening SQLite. An absent/empty hash currently skips comparison.

## Reference revisions

The example is from results `490451b`. Field definitions and behavior are pinned
to engine `2104bfe`: [schema](https://github.com/spatial-bench/spatial-bench-core/blob/2104bfe8b2ba04c31ee484e9a9aee4f0c5df0f79/crates/spatial-bench-core/src/schema.rs),
[run assembly](https://github.com/spatial-bench/spatial-bench-core/blob/2104bfe8b2ba04c31ee484e9a9aee4f0c5df0f79/crates/spatial-bench-core/src/run.rs),
[fingerprint checks](https://github.com/spatial-bench/spatial-bench-core/blob/2104bfe8b2ba04c31ee484e9a9aee4f0c5df0f79/crates/spatial-bench-core/src/fingerprint.rs)
and [submission/collation](https://github.com/spatial-bench/spatial-bench-core/blob/2104bfe8b2ba04c31ee484e9a9aee4f0c5df0f79/crates/spatial-bench-cli/src/main.rs).
