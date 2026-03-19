# Nox Action

This action runs a [nox](https://github.com/wntrblm/nox/) session.

> [!WARNING]
> This action executes `noxfile.py` and potentially build scripts
> (`setup.py`/`pyproject.toml`) from the working directory. It **MUST NOT** be
> used with `pull_request_target` workflows that check out untrusted code.
> Doing so could allow an attacker to execute arbitrary code with the
> workflow's permissions and secrets.

Here is an example demonstrating how to use it in a workflow with a matrix job:

```yaml
jobs:
  nox:
    name: Test with nox
    permissions:
      # Required for the checkout step, not this action itself.
      contents: read
    strategy:
      fail-fast: false
      matrix:
        os:
          - ubuntu-slim
        python-version:
          - "3.11"
        nox-session:
          # To speed things up a bit we use the special ci_checks_max session
          # that uses the same venv to run multiple linting sessions
          - "ci_checks_max"
          - "pytest_min"
    runs-on: ${{ matrix.os }}

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Run nox
        uses: frequenz-floss/gh-action-nox@<hash> # v1.0.0
        with:
          python-version: ${{ matrix.python-version }}
          nox-session: ${{ matrix.nox-session }}
```

## Inputs

This action expects the workflow to check out the repository before running
`nox`.

* `python-version`: The python version to use. Required.

  This is passed to the
  [`actions/gh-action-setup-python-with-deps`](https://github.com/frequenz-floss/gh-action-setup-python-with-deps/)
  action.

* `nox-session`: The nox session to run. Required.

* `nox-dependencies`: The dependencies to install using `pip` to run `nox`.
  Optional. Default: `".[dev-noxfile]"`.

  Projects not having any extra dependency to run nox can just use `"nox"` here.

## Permissions

This action does not require any GitHub token permissions by itself.

If the calling workflow uses `actions/checkout`, grant whatever permissions that
step needs separately.

## Recommended use with matrix jobs

When using a matrix, it is recommended to create a dummy job to *merge* all the
matrix jobs, specially if you want to require all matrix jobs to pass to allow
merging a pull request. If you do this, you only need to add the dummy job as
a requirement and you don't need to update your requirements each time you
update your matrix.

```yaml
  # This job runs if all the `nox` matrix jobs ran and succeeded.
  # It is only used to have a single job that we can require in branch
  # protection rules, so we don't have to update the protection rules each time
  # we add or remove a job from the matrix.
  nox-all:
    # The job name should match the name of the `nox` job.
    name: Test with nox
    needs: ["nox"]
    # We skip this job only if nox was also skipped
    if: always() && needs.nox.result != 'skipped'
    runs-on: ubuntu-slim
    env:
      DEPS_RESULT: ${{ needs.nox.result }}
    steps:
      - name: Check matrix job result
        run: test "$DEPS_RESULT" = "success"
```
