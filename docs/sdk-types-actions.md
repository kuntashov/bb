# Plugin SDK declaration builds in the fork

`.github/workflows/sdk-types.yml` runs the real SDK declaration builder on a
standard GitHub-hosted `ubuntu-24.04` runner in `kuntashov/bb`. It uses Node 22.x
(the repository CI version), pnpm from the target's `packageManager` (currently
9.15.0), and a frozen lockfile. Installation skips lifecycle scripts; Turbo runs
the declaration task and its declared generators explicitly:

```sh
pnpm exec turbo run build:types --filter=@get-bb/plugin-sdk --force --concurrency=1 --output-logs=full
```

The task invokes `packages/plugin-sdk/scripts/build-bundled-dts.mjs`. It does not
build the frontend. `--force` requires real execution instead of a Turbo cache
hit. No remote cache, custom runner, secrets, or new services are configured.
Do not run this build on user servers.

## Publication and invocation

Publish only after independent review and root approval of the exact commit.
The push trigger is restricted to `ci/sdk-types-github-hosted`; publishing that
branch runs the committed source. Existing inherited workflows may also respond
to pushes; this change does not alter them.

Manual dispatch requires this workflow to exist on the fork's default branch,
`main`. After root approves that specific update, fast-forward the reviewed
workflow/docs commit onto the unchanged fork `main` base. Recheck its SHA and
protection immediately before updating; stop on drift. Do not update upstream
`get-bb/bb`, other PR branches, protection rules, or the default-branch setting.
See [GitHub's manual dispatch requirements](https://docs.github.com/en/actions/how-tos/manage-workflow-runs/manually-run-a-workflow).

After publication on `main`, dispatch via the REST API:

```text
POST /repos/kuntashov/bb/actions/workflows/sdk-types.yml/dispatches
{"ref":"main","inputs":{"target_ref":"<approved full target SHA>"}}
```

The first approved baseline target is
`6a639cfc0d66e0c789a1bd7309266973228d1a80` (the fork's current `main`).
That run verifies the SDK build on GitHub, not an unpublished PR diff. PR #5778
is not the target of this setup. Record the actual checkout SHA even when a
branch name was supplied.

`target_ref` accepts a fork branch or reachable commit. Prefer a full approved
SHA for review evidence. Empty input builds the workflow ref. The workflow uses
the fork repository only; it does not copy another thread's working tree or
include unpublished changes. A PR update must be published by its owning thread
before its exact SHA can be used here. Concurrent runs for the same workflow ref
and target input cancel the older run.

## Acceptance

Record the run URL, workflow commit, actual checked-out SHA, runner identity,
conclusion, build exit, elapsed time, maximum RSS, and artifact URL. The job has
a 45 minute limit, installation 10 minutes, and the build 30 minutes. Success
requires the real Turbo build step to pass and every exported declaration file
to exist and be nonempty. Download the artifact and confirm the evidence and
declarations match the recorded SHA. YAML validation alone is not acceptance.

The `sdk-types-<run_id>-<attempt>` artifact contains provenance, lockfile digest,
build log, GNU time resource statistics, and generated declarations. Retention
is three days. Failed builds still attempt to upload evidence; a hard runner
termination can prevent upload. GitHub's step logs remain the source for install
or checkout failures. Generated types remain untracked and are never committed.
