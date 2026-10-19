# Contributing results

After following the engine's
[run guide](https://github.com/spatial-bench/spatial-bench-core/blob/main/docs/running-benchmarks.md),
inspect the completed records and propose them in a pull request. Adapter changes
should have their own
[bencher review](https://github.com/spatial-bench/spatial-bench-benchers/blob/master/CONTRIBUTING.md).

## Prepare a completed run

Records normally appear below
`${XDG_DATA_HOME:-$HOME/.local/share}/spatial-bench/runs/`.
[Inspect the workload, source and machine fields](docs/format-and-provenance.md)
before selecting runs for submission.

The PR should explain the workload selection, identify the runs and link evidence
that the adapter answers the intended query. Include the invocation with its seed,
engine/bencher revisions and generated dependency lockfiles. Record relevant
compiler/interpreter versions and thread/affinity settings. Retain logs and raw
samples needed to investigate the estimates. Explain failed or omitted cases,
selection among reruns and any intended replacement of existing records.

These are review expectations. Automated submission checks cover a smaller set
of [eligibility conditions](docs/format-and-provenance.md#submission-checks).

## Propose the results

With Git push access to this repository and an authenticated `gh`, the engine can
prepare and open the PR:

```sh
spatial-bench submit --to /path/to/clean/spatial-bench-results
```

> Use a dedicated, clean checkout whose local `main` is current. The command
> submits **all eligible local runs**, stages **all destination changes**, commits
> and pushes. It has no dry-run or run-ID filter.

The command copies records and extracts machine TOML, then creates a branch from
local `main` using `git checkout -B`. It does not fetch that branch first. `--to`
selects the checkout; PR creation still targets
`spatial-bench/spatial-bench-results`. Review the local run directory and
`git status` before invoking it.

Without central push access, prepare a normal fork PR. Preserve each run's
filename under `datasets/YYYY-MM/`. For a new machine, include
`machines/<machine_hash>.toml` containing `run.machine` converted to TOML with
null values omitted; compare an existing record with the run before reusing it.
Review the complete diff before committing.

If submission fails, inspect the checkout and remote branch before retrying.
Copying precedes branch creation, and PR creation can fail after a successful
push. An already pushed branch can be used to finish opening the PR.

## Review and publication

A reviewer should check the adapter's operation and timing boundary, source
identity, machine context and selection of measurements. Registration conformance
checks declared coverage; query-answer correctness needs separate evidence.
The [methodology](https://spatial-bench.org/methodology) defines the estimators and
controls to consider when comparing runs.

[Collate locally](docs/publication.md#build-a-local-database) and confirm that the
expected runs and cases appear. This checks parsing and database insertion.
After merge, publication must upload both the database and its pointer before
readers can load the new snapshot. Follow the
[publication verification](docs/publication.md#publish-after-review) procedure.

## Correct a result

Open an [issue](https://github.com/spatial-bench/spatial-bench-results/issues) or
PR with the snapshot revision, run ID and case tags. Explain the error and supply
a reproduction or corrected adapter. Maintainer review determines whether to add
a rerun, edit a record or remove it. Preserve the distinction between measured
results and corrected metadata.

Adding a run leaves earlier records intact. Editing or removing JSON changes the
next full collation; there is no exclusion flag or automatic purge of older
snapshots. Retention rules or mandatory independent reruns require a separate
policy decision. Changes to schema, metrics or publication should state their
documentation impact and identify coordinated engine or website changes.
