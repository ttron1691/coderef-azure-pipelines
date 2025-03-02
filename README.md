# coderef-azure-pipelines

# Azure Pipelines Reference Card

## Basic Components

### Pipeline File Structure
```yaml
# azure-pipelines.yml
trigger:
  - main

pool:
  vmImage: 'ubuntu-latest'

variables:
  buildConfiguration: 'Release'

stages:
  - stage: Build
    jobs:
      - job: BuildJob
        steps:
          - script: echo Hello, world!
```

### Key Components
- **Pipeline**: Top-level entity containing stages, jobs, and steps
- **Stage**: Logical grouping of jobs (e.g., Build, Test, Deploy)
- **Job**: A sequence of steps that run on an agent
- **Step**: Individual task or script that performs an action
- **Agent**: Computing resource that runs the pipeline
- **Artifact**: Build outputs stored for later use

### Common Tasks
```yaml
steps:
  # Run a command line script
  - script: echo Hello, world!
    displayName: 'Run a script'
  
  # Run a PowerShell script
  - powershell: |
      Write-Host "Running PowerShell"
    displayName: 'Run PowerShell'
  
  # Use predefined tasks
  - task: NuGetToolInstaller@1
  
  - task: UseDotNet@2
    inputs:
      packageType: 'sdk'
      version: '6.0.x'
  
  # Publish artifacts
  - task: PublishBuildArtifacts@1
    inputs:
      pathToPublish: '$(Build.ArtifactStagingDirectory)'
      artifactName: 'drop'
```

## Triggers and Schedules

### Branch and Path Triggers
```yaml
trigger:
  branches:
    include:
      - main
      - releases/*
    exclude:
      - releases/old*
  paths:
    include:
      - src/*
    exclude:
      - docs/*
```

### Pull Request Triggers
```yaml
pr:
  branches:
    include:
      - main
      - feature/*
  paths:
    exclude:
      - README.md
```

### Scheduled Triggers
```yaml
schedules:
  - cron: '0 0 * * *'  # Daily at midnight
    displayName: 'Daily midnight build'
    branches:
      include:
        - main
    always: true  # Run even if no code changes
```

### Manual Triggers (Resources)
```yaml
resources:
  pipelines:
    - pipeline: MyAppBuild
      source: MyAppBuildPipeline
      trigger: true  # Run when the referenced pipeline completes
```

## Variables and Parameters

### Variable Types
```yaml
variables:
  # Scalar variable
  myString: 'a string value'
  
  # Variable group reference
  - group: 'my-variable-group'
  
  # Variable template reference
  - template: variables.yml
  
  # Variable defined in the UI
  - name: 'uiVariable'
    value: $[variables.UiDefinedVariable]
```

### Variable Usage
```yaml
steps:
  - script: echo $(myString)
  - script: echo $(Build.BuildNumber)  # Predefined system variable
  - script: echo $(Build.SourceBranch)
  - script: |
      echo "Setting environment variable"
      echo "##vso[task.setvariable variable=customVar]value"
    displayName: 'Set variable'
  - script: echo $(customVar)
    displayName: 'Use set variable'
```

### Parameters
```yaml
parameters:
  - name: environment
    displayName: 'Environment to deploy to'
    type: string
    default: 'Development'
    values:
      - Development
      - Staging
      - Production

steps:
  - script: echo Deploying to ${{ parameters.environment }}
```

## Conditions and Expressions

### Conditions
```yaml
steps:
  - script: echo Success!
    condition: succeeded()
  
  - script: echo Failed!
    condition: failed()
  
  - script: echo Always run
    condition: always()
  
  - script: echo Skip this
    condition: eq(variables['Build.SourceBranch'], 'refs/heads/main')

jobs:
  - job: ConditionalJob
    condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
```

### Expressions
```yaml
variables:
  isMain: $[eq(variables['Build.SourceBranch'], 'refs/heads/main')]
  buildYear: $[format('{0:yyyy}', pipeline.startTime)]

steps:
  - script: echo Running on main
    condition: eq(variables.isMain, 'True')
  
  - script: echo Running with custom condition
    condition: ${{ contains(variables['Build.SourceVersionMessage'], '#run-this') }}
```

## Advanced Features

### Templates
```yaml
# template.yml
parameters:
  jobName: 'DefaultJob'
  vmImage: 'ubuntu-latest'

jobs:
  - job: ${{ parameters.jobName }}
    pool:
      vmImage: ${{ parameters.vmImage }}
    steps:
      - script: echo Running steps from template

# azure-pipelines.yml
jobs:
  - template: template.yml
    parameters:
      jobName: 'CustomJob'
      vmImage: 'windows-latest'
```

### Multi-Stage Pipelines
```yaml
stages:
  - stage: Build
    jobs:
      - job: BuildJob
        steps:
          - script: echo Building
          - task: PublishBuildArtifacts@1
            inputs:
              pathToPublish: '$(Build.ArtifactStagingDirectory)'
              artifactName: 'drop'
  
  - stage: Test
    dependsOn: Build
    jobs:
      - job: TestJob
        steps:
          - task: DownloadBuildArtifacts@0
            inputs:
              buildType: 'current'
              artifactName: 'drop'
  
  - stage: Deploy
    dependsOn: Test
    jobs:
      - deployment: DeployJob
        environment: 'Production'
        strategy:
          runOnce:
            deploy:
              steps:
                - script: echo Deploying
```

### Deployment Jobs
```yaml
jobs:
  - deployment: DeployWeb
    environment: 'Production'
    strategy:
      # Deployment strategies
      runOnce:
        deploy:
          steps:
            - script: echo Deploy once
      
      # Rolling strategy
      rolling:
        maxParallel: 2  # Max number of concurrent deployments
        deploy:
          steps:
            - script: echo Rolling deployment
      
      # Canary strategy
      canary:
        increments: [10, 20, 70]  # Percentage of targets for each increment
        deploy:
          steps:
            - script: echo Canary deployment
```

### Service Connections
```yaml
resources:
  repositories:
    - repository: MyRepo
      type: git
      name: MyProject/MyRepo
      ref: refs/heads/main

  containers:
    - container: MyContainer
      image: mcr.microsoft.com/windows/servercore:ltsc2019

  pipelines:
    - pipeline: MyPipe
      source: OtherPipeline
      trigger: true

  packages:
    - package: MyPackage
      type: Npm
      connection: MyNpmConnection
      name: MyPackageName
      version: 1.0.0
```

### Matrix Strategy
```yaml
jobs:
  - job: BuildMatrix
    strategy:
      matrix:
        linux_node_14:
          os: 'ubuntu-latest'
          node_version: '14.x'
        linux_node_16:
          os: 'ubuntu-latest'
          node_version: '16.x'
        windows_node_14:
          os: 'windows-latest'
          node_version: '14.x'
        windows_node_16:
          os: 'windows-latest'
          node_version: '16.x'
    pool:
      vmImage: $(os)
    steps:
      - task: NodeTool@0
        inputs:
          versionSpec: $(node_version)
      - script: npm test
```

### Parallel Jobs
```yaml
jobs:
  - job: Job1
    steps:
      - script: echo Job1
  
  - job: Job2
    steps:
      - script: echo Job2
  
  - job: Job3
    dependsOn:
      - Job1
      - Job2
    steps:
      - script: echo Job3 runs after Job1 and Job2
```

## Security and Secrets

### Secret Variables
```yaml
# Define in UI and reference in yaml
steps:
  - script: echo Connecting with $(mySecret)
    env:
      mySecret: $(secretVariable)  # Don't echo variable directly
```

### Key Vault Integration
```yaml
steps:
  - task: AzureKeyVault@2
    inputs:
      azureSubscription: 'MyServiceConnection'
      keyVaultName: 'MyKeyVault'
      secretsFilter: 'DBPassword,APIKey'
  
  - script: echo Using $(DBPassword)
```

## Best Practices

### Performance Optimization
- Use parallel jobs where possible
- Cache dependencies
- Use small, reusable templates
- Optimize container usage

```yaml
steps:
  # Cache dependencies
  - task: Cache@2
    inputs:
      key: 'npm | "$(Agent.OS)" | package-lock.json'
      restoreKeys: npm | "$(Agent.OS)"
      path: $(npm_config_cache)
    displayName: 'Cache npm packages'
  
  # Use fast containers
  - task: Docker@2
    inputs:
      command: 'build'
      Dockerfile: '**/Dockerfile'
      buildContext: '.'
      arguments: '--cache-from=myregistry.azurecr.io/myimage:latest'
```

### Maintainability
- Organize complex pipelines into stages
- Use templates for repeated patterns
- Document with displayName
- Keep jobs focused on a single responsibility

```yaml
steps:
  - script: npm ci
    displayName: 'Install dependencies'
  
  - script: npm run lint
    displayName: 'Run linting'
  
  - script: npm test
    displayName: 'Run tests'
```

### Error Handling
- Use timeouts for external dependencies
- Add retry logic for transient failures
- Include clear error messages
- Set appropriate timeouts

```yaml
jobs:
  - job: JobWithRetry
    timeoutInMinutes: 10
    steps:
      - task: PowerShell@2
        inputs:
          targetType: 'inline'
          script: |
            $retryCount = 3
            $retryDelay = 10
            $success = $false
            
            for ($i = 1; $i -le $retryCount; $i++) {
              try {
                # Attempt the operation
                echo "Attempt $i"
                # If successful, set $success to true and break
                $success = $true
                break
              }
              catch {
                if ($i -lt $retryCount) {
                  echo "Attempt $i failed, retrying in $retryDelay seconds..."
                  Start-Sleep -Seconds $retryDelay
                }
              }
            }
            
            if (-not $success) {
              throw "Operation failed after $retryCount attempts"
            }
```

### Environment Management
- Use approvals for production deployments
- Implement environment-specific variable groups
- Use deployment gates

```yaml
stages:
  - stage: DeployToProd
    dependsOn: Test
    jobs:
      - deployment: DeployWeb
        environment: 'Production'  # Environment with approvals configured
        strategy:
          runOnce:
            deploy:
              steps:
                - script: echo Deploying to Production
```

## Common Patterns

### CI/CD Pipeline
```yaml
trigger:
  - main

stages:
  - stage: Build
    jobs:
      - job: BuildJob
        pool:
          vmImage: 'ubuntu-latest'
        steps:
          - script: dotnet build
          - task: DotNetCoreCLI@2
            inputs:
              command: 'test'
          - task: PublishBuildArtifacts@1
            inputs:
              pathToPublish: '$(Build.ArtifactStagingDirectory)'
              artifactName: 'drop'
  
  - stage: DeployToDev
    dependsOn: Build
    jobs:
      - deployment: DeployJob
        environment: 'Development'
        strategy:
          runOnce:
            deploy:
              steps:
                - download: current
                  artifact: drop
                - script: echo Deploying to Dev
  
  - stage: DeployToProd
    dependsOn: DeployToDev
    jobs:
      - deployment: DeployJob
        environment: 'Production'
        strategy:
          runOnce:
            deploy:
              steps:
                - download: current
                  artifact: drop
                - script: echo Deploying to Prod
```

### Build-Test Matrix
```yaml
trigger:
  - main

pool:
  vmImage: 'ubuntu-latest'

strategy:
  matrix:
    DotNet5:
      dotnetVersion: '5.0.x'
    DotNet6:
      dotnetVersion: '6.0.x'
    DotNet7:
      dotnetVersion: '7.0.x'

steps:
  - task: UseDotNet@2
    inputs:
      packageType: 'sdk'
      version: $(dotnetVersion)
  
  - script: dotnet build
    displayName: 'Build with .NET $(dotnetVersion)'
  
  - script: dotnet test
    displayName: 'Test with .NET $(dotnetVersion)'
```

### Multi-Environment Deployment
```yaml
stages:
  - stage: Build
    jobs:
      - job: BuildJob
        steps:
          - script: echo Building
          - task: PublishBuildArtifacts@1
            inputs:
              pathToPublish: '$(Build.ArtifactStagingDirectory)'
              artifactName: 'drop'

  - stage: DeployDev
    dependsOn: Build
    condition: succeeded()
    variables:
      - group: dev-variables
    jobs:
      - deployment: DeployWeb
        environment: Development
        strategy:
          runOnce:
            deploy:
              steps:
                - script: echo "Connection string: $(ConnectionString)"

  - stage: DeployQA
    dependsOn: DeployDev
    condition: succeeded()
    variables:
      - group: qa-variables
    jobs:
      - deployment: DeployWeb
        environment: QA
        strategy:
          runOnce:
            deploy:
              steps:
                - script: echo "Connection string: $(ConnectionString)"

  - stage: DeployProd
    dependsOn: DeployQA
    condition: succeeded()
    variables:
      - group: prod-variables
    jobs:
      - deployment: DeployWeb
        environment: Production
        strategy:
          runOnce:
            deploy:
              steps:
                - script: echo "Connection string: $(ConnectionString)"
```
