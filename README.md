# Custom Action: Datadog CI

## Purpose

This custom action was created to integrate Datadog CI monitoring into your CI/CD pipeline. It allows you to track and analyze the performance and health of your pipelines during the build and deployment process.

### Usage

To use this Datadog CI action, add it to your pipeline workflow YAML file. Here are examples of adding traces to the pipeline depending on your needs.

### How it works?

1. It downloads latest datadog-ci Linux binary wia curl
2. Reads `DatadogGithubActionsApiKey` AWS Secret this using aws cli (preinstalled on all GitHub agents) and using it for `datadog-ci` command execution.

**There is `DatadogGithubActionsApiKey` secret stored at AWS SSM in Flipdish Management account and separate role created for GitHub Actions which can be assumed only by Flipdish Org GitHub Actions repositories and has IAM permissions only for getting this secret value.**

**P.S. this role can't be assumed by any repo outside of Flipdish GitHub Org.**

### How to use?

#### `flipdishbytes/datadog-ci@v1.1` - no need for DD_API_KEY being set. Execution time 5s.
Should be used by default in Flipdish Org. It doesn't require `DD_API_KEY` secret being set in the repository.

```yaml
name: GH Action workflow without DD_API_KEY secret being set

on:
  pull_request:
    types:
      - opened
      - reopened
      - synchronize
      - edited
    branches:
      - 'main'

permissions: # v1.1 requires id-token write permission to assume AWS role. Makes sure you added this to your yml.
  contents: read
  id-token: write

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Add tags to pipeline traces
        uses: flipdishbytes/datadog-ci@v1.1
        with:
          COMMAND: 'tag --level pipeline --tags service:aws-governance --tags team:de-team --tags env:production'
```

#### `flipdishbytes/datadog-ci@v1.0` - DD_API_KEY is required. Execution time 3s.
Should be used only if you want to set DD_API_KEY and DD_SITE in your repository. It requires `DD_API_KEY` secret being set in the repository. `DD_SITE` is set to us3.datadoghq.com by default.
```yaml
name: GH Action workflow with DD_API_KEY secret being set up

on:
  pull_request:
    types:
      - opened
      - reopened
      - synchronize
      - edited
    branches:
      - 'main'

jobs:
  ci-build-and-test:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Add tags to pipeline traces
        uses: flipdishbytes/datadog-ci@v1.0
        with:
          COMMAND: 'tag --level pipeline --tags service:aws-governance --tags team:de-team --tags env:production'
          DD_API_KEY: ${{ secrets.DD_API_KEY }} # v1.0 requires DD_API_KEY being set.
        #   DD_SITE: 'us3.datadoghq.com' # DD_SITE is set to us3.datadoghq.com by default.
```