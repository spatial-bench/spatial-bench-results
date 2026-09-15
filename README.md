# spatial-bench-results

Run records and machine information behind [spatial-bench](https://spatial-bench.org).
The repository holds the reviewable input to the published comparison database;
Git history records changes to that input. The engine measures workloads, the
[benchers repository](https://github.com/spatial-bench/spatial-bench-benchers)
defines library adapters, and the [website](https://spatial-bench.org/explore)
displays a collated snapshot.

- **Interpret a comparison:** [reading guide](https://spatial-bench.org/guide) and
  [methodology](https://spatial-bench.org/methodology).
- **Trace a plotted point:** [format and provenance](docs/format-and-provenance.md).
- **Propose or correct results:** [contributing](CONTRIBUTING.md).
- **Build and inspect a snapshot locally:** [publication](docs/publication.md).

`datasets/YYYY-MM/*.json` contains one document per run, including its points,
subject identities and captured environment. `machines/<machine-hash>.toml`
contains extracted machine information. Neither path contains the generated
construction points used by the benchmark.

The [publication workflow](.github/workflows/publish.yml) runs on pushes to `main`
and manual dispatch. It collates JSON/TOML into SQLite, uploads
`benchmarks-<12-character-results-revision>.sqlite.gz` to Cloudflare R2, then
updates `latest.json`. Its SHA-256 covers the **uncompressed SQLite bytes**;
`bytes` describes the compressed file. A merge is not evidence of successful
publication: inspect the workflow and pointer before expecting new website data.

Documentation verified on 14 September 2026 against results `895a701` and engine
`2104bfe`. The local example collates that results revision into 50 runs, 1,512
points and one machine; these counts describe the audited snapshot, not a fixed
project capacity or a current live-site assertion.
