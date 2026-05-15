# Azure DevOps Pipeline Strategy

## Genel Bakış

Bu doküman, branch stratejimizi destekleyen Azure Pipeline yapısını, otomatik senkronizasyon mekanizmalarını ve CI/CD süreçlerini detaylı olarak açıklar.

## Pipeline Mimarisi

### Pipeline Kategorileri

1. **CI Pipelines**: Kod kalitesi ve build validasyonu
2. **CD Pipelines**: Deployment ve release otomasyonu
3. **Sync Pipelines**: Branch senkronizasyon otomasyonu
4. **Utility Pipelines**: Maintenance ve yardımcı işlemler

## Core Pipeline'lar

### 1. CI Pipeline (Continuous Integration)

#### CI-Main-Validation Pipeline
```yaml
# File: .azure-pipelines/ci-main-validation.yml
trigger:
  branches:
    include:
      - main
  paths:
    exclude:
      - docs/*
      - README.md

pr:
  branches:
    include:
      - main
  paths:
    exclude:
      - docs/*

pool:
  vmImage: 'ubuntu-latest'

variables:
  buildConfiguration: 'Release'
  COVERAGE_THRESHOLD: 80

stages:
  - stage: Build
    displayName: 'Build and Validate'
    jobs:
      - job: BuildJob
        displayName: 'Build Application'
        steps:
          - checkout: self
            clean: true
            fetchDepth: 0
          
          - task: UseDotNet@2
            displayName: 'Use .NET SDK'
            inputs:
              packageType: 'sdk'
              version: '8.x'
          
          - task: DotNetCoreCLI@2
            displayName: 'Restore Dependencies'
            inputs:
              command: 'restore'
              projects: '**/*.csproj'
          
          - task: DotNetCoreCLI@2
            displayName: 'Build Solution'
            inputs:
              command: 'build'
              projects: '**/*.csproj'
              arguments: '--configuration $(buildConfiguration) --no-restore'
          
          - task: DotNetCoreCLI@2
            displayName: 'Run Unit Tests'
            inputs:
              command: 'test'
              projects: '**/*Tests.csproj'
              arguments: '--configuration $(buildConfiguration) --no-build --collect:"XPlat Code Coverage" --logger trx'
              publishTestResults: true
          
          - task: PublishCodeCoverageResults@1
            displayName: 'Publish Code Coverage'
            inputs:
              codeCoverageTool: 'Cobertura'
              summaryFileLocation: '$(Agent.TempDirectory)/**/coverage.cobertura.xml'
              failIfCoverageEmpty: true
          
          - script: |
              COVERAGE=$(grep -oP 'line-rate="\K[0-9.]+' $(Agent.TempDirectory)/**/coverage.cobertura.xml | head -1)
              COVERAGE_PERCENT=$(echo "$COVERAGE * 100" | bc)
              if (( $(echo "$COVERAGE_PERCENT < $(COVERAGE_THRESHOLD)" | bc -l) )); then
                echo "##vso[task.logissue type=error]Code coverage ($COVERAGE_PERCENT%) is below threshold ($(COVERAGE_THRESHOLD)%)"
                exit 1
              fi
              echo "Code coverage: $COVERAGE_PERCENT%"
            displayName: 'Validate Code Coverage Threshold'
          
          - task: PublishBuildArtifacts@1
            displayName: 'Publish Build Artifacts'
            inputs:
              PathtoPublish: '$(Build.ArtifactStagingDirectory)'
              ArtifactName: 'drop'

  - stage: Security
    displayName: 'Security Scanning'
    dependsOn: Build
    jobs:
      - job: SecurityScan
        displayName: 'Security and Compliance Scan'
        steps:
          - task: SonarQubePrepare@5
            displayName: 'Prepare SonarQube Analysis'
            inputs:
              SonarQube: 'SonarQube-Connection'
              scannerMode: 'MSBuild'
              projectKey: 'your-project-key'
              projectName: 'Your Project'
          
          - task: DotNetCoreCLI@2
            displayName: 'Build for SonarQube'
            inputs:
              command: 'build'
              projects: '**/*.csproj'
          
          - task: SonarQubeAnalyze@5
            displayName: 'Run SonarQube Analysis'
          
          - task: SonarQubePublish@5
            displayName: 'Publish SonarQube Results'
            inputs:
              pollingTimeoutSec: '300'
          
          - task: DependencyCheck@1
            displayName: 'OWASP Dependency Check'
            inputs:
              projectName: 'Your Project'
              scanPath: '$(Build.SourcesDirectory)'
              format: 'HTML,JSON'
          
          - task: WhiteSource@21
            displayName: 'WhiteSource Security Scan'
            inputs:
              cwd: '$(Build.SourcesDirectory)'
```

#### CI-Release-Validation Pipeline
```yaml
# File: .azure-pipelines/ci-release-validation.yml
trigger:
  branches:
    include:
      - release/*

pr:
  branches:
    include:
      - release/*

pool:
  vmImage: 'ubuntu-latest'

stages:
  - stage: Build
    displayName: 'Build and Test'
    jobs:
      - job: BuildAndTest
        steps:
          # Similar to CI-Main but with:
          # - Code coverage threshold: 75%
          # - Additional UAT test execution
          # - Database migration validation
          - template: templates/build-steps.yml
          - template: templates/uat-tests.yml
          - template: templates/regression-tests.yml
          - template: templates/db-migration-check.yml
```

#### CI-Sandbox-Validation Pipeline
```yaml
# File: .azure-pipelines/ci-sandbox-validation.yml
trigger:
  branches:
    include:
      - sandbox

pr:
  branches:
    include:
      - sandbox

pool:
  vmImage: 'ubuntu-latest'

variables:
  COVERAGE_THRESHOLD: 60

stages:
  - stage: Build
    displayName: 'Build and Unit Test'
    jobs:
      - job: QuickValidation
        steps:
          - template: templates/build-steps.yml
          - template: templates/unit-tests.yml
          
  - stage: CodeQuality
    displayName: 'Code Quality Analysis'
    dependsOn: Build
    condition: succeeded()
    jobs:
      - job: QualityChecks
        steps:
          - task: SonarQubePrepare@5
            displayName: 'SonarQube Analysis'
          - template: templates/code-quality.yml
```

### 2. CD Pipeline (Continuous Deployment)

#### CD-Production Pipeline
```yaml
# File: .azure-pipelines/cd-production.yml
trigger:
  branches:
    include:
      - main
  tags:
    include:
      - v*

pool:
  vmImage: 'ubuntu-latest'

variables:
  - group: Production-Variables
  - name: environment
    value: 'Production'

stages:
  - stage: PreDeployment
    displayName: 'Pre-Deployment Checks'
    jobs:
      - job: ValidationGate
        displayName: 'Deployment Validation'
        steps:
          - task: ManualValidation@0
            displayName: 'Manual Approval Gate'
            inputs:
              notifyUsers: 'release-managers@company.com'
              instructions: 'Please review and approve production deployment'
              timeoutInMinutes: 1440 # 24 hours
          
          - script: |
              # Verify release tag
              git describe --tags --exact-match HEAD || exit 1
            displayName: 'Verify Release Tag'
          
          - task: AzureCLI@2
            displayName: 'Check Production Health'
            inputs:
              azureSubscription: 'Azure-Production'
              scriptType: 'bash'
              scriptLocation: 'inlineScript'
              inlineScript: |
                # Check if production is healthy before deployment
                HEALTH_ENDPOINT="https://api.production.com/health"
                STATUS=$(curl -s -o /dev/null -w "%{http_code}" $HEALTH_ENDPOINT)
                if [ $STATUS -ne 200 ]; then
                  echo "Production health check failed: $STATUS"
                  exit 1
                fi

  - stage: Deployment
    displayName: 'Deploy to Production'
    dependsOn: PreDeployment
    jobs:
      - deployment: DeployProduction
        displayName: 'Deploy to Production Environment'
        environment: 'Production'
        strategy:
          runOnce:
            deploy:
              steps:
                - download: current
                  artifact: drop
                
                - task: AzureWebApp@1
                  displayName: 'Deploy to Azure App Service'
                  inputs:
                    azureSubscription: 'Azure-Production'
                    appType: 'webApp'
                    appName: '$(AppServiceName)'
                    package: '$(Pipeline.Workspace)/drop/**/*.zip'
                    deploymentMethod: 'zipDeploy'
                
                - task: AzureCLI@2
                  displayName: 'Run Database Migrations'
                  inputs:
                    azureSubscription: 'Azure-Production'
                    scriptType: 'bash'
                    scriptLocation: 'inlineScript'
                    inlineScript: |
                      # Run EF Core migrations or SQL scripts
                      dotnet ef database update --connection "$(ConnectionString)"
                
                - script: |
                    # Smoke tests after deployment
                    sleep 30
                    curl -f https://api.production.com/health || exit 1
                  displayName: 'Smoke Test'

  - stage: PostDeployment
    displayName: 'Post-Deployment'
    dependsOn: Deployment
    jobs:
      - job: CreateTag
        displayName: 'Create Release Tag'
        steps:
          - checkout: self
            persistCredentials: true
          
          - script: |
              TAG_NAME="v$(Build.BuildNumber)"
              git tag -a $TAG_NAME -m "Production release $TAG_NAME"
              git push origin $TAG_NAME
            displayName: 'Create and Push Git Tag'
          
      - job: Notification
        displayName: 'Send Notifications'
        steps:
          - task: SendEmail@1
            displayName: 'Email Notification'
            inputs:
              To: 'team@company.com'
              Subject: 'Production Deployment Complete: v$(Build.BuildNumber)'
              Body: |
                Production deployment successful!
                Version: v$(Build.BuildNumber)
                Deployed by: $(Build.RequestedFor)
                Build: $(Build.BuildUri)
```

#### CD-Staging Pipeline
```yaml
# File: .azure-pipelines/cd-staging.yml
trigger:
  branches:
    include:
      - release/*

pool:
  vmImage: 'ubuntu-latest'

variables:
  - group: Staging-Variables
  - name: environment
    value: 'Staging'

stages:
  - stage: Deployment
    displayName: 'Deploy to Staging'
    jobs:
      - deployment: DeployStaging
        displayName: 'Deploy to Staging Environment'
        environment: 'Staging'
        strategy:
          runOnce:
            deploy:
              steps:
                - template: templates/deploy-webapp.yml
                  parameters:
                    environment: 'Staging'
                    azureSubscription: 'Azure-Staging'
                    appName: '$(StagingAppName)'
```

#### CD-Development Pipeline
```yaml
# File: .azure-pipelines/cd-development.yml
trigger:
  branches:
    include:
      - sandbox

pool:
  vmImage: 'ubuntu-latest'

variables:
  - group: Development-Variables
  - name: environment
    value: 'Development'

stages:
  - stage: Deployment
    displayName: 'Deploy to Development'
    jobs:
      - deployment: DeployDevelopment
        displayName: 'Deploy to Development Environment'
        environment: 'Development'
        strategy:
          runOnce:
            deploy:
              steps:
                - template: templates/deploy-webapp.yml
                  parameters:
                    environment: 'Development'
                    azureSubscription: 'Azure-Development'
                    appName: '$(DevAppName)'
```

### 3. Hotfix Sync Pipeline (Kritik)

```yaml
# File: .azure-pipelines/hotfix-sync.yml
# Bu pipeline, hotfix merge sonrası otomatik olarak tetiklenir
# ve değişiklikleri release ve sandbox branch'lerine senkronize eder

trigger:
  branches:
    include:
      - main
  paths:
    exclude:
      - docs/*

# Sadece hotfix merge'lerinde çalışması için condition
parameters:
  - name: verifyHotfix
    displayName: 'Verify this is a hotfix merge'
    type: boolean
    default: true

pool:
  vmImage: 'ubuntu-latest'

variables:
  - name: isHotfixMerge
    value: false

stages:
  - stage: Verification
    displayName: 'Verify Hotfix Merge'
    jobs:
      - job: CheckHotfix
        displayName: 'Check if this is a hotfix merge'
        steps:
          - checkout: self
            fetchDepth: 0
            persistCredentials: true
          
          - script: |
              # Son commit mesajını kontrol et
              COMMIT_MSG=$(git log -1 --pretty=%B)
              
              # Hotfix merge olup olmadığını kontrol et
              if echo "$COMMIT_MSG" | grep -iq "hotfix"; then
                echo "##vso[task.setvariable variable=isHotfixMerge;isOutput=true]true"
                echo "Hotfix merge detected: $COMMIT_MSG"
              else
                # Son merge edilen branch'i kontrol et
                MERGED_BRANCH=$(git log -1 --merges --pretty=%s | grep -oP 'from \K[^ ]+' || echo "")
                if echo "$MERGED_BRANCH" | grep -iq "hotfix"; then
                  echo "##vso[task.setvariable variable=isHotfixMerge;isOutput=true]true"
                  echo "Hotfix merge detected from branch: $MERGED_BRANCH"
                else
                  echo "##vso[task.setvariable variable=isHotfixMerge;isOutput=true]false"
                  echo "Not a hotfix merge, skipping sync"
                fi
              fi
            name: hotfixCheck
            displayName: 'Detect Hotfix Merge'

  - stage: SyncToRelease
    displayName: 'Sync to Release Branch'
    dependsOn: Verification
    condition: and(succeeded(), eq(dependencies.Verification.outputs['CheckHotfix.hotfixCheck.isHotfixMerge'], 'true'))
    jobs:
      - job: MergeToRelease
        displayName: 'Merge Hotfix to Release'
        steps:
          - checkout: self
            fetchDepth: 0
            persistCredentials: true
          
          - script: |
              git config user.email "devops@company.com"
              git config user.name "Azure DevOps Bot"
              
              # Get active release branch (latest)
              RELEASE_BRANCH=$(git branch -r | grep 'origin/release/' | sort -V | tail -1 | sed 's/.*origin\///')
              
              if [ -z "$RELEASE_BRANCH" ]; then
                echo "No release branch found, skipping..."
                exit 0
              fi
              
              echo "Syncing to $RELEASE_BRANCH"
              git checkout $RELEASE_BRANCH
              git pull origin $RELEASE_BRANCH
              
              # Try to merge
              if git merge origin/main --no-edit; then
                echo "Merge successful"
                git push origin $RELEASE_BRANCH
                echo "##vso[task.setvariable variable=mergeStatus]success"
              else
                echo "##vso[task.logissue type=warning]Merge conflict detected"
                echo "##vso[task.setvariable variable=mergeStatus]conflict"
                git merge --abort
              fi
            displayName: 'Merge to Release Branch'
            name: releaseSync
          
          - task: CreateWorkItem@1
            displayName: 'Create PR for Conflict Resolution'
            condition: eq(variables['mergeStatus'], 'conflict')
            inputs:
              workItemType: 'Task'
              title: 'Resolve Hotfix Sync Conflict in Release Branch'
              assignedTo: '$(Build.RequestedForEmail)'
              areaPath: '$(System.TeamProject)'
              iterationPath: '$(System.TeamProject)'
              fieldMappings: |
                Description=Automatic hotfix sync to release branch failed due to conflicts.
                Please manually resolve conflicts and merge hotfix changes.
                
                Hotfix commit: $(Build.SourceVersion)
                Build: $(Build.BuildUri)
              associate: true

  - stage: SyncToSandbox
    displayName: 'Sync to Sandbox Branch'
    dependsOn: 
      - Verification
      - SyncToRelease
    condition: and(succeeded(), eq(dependencies.Verification.outputs['CheckHotfix.hotfixCheck.isHotfixMerge'], 'true'))
    jobs:
      - job: MergeToSandbox
        displayName: 'Merge Hotfix to Sandbox'
        steps:
          - checkout: self
            fetchDepth: 0
            persistCredentials: true
          
          - script: |
              git config user.email "devops@company.com"
              git config user.name "Azure DevOps Bot"
              
              git checkout sandbox
              git pull origin sandbox
              
              # Try to merge
              if git merge origin/main --no-edit; then
                echo "Merge successful"
                git push origin sandbox
                echo "##vso[task.setvariable variable=mergeStatus]success"
              else
                echo "##vso[task.logissue type=warning]Merge conflict detected"
                echo "##vso[task.setvariable variable=mergeStatus]conflict"
                git merge --abort
              fi
            displayName: 'Merge to Sandbox Branch'
            name: sandboxSync
          
          - task: CreateWorkItem@1
            displayName: 'Create PR for Conflict Resolution'
            condition: eq(variables['mergeStatus'], 'conflict')
            inputs:
              workItemType: 'Task'
              title: 'Resolve Hotfix Sync Conflict in Sandbox Branch'
              assignedTo: '$(Build.RequestedForEmail)'
              areaPath: '$(System.TeamProject)'
              iterationPath: '$(System.TeamProject)'
              fieldMappings: |
                Description=Automatic hotfix sync to sandbox branch failed due to conflicts.
                Please manually resolve conflicts and merge hotfix changes.
                
                Hotfix commit: $(Build.SourceVersion)
                Build: $(Build.BuildUri)
              associate: true

  - stage: Notification
    displayName: 'Send Notifications'
    dependsOn:
      - SyncToRelease
      - SyncToSandbox
    condition: always()
    jobs:
      - job: NotifyTeam
        displayName: 'Send Sync Status Notification'
        steps:
          - task: SendEmail@1
            displayName: 'Email Notification'
            inputs:
              To: 'devops-team@company.com'
              Subject: 'Hotfix Sync Status - $(Build.BuildNumber)'
              Body: |
                Hotfix automatic synchronization completed.
                
                Status:
                - Release Branch: $(releaseSyncStatus)
                - Sandbox Branch: $(sandboxSyncStatus)
                
                Build: $(Build.BuildUri)
                Commit: $(Build.SourceVersion)
```

### 4. Release Cleanup Pipeline

```yaml
# File: .azure-pipelines/release-cleanup.yml
# Bu pipeline, eski release branch'lerini temizler (son 5'i korur)

schedules:
  - cron: "0 2 * * 0"  # Her Pazar 02:00
    displayName: Weekly release cleanup
    branches:
      include:
        - main
    always: true

trigger: none

pool:
  vmImage: 'ubuntu-latest'

steps:
  - checkout: self
    fetchDepth: 0
    persistCredentials: true

  - script: |
      git config user.email "devops@company.com"
      git config user.name "Azure DevOps Bot"
      
      # Get all release branches sorted by version
      RELEASE_BRANCHES=$(git branch -r | grep 'origin/release/' | sed 's/.*origin\///' | sort -V)
      TOTAL_COUNT=$(echo "$RELEASE_BRANCHES" | wc -l)
      
      echo "Total release branches: $TOTAL_COUNT"
      
      # Keep last 5, delete older ones
      if [ $TOTAL_COUNT -gt 5 ]; then
        DELETE_COUNT=$((TOTAL_COUNT - 5))
        BRANCHES_TO_DELETE=$(echo "$RELEASE_BRANCHES" | head -n $DELETE_COUNT)
        
        echo "Branches to delete:"
        echo "$BRANCHES_TO_DELETE"
        
        # Create tags before deletion for archive
        for BRANCH in $BRANCHES_TO_DELETE; do
          TAG_NAME="archive/$(echo $BRANCH | sed 's/release\///')-$(date +%Y%m%d)"
          git tag -a $TAG_NAME refs/remotes/origin/$BRANCH -m "Archive of $BRANCH before cleanup"
          git push origin $TAG_NAME
          echo "Created archive tag: $TAG_NAME"
        done
        
        # Wait 30 days grace period check (implement your logic)
        # For demo, we'll delete immediately after tagging
        
        for BRANCH in $BRANCHES_TO_DELETE; do
          git push origin --delete $BRANCH
          echo "Deleted branch: $BRANCH"
        done
        
        echo "##vso[task.complete result=Succeeded;]Cleanup completed. Deleted $DELETE_COUNT branches."
      else
        echo "##vso[task.complete result=Succeeded;]No cleanup needed. Only $TOTAL_COUNT branches exist."
      fi
    displayName: 'Cleanup Old Release Branches'

  - task: SendEmail@1
    displayName: 'Send Cleanup Report'
    inputs:
      To: 'devops-team@company.com'
      Subject: 'Release Branch Cleanup Report'
      Body: |
        Release branch cleanup completed.
        Check build logs for details: $(Build.BuildUri)
```

### 5. Branch Health Monitoring Pipeline

```yaml
# File: .azure-pipelines/branch-health-monitor.yml
# Bu pipeline, branch sağlığını ve metrics'leri izler

schedules:
  - cron: "0 9 * * *"  # Her gün 09:00
    displayName: Daily branch health check
    branches:
      include:
        - main
    always: true

trigger: none

pool:
  vmImage: 'ubuntu-latest'

steps:
  - checkout: self
    fetchDepth: 0

  - task: PowerShell@2
    displayName: 'Analyze Branch Health'
    inputs:
      targetType: 'inline'
      script: |
        # Feature branch age analysis
        Write-Host "Analyzing feature branches..."
        $featureBranches = git branch -r | Where-Object { $_ -match 'origin/feature/' }
        
        foreach ($branch in $featureBranches) {
          $branch = $branch.Trim()
          $lastCommit = git log -1 --format="%ci" $branch
          $age = (Get-Date) - (Get-Date $lastCommit)
          
          if ($age.Days -gt 14) {
            Write-Host "##vso[task.logissue type=warning]Stale branch detected: $branch (Age: $($age.Days) days)"
          }
        }
        
        # PR metrics
        Write-Host "Checking open PRs..."
        # Use Azure DevOps REST API to get PR metrics
        
        # Branch divergence check
        Write-Host "Checking branch divergence..."
        $sandbox = git rev-parse origin/sandbox
        $main = git rev-parse origin/main
        $divergence = git rev-list --left-right --count origin/main...origin/sandbox
        Write-Host "Main vs Sandbox divergence: $divergence"

  - task: PublishBuildArtifacts@1
    displayName: 'Publish Health Report'
    inputs:
      PathtoPublish: '$(Build.ArtifactStagingDirectory)/health-report.json'
      ArtifactName: 'branch-health-report'
```

## Pipeline Templates

### Build Steps Template
```yaml
# File: .azure-pipelines/templates/build-steps.yml
steps:
  - task: UseDotNet@2
    displayName: 'Install .NET SDK'
    inputs:
      packageType: 'sdk'
      version: '8.x'

  - task: DotNetCoreCLI@2
    displayName: 'Restore NuGet Packages'
    inputs:
      command: 'restore'
      projects: '**/*.csproj'
      feedsToUse: 'select'

  - task: DotNetCoreCLI@2
    displayName: 'Build Solution'
    inputs:
      command: 'build'
      projects: '**/*.csproj'
      arguments: '--configuration Release --no-restore'

  - task: DotNetCoreCLI@2
    displayName: 'Publish Application'
    inputs:
      command: 'publish'
      publishWebProjects: true
      arguments: '--configuration Release --output $(Build.ArtifactStagingDirectory)'
      zipAfterPublish: true
```

### Deploy WebApp Template
```yaml
# File: .azure-pipelines/templates/deploy-webapp.yml
parameters:
  - name: environment
    type: string
  - name: azureSubscription
    type: string
  - name: appName
    type: string

steps:
  - download: current
    artifact: drop

  - task: AzureWebApp@1
    displayName: 'Deploy to ${{ parameters.environment }}'
    inputs:
      azureSubscription: ${{ parameters.azureSubscription }}
      appType: 'webApp'
      appName: ${{ parameters.appName }}
      package: '$(Pipeline.Workspace)/drop/**/*.zip'
      deploymentMethod: 'zipDeploy'

  - task: AzureAppServiceManage@0
    displayName: 'Restart App Service'
    inputs:
      azureSubscription: ${{ parameters.azureSubscription }}
      Action: 'Restart Azure App Service'
      WebAppName: ${{ parameters.appName }}

  - script: |
      echo "Waiting for app to start..."
      sleep 30
      curl -f https://${{ parameters.appName }}.azurewebsites.net/health || exit 1
    displayName: 'Health Check'
```

## Pipeline Trigger Strategy

### Branch-based Triggers

```yaml
# Main branch: Production deployment
trigger:
  branches:
    include:
      - main
  tags:
    include:
      - v*

# Release branches: Staging deployment
trigger:
  branches:
    include:
      - release/*

# Sandbox: Development deployment
trigger:
  branches:
    include:
      - sandbox

# Feature branches: CI validation only
pr:
  branches:
    include:
      - sandbox
  paths:
    exclude:
      - docs/*
      - '*.md'
```

### Path-based Triggers

```yaml
# Infrastructure changes
trigger:
  branches:
    include:
      - main
  paths:
    include:
      - infrastructure/*
      - '*.bicep'
      - '*.tf'

# Database changes
trigger:
  branches:
    include:
      - main
      - release/*
  paths:
    include:
      - database/migrations/*
```

## Variable Groups

### Development Variables
```yaml
# Azure DevOps → Pipelines → Library → Variable Groups
Name: Development-Variables
Variables:
  - AppServiceName: 'app-dev-001'
  - ResourceGroup: 'rg-dev-001'
  - EnvironmentName: 'Development'
  - DatabaseConnectionString: '$(DevDbConnectionString)'  # Secret
```

### Staging Variables
```yaml
Name: Staging-Variables
Variables:
  - AppServiceName: 'app-staging-001'
  - ResourceGroup: 'rg-staging-001'
  - EnvironmentName: 'Staging'
  - DatabaseConnectionString: '$(StagingDbConnectionString)'  # Secret
```

### Production Variables
```yaml
Name: Production-Variables
Variables:
  - AppServiceName: 'app-prod-001'
  - ResourceGroup: 'rg-prod-001'
  - EnvironmentName: 'Production'
  - DatabaseConnectionString: '$(ProdDbConnectionString)'  # Secret
  - EnableMonitoring: 'true'
```

## Service Connections

### Azure Service Connections
```
Name: Azure-Development
Type: Azure Resource Manager
Scope: Subscription
Usage: Development environment deployments

Name: Azure-Staging
Type: Azure Resource Manager
Scope: Subscription
Usage: Staging environment deployments

Name: Azure-Production
Type: Azure Resource Manager
Scope: Subscription
Usage: Production environment deployments
Approval: Required (Pre-deployment)
```

### SonarQube Connection
```
Name: SonarQube-Connection
Type: SonarQube
Server URL: https://sonarqube.company.com
Token: $(SonarQubeToken)  # Secure
```

## Approval Gates

### Pre-Deployment Approvals

**Production Environment**:
- Required Approvers: 2
- Approver Groups: Release Managers, DevOps Lead
- Timeout: 24 hours
- Approval Required For: All deployments
- Auto-approve: Disabled

**Staging Environment**:
- Required Approvers: 1
- Approver Groups: DevOps Team
- Timeout: 8 hours
- Auto-approve: Disabled

**Development Environment**:
- Required Approvers: 0
- Auto-approve: Enabled

### Post-Deployment Gates

**Production**:
- Health Check: Required (30 min timeout)
- Monitoring Alert: Check for critical alerts
- Performance Test: Run smoke tests

**Staging**:
- Health Check: Required (15 min timeout)
- Basic Smoke Test: Required

## Notifications

### Email Notifications
```yaml
# Failed builds
Notify:
  - Build failed: Build initiator, DevOps team
  - Build succeeded (main): Release managers
  - Deployment failed: DevOps team, On-call engineer
  - Approval pending: Approvers

# Teams notifications via webhook
Teams Channel: DevOps Notifications
Events:
  - Deployment started (Production)
  - Deployment completed (Production)
  - Hotfix merged to main
  - Release branch created
```

### Slack Integration
```yaml
Webhook: $(SlackWebhookUrl)
Channel: #deployments
Notifications:
  - Production deployment status
  - Hotfix sync status
  - Build failures (main, release)
```

## Pipeline Best Practices

### 1. Pipeline Organization
- ✅ Use templates for reusable steps
- ✅ Separate CI and CD pipelines
- ✅ Use stages for logical grouping
- ✅ Implement proper error handling

### 2. Security
- ✅ Use service connections for Azure
- ✅ Store secrets in variable groups
- ✅ Enable pipeline permissions
- ✅ Audit pipeline runs

### 3. Performance
- ✅ Use caching for dependencies
- ✅ Parallel job execution
- ✅ Incremental builds when possible
- ✅ Optimize artifact size

### 4. Maintainability
- ✅ Document pipeline purpose
- ✅ Use meaningful names
- ✅ Version control all pipeline files
- ✅ Regular pipeline reviews

## Troubleshooting

### Common Issues

**Issue**: Pipeline fails on merge conflict
**Solution**: Check hotfix-sync.yml, manual resolution required

**Issue**: Deployment timeout
**Solution**: Increase timeout, check App Service health

**Issue**: SonarQube quality gate fails
**Solution**: Review code quality issues, fix or create exception

## Monitoring ve Metrics

### Pipeline Dashboards

**Pipeline Performance Dashboard**:
- Average build duration
- Success rate
- Failed build trends
- Queue time analysis

**Deployment Dashboard**:
- Deployment frequency
- Lead time for changes
- Mean time to recovery (MTTR)
- Change failure rate

## Ek Kaynaklar

- [Branch Strategy](./BRANCH_STRATEGY.md)
- [Branch Policies](./BRANCH_POLICIES.md)
- [Azure DevOps Setup](./AZURE_DEVOPS_SETUP.md)
- [Developer Workflow](./DEVELOPER_WORKFLOW.md)

---

**Son Güncelleme**: 2025-12-16  
**Doküman Versiyonu**: 1.0  
**Onaylayan**: DevOps Team
