# Contributing results

Propose measurements with enough evidence for another person to inspect the
workload, driver, source identity and machine context. First follow the engine's
[run guide](https://github.com/spatial-bench/spatial-bench-core/blob/main/docs/running-benchmarks.md),
then inspect the completed JSON using [format and provenance](docs/format-and-provenance.md).
Library/driver changes belong in the
[benchers contribution workflow](https://github.com/spatial-bench/spatial-bench-benchers/blob/master/CONTRIBUTING.md).

## Prepare the review

Run documents normally live below
`${XDG_DATA_HOME:-$HOME/.local/share}/spatial-bench/runs/`. A run completing
successfully does not establish that its provenance is complete or its query
answers are correct. Before proposing it, record:

- The exact invocation, seed, engine and bencher commit IDs, selected manifests,
  library pins and generated build lockfiles. Keep raw samples/logs when they are
  needed to assess a discrepancy; they are not stored in the normal JSON.
- The machine hash and captured context, including missing/degraded fields;
  compiler/interpreter versions, affinity, thread controls and other settings
  not captured by the schema.
- Evidence that the driver answers the intended query, and that setup, allocation,
  result consumption and default/tuned labels describe what it actually did.
- How cases, configurations and repeated runs were selected. Explain omitted or
  failed cases and any intended replacement of prior results.

These are **review expectations**. The submit command does not enforce this
checklist, independent review, correct answers or stable machine conditions.
For statistical interpretation use the
[methodology](https://spatial-bench.org/methodology), especially the distinction
between stored mean, plotted median and within-run confidence bounds.

## Submit deliberately

`spatial-bench submit` is a remote write operation: it copies all eligible local
run documents, stages the destination repository, commits, pushes a branch, and
attempts to open a GitHub PR. It has no dry-run, run-ID filter or local-only mode.
Do not use it as an inspection command.

Use a dedicated, clean results checkout with a current local `main`. The tool
bases its branch on **local `main`**, does not fetch it, uses `git checkout -B`,
and stages **all** destination changes with `git add -A`. It copies documents
before changing branch. Protect unrelated work by using a fresh clone; confirm
what is in the local runs directory because previously completed eligible runs
are included too. `--to` selects the checkout, while PR creation still names
`spatial-bench/spatial-bench-results` explicitly; it is not a general repository
submission option.

Once the run set and destination are reviewed, this is the submission command
(from a shell with the built engine on `PATH`):

```sh
spatial-bench submit --to /path/to/clean/spatial-bench-results
```

It requires Git push access to the destination `origin` and an authenticated
`gh` with permission to open the PR. The standard workflow can supply those
credentials; never commit them. For a contributor without that push access,
prepare the JSON and extracted machine record on a normal fork branch and open a
PR using the usual GitHub workflow. Preserve generated filenames and inspect the
whole diff before committing.

Current eligibility checks are narrow: schema version 1, `run.source.git_dirty`
false, and a nonempty `sha` for every subject present. The current code permits a
missing or empty subject map and does not require a fingerprint, reject `UNKNOWN`
machine hashes, validate digest syntax, or check engine/bencher Git cleanliness.
Unreadable/malformed JSON is skipped. `git_dirty` currently marks subject-path
overrides, so false is not proof of a clean engine checkout. Do not edit these
fields to bypass a refusal; resolve the provenance gap and rerun as needed.
[Audited submission implementation](https://github.com/spatial-bench/spatial-bench-core/blob/2104bfe8b2ba04c31ee484e9a9aee4f0c5df0f79/crates/spatial-bench-cli/src/main.rs).

If there are no eligible runs, the command reports “no submittable runs found”.
If there are no new destination changes, it can return after switching to its
submission branch. A failure after copying/staging can leave local changes; a
push followed by a `gh` failure can leave a remote branch without a PR. Inspect
`git status`, the branch and remote state before retrying. Complete PR creation
for an already pushed branch rather than assuming the whole operation rolled
back.

## Review, merge and publication

A reviewer should compare the JSON and machine record, identify all measured
implementations, inspect source pins and look for unsupported methodological
claims. A populated fingerprint path is a recorded path, not cryptographic proof
of the machine or operator. Review the known
[provenance gaps](docs/format-and-provenance.md#what-the-record-cannot-establish).
Run the [local collation and inspection example](docs/publication.md#collate-locally)
to detect malformed files, duplicate run IDs and unexpected counts before merge.
Collation is a structural check, not a scientific review.

Merging changes to `main` starts the publisher. Success means the database object
and `latest.json` were uploaded; website availability additionally depends on its
Worker/bucket binding and the browser loading that snapshot. Follow the
[publication checks and recovery steps](docs/publication.md#maintainer-publication).

## Corrections, exclusions and removals

There is no current exclusion field, tombstone mechanism, automatic adjudication
policy or immutable-record enforcement in the publisher. It reads the JSON files
present at the selected revision. Adding a new run preserves the previous run;
editing or removing a document in a reviewed PR changes the next full collation.
Removing a source record does not itself delete older R2 snapshot objects.

Require maintainer review for a proposed correction or removal. Explain the
fault, identify affected run IDs and comparisons, supply the corrected adapter or
reproduction evidence, and state whether the PR adds a rerun or removes/replaces
old input. Do not silently relabel old measurements as though they had been
rerun. For disputed results, open an
[issue](https://github.com/spatial-bench/spatial-bench-results/issues) with the
snapshot revision, run ID and tags so the report remains identifiable.

A formal retention period, exclusion registry or requirement for independent
reruns would be a **policy proposal**, not an existing automated guarantee. If
needed, propose it separately with migration/publication consequences. Changes
to the result schema, metric interpretation or publication contract should state
their documentation impact and coordinate with engine and website maintainers.
