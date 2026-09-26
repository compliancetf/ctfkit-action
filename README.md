# ctfkit by compliance.tf: GitHub Action

This action runs ctfkit by compliance.tf in GitHub Actions. ctfkit runs the same Terraform and
OpenTofu checks in every pipeline and names the compliance.tf module that closes each gap it finds.

Status: **preview**. The action is public. The ctfkit binary it downloads is published in a private
repository and is available to compliance.tf customers on request:
[request access](https://compliance.tf/contact/?category=ctfkit).

Full documentation: https://docs.compliance.tf/ctfkit/github-actions/

## Before you start

1. Request access. Once approved, your GitHub user can read the `compliancetf/ctfkit` releases.
2. Create a fine-grained personal access token with resource owner `compliancetf`, repository
   `ctfkit` and the `Contents: read` permission.
3. Store it as a repository or organization secret, for example `CTFKIT_RELEASE_TOKEN`.

## Example

```yaml
on: pull_request

permissions:
  contents: read
  pull-requests: write # only for pr-comment

jobs:
  ctfkit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: compliancetf/ctfkit-action@v0.2.1
        with:
          command: scan
          stage: pre_plan
          chdir: infra
          version: v0.2.1
          github-token: ${{ secrets.CTFKIT_RELEASE_TOKEN }}
          pr-comment: 'true'
          sarif: 'false' # set 'true' when the repository has code scanning
```

The action checks the downloaded archive against the release `checksums.txt` before it runs. The
job fails on findings at or above `fail-on` (default `error`).

Inputs, outputs, the `report` command and multi-root setups are described in the
[documentation](https://docs.compliance.tf/ctfkit/github-actions/).

## License

Apache-2.0
