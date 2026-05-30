# Reusable Build Workflows

This repository contains reusable GitHub Actions workflows for common build pipelines:

- Node.js build
- Python build and test
- Maven build
- Docker image build and optional push

It also includes a custom composite action at `.github/actions/setup-build` for repositories that want one shared setup step inside their own workflows.

## Repository Layout
### Customized Actions

docker-build-push
      action.yml
ssh-deploy
      action.yml

### Workflow
  workflows/
    nodejs-build.yml
    python-build.yml
    maven-build.yml
    docker-build.yml


## How To Use

Reference these workflows from another repository with `uses`.

```yaml
name: Node CI

on:
  pull_request:
  push:
    branches: [main]

jobs:
  node:
    uses: your-org/reusable-workflow-repo/.github/workflows/nodejs-build.yml@main
    with:
      node-version: "20"
      package-manager: npm
      build-command: npm run build
      test-command: npm test
```

For private repositories or workflows that need secrets, add:

```yaml
    secrets: inherit
```

## Custom Action

The composite action can be called directly by repositories that want shared setup without calling a full reusable workflow:

```yaml
steps:
  - uses: your-org/reusable-workflow-repo/.github/actions/setup-build@v1.0.0
    with:
      language: node
      node-version: "20"
      package-manager: npm
```

The reusable workflows below are self-contained because GitHub resolves local `./.github/actions/...` paths against the caller repository after checkout.

## Workflows

### Node.js

`.github/workflows/nodejs-build.yml`

Inputs:

| Input | Default | Description |
| --- | --- | --- |
| `node-version` | `20` | Node.js version. |
| `package-manager` | `npm` | One of `npm`, `yarn`, or `pnpm`. |
| `working-directory` | `.` | Directory containing the Node project. |
| `install-command` | Auto-detected | Override dependency install command. |
| `build-command` | `npm run build --if-present` | Build command. |
| `test-command` | `npm test --if-present` | Test command. |
| `pre-build-command` | empty | Optional command before install/build. |

### Python

`.github/workflows/python-build.yml`

Inputs:

| Input | Default | Description |
| --- | --- | --- |
| `python-version` | `3.12` | Python version. |
| `working-directory` | `.` | Directory containing the Python project. |
| `requirements-file` | `requirements.txt` | Requirements file to install when present. |
| `install-command` | empty | Optional install command override/addition. |
| `lint-command` | empty | Optional lint command. |
| `test-command` | `pytest` | Test command. |
| `pre-build-command` | empty | Optional command before install/test. |

### Maven

`.github/workflows/maven-build.yml`

Inputs:

| Input | Default | Description |
| --- | --- | --- |
| `java-version` | `17` | Java version. |
| `distribution` | `temurin` | Java distribution for `actions/setup-java`. |
| `working-directory` | `.` | Directory containing the Maven project. |
| `maven-command` | `mvn -B verify` | Maven command. |
| `pre-build-command` | empty | Optional command before Maven runs. |

### Docker

`.github/workflows/docker-build.yml`

Inputs:

| Input | Default | Description |
| --- | --- | --- |
| `image-name` | required | Image name, for example `ghcr.io/org/app`. |
| `registry` | empty | Optional registry host for login, for example `ghcr.io`. |
| `context` | `.` | Docker build context. |
| `dockerfile` | `Dockerfile` | Dockerfile path. |
| `platforms` | `linux/amd64` | Build platforms. |
| `push` | `false` | Whether to push the image. |
| `tags` | empty | Optional docker/metadata-action tag rules. |
| `pre-build-command` | empty | Optional command before Docker build. |

Secrets:

| Secret | Required | Description |
| --- | --- | --- |
| `registry-username` | false | Registry username. |
| `registry-password` | false | Registry password/token. |

## Versioning Recommendation

Create immutable releases such as `v1.0.0` and have application repositories call that tag:

```yaml
uses: your-org/reusable-workflow-repo/.github/workflows/maven-build.yml@v1.0.0
```

Use `main` only when callers intentionally want the latest workflow changes.


# Reusable Terraform GitHub Actions Workflow

Create this file in the shared workflow repository:

`.github/workflows/terraform.yml`

```yaml
name: Reusable Terraform

on:
  workflow_call:
    inputs:
      working-directory:
        description: Directory containing Terraform configuration.
        required: false
        type: string
        default: .
      terraform-version:
        description: Terraform CLI version or version constraint.
        required: false
        type: string
        default: 1.9.8
      run-apply:
        description: Run terraform apply after plan.
        required: false
        type: boolean
        default: false
      plan-artifact-name:
        description: Name for uploaded Terraform plan artifact.
        required: false
        type: string
        default: terraform-plan
    secrets:
      AWS_ROLE_TO_ASSUME:
        required: false
      AWS_REGION:
        required: false

permissions:
  contents: read
  id-token: write
  pull-requests: write

jobs:
  terraform:
    name: Terraform
    runs-on: ubuntu-latest

    defaults:
      run:
        working-directory: ${{ inputs.working-directory }}
    env:
      AWS_ROLE_TO_ASSUME: ${{ secrets.AWS_ROLE_TO_ASSUME }}
      AWS_REGION: ${{ secrets.AWS_REGION }}

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Configure AWS credentials
        if: ${{ env.AWS_ROLE_TO_ASSUME != '' && env.AWS_REGION != '' }}
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ env.AWS_ROLE_TO_ASSUME }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ inputs.terraform-version }}

      - name: Terraform format check
        run: terraform fmt -check -recursive

      - name: Terraform init
        run: terraform init -input=false

      - name: Terraform validate
        run: terraform validate

      - name: Terraform plan
        run: terraform plan -input=false -out=tfplan

      - name: Upload Terraform plan
        uses: actions/upload-artifact@v4
        with:
          name: ${{ inputs.plan-artifact-name }}
          path: ${{ inputs.working-directory }}/tfplan
          if-no-files-found: error

      - name: Terraform apply
        if: ${{ inputs.run-apply }}
        run: terraform apply -input=false -auto-approve tfplan
```

## Caller Workflow

Create this file in each Terraform repository:

`.github/workflows/terraform.yml`

```yaml
name: Terraform

on:
  pull_request:
    paths:
      - infra/**
      - .github/workflows/terraform.yml
  push:
    branches:
      - main
    paths:
      - infra/**
      - .github/workflows/terraform.yml
  workflow_dispatch:
    inputs:
      apply:
        description: Run terraform apply.
        required: true
        type: boolean
        default: false

jobs:
  terraform:
    uses: ORG/SHARED-WORKFLOWS-REPO/.github/workflows/terraform.yml@main
    with:
      working-directory: infra
      terraform-version: 1.9.8
      run-apply: ${{ github.event_name == 'push' || (github.event_name == 'workflow_dispatch' && inputs.apply) }}
      plan-artifact-name: terraform-plan-${{ github.run_id }}
    secrets:
      AWS_ROLE_TO_ASSUME: ${{ secrets.AWS_ROLE_TO_ASSUME }}
      AWS_REGION: ${{ secrets.AWS_REGION }}
```

Replace `ORG/SHARED-WORKFLOWS-REPO` with the repository that stores the reusable workflow.

## Notes

- Put environment-specific backend config in the Terraform code or pass it through additional workflow inputs.
- Prefer OIDC with `AWS_ROLE_TO_ASSUME` instead of long-lived cloud keys.
- Keep `run-apply` false for pull requests. Apply should normally happen only on protected branches or manual dispatch.
- If you use Azure or GCP, swap the AWS credentials step for the relevant login action and secrets.
