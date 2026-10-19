# Build and publish a snapshot

The website's database is built from run JSON and machine TOML in this repository.
The engine's `publish` command creates local files; the GitHub workflow uploads
them after review. Local collation needs no production credentials or benchmark
execution.

## Build a local database

Start in the **results repository root**, with an engine checkout at sibling
`../spatial-bench`. Install its
[build prerequisites](https://github.com/spatial-bench/spatial-bench-core/blob/main/docs/development.md)
and Python 3 with the standard `sqlite3` module.

```sh
export SPATIAL_RESULTS_OUT=$(mktemp -d)
SPATIAL_RESULTS_SHA=$(git rev-parse --short=12 HEAD)
cargo run --release --manifest-path ../spatial-bench/Cargo.toml -p spatial-bench -- \
  publish --results "$PWD" \
  --out "$SPATIAL_RESULTS_OUT/benchmarks-$SPATIAL_RESULTS_SHA.sqlite" \
  --sha "$SPATIAL_RESULTS_SHA"
```

The output directory contains SQLite and an intermediate `latest.json`. At results
revision `490451b`, collation produces **62 runs, 1,980 points and one machine**.
Use a clean checkout and record the engine revision when reproducing a snapshot:
`--sha` is a caller-supplied label, without working-tree verification. Existing
output databases are replaced, which is why the example creates a new directory.

## Inspect and package the snapshot

In the same shell, check SQLite and trace a point to its run:

```sh
python3 - <<'PY'
import json, os, pathlib, sqlite3
out = pathlib.Path(os.environ['SPATIAL_RESULTS_OUT'])
manifest = json.loads((out / 'latest.json').read_text())
with sqlite3.connect(f"file:{out / manifest['db']}?mode=ro", uri=True) as db:
    assert db.execute('PRAGMA integrity_check').fetchone()[0] == 'ok'
    for table in ('runs', 'points', 'machines'):
        print(table, db.execute(f'SELECT count(*) FROM {table}').fetchone()[0])
    print(db.execute('''SELECT p.run_id,p.impl,p.version,r.machine_hash
        FROM points p JOIN runs r ON r.id=p.run_id
        ORDER BY r.started_at DESC,p.id LIMIT 1''').fetchone())
PY
```

For that revision, the selected run is `01M2FM95CY25TZ83AAV6S66ASZ`, measuring
pykdtree `1.4.3` on `anxrmnkpfa-qxmvv`. Continue with the
[annotated source record](format-and-provenance.md#locate-the-run) to inspect its
full tags, estimates and provenance.

Create the gzip artifact and replace the intermediate manifest with the website
format:

```sh
python3 - <<'PY'
import datetime, gzip, hashlib, json, os, pathlib
out = pathlib.Path(os.environ['SPATIAL_RESULTS_OUT'])
manifest = json.loads((out / 'latest.json').read_text())
path = out / manifest['db']
raw = path.read_bytes()
assert raw.startswith(b'SQLite format 3\0')
packed = gzip.compress(raw, compresslevel=9, mtime=0)
name = path.name + '.gz'
(out / name).write_bytes(packed)
latest = {
    'db': name,
    'sha': manifest['sha'],
    'sha256': hashlib.sha256(raw).hexdigest(),
    'bytes': len(packed),
    'generated_at': datetime.datetime.now(datetime.timezone.utc).isoformat(),
}
(out / 'latest.json').write_text(json.dumps(latest, indent=2) + '\n')
assert gzip.decompress((out / name).read_bytes()) == raw
print(json.dumps(latest, indent=2))
PY
```

A local build of the example revision produced this manifest:

```json
{
  "db": "benchmarks-490451bb996d.sqlite.gz",
  "sha": "490451bb996d",
  "sha256": "f6bfb0d5eee69341558257093578164a243aab6e7776afb9801725f196eac4f3",
  "bytes": 282510,
  "generated_at": "2026-09-17T06:05:14.606514+00:00"
}
```

`db` is the artifact filename, `sha` the results revision and `generated_at` the
UTC packaging time. `sha256` covers **raw SQLite bytes**; `bytes` counts the
**compressed gzip file**. Gzip timestamps can differ from the production workflow
without changing the decompressed content or its hash.

Copy `latest.json` and the named gzip file into the data location described in
[web development](https://github.com/spatial-bench/spatial-bench-web/blob/main/CONTRIBUTING.md).
The packaging step expects the intermediate manifest, so repeat the example from
collation into a new directory.

## Publish after review

The [workflow](../.github/workflows/publish.yml) runs on pushes to `main` or manual
dispatch. It checks out engine `main`, builds with stable Rust, collates the
results revision and computes SHA-256 before `gzip -9`. It uploads
`benchmarks-<12-character-results-revision>.sqlite.gz`, then updates `latest.json`.

The upload needs `R2_ACCESS_KEY_ID`, `R2_SECRET_ACCESS_KEY`, `R2_ENDPOINT` and
`R2_BUCKET` secrets, and AWS CLI access to that endpoint. The website Worker's
`DATASET` binding must reference the same bucket. Upload metadata sets SQLite
content type and gzip content encoding; the database receives a one-year
`immutable` cache policy and the pointer receives `no-cache`.

Object names derive from the results commit. A retry against the same revision
can use a newer engine `main` and overwrite that key with different bytes. Record
the engine checkout from workflow logs and avoid replacing a cached artifact with
different content; a new reviewed results commit provides a new key.

After publication, inspect the workflow's revisions and counts, then fetch the
site's pointer and named artifact. Decompress if necessary, compare the raw
SHA-256, run `PRAGMA integrity_check` and verify representative run IDs. Confirm
that the explorer loads that snapshot; an already open page may still hold the
previous one.

## Recover a failed publication

For a collation error, inspect the named input or duplicate-run constraint and
rebuild into a fresh directory. A failed command can leave a partial database.
For an upload error, inspect the object and pointer separately: the object may
have uploaded while the old pointer remains. Verify the object before completing
an authorized pointer update or retry.

For a browser integrity error, compare the pointer against the decompressed
SQLite hash. Keep the integrity check enabled. A maintainer-approved rollback can
restore a previously verified pointer; there is no dedicated rollback action or
automated retention policy. Source-record
[corrections](../CONTRIBUTING.md#correct-a-result) affect future collation and do
not purge earlier objects or client caches.

## Reference revisions

[Publisher at results 490451b](https://github.com/spatial-bench/spatial-bench-results/blob/490451bb996d646eff2fe73e7bb75bc1bce1973e/.github/workflows/publish.yml),
[collator at engine 2104bfe](https://github.com/spatial-bench/spatial-bench-core/blob/2104bfe8b2ba04c31ee484e9a9aee4f0c5df0f79/crates/spatial-bench-cli/src/main.rs)
and [browser loader at web 7551596](https://github.com/spatial-bench/spatial-bench-web/blob/755159672d2601f6eb5a40a16e9b0fb9c0e96b8a/src/engine/dataset.ts).
