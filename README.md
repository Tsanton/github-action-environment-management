# **GitHub Actions environment management**

This repository holds reusable workflows and composite actions that are securely piped secrets and environent variables to dynamically configure actions.

## **Reusable workflows**

Reusable workflows must be invocated with `"secrets: inherit"` and `"env: ${{ secrets }}"` must be configured for downstream actions utilised by the reusable workflow:

```yaml
### Invoking workflow ###
jobs:
  ci:
    name: invocation
    uses: ./.github/workflows/reusable-secret-env.yaml
    # 'secrets: inherit' along with the 'env: ${{ secrets }}' in downstream actions will make all secrets available as env variables in downstream actions
    secrets: inherit
    with:
      env_vars: |
        ENV_VARIABLE_FOO=HELLO
        ENV_VARIABLE_BAR=WORLD
---
### Reusable workflow ###
name: secret-env

on:
  workflow_call:
    inputs:
      env_vars:
        description: A literal block scalar of environment variables to set up, given in env=value format.
        required: false
        type: string
jobs:
  build:
    name: secret-env-variables
    runs-on: ubuntu-latest
    steps:
      - name: Set environment variables
        if: ${{ inputs.env_vars }}
        run: |
          for i in "${{ inputs.env_vars }}"
          do
            printf "%s\n" $i >> $GITHUB_ENV
          done

      - name: echo environment variables
        env: ${{ secrets }}
        run: |
          printenv
```

The output from the **echo environment variables** action is as follows:

```txt
printenv
shell: /usr/bin/bash -e {0}
env:
    ENV_VARIABLE_FOO: HELLO
    ENV_VARIABLE_BAR: WORLD
    github_token: ***
    RANDOM_TEST_SECRET: ***
```

Where **RANDOM_TEST_SECRET** is a secret that lives in the context of the repository

**`As we can see: no secrets are leaked.`**


## **Reusable workflows with composite actions**

The behaviour is the same when utilizing reusable workflows calling composite actions. \
You must only pipe `"env: ${{ secrets }}"` to the invocation of the composite action. \
This entails that you do **`not`** have to setup `env` for the separate downstream actions orchastrated within the composite action:

```yaml

### Reusable workflow ###
name: reusable-secret-env-composite

on:
  workflow_call:
    inputs:
      env_vars:
        description: A literal block scalar of environment variables to set up, given in env=value format.
        required: false
        type: string
jobs:
  build:
    name: secret-env-variables
    runs-on: ubuntu-latest
    steps:
      - name: echo env secrets
        uses: tsanton/github-action-environment-management/actions/echo-env-secrets@main
        env: ${{ secrets }}
        with:
          env_vars: ${{ inputs.env_vars }}
---
### Composite action ###
name: echo-env-secret
description: Ensure that we're able to pass secrets as environment variables to steps in composite actions used by reusable workflows

inputs:
  env_vars:
    description: A literal block scalar of environment variables to set up, given in env=value format.
    required: false
    type: string

runs:
  using: "composite"
  steps:
    - name: Set environment variables
      if: ${{ inputs.env_vars }}
      shell: bash
      run: |
        for i in "${{ inputs.env_vars }}"
        do
          printf "%s\n" $i >> $GITHUB_ENV
        done

    - name: echo environment variables
      shell: bash
      run: |
        printenv
```

Again, the output from the **echo environment variables** action is as follows:

```txt
printenv
shell: /usr/bin/bash -e {0}
env:
    ENV_VARIABLE_FOO: HELLO
    ENV_VARIABLE_BAR: WORLD
    github_token: ***
    RANDOM_TEST_SECRET: ***
```

Where **RANDOM_TEST_SECRET** is a secret that lives in the context of the repository. 

**`As before: we can see that no secrets are leaked.`**