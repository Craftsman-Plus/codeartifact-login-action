# AWS CodeArtifact login Action

Log into [AWS CodeArtifact](https://aws.amazon.com/codeartifact/) using AWS credentials and get 
a temporary token for usage with Python package managers such as `pip` and `poetry`.

## Usage

```yaml
name: CI

on:
  push:
    branches:
      - main

jobs:
  do-stuff:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - name: Set up Python
        id: setup-python
        uses: actions/setup-python@v2
        with:
          python-version: "3.9.10"

      - name: Install Poetry
        uses: snok/install-poetry@v1
        with:
          virtualenvs-create: true
          virtualenvs-in-project: true
          installer-parallel: true
    
      - name: Authenticate to CodeArtifact
        uses: source-ag/codeartifact-login-action@v1
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: eu-central-1
          role-to-assume: ${{ secrets.PACKAGE_REPOSITORY_ROLE }}
          codeartifact-domain: my-domain
          codeartifact-domain-owner: 123456789012
          codeartifact-repository: my-python-packages
          configure-poetry: true

      - name: Install dependencies
        run: poetry install
```

## Outputs

This action supplies the following outputs:

```
codeartifact-token: Temporary token to authenticate with AWS CodeArtifact repositories
codeartifact-user: Username for usage with package tools such as pip and Poetry
codeartifact-repo-url: URL for the specified repository
```

## Pointing the default registry at a store repository

By default this action configures only the **scoped** registry (`@cmp`), so public
packages still resolve from npmjs.com. On self-hosted runners that egress is
charged as NAT gateway data processing — in `cmp-tools` it was the single largest
line item, larger than all compute in the account (API-2126).

Pass `defaultRepository` to also point the tool's default registry at a
CodeArtifact repository holding an external connection to the public registry:

```yaml
- uses: Craftsman-Plus/codeartifact-login-action@v1.1.0
  with:
    domain: ${{ vars.CODE_ARTIFACT_NPM_REPOSITORY_DOMAIN }}
    scope: "@cmp"
    repository: ${{ vars.CODE_ARTIFACT_NPM_REPOSITORY_NAME }}
    defaultRepository: npm-store
    region: ${{ vars.AWS_REGION }}
    accountId: ${{ vars.AWS_PROD_ACCOUNT_ID }}
```

The two logins write different `.npmrc` keys — `@cmp:registry` and `registry` —
so our own packages keep resolving from their own repository. Verified:

```
@cmp:registry=https://<domain>.d.codeartifact.<region>.amazonaws.com/npm/cmp/
registry=https://<domain>.d.codeartifact.<region>.amazonaws.com/npm/npm-store/
```

Omitting the input leaves behaviour exactly as before.
