# Contributing results

Follow the engine's
[run guide](https://github.com/spatial-bench/spatial-bench-core/blob/main/docs/running-benchmarks.md)
first, then inspect the completed records and propose them in a pull request. If
your work also changed an adapter, that change needs its own
[bencher review](https://github.com/spatial-bench/spatial-bench-benchers/blob/master/CONTRIBUTING.md).

## Prepare a completed run

Records normally appear below
`${XDG_DATA_HOME:-$HOME/.local/share}/spatial-bench/runs/`. Before choosing runs to
submit, [inspect their workload, source and machine fields](docs/format-and-provenance.md).

The pull request should explain the workload selection, identify the runs it adds,
and link evidence that the adapter answers the intended query. Include the
invocation with its seed, the engine and bencher revisions it used, and the
generated dependency lockfiles. Record the compiler or interpreter versions and any
thread or affinity settings. Keep the logs and raw samples that would be needed to
investigate the estimates, and explain failed or omitted cases, how you chose among
reruns, and whether existing records are meant to be replaced.

These are review expectations, not what the tooling enforces. Automated submission
checks cover a smaller set of
[eligibility conditions](docs/format-and-provenance.md#submission-checks).

## Propose the results

If you have Git push access to this repository and an authenticated `gh`, the engine
can prepare and open the PR for you:

```sh
spatial-bench submit --to /path/to/clean/spatial-bench-results
```

> Use a dedicated, clean checkout whose local `main` is current. The command
> submits **all eligible local runs**, stages **all destination changes**, commits
> and pushes, and has no dry-run mode or run-ID filter.

The command copies records and extracts machine TOML, then creates a branch from
local `main` with `git checkout -B`. It does not fetch that branch first, so make
sure the checkout is up to date. `--to` selects the destination checkout, while PR
creation still targets `spatial-bench/spatial-bench-results`. Review the local run
directory and `git status` before running it.

Without central push access, prepare an ordinary fork PR. Preserve each run's
filename under `datasets/YYYY-MM/`. For a new machine, include
`machines/<machine_hash>.toml` containing `run.machine` converted to TOML with null
values omitted, and compare it against an existing record before reusing any of it.
Review the complete diff before committing.

If submission fails, inspect the checkout and remote branch before retrying: the
copy happens before branch creation, and PR creation can fail even after a
successful push. An already-pushed branch can be used to finish opening the PR.

## Review and publication

A reviewer checks the adapter's operation and timing boundary, the source identity,
the machine context, and whether the right measurements were selected. Registration
conformance verifies declared coverage; query-answer correctness needs separate
evidence. The [methodology](https://spatial-bench.org/methodology) defines the
estimators and controls to keep in mind when comparing runs.

[Collate the database locally](docs/publication.md#build-a-local-database) and
confirm the expected runs and cases appear, which exercises both parsing and
database insertion. After merge, publication has to upload the database and its
pointer before readers can load the new snapshot; follow the
[publication verification](docs/publication.md#publish-after-review) procedure.

## Correct a result

Open an [issue](https://github.com/spatial-bench/spatial-bench-results/issues) or a
PR with the snapshot revision, run ID and case tags, explain the error, and supply a
reproduction or a corrected adapter. It is maintainer review that decides whether
the answer is a rerun, an edit to the record, or a removal, and the distinction
between a measured result and corrected metadata should be preserved either way.

Adding a run leaves earlier records intact. Editing or removing JSON changes the
next full collation, but there is no exclusion flag and no automatic purge of older
snapshots. Retention rules or mandatory independent reruns would require a separate
policy decision. Changes to schema, metrics or publication should state their
documentation impact and identify any coordinated engine or website changes.