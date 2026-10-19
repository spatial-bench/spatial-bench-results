# Collate, inspect and publish a snapshot

A completed run becomes website data only after it is proposed, reviewed, merged,
collated and uploaded. [Contributing](../CONTRIBUTING.md) covers submission and
review. This page covers local verification without credentials and the separate
maintainer publication path. Verified on 14 September 2026 with engine `2104bfe`
and results `895a701`; read the
[provenance limits](format-and-provenance.md) before treating a successful build
as complete reproduction evidence.

## Collate locally

Prerequisites: an engine source checkout, its Rust toolchain/build prerequisites,
and Python 3 with the standard `sqlite3` module. No database server, production
credentials, benchmark run or dataset generator is required for collation. For
engine setup see
[development](https://github.com/spatial-bench/spatial-bench-core/blob/main/docs/development.md).
The commands below run from the **results repository root**, with a sibling
engine checkout; adjust only the engine path if your layout differs.

```sh
SPATIAL_BENCH_ENGINE=../spatial-bench-core
cargo build --release --manifest-path "$SPATIAL_BENCH_ENGINE/Cargo.toml" -p spatial-bench
export SPATIAL_BENCH_SNAPSHOT=$(mktemp -d)
SPATIAL_BENCH_RESULTS_SHA=$(git rev-parse --short=12 HEAD)
"$SPATIAL_BENCH_ENGINE/target/release/spatial-bench" publish \
  --results "$PWD" \
  --out "$SPATIAL_BENCH_SNAPSHOT/benchmarks-$SPATIAL_BENCH_RESULTS_SHA.sqlite" \
  --sha "$SPATIAL_BENCH_RESULTS_SHA"
```

The output directory holds SQLite plus an intermediate `latest.json`. `publish`
is a local conversion command despite its name; it does not upload anything.
It reads the files in the working tree, so `--sha` is a label supplied by the
caller, not verification that those files equal that Git revision. Use a clean
checkout and record both engine/results revisions for an auditable snapshot.
If `CARGO_TARGET_DIR` is set, use the executable from that target directory.

At results `895a701c0dc89dfcfb60864ad30e67cf1c2d04e5` this reports **50 runs,
1,512 points, one machine**. Counts can change after that revision. Inspect the
actual generated database:

```sh
python3 - <<'PY'
import json, os, pathlib, sqlite3
out = pathlib.Path(os.environ['SPATIAL_BENCH_SNAPSHOT'])
manifest = json.loads((out / 'latest.json').read_text())
with sqlite3.connect(f"file:{out / manifest['db']}?mode=ro", uri=True) as db:
    db.row_factory = sqlite3.Row
    assert db.execute('PRAGMA integrity_check').fetchone()[0] == 'ok'
    for table in ('runs', 'points', 'machines'):
        print(table, db.execute(f'SELECT count(*) FROM {table}').fetchone()[0])
    row = db.execute('''
        SELECT p.id, p.run_id, p.impl, p.version, p.sha,
               p.latency_ns, p.median_ns, r.machine_hash
        FROM points p JOIN runs r ON r.id = p.run_id
        ORDER BY r.started_at DESC, p.id LIMIT 1
    ''').fetchone()
    print(json.dumps(dict(row), indent=2))
PY
```

At the audited revision the selected point has snapshot-local ID `1477`, run ID
`01M2DFHAPS3CVQ78Z37ZNCMZPT`, library `kdtree` version `0.8.1`, and machine
`anxrmnkpfa-qxmvv`. Trace that record back to the original file, from the same
results repository root:

```sh
python3 - <<'PY'
import json, pathlib
run_id = '01M2DFHAPS3CVQ78Z37ZNCMZPT'
for path in pathlib.Path('datasets').rglob('*.json'):
    doc = json.loads(path.read_text())
    if doc['run']['run_id'] == run_id:
        print(path)
        print(json.dumps(doc['run'], indent=2))
        print(pathlib.Path('machines', doc['run']['machine_hash'] + '.toml'))
        break
else:
    raise SystemExit('run absent: check the results revision')
PY
```

The [annotated record](format-and-provenance.md#trace-from-the-explorer-to-the-source)
explains why the raw JSON/TOML is needed beyond these SQL columns. For another
point, use its actual run ID and match all tags rather than assuming the first
point in the run.

## Make a browser-compatible local snapshot

The collator's intermediate manifest has counts but lacks the browser's normal
integrity fields. Run this next, using the same `SPATIAL_BENCH_SNAPSHOT` directory.
It leaves the raw SQLite file, creates a gzip artifact, replaces `latest.json`
and checks the round trip. The resulting filename follows the Worker route's
12-hex-character revision convention.

```sh
python3 - <<'PY'
import datetime, gzip, hashlib, json, os, pathlib
out = pathlib.Path(os.environ['SPATIAL_BENCH_SNAPSHOT'])
manifest = json.loads((out / 'latest.json').read_text())
raw_path = out / manifest['db']
raw = raw_path.read_bytes()
assert raw.startswith(b'SQLite format 3\0')
compressed = gzip.compress(raw, compresslevel=9, mtime=0)
name = raw_path.name + '.gz'
(out / name).write_bytes(compressed)
latest = {
    'db': name,
    'sha': manifest['sha'],
    'sha256': hashlib.sha256(raw).hexdigest(),
    'bytes': len(compressed),
    'generated_at': datetime.datetime.now(datetime.timezone.utc).strftime('%Y-%m-%dT%H:%M:%SZ'),
}
(out / 'latest.json').write_text(json.dumps(latest, indent=2) + '\n')
assert gzip.decompress((out / name).read_bytes()) == raw
assert hashlib.sha256(gzip.decompress((out / name).read_bytes())).hexdigest() == latest['sha256']
print(json.dumps(latest, indent=2))
PY
```

`sha256` hashes **raw SQLite**, while `bytes` counts **gzip**. The workflow uses
`gzip -9`; this local example fixes the gzip timestamp to zero. Compressed bytes
need not match the workflow's gzip header for the decompressed hash to agree.
SHA-256 detects an artifact mismatch against the pointer; it does not authenticate
who generated either file.

To view it locally, use the web repository's
[local data instructions](https://github.com/spatial-bench/spatial-bench-web/blob/main/CONTRIBUTING.md)
and copy the generated `latest.json` and named gzip file into its documented data
location. Do not replace `sha256` with a hash of the `.gz` file. Rebuild from
collation into a fresh directory to repeat this example; the conversion step
expects the intermediate manifest and is not an in-place publisher retry.

## Maintainer publication

The checked-in [workflow](../.github/workflows/publish.yml) triggers on pushes to
`main` and `workflow_dispatch`. It checks out the results revision, checks out
engine **`main`**, installs stable Rust, builds `spatial-bench`, and collates the
working tree. It names the database from the first 12 characters of
`GITHUB_SHA`, computes SHA-256 **before** `gzip -9`, uploads the gzip object, then
uploads `latest.json` last.

The gzip upload sets `Content-Type: application/sqlite3`,
`Content-Encoding: gzip` and `Cache-Control: public, max-age=31536000, immutable`.
The JSON pointer sets `Content-Type: application/json` and `Cache-Control:
no-cache`. Snapshot names are **results-revision-addressed**, not the SHA-256 of
the content. The workflow does not prevent overwriting the same object key. A
retry at the same results commit can use a newer engine `main`, so it must not be
assumed to recreate identical SQLite bytes or a genuinely immutable object.
Record the engine checkout/build details from the workflow logs when auditing a
publication. Pinning the collator revision would be a future reproducibility
improvement, not current behavior.

The workflow needs repository secrets `R2_ACCESS_KEY_ID`,
`R2_SECRET_ACCESS_KEY`, `R2_ENDPOINT`, and `R2_BUCKET`, plus an AWS CLI able to
access that endpoint. These are configuration names, never values to include in
a PR. The website's Worker needs its `DATASET` R2 binding to the same bucket and
serves `/data/latest.json` and allowed snapshot filenames. Uploading to R2 alone
does not configure or deploy the website.
[Audited Worker](https://github.com/spatial-bench/spatial-bench-web/blob/755159672d2601f6eb5a40a16e9b0fb9c0e96b8a/src/worker.ts),
[web deployment instructions](https://github.com/spatial-bench/spatial-bench-web/blob/main/CONTRIBUTING.md).

After an authorized publication, check the workflow's engine/results revisions,
collation counts and both upload steps. Read the site's `latest.json`, confirm
its revision and object name, fetch that exact object, decompress if necessary,
and hash the resulting SQLite bytes. Compare with `sha256`, run SQLite's
`PRAGMA integrity_check`, and inspect representative points and their source run
IDs. Finally load the explorer and confirm it is using that revision. A client
can remain on an older loaded snapshot until it reloads; a successful workflow
is not proof that every open browser has refreshed.

## Failures and recovery

| Failure | What to inspect and recover |
| --- | --- |
| Local collation fails | Read the named JSON/TOML error or duplicate run-ID constraint. Correct input in a reviewed change. Rebuild into a fresh output directory; an existing output was removed before conversion and a partial file may remain. |
| Build fails in CI | Inspect the engine revision/toolchain actually checked out. Results `main` and engine `main` advance independently. No data upload should be inferred from a build failure. |
| Database upload fails | Verify endpoint, bucket and upload permission. The pointer step normally has not run; inspect it before retrying. |
| Pointer upload fails after database upload | The old pointer normally remains usable and the new object can be unreferenced. Verify the uploaded object before completing an authorized pointer update or retry. |
| Browser reports a hash mismatch | Verify the raw SQLite hash after optional decompression, then compare the pointer's object name, revision and hash. Do not disable integrity checking or hash gzip to make it pass. |
| New pointer works but a comparison is missing | Check the source run, SQL projection, active filters, grouping/deduplication and client snapshot revision. Missing coverage is not a zero-cost measurement. |

Recovery is a maintainer operation because it changes what readers receive. Keep
the previous pointer and verified object available while investigating. If a
rollback is chosen, point to a previously verified complete snapshot and record
the reason; the checked-in workflow does not provide a dedicated rollback action
or enforce a retention policy. Avoid overwriting a cached revision-named object
with different content: immutable cache headers can leave different clients with
different bytes. A fresh reviewed results commit gives publication a new key.

[Corrections and removals](../CONTRIBUTING.md#corrections-exclusions-and-removals)
change future collation input. They do not automatically purge previously
published artifacts, browser copies or caches; any retention or deletion policy
requires explicit maintainer agreement.
