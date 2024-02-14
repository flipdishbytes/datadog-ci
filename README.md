# Purpose

This custom GitHub Action & Azure DevOps Template were created to integrate Datadog CI monitoring into your CI/CD pipeline. It allows you to track and analyze the performance and health of your pipelines during the build and deployment process.

# Github Action: Datadog CI `flipdishbytes/datadog-ci@v1.1`

To use this Datadog CI action, add it to your pipeline workflow YAML file. Here are examples of adding traces to the pipeline depending on your needs.

### How it works?

1. Downloads latest datadog-ci Linux binary wia curl
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

# Azure DevOps Template: Datadog CI `- template: '.azure/datadog-ci.yml@DatadogCI'`

To use this Datadog CI template, add it to your pipeline YAML file stages. Here are example of adding traces to the pipeline depending on your needs.

### How it works?

1. Downloads latest datadog-ci Linux binary wia curl if it's not preinstalled
2. Executing `datadog-ci` with the provided command.

### How to use?

#### `- template: '.azure/datadog-ci.yml@DatadogCI'` - no need for DD_API_KEY being set. Execution time 5s.
Should be used by default in Flipdish Org. It doesn't require `DD_API_KEY` secret being set in the repository.

```yaml
trigger:
  branches:
    include:
      - main
pr: none

resources: # resources are required to load the template
  repositories:
    - repository: DatadogCI
      endpoint: 'flipdishbytes-Web Order' # this could be different for each Azure DevOps Project.
      type: github
      name: flipdishbytes/datadog-ci
      ref: main
      trigger: none

variables:
    - group: 'Datadog API Keys' # this variable group is required to provide datadogAPIKeyAzureDevOps secret
    - name: environmentName
      value: 'production'
    - name: Version
      value: $(Build.BuildNumber)

stages:
- stage: 'deploy'
  displayName: 'WAF Rules Deploy'
  dependsOn: []
  jobs:
    - template: '.azure/datadog-ci.yml@DatadogCI' #the only template you need to add to your stages
      parameters:
        displayName: 'Add tags to pipeline traces'
        DD_API_KEY: $(datadogAPIKeyAzureDevOps)
        COMMAND: 'tag --level pipeline --tags service:aws-governance --tags team:de-team --tags env:production'

    - job: GitHubRelease #just for example to show pipeline
        displayName: 'Add Build Tags'
        pool: Development Pipeline
        steps:
          - checkout: self
          - task: GitHubRelease@1
            displayName: 'GitHub release (create)'
            inputs:
              gitHubConnection: 'github'
              tagSource: userSpecifiedTag
              tag: '$(Version)'
              title: 'Datadog-CI $(Version)'
```