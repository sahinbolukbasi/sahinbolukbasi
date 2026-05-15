# Azure DevOps Setup Guide

## Genel Bakış

Bu kılavuz, Azure DevOps'ta branch strategy, policies ve pipeline'ları sıfırdan kurulum için adım adım talimatlar içerir.

## Prerequisites

### Gereksinimler

1. **Azure DevOps Organization**
   - Organization admin erişimi
   - Project admin erişimi

2. **Azure Subscription**
   - Development, Staging, Production subscription'ları
   - Service Principal oluşturma yetkisi

3. **Tools**
   - Git (2.30+)
   - Azure CLI (2.40+)
   - Azure DevOps CLI Extension

### Azure CLI Kurulumu

```bash
# Azure CLI kurulumu
# Windows (PowerShell)
Invoke-WebRequest -Uri https://aka.ms/installazurecliwindows -OutFile .\AzureCLI.msi
Start-Process msiexec.exe -Wait -ArgumentList '/I AzureCLI.msi /quiet'

# macOS
brew update && brew install azure-cli

# Linux (Ubuntu/Debian)
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash

# Azure DevOps extension ekleme
az extension add --name azure-devops

# Login
az login
az devops configure --defaults organization=https://dev.azure.com/your-org project=your-project
```

## 1. Repository Oluşturma

### 1.1 Yeni Repository Oluşturma

```bash
# Azure DevOps Portal
1. Organization → Project seçin
2. Repos → New repository
   - Repository name: your-repo-name
   - Add a README: Yes
   - .gitignore template: Visual Studio (veya uygun template)

# CLI ile
az repos create --name your-repo-name --project your-project
```

### 1.2 İlk Branch Yapısını Oluşturma

```bash
# Repository'yi clone edin
git clone https://dev.azure.com/your-org/your-project/_git/your-repo-name
cd your-repo-name

# Main branch (default olarak var)
git checkout main

# Sandbox branch oluştur
git checkout -b sandbox
git push -u origin sandbox

# İlk release branch oluştur
git checkout -b release/v1.0
git push -u origin release/v1.0

# Default branch'i main olarak ayarla (portal'dan)
# Project Settings → Repositories → your-repo → Default branch → main
```

## 2. Branch Policies Kurulumu

### 2.1 Main Branch Policies

```bash
# Azure DevOps Portal
1. Project Settings → Repositories → your-repo
2. Policies → main branch
3. Configure policies:

# Branch Policies Ekranında:

# 1. Require a minimum number of reviewers
✅ Enable
   - Minimum number: 2
   - ✅ Allow requestors to approve their own changes: NO
   - ✅ Prohibit the most recent pusher from approving: YES
   - ✅ Reset votes when there are new changes: YES

# 2. Check for linked work items
✅ Enable
   - Required: Yes

# 3. Check for comment resolution
✅ Enable
   - Required: Yes

# 4. Limit merge types
✅ Enable
   - ✅ Allow merge (no fast-forward)
   - ❌ Squash merge
   - ❌ Rebase and fast-forward
   - ❌ Rebase with merge commit
```

### 2.2 Release Branch Policies (Pattern-based)

```bash
# Azure DevOps Portal
1. Project Settings → Repositories → your-repo
2. Policies → Add branch pattern
3. Branch name pattern: release/*
4. Configure policies:

# 1. Require a minimum number of reviewers
✅ Enable
   - Minimum number: 1
   - ✅ Reset votes when there are new changes: YES

# 2. Check for linked work items
✅ Enable
   - Required: Yes

# 3. Limit merge types
✅ Enable
   - ✅ Allow merge (no fast-forward) only
```

### 2.3 Sandbox Branch Policies

```bash
# Azure DevOps Portal
1. Project Settings → Repositories → your-repo
2. Policies → sandbox branch
3. Configure policies:

# 1. Require a minimum number of reviewers
✅ Enable
   - Minimum number: 1
   - ❌ Allow requestors to approve their own changes: NO
   - ❌ Prohibit the most recent pusher from approving: NO
   - ❌ Reset votes when there are new changes: NO (retain votes)

# 2. Check for linked work items
✅ Enable
   - Required: Optional

# 3. Limit merge types
✅ Enable
   - ✅ Allow merge (no fast-forward)
   - ✅ Allow squash merge (default)
   - ❌ Rebase options
```

### 2.4 CLI ile Policy Oluşturma

```bash
# Main branch policy
az repos policy approver-count create \
  --blocking true \
  --branch main \
  --enabled true \
  --minimum-approver-count 2 \
  --creator-vote-counts false \
  --allow-downvotes false \
  --reset-on-source-push true \
  --repository-id $(az repos show --repository your-repo --query id -o tsv)

# Build validation policy
az repos policy build create \
  --blocking true \
  --branch main \
  --enabled true \
  --build-definition-id <build-definition-id> \
  --display-name "CI Validation" \
  --manual-queue-only false \
  --queue-on-source-update-only false \
  --repository-id $(az repos show --repository your-repo --query id -o tsv)

# Work item linking policy
az repos policy work-item-linking create \
  --blocking true \
  --branch main \
  --enabled true \
  --repository-id $(az repos show --repository your-repo --query id -o tsv)

# Comment resolution policy
az repos policy comment-required create \
  --blocking true \
  --branch main \
  --enabled true \
  --repository-id $(az repos show --repository your-repo --query id -o tsv)
```

## 3. Service Connections Kurulumu

### 3.1 Azure Resource Manager Connection

```bash
# Portal'dan
1. Project Settings → Service connections
2. New service connection → Azure Resource Manager
3. Authentication method: Service Principal (automatic)
4. Subscription: Select subscription
5. Resource group: (leave empty for subscription scope)
6. Service connection name: Azure-Development
7. Grant access permission to all pipelines: NO (güvenlik için)
8. Save

# Development, Staging, Production için tekrarla
```

### 3.2 Service Principal ile Manuel Oluşturma

```bash
# Azure CLI ile Service Principal oluştur
az ad sp create-for-rbac \
  --name "sp-azdevops-development" \
  --role Contributor \
  --scopes /subscriptions/<subscription-id>/resourceGroups/<rg-name>

# Output:
{
  "appId": "xxx",
  "displayName": "sp-azdevops-development",
  "password": "xxx",
  "tenant": "xxx"
}

# Azure DevOps'ta Service Connection oluştur
1. Project Settings → Service connections
2. New service connection → Azure Resource Manager
3. Authentication method: Service Principal (manual)
4. Subscription ID: <your-subscription-id>
5. Subscription Name: Development
6. Service Principal Id: <appId from above>
7. Service Principal Key: <password from above>
8. Tenant ID: <tenant from above>
9. Service connection name: Azure-Development
10. Verify ve Save
```

### 3.3 SonarQube Connection

```bash
# SonarQube'de token oluştur
1. SonarQube → My Account → Security → Generate Token
2. Token kopyala

# Azure DevOps'ta connection oluştur
1. Project Settings → Service connections
2. New service connection → SonarQube
3. Server URL: https://sonarqube.company.com
4. Token: <generated-token>
5. Service connection name: SonarQube-Connection
6. Save
```

## 4. Pipeline Kurulumu

### 4.1 Pipeline Dizin Yapısı

```bash
# Repository'de pipeline dosyalarını oluştur
mkdir -p .azure-pipelines/templates
mkdir -p .azure-pipelines/pipelines

# Dizin yapısı:
.azure-pipelines/
├── pipelines/
│   ├── ci-main-validation.yml
│   ├── ci-release-validation.yml
│   ├── ci-sandbox-validation.yml
│   ├── cd-production.yml
│   ├── cd-staging.yml
│   ├── cd-development.yml
│   ├── hotfix-sync.yml
│   └── release-cleanup.yml
└── templates/
    ├── build-steps.yml
    ├── deploy-webapp.yml
    ├── unit-tests.yml
    └── code-quality.yml
```

### 4.2 CI Main Validation Pipeline Oluşturma

```bash
# Portal'dan
1. Pipelines → New pipeline
2. Where is your code? → Azure Repos Git
3. Select repository: your-repo
4. Configure: Existing Azure Pipelines YAML file
5. Path: /.azure-pipelines/pipelines/ci-main-validation.yml
6. Run

# Pipeline'ı trigger olarak ayarla
1. Edit pipeline
2. Triggers → Enable continuous integration
3. Branch filters: main
4. Path filters: Exclude docs/*, *.md
5. Save

# Build validation policy olarak ekle
1. Project Settings → Repositories → Policies → main
2. Build validation → Add build policy
3. Build pipeline: CI-Main-Validation
4. Policy requirement: Required
5. Build expiration: Immediately
6. Save
```

### 4.3 Hotfix Sync Pipeline Oluşturma

```yaml
# .azure-pipelines/pipelines/hotfix-sync.yml dosyasını oluştur
# (Detaylı içerik PIPELINE_STRATEGY.md'de)

# Pipeline'ı oluştur
1. Pipelines → New pipeline
2. Existing YAML: /.azure-pipelines/pipelines/hotfix-sync.yml
3. Save (Run değil!)

# Trigger ayarları
1. Edit pipeline
2. Triggers → Enable CI
3. Branch: main
4. Path exclusion: docs/*, *.md
5. Save

# Pipeline'a özel permissions
1. Pipeline → Edit → More actions → Security
2. Pipeline permissions → Add
   - Build Service Account: Allow
3. Repository permissions → Add
   - Allow contribute to pull requests
   - Allow create branch
   - Allow create tag
```

### 4.4 CD Production Pipeline Oluşturma

```bash
# Environment oluştur (Production)
1. Pipelines → Environments
2. New environment
   - Name: Production
   - Description: Production environment
   - Resource: None (for now)
3. Approvals and checks
   - Approvals: Add
     - Approvers: Release Managers, DevOps Lead
     - Minimum number of approvals: 2
     - Timeout: 24 hours
   - Save

# Pipeline oluştur
1. Pipelines → New pipeline
2. Existing YAML: /.azure-pipelines/pipelines/cd-production.yml
3. Save

# Trigger ayarları
1. Edit pipeline
2. Triggers → Enable CI
3. Branch: main
4. Include: main, tags: v*
5. Save

# Variables ayarla
1. Edit pipeline → Variables
2. Add variable group: Production-Variables
   - Link an existing variable group
   - Select: Production-Variables
3. Save
```

### 4.5 Schedule-based Pipeline (Release Cleanup)

```bash
# Pipeline oluştur
1. Pipelines → New pipeline
2. Existing YAML: /.azure-pipelines/pipelines/release-cleanup.yml

# Schedule ekle (YAML'de tanımlı ama verify edin)
schedules:
  - cron: "0 2 * * 0"  # Her Pazar 02:00
    displayName: Weekly release cleanup
    branches:
      include:
        - main
    always: true

# Save ve enable
```

## 5. Variable Groups Kurulumu

### 5.1 Development Variables

```bash
# Portal'dan
1. Pipelines → Library → Variable groups
2. + Variable group
3. Variable group name: Development-Variables
4. Variables:
   - AppServiceName: app-dev-yourapp
   - ResourceGroup: rg-dev-yourapp
   - EnvironmentName: Development
5. Add secret variables:
   - DatabaseConnectionString: (value) [Lock icon tıkla]
   - ApiKey: (value) [Lock icon tıkla]
6. Save

# Pipeline permissions
1. Pipeline permissions → Add pipeline
2. Select pipelines that can use this group:
   - CI-Sandbox-Validation
   - CD-Development
3. Save
```

### 5.2 Staging Variables

```bash
# Staging-Variables variable group
Variables:
  - AppServiceName: app-staging-yourapp
  - ResourceGroup: rg-staging-yourapp
  - EnvironmentName: Staging
Secrets:
  - DatabaseConnectionString: [locked]
  - ApiKey: [locked]

Pipelines:
  - CI-Release-Validation
  - CD-Staging
```

### 5.3 Production Variables

```bash
# Production-Variables variable group
Variables:
  - AppServiceName: app-prod-yourapp
  - ResourceGroup: rg-prod-yourapp
  - EnvironmentName: Production
  - EnableMonitoring: true
Secrets:
  - DatabaseConnectionString: [locked]
  - ApiKey: [locked]
  - AlertWebhook: [locked]

Pipelines:
  - CI-Main-Validation
  - CD-Production

# Variable group approvals (extra security)
1. More actions → Approvals and checks
2. Approvals: Add
   - Approvers: Security Team
   - Minimum: 1
3. Save
```

### 5.4 CLI ile Variable Group Oluşturma

```bash
# Variable group oluştur
az pipelines variable-group create \
  --name "Development-Variables" \
  --variables \
    AppServiceName=app-dev-yourapp \
    ResourceGroup=rg-dev-yourapp \
    EnvironmentName=Development \
  --authorize true

# Secret variable ekle
az pipelines variable-group variable create \
  --group-id <group-id> \
  --name DatabaseConnectionString \
  --value "Server=..." \
  --secret true
```

## 6. Environments Kurulumu

### 6.1 Development Environment

```bash
# Portal'dan
1. Pipelines → Environments → New environment
2. Name: Development
3. Description: Development environment
4. Resource: Azure App Service (optional)
   - Azure subscription: Azure-Development
   - App Service name: app-dev-yourapp
5. Create

# Approvals: None (auto-deploy)
```

### 6.2 Staging Environment

```bash
# Staging environment
Name: Staging
Resource: Azure App Service
  - Subscription: Azure-Staging
  - App name: app-staging-yourapp

Approvals:
  - Approvers: DevOps Team
  - Minimum: 1
  - Timeout: 8 hours
```

### 6.3 Production Environment

```bash
# Production environment
Name: Production
Resource: Azure App Service
  - Subscription: Azure-Production
  - App name: app-prod-yourapp

Approvals and Checks:
  1. Approvals:
     - Approvers: Release Managers, DevOps Lead
     - Minimum: 2
     - Timeout: 24 hours
  
  2. Business Hours:
     - Only allow deployments during business hours
     - Time zone: (GMT+3) Istanbul
     - Business hours: Monday-Friday, 09:00-18:00
  
  3. Invoke Azure Function (optional):
     - Health check before deployment
     - Function URL: https://health-check.azurewebsites.net/api/check
  
  4. Query Work Items:
     - Ensure all linked work items are in "Done" state

Security:
  - Restrict access: Only Production-CD pipeline
```

## 7. Permissions ve Security

### 7.1 Repository Permissions

```bash
# Portal'dan
1. Project Settings → Repositories → your-repo → Security
2. Groups:

# Project Administrators
- All permissions: Allow

# DevOps Team
- Contribute: Allow
- Create branch: Allow
- Create tag: Allow
- Manage permissions: Deny
- Force push: Deny
- Remove others' locks: Deny

# Developers
- Read: Allow
- Contribute to pull requests: Allow
- Create branch: Allow (feature/* only)
- Force push: Deny

# Readers
- Read: Allow
- All others: Deny
```

### 7.2 Branch Permissions

```bash
# Main branch
1. Repos → Branches → main → Security
2. Set permissions:

# Project Administrators
- All: Allow

# Build Service
- Contribute: Allow (for auto-tagging)
- Create branch: Deny
- Delete branch: Deny
- Force push: Deny

# Developers
- Contribute: Deny (PR only)
- Read: Allow

# Configure for release/* pattern similarly
```

### 7.3 Pipeline Permissions

```bash
# Pipeline-level permissions
1. Pipelines → CI-Main-Validation → Edit → More actions → Security
2. Pipeline permissions:

# Project Administrators
- All: Allow

# DevOps Team
- Queue builds: Allow
- Edit build pipeline: Allow
- Delete build pipeline: Deny

# Developers
- View builds: Allow
- Queue builds: Deny
```

## 8. Notifications Kurulumu

### 8.1 Team Notifications

```bash
# Portal'dan
1. Project Settings → Notifications
2. New subscription → Team

# Build completed (main branch)
Event: Build completed
Filters:
  - Build pipeline: CI-Main-Validation
  - Build status: Failed
Delivery:
  - Deliver to: DevOps Team
  - Email
  
# Pull request created
Event: Pull request created
Filters:
  - Target branch: main
Delivery:
  - Teams channel webhook
  - Channel: #devops-notifications
```

### 8.2 Personal Notifications

```bash
# User Settings → Notifications
1. New subscription → Personal

# PR assigned to me
Event: Pull request reviewers updated
Filters:
  - I am a reviewer
Delivery:
  - Email
  - Browser notification

# Build failed (my builds)
Event: Build completed
Filters:
  - Initiated by me
  - Status: Failed
Delivery:
  - Email (immediate)
```

### 8.3 Service Hooks (Slack/Teams)

```bash
# Slack Integration
1. Project Settings → Service hooks
2. Create subscription → Slack
3. Trigger: Build completed
4. Filters:
   - Build status: Failed, Succeeded
   - Build pipeline: CD-Production
5. Action:
   - Webhook URL: https://hooks.slack.com/services/...
   - Channel: #deployments
   - Message format:
     Build {{buildNumber}} {{buildStatus}}
     Pipeline: {{buildDefinitionName}}
     Branch: {{sourceBranch}}
6. Test → Finish

# Microsoft Teams
1. Teams channel → Connectors → Azure DevOps
2. Configure Azure DevOps connector
3. Select events:
   - Pull requests
   - Releases
   - Builds
4. Save
```

## 9. Code Owners (CODEOWNERS)

### 9.1 CODEOWNERS File Oluşturma

```bash
# Repository root'da dosya oluştur
# .azuredevops/CODEOWNERS

# Global owners
* @devops-team

# Backend code
/src/backend/** @backend-team
/src/api/** @api-team

# Frontend code
/src/frontend/** @frontend-team
/src/web/** @frontend-team

# Infrastructure
/infrastructure/** @infrastructure-team @security-team
*.bicep @infrastructure-team
*.tf @infrastructure-team

# Database
/database/** @database-team
/migrations/** @database-team @backend-team

# CI/CD
/.azure-pipelines/** @devops-team
/azure-pipelines.yml @devops-team

# Documentation
/docs/** @tech-writers @devops-team
*.md @tech-writers

# Security-sensitive
/src/security/** @security-team
/src/authentication/** @security-team @backend-team

# Configuration
/config/** @devops-team @security-team
appsettings*.json @devops-team @backend-team
```

### 9.2 CODEOWNERS Policy Aktif Etme

```bash
# Portal'dan
1. Project Settings → Repositories → your-repo → Policies
2. Branch: main
3. Add policy → Require a minimum number of reviewers
4. ✅ Require at least one approval from Code Owners
5. Save

# Tekrarla: release/*, sandbox branches için
```

## 10. Dashboard ve Reporting

### 10.1 Pipeline Dashboard

```bash
# Portal'dan
1. Overview → Dashboards → New Dashboard
2. Name: CI/CD Pipeline Dashboard
3. Add widgets:

# Build History
- Widget: Build History
- Build pipeline: All
- Time period: Last 30 days

# Deployment Status
- Widget: Deployment Status
- Environment: Production
- Time period: Last 7 days

# Test Results
- Widget: Test Results Trend
- Build pipeline: CI-Main-Validation

# Code Coverage
- Widget: Code Coverage
- Build pipeline: CI-Main-Validation

4. Save
```

### 10.2 Branch Health Dashboard

```bash
# New Dashboard: Branch Health
Widgets:
1. Pull Request Status
   - Active PRs
   - PR age distribution
   
2. Branch Age
   - Stale branches (>14 days)
   - Feature branch count
   
3. Build Success Rate
   - Per branch success rate
   - Trend over time
   
4. Deployment Frequency
   - Deployments per week
   - Environment breakdown
```

### 10.3 Custom Queries (Work Items)

```bash
# Queries → New query → Flat list

# Open PRs by Team Member
SELECT [System.Id], [System.Title], [System.State]
FROM workitems
WHERE [System.WorkItemType] = 'Pull Request'
AND [System.State] = 'Active'
AND [System.AssignedTo] = @Me

# Feature Branch Work Items
SELECT [System.Id], [System.Title], [System.State]
FROM workitems
WHERE [System.WorkItemType] IN ('User Story', 'Bug')
AND [System.Tags] CONTAINS 'feature-branch'
AND [System.State] <> 'Done'

# Save query and add to dashboard
```

## 11. Monitoring ve Alerts

### 11.1 Azure Monitor Integration

```bash
# Azure Portal
1. Application Insights → your-app
2. Continuous export → Add
3. Export to: Azure DevOps Work Items
4. Configure:
   - Event: Exception
   - Severity: Error or higher
   - Action: Create work item
   - Work item type: Bug
   - Area path: your-project
   - Assigned to: On-call engineer

# Azure DevOps'ta görünür
# Boards → Work Items → Filter: Created by Azure Monitor
```

### 11.2 Pipeline Failure Alerts

```bash
# Azure DevOps → Notifications
1. New subscription → Build completed
2. Filters:
   - Status: Failed
   - Build pipeline: CD-Production
3. Action:
   - Create work item
   - Type: Incident
   - Priority: 1
   - Assigned to: On-call engineer
   - Title: "Production deployment failed: {{buildNumber}}"
4. Save
```

## 12. Backup ve Disaster Recovery

### 12.1 Repository Backup

```bash
# Automated backup script (scheduled pipeline)
# Create: .azure-pipelines/backup-repo.yml

schedules:
  - cron: "0 0 * * *"  # Daily
    branches:
      include:
        - main

steps:
  - checkout: self
    fetchDepth: 0
    
  - script: |
      git bundle create repo-backup-$(date +%Y%m%d).bundle --all
      
  - task: AzureCLI@2
    inputs:
      azureSubscription: 'Azure-Production'
      scriptType: 'bash'
      inlineScript: |
        az storage blob upload \
          --account-name backupstorage \
          --container-name repo-backups \
          --name repo-backup-$(date +%Y%m%d).bundle \
          --file repo-backup-$(date +%Y%m%d).bundle
```

### 12.2 Pipeline Export

```bash
# Export all pipelines
az pipelines list --output json > pipelines-backup.json

# Export variable groups
az pipelines variable-group list --output json > variable-groups-backup.json

# Store in secure location
# Encrypt if contains sensitive data
```

## 13. Troubleshooting

### Common Setup Issues

#### Issue: Service Connection Authentication Failed

```bash
# Solution
1. Verify Service Principal credentials
2. Check RBAC permissions in Azure
   az role assignment list --assignee <sp-app-id>
3. Recreate Service Principal if needed
4. Update Service Connection with new credentials
```

#### Issue: Pipeline Can't Access Repository

```bash
# Solution
1. Pipeline Settings → Triggers → YAML → Get sources
2. ✅ Grant access permission to all pipelines
   
Or:
1. Project Settings → Repositories → Security
2. Build Service → Allow: Contribute, Create branch
```

#### Issue: Build Validation Not Triggering

```bash
# Solution
1. Branch Policy → Build validation
2. Verify:
   - ✅ Build definition selected correctly
   - ✅ Branch matches (exact match or pattern)
   - ✅ Path filters correct
   - ✅ Policy enabled
3. Test: Create PR and verify build queues
```

## 14. Validation Checklist

### Post-Setup Verification

```markdown
## Repository Setup
- [ ] Main branch exists and is default
- [ ] Sandbox branch created
- [ ] Release/v1.0 branch created
- [ ] .gitignore configured
- [ ] CODEOWNERS file created

## Branch Policies
- [ ] Main branch policies configured (2 reviewers)
- [ ] Release/* pattern policies configured
- [ ] Sandbox policies configured (1 reviewer)
- [ ] Build validation enabled for all protected branches
- [ ] Work item linking required
- [ ] Comment resolution required

## Pipelines
- [ ] CI-Main-Validation created and working
- [ ] CI-Release-Validation created
- [ ] CI-Sandbox-Validation created
- [ ] CD-Production created with approvals
- [ ] CD-Staging created
- [ ] CD-Development created
- [ ] Hotfix-Sync pipeline created
- [ ] Release-Cleanup scheduled pipeline created

## Service Connections
- [ ] Azure-Development connection working
- [ ] Azure-Staging connection working
- [ ] Azure-Production connection working
- [ ] SonarQube connection working (if applicable)

## Variable Groups
- [ ] Development-Variables created
- [ ] Staging-Variables created
- [ ] Production-Variables created
- [ ] Secrets properly secured
- [ ] Pipeline permissions assigned

## Environments
- [ ] Development environment created
- [ ] Staging environment created with approval
- [ ] Production environment created with approvals
- [ ] Resource mappings configured

## Permissions
- [ ] Repository permissions assigned
- [ ] Branch permissions configured
- [ ] Pipeline permissions set
- [ ] Build service has required permissions

## Notifications
- [ ] Team notifications configured
- [ ] Service hooks for Slack/Teams working
- [ ] Email notifications working

## Testing
- [ ] Test PR to sandbox (should work)
- [ ] Test PR to main (should require 2 reviews)
- [ ] Test build trigger on main
- [ ] Test deployment to Development
- [ ] Test hotfix flow (create branch, PR, sync)
```

## Next Steps

1. **Team Onboarding**
   - Share documentation with team
   - Conduct training session
   - Create demo repository for practice

2. **Monitoring Setup**
   - Configure dashboards
   - Set up alerts
   - Regular health checks

3. **Continuous Improvement**
   - Collect team feedback
   - Iterate on policies
   - Update documentation

## Support

### Resources
- [Branch Strategy](./BRANCH_STRATEGY.md)
- [Branch Policies](./BRANCH_POLICIES.md)
- [Pipeline Strategy](./PIPELINE_STRATEGY.md)
- [Developer Workflow](./DEVELOPER_WORKFLOW.md)

### Contacts
- **Setup Help**: devops-team@company.com
- **Azure Support**: azure-support@company.com
- **Documentation**: Update via PR to docs/

---

**Son Güncelleme**: 2025-12-16  
**Doküman Versiyonu**: 1.0  
**Hazırlayan**: DevOps Team
