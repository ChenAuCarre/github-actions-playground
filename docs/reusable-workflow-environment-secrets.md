# Environment Secrets in Reusable Workflows

This note records an observed GitHub Actions behavior when a reusable workflow
uses environment secrets. It compares two runs of the same caller/called
workflow pair before stating the conclusion.

## Observation 1: Environment Secrets Were Empty

[Run 34585555638](https://github.com/ChenAuCarre/github-actions-playground/actions/runs/34585555638)
ran from commit
[`5914aee`](https://github.com/ChenAuCarre/github-actions-playground/commit/5914aee7360816d635b796dcc689933286211ead).

The caller passed only the repository secret:

```yaml
jobs:
  pass-secrets:
    uses: ./.github/workflows/read-secrets.yml
    secrets:
      MY_CREDENTIAL: ${{ secrets.MY_CREDENTIAL }}
```

The called workflow declared only that secret in its `workflow_call` contract:

```yaml
on:
  workflow_call:
    secrets:
      MY_CREDENTIAL:
        description: "The credential to read."
        required: true
```

Its job selected the `debug` environment and attempted to read two additional
secrets:

```yaml
jobs:
  read-secrets:
    runs-on: ubuntu-latest
    environment: debug
    steps:
      - name: Read secret
        env:
          MY_CREDENTIAL: ${{ secrets.MY_CREDENTIAL }}
          MY_VAULT_ROLE_ID: ${{ secrets.MY_VAULT_ROLE_ID }}
          MY_VAULT_SECRET_ID: ${{ secrets.MY_VAULT_SECRET_ID }}
```

Both Vault secrets existed in the `debug` environment, but the log reported:

```text
MY_VAULT_ROLE_ID is set: no
MY_VAULT_SECRET_ID is set: no
```

In this run, selecting `environment: debug` did not by itself make undeclared
secret names available through the reusable workflow's `secrets` context.

## Observation 2: Declared and Passed Secret Names Worked

[Run 34586295566](https://github.com/ChenAuCarre/github-actions-playground/actions/runs/34586295566)
ran from commit
[`15a80b2`](https://github.com/ChenAuCarre/github-actions-playground/commit/15a80b2a7705468f0488057749853bc765ff0bcf).

By this run, the called workflow declared all three secret names:

```yaml
on:
  workflow_call:
    secrets:
      MY_CREDENTIAL:
        required: true
      MY_VAULT_ROLE_ID:
        required: true
      MY_VAULT_SECRET_ID:
        required: true
```

The caller also supplied all three names:

```yaml
jobs:
  pass-secrets:
    uses: ./.github/workflows/read-secrets.yml
    secrets:
      MY_CREDENTIAL: ${{ secrets.MY_CREDENTIAL }}
      MY_VAULT_ROLE_ID: ${{ secrets.MY_VAULT_ROLE_ID }}
      MY_VAULT_SECRET_ID: ${{ secrets.MY_VAULT_SECRET_ID }}
```

The caller job did not select the `debug` environment, so the caller itself did
not gain access to that environment merely because the called job used it.
Nevertheless, passing the names satisfied the called workflow's required-secret
contract.

The called job still selected `environment: debug`. At execution time, GitHub
made the `debug` environment secrets available to that job. For matching names,
the environment values took precedence over values passed by the caller. The
Vault secrets were therefore set in this run.

## Conclusion

For this reusable workflow, `environment: debug` was not sufficient on its own.
The successful configuration did all three of the following:

1. Declared each secret name under `on.workflow_call.secrets` in the called
   workflow.
2. Passed each declared name from the caller under `jobs.<job_id>.secrets`.
3. Selected the `debug` environment on the job inside the called workflow.

This separates two responsibilities:

- `on.workflow_call.secrets` and the caller's `secrets` mapping define and
  satisfy the reusable workflow's secret interface.
- `jobs.<job_id>.environment` determines which environment-scoped values are
  available when the called job executes.

When the caller-provided secret and the called job's environment secret use the
same name, GitHub uses the environment secret in the called job. In this case,
the caller mappings establish the reusable workflow contract, while the actual
Vault values come from the `debug` environment.

This conclusion is based on the two runs above and their corresponding workflow
commits. It should be read as an observed behavior of this configuration, with
the environment-secret precedence rule matching GitHub's reusable workflow
documentation.
