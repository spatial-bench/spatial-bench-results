# Build and publish a snapshot

The website's database is built from the run JSON and machine TOML stored in this
repository. The engine's `publish` command produces the local files, and the GitHub
workflow uploads them after review. Building a snapshot locally requires no
production credentials and no benchmark execution.

## Build a local database

Start in the **results repository root**, with an engine checkout at the sibling
path `../spatial-bench`. Install the engine's
[build prerequisites](https://github.com/spatial-bench/spatial-bench-core/blob/main/docs/development.md)
along with Python 3 and its standard `sqlite3` module.

```sh
export SPATIAL_RESULTS_OUT=$(mktemp -d)
SPATIAL_RESULTS_SHA=$(git rev-parse --short=12 HEAD)
cargo run --release --manifest-path ../spatial-bench/Cargo.toml -p spatial-bench -- \
  publish --results "$PWD" \
  --out "$SPATIAL_RESULTS_OUT/benchmarks-$SPATIAL_RESULTS_SHA.sqlite" \
  --sha "$SPATIAL_RESULTS_SHA"
```

The output directory will contain the SQLite database and an intermediate
`latest.json`. At results revision `490451b`, collation produces **62 runs, 1,980
points and one machine**. Use a clean checkout and record the engine revision when
reproducing a snapshot, since `--sha` is a caller-supplied label with no
working-tree verification behind it. Existing output databases are replaced, which
is why the example writes into a fresh directory.

## Inspect and package the snapshot

Still in the same shell, check the database and trace a point back to its run:

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

For that revision the selected run is `01M2FM95CY25TZ83AAV6S66ASZ`, measuring
pykdtree `1.4.3` on `anxrmnkpfa-qxmvv`. You can continue with the
[annotated source record](format-and-provenance.md#locate-the-run) to look at its
full tags, estimates and provenance.

Next, create the gzip artifact and rewrite the intermediate manifest in the format
the website expects:

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

Here `db` is the artifact filename, `sha` the results revision, and `generated_at`
the UTC packaging time. `sha256` covers the **raw SQLite bytes** while `bytes`
counts the **compressed gzip file**. Gzip timestamps can differ from the production
workflow without changing the decompressed content or its hash.

Copy `latest.json` and the named gzip file into the data location described in
[web development](https://github.com/spatial-bench/spatial-bench-web/blob/main/CONTRIBUTING.md).
Because the packaging step expects the intermediate manifest, run the whole example
again from collation if you need a new directory.

## Publish after review

The [workflow](../.github/workflows/publish.yml) runs on pushes to `main` or on
manual dispatch. It checks out engine `main`, builds with stable Rust, collates the
results revision, and computes SHA-256 before running `gzip -9`. It then uploads
`benchmarks-<12-character-results-revision>.sqlite.gz` and updates `latest.json`.

The upload needs `R2_ACCESS_KEY_ID`, `R2_SECRET_ACCESS_KEY`, `R2_ENDPOINT` and
`R2_BUCKET` secrets, along with AWS CLI access to that endpoint; the website Worker's
`DATASET` binding must reference the same bucket. Upload metadata sets the SQLite
content type and gzip content encoding, and the database receives a one-year
`immutable` cache policy while the pointer is marked `no-cache`.

Object names are derived from the results commit. A retry against the same revision
can pick up a newer engine `main` and overwrite that key with different bytes, so
record the engine checkout from the workflow logs and avoid replacing a cached
artifact with different content. A new, reviewed results commit gives you a new key
to work with.

Once published, inspect the revisions and counts in the workflow log, then fetch the
site's pointer and the artifact it names. Decompress if necessary, compare the raw
SHA-256, run `PRAGMA integrity_check` and verify a few representative run IDs.
Finally, confirm the explorer loads that snapshot, remembering that an already-open
page may still be holding the previous one.

## Recover a failed publication

For a collation error, inspect the named input or the duplicate-run constraint and
rebuild into a fresh directory; a failed command can leave a partial database
behind. For an upload error, examine the object and the pointer separately, since
the object may have uploaded successfully while the old pointer remains. Verify the
object before completing an authorized pointer update or retrying.

For a browser integrity error, compare the pointer against the decompressed SQLite
hash. Leave the integrity check enabled. A maintainer-approved rollback can restore
a previously verified pointer, but there is no dedicated rollback action and no
automated retention policy. Source-record
[corrections](../CONTRIBUTING.md#correct-a-result) affect future collation and do
not purge earlier objects or client caches.

## Reference revisions

[Publisher at results 490451b](https://github.com/spatial-bench/spatial-bench-results/blob/490451bb996d646eff2fe73e7bb75bc1bce1973e/.github/workflows/publish.yml),
[collator at engine 2104bfe](https://github.com/spatial-bench/spatial-bench-core/blob/2104bfe8b2ba04c31ee484e9a9aee4f0c5df0f79/crates/spatial-bench-cli/src/main.rs)
and [browser loader at web 7551596](https://github.com/spatial-bench/spatial-bench-web/blob/755159672d2601f6eb5a40a16e9b0fb9c0e96b8a/src/engine/dataset.ts).