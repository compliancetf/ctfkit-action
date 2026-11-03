# ctfkit by compliance.tf: GitHub Action

Add this action to a pull request workflow and every Terraform or OpenTofu change it runs on gets
the same compliance checks. Findings name the compliance.tf module that closes the gap. The action
installs a pinned ctfkit release, runs it over what your workflow produced, writes a report to the
job summary, uploads the findings to the Security tab when `sarif` is on and, if you ask, keeps a
comment on the pull request up to date.

Status: **preview**. The action is public. The ctfkit binary it downloads is published in the
private `compliancetf/ctfkit` repository and is available to compliance.tf customers:
[request access](https://compliance.tf/contact/?category=ctfkit).

Full documentation: https://compliance.tf/docs/ctfkit/github-actions/

## Quick start

1. Request access. Once approved, your GitHub user can read the `compliancetf/ctfkit` releases.
2. Create a fine-grained personal access token: resource owner `compliancetf`, repository
   `ctfkit`, permission `Contents: read`.
3. Store it as a repository or organization secret, for example `CTFKIT_RELEASE_TOKEN`.

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
      - uses: compliancetf/ctfkit-action@v0.3.0
        with:
          command: scan
          stage: pre_plan
          chdir: infra
          version: v0.3.0-preview.2 # the ctfkit release to install
          github-token: ${{ secrets.CTFKIT_RELEASE_TOKEN }}
          pr-comment: 'true'
          sarif: 'false' # 'true' when the repository has code scanning
```

The job fails on findings at or above `fail-on` (default `error`). Start a rollout with
`fail-on: never` to see the findings without blocking anyone. Pull requests from forks get no
secrets, so the download step fails there; the documentation covers fork setups.

## Commands

| `command` | Works on | Run before it |
|---|---|---|
| `scan` with `stage: pre_plan` | configuration in `chdir` | checkout only |
| `scan` | plan JSON from `show -json` | `init`, `plan`, `show -json` |
| `report` (default) | the module cache in `path`, plus `test -json` output if you pass it | `init` |
| `plan` (preview) | configuration in `chdir`; creates the artifact set in `artifacts` | checkout, terraform or tofu installed |
| `sign`, `apply` (preview) | the artifact set in `artifacts` | the previous job's artifact downloaded, the same terraform or tofu binary as `plan` |

`plan`, `sign` and `apply` split a governed apply across three jobs. `plan` writes the plan and a
candidate, `sign` adds a verdict signed by the `signer` you name, and `apply` applies the sealed
plan. With an AWS KMS key (`awskms:<key arn>`) the signer reads the default AWS credential chain,
so assume a role with OIDC in an earlier step. `sign` and `apply` refuse a set with a changed,
missing or extra file, since each file must match its digest in the candidate, and `apply` also
checks that the verdict signature binds the plan. The
[documentation](https://compliance.tf/docs/ctfkit/github-actions/) links to a copyable three-job
workflow in the ctfkit repository (readable with access).

## What a pull request cannot change

The repository under analysis is treated as untrusted input:

- `fail-on` is always passed to ctfkit (`report`, `scan` and `sign`), and so are `config`,
  `exceptions` and `catalog` (`report` and `scan`). ctfkit never discovers them in the checkout,
  so a committed `.ctfkit.yaml` or exception file cannot lower the threshold or grant itself an
  exception. A `config`, `exceptions` or `catalog` path you set must name a file the pull request
  cannot change. `config` and `exceptions` default to the files in [`defaults/`](defaults):
  every setting at the ctfkit default, and an empty exception registry. `catalog` defaults to
  `embedded`, the catalog compiled into the pinned binary. `plan`, `sign` and `apply` take their
  policy from signed inputs: `profile`, `floor` and `bundle-dir`.
- In the `report` and `scan` output, module sources and finding messages are escaped before they
  reach the job summary or the comment, so they cannot add HTML or an `@` mention, or break out of
  a table or code span.
- Workflow commands are switched off while ctfkit runs, so its log output cannot set an output or
  write an annotation.
- The comment step only updates a comment its own token wrote. There is one comment per command
  and directory, so a matrix over several roots keeps one each.

## Pinning the binary

`version` is required and has no default: pin the release you reviewed. The archive is checked
against the release `checksums.txt` before it is unpacked. That file comes from the same release
and token, so it proves the download is intact, not who published it. Two optional inputs go
further:

- `sha256`: the digest of the archive you reviewed. A different archive fails the step.
- `verify-attestation: 'true'`: checks the GitHub build provenance attestation from the ctfkit
  release workflow for that tag. Current releases have no attestation, so turning it on fails the
  step today; it is off by default.

`verify-lock: 'true'` also runs `ctfkit verify-lock` before the command and fails the step when a
lock file lacks the hashed compliance.tf wrapper entry or still installs the upstream provider it
wraps.

## Results

The job summary, the Security tab upload and the pull request comment run before the exit code is
applied, so a failing run still shows what it found. Each one runs only when there is something to
publish and its input is on.

| Output | Value |
|---|---|
| `receipt` | path to the evidence receipt; `report` only |
| `sarif` | path to the SARIF findings |
| `markdown` | path to the markdown written to the job summary |
| `pr-comment` | `created`, `updated`, `skipped` or `failed` |
| `exit-code` | `0` ok; `1` findings at or above `fail-on`, or a `plan`, `sign` or `apply` that did not pass; `2` refusal or environment error |

The receipt records which controls the compliance.tf modules in the configuration enforce, as read
from the manifests written at `init`. It is not a live-state assertion and not an attestation by
compliance.tf.

Every input, the multi-root setup and the fork rules are in the
[documentation](https://compliance.tf/docs/ctfkit/github-actions/).

## License

Apache-2.0
