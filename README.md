# spatial-bench-results

Recorded measurements behind [spatial-bench](https://spatial-bench.org). Each run
contains its measured cases, estimates, library identities and machine context.
The website reads a SQLite snapshot built from these records.

- [Inspect a measurement](docs/format-and-provenance.md) to trace a plotted point
  to its workload, source and machine.
- [Contribute results](CONTRIBUTING.md) to propose completed runs or correct an
  existing record.
- [Build and publish a snapshot](docs/publication.md) to inspect the database
  locally or maintain publication.

Run documents are in `datasets/YYYY-MM/*.json`; extracted machine information is
in `machines/*.toml`. The [publication workflow](.github/workflows/publish.yml)
collates them on pushes to `main`, then uploads a gzip-compressed SQLite file and
its `latest.json` pointer.

For interpreting comparisons, read the [guide](https://spatial-bench.org/guide)
and [methodology](https://spatial-bench.org/methodology). Benchmark execution
belongs to [the engine](https://github.com/spatial-bench/spatial-bench-core);
library adapters belong to
[the benchers repository](https://github.com/spatial-bench/spatial-bench-benchers).
