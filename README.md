# spatial-bench-results

This repository stores the measurements recorded by spatial-bench. Each run keeps
its measured cases and estimates alongside the library identities and machine
context that produced them, and the website reads a SQLite snapshot built from
those records.

- [Inspect a measurement](docs/format-and-provenance.md) to trace a plotted point
  back to its workload, source and machine.
- [Contribute results](CONTRIBUTING.md) to propose completed runs or correct an
  existing record.
- [Build and publish a snapshot](docs/publication.md) to collate the database
  locally or work on publication.

Run documents live in `datasets/YYYY-MM/*.json`, and extracted machine information
in `machines/*.toml`. On every push to `main`, the
[publication workflow](.github/workflows/publish.yml) collates them and uploads a
gzip-compressed SQLite file together with its `latest.json` pointer.

For help interpreting a comparison, read the
[guide](https://spatial-bench.org/guide) and
[methodology](https://spatial-bench.org/methodology). Running benchmarks belongs to
[the engine](https://github.com/spatial-bench/spatial-bench-core) and writing
adapters to the
[benchers repository](https://github.com/spatial-bench/spatial-bench-benchers).