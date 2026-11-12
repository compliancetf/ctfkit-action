# ctfkit by compliance.tf: GitHub Action

Add this action to a pull request workflow and every Terraform or OpenTofu change it runs on gets
the same compliance checks. Findings name the compliance.tf module that closes the gap. The action
installs a ctfkit release, the newest unless you pin one, runs it over what your workflow produced, writes a report to the
job summary, uploads the findings to the Security tab when `sarif` is on and, if you ask, keeps a
comment on the pull request up to date.

The action and ctfkit are open source: the action downloads the binary from
the public releases of [`compliancetf/ctfkit`](https://github.com/compliancetf/ctfkit) with the
default job token, so a workflow needs no extra secret.

Full documentation: https://compliance.tf/docs/ctfkit/github-actions/

## Quick start

This workflow scans configuration on every pull request. It needs no `init`, no plan and no cloud
credentials:

```yaml
on: pull_request

permissions:
  contents: read
  pull-requests: write # only for pr-comment
  # security-events: write # add with sarif: 'true'

jobs:
  ctfkit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: compliancetf/ctfkit-action@v0
        with:
          command: scan
          stage: pre_plan
          chdir: infra
          pr-comment: 'true'
          sarif: 'false' # 'true' when the repository has code scanning
```

The job fails on findings at or above `fail-on` (default `error`). Start a rollout with
`fail-on: never` to see the findings without blocking anyone. Pull requests from forks get a
read-only job token: the download still works, the comment and the Security tab upload do not; the
documentation covers fork setups.

## Commands

| `command` | Works on | Run before it |
|---|---|---|
| `scan` with `stage: pre_plan` | configuration in `chdir` | checkout only |
| `scan` | plan JSON from `show -json` | `init`, `plan`, `show -json` |
| `report` (default) | the module cache in `path`, plus `test -json` output if you pass it | `init` |
| `plan` | configuration in `chdir`; creates the artifact set in `artifacts` | checkout, terraform or tofu installed |
| `sign`, `apply` | the artifact set in `artifacts` | the previous job's artifact downloaded, the same terraform or tofu binary as `plan` |

`plan`, `sign` and `apply` split a governed apply across three jobs. `plan` writes the plan and a
candidate, `sign` adds a verdict signed by the `signer` you name, and `apply` applies the sealed
plan. With an AWS KMS key (`awskms:<key arn>`) the signer reads the default AWS credential chain,
so assume a role with OIDC in an earlier step. `sign` and `apply` refuse a set with a changed,
missing or extra file, since each file must match its digest in the candidate, and `apply` also
checks that the verdict signature binds the plan. The
[documentation](https://compliance.tf/docs/ctfkit/github-actions/) links to a copyable three-job
workflow in the ctfkit repository.

## What a pull request cannot change

The repository under analysis is treated as untrusted input:

- `fail-on` is always passed to ctfkit (`report`, `scan` and `sign`), and so are `config`,
  `exceptions` and `catalog` (`report` and `scan`). ctfkit never discovers them in the checkout,
  so a committed `.ctfkit.yaml` or exception file cannot lower the threshold or grant itself an
  exception. A `config`, `exceptions` or `catalog` path you set must name a file the pull request
  cannot change. `config` and `exceptions` default to the files in [`defaults/`](defaults):
  every setting at the ctfkit default, and an empty exception registry. `catalog` defaults to
  `embedded`, the catalog compiled into the pinned binary (the CIS AWS Benchmark v6.0 controls).
  `catalog: pull` downloads the full catalog with `ctfkit catalog pull` (ctfkit 0.7.0 or later, a
  paid plan, and the registry token in `TF_TOKEN_registry_compliance_tf` on the step) into a
  directory outside the output directory, and deletes it when the step ends. A failed pull fails
  the step with exit code `2`. `plan`, `sign` and `apply` take their policy from signed inputs:
  `profile`, `floor` and `bundle-dir`.
- In the `report` and `scan` output, module sources and finding messages are escaped before they
  reach the job summary or the comment, so they cannot add HTML or an `@` mention, or break out of
  a table or code span.
- Workflow commands are switched off while ctfkit runs, so its log output cannot set an output or
  write an annotation.
- The comment step only updates a comment its own token wrote. There is one comment per command
  and directory, so a matrix over several roots keeps one each.

## Pinning

`@v0` follows the newest v0 release of this action, and `version` defaults to `latest`, the newest
ctfkit release when the job runs, so the quick start never goes stale. For a reproducible
pipeline, pin both: the action to a release commit SHA, which Dependabot can bump, and `version`
to the ctfkit release you reviewed:

```yaml
      - uses: compliancetf/ctfkit-action@<release commit SHA> # vX.Y.Z
        with:
          version: v1.2.3 # the ctfkit release you reviewed
```

The `ctfkit-version` output names the release that was installed, with `latest` resolved; pass it
to later jobs that must run the same release. The archive is checked
against the release `checksums.txt` before it is unpacked. That file comes from the same release
and token, so it proves the download is intact, not who published it. Two optional inputs go
further:

- `sha256`: the digest of the archive you reviewed. A different archive fails the step. It needs a
  pinned `version`.
- `verify-attestation: 'true'`: checks the GitHub build provenance attestation from the ctfkit
  release workflow for that tag. Every ctfkit release carries one; a release from a private copy
  without that workflow fails the check, so it is off by default.

`verify-lock: 'true'` also runs `ctfkit verify-lock` before the command and fails the step when a
lock file lacks the hashed compliance.tf wrapper entry or still installs the upstream provider it
wraps.

## Results

The job summary, the Security tab upload and the pull request comment run before the exit code is
applied, so a failing run still shows what it found. Each one runs only when there is something to
publish and its input is on.

| Output | Value |
|---|---|
| `ctfkit-version` | the ctfkit release installed, such as `v1.2.3` |
| `receipt` | path to the evidence receipt; `report` only |
| `sarif` | path to the SARIF findings |
| `markdown` | path to the markdown written to the job summary |
| `pr-comment` | `created`, `updated`, `skipped` or `failed` |
| `exit-code` | `0` ok; `1` findings at or above `fail-on`, or a `plan`, `sign` or `apply` that did not pass; `2` refusal or environment error |

The receipt records which controls the compliance.tf modules in the configuration enforce, as read
from the manifests written at `init`. It is not a live-state assertion and not an attestation by
compliance.tf.

## Evidence pack on the release

`report` (ctfkit 0.8.0 or later) also writes the evidence pack: `evidence-pack.pdf` for an auditor
to open directly, and `evidence-pack.zip` with the PDF, the pack as Markdown and JSON, the module
manifests, the receipt and the SARIF. With `release-assets: 'true'` the action uploads both files
to the GitHub release the workflow runs for, so QA staff and auditors download them from the
release page:

```yaml
on:
  release:
    types: [published]

permissions:
  contents: write # to upload release assets

jobs:
  ctfkit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: tofu init -input=false
      - uses: compliancetf/ctfkit-action@v0
        with:
          release-assets: 'true'
```

- A `release` event counts, and so does a push of a tag whose release exists. Any other run with
  `release-assets: 'true'` fails the step.
- The upload uses `github-token`, which needs `contents: write`. A re-run replaces the files.
- A report with findings (exit `1`) still uploads its pack. After any other exit code nothing is
  uploaded and the step fails.
- The pack is not signed.

Every input, the multi-root setup and the fork rules are in the
[documentation](https://compliance.tf/docs/ctfkit/github-actions/).

## License

Apache-2.0
