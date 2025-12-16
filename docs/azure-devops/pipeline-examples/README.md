# Azure Pipeline Örnekleri

Bu dizin, Azure DevOps branch ve pipeline stratejimiz için kullanılabilecek hazır pipeline YAML dosyalarını içerir.

## 📂 Dosya Yapısı

```
pipeline-examples/
├── ci-main-validation.yml          # Main branch CI validation
├── ci-release-validation.yml       # Release branch CI validation
├── ci-sandbox-validation.yml       # Sandbox branch CI validation
├── cd-production.yml                # Production deployment
├── cd-staging.yml                   # Staging deployment
├── cd-development.yml               # Development deployment
├── hotfix-sync.yml                  # Hotfix automatic sync
├── release-cleanup.yml              # Release branch cleanup
└── templates/
    ├── build-steps.yml              # Common build steps
    ├── deploy-webapp.yml            # Web app deployment template
    ├── unit-tests.yml               # Unit test template
    └── code-quality.yml             # Code quality checks template
```

## 🚀 Kullanım

### 1. Repository'nize Kopyalama

```bash
# Pipeline dosyalarını kopyalayın
cp -r docs/azure-devops/pipeline-examples/.azure-pipelines /path/to/your/repo/

# Veya manuel olarak
mkdir -p .azure-pipelines/pipelines
mkdir -p .azure-pipelines/templates

# İhtiyacınız olan pipeline'ları kopyalayın
```

### 2. Pipeline'ı Özelleştirme

Her pipeline dosyasında aşağıdaki alanları projenize göre güncelleyin:

```yaml
# Değiştirilmesi gerekenler:

variables:
  buildConfiguration: 'Release'          # ✅ Genelde değiştirme
  COVERAGE_THRESHOLD: 80                  # ⚠️ İhtiyaca göre ayarla
  
pool:
  vmImage: 'ubuntu-latest'               # ⚠️ Windows için: 'windows-latest'

# Service connection isimleri
azureSubscription: 'Azure-Production'    # 🔴 Mutlaka değiştir
appName: '$(AppServiceName)'             # 🔴 Variable group'tan gelir

# Build commands (projenize göre)
task: DotNetCoreCLI@2                    # ⚠️ Node.js için: Npm@1
                                         # ⚠️ Python için: UsePythonVersion@0
```

### 3. Azure DevOps'ta Pipeline Oluşturma

```bash
# Portal'dan:
1. Pipelines → New pipeline
2. Azure Repos Git → Select repository
3. Existing Azure Pipelines YAML file
4. Path: /.azure-pipelines/pipelines/ci-main-validation.yml
5. Review and Run

# CLI ile:
az pipelines create \
  --name "CI-Main-Validation" \
  --repository your-repo \
  --repository-type tfsgit \
  --branch main \
  --yml-path .azure-pipelines/pipelines/ci-main-validation.yml
```

## 📋 Pipeline Açıklamaları

### CI Pipelines

#### ci-main-validation.yml
- **Trigger**: main branch'e PR veya commit
- **Amaç**: Production quality validation
- **Stages**: Build → Test → Security Scan
- **Coverage**: 80% minimum
- **Duration**: ~5-10 dakika

#### ci-release-validation.yml
- **Trigger**: release/* branch'lerine PR veya commit
- **Amaç**: Staging quality validation
- **Stages**: Build → UAT Tests → Regression Tests
- **Coverage**: 75% minimum
- **Duration**: ~10-15 dakika

#### ci-sandbox-validation.yml
- **Trigger**: sandbox branch'e PR veya commit
- **Amaç**: Quick development feedback
- **Stages**: Build → Unit Tests
- **Coverage**: 60% minimum
- **Duration**: ~3-5 dakika

### CD Pipelines

#### cd-production.yml
- **Trigger**: main branch commit, version tags
- **Environment**: Production
- **Approval**: 2 approvers required
- **Stages**: Pre-checks → Deploy → Post-deploy validation
- **Rollback**: Automatic on failure

#### cd-staging.yml
- **Trigger**: release/* branch commit
- **Environment**: Staging
- **Approval**: 1 approver
- **Stages**: Deploy → Smoke tests
- **Purpose**: QA ve UAT

#### cd-development.yml
- **Trigger**: sandbox branch commit
- **Environment**: Development
- **Approval**: None (automatic)
- **Stages**: Fast deploy
- **Purpose**: Developer testing

### Utility Pipelines

#### hotfix-sync.yml
- **Trigger**: main branch commit (hotfix merge detection)
- **Amaç**: Hotfix'leri otomatik olarak release ve sandbox'a sync et
- **Logic**: 
  1. Hotfix merge tespit et
  2. Release branch'e merge
  3. Sandbox branch'e merge
  4. Conflict varsa work item oluştur

#### release-cleanup.yml
- **Schedule**: Haftalık (Pazar 02:00)
- **Amaç**: Eski release branch'lerini temizle
- **Logic**:
  1. Release branch'lerini listele
  2. Son 5'i koruy
  3. Eskilerini tag'le (arşiv için)
  4. Sil

## 🔧 Template'ler

### templates/build-steps.yml
Ortak build adımları:
- SDK/Runtime kurulumu
- Dependency restore
- Build
- Artifact publish

**Kullanım**:
```yaml
steps:
  - template: templates/build-steps.yml
```

### templates/deploy-webapp.yml
Web app deployment template:
- Parametre: environment, subscription, appName
- Azure Web App deployment
- Health check

**Kullanım**:
```yaml
steps:
  - template: templates/deploy-webapp.yml
    parameters:
      environment: 'Production'
      azureSubscription: 'Azure-Production'
      appName: 'my-app-prod'
```

### templates/unit-tests.yml
Unit test execution template:
- Test çalıştırma
- Coverage collection
- Results publish

**Kullanım**:
```yaml
steps:
  - template: templates/unit-tests.yml
```

### templates/code-quality.yml
Code quality checks:
- SonarQube analysis
- Code style checks
- Complexity analysis

**Kullanım**:
```yaml
steps:
  - template: templates/code-quality.yml
    parameters:
      sonarConnection: 'SonarQube-Connection'
      projectKey: 'my-project'
```

## 🎯 Technology Stack Örnekleri

### .NET Core

```yaml
# Build
- task: UseDotNet@2
  inputs:
    packageType: 'sdk'
    version: '8.x'

- task: DotNetCoreCLI@2
  inputs:
    command: 'build'
    projects: '**/*.csproj'

# Test
- task: DotNetCoreCLI@2
  inputs:
    command: 'test'
    projects: '**/*Tests.csproj'
```

### Node.js

```yaml
# Build
- task: NodeTool@0
  inputs:
    versionSpec: '18.x'

- task: Npm@1
  inputs:
    command: 'install'

- task: Npm@1
  inputs:
    command: 'custom'
    customCommand: 'run build'

# Test
- task: Npm@1
  inputs:
    command: 'custom'
    customCommand: 'test'
```

### Python

```yaml
# Build
- task: UsePythonVersion@0
  inputs:
    versionSpec: '3.11'

- script: |
    python -m pip install --upgrade pip
    pip install -r requirements.txt
  displayName: 'Install dependencies'

# Test
- script: |
    pip install pytest pytest-cov
    pytest --cov=. --cov-report=xml
  displayName: 'Run tests'
```

### Java/Maven

```yaml
# Build
- task: Maven@3
  inputs:
    mavenPomFile: 'pom.xml'
    goals: 'clean package'
    javaHomeOption: 'JDKVersion'
    jdkVersionOption: '17'

# Test
- task: Maven@3
  inputs:
    mavenPomFile: 'pom.xml'
    goals: 'test'
    publishJUnitResults: true
    testResultsFiles: '**/TEST-*.xml'
```

## 🔐 Secrets ve Variables

### Variable Groups Gereksinimi

Pipeline'lar aşağıdaki variable group'ları bekler:

#### Development-Variables
```yaml
Variables:
  - AppServiceName: 'your-app-dev'
  - ResourceGroup: 'rg-dev'
  - EnvironmentName: 'Development'
Secrets:
  - DatabaseConnectionString
  - ApiKey
```

#### Staging-Variables
```yaml
Variables:
  - AppServiceName: 'your-app-staging'
  - ResourceGroup: 'rg-staging'
  - EnvironmentName: 'Staging'
Secrets:
  - DatabaseConnectionString
  - ApiKey
```

#### Production-Variables
```yaml
Variables:
  - AppServiceName: 'your-app-prod'
  - ResourceGroup: 'rg-prod'
  - EnvironmentName: 'Production'
Secrets:
  - DatabaseConnectionString
  - ApiKey
  - AlertWebhook
```

### Variable Group Oluşturma

```bash
az pipelines variable-group create \
  --name "Development-Variables" \
  --variables \
    AppServiceName=your-app-dev \
    ResourceGroup=rg-dev \
    EnvironmentName=Development \
  --authorize true

# Secret ekle
az pipelines variable-group variable create \
  --group-id <group-id> \
  --name DatabaseConnectionString \
  --value "your-connection-string" \
  --secret true
```

## 📊 Pipeline Best Practices

### Performance Optimization

```yaml
# Cache kullanımı
- task: Cache@2
  inputs:
    key: 'npm | "$(Agent.OS)" | package-lock.json'
    restoreKeys: |
      npm | "$(Agent.OS)"
    path: $(npm_config_cache)
  displayName: 'Cache npm packages'

# Parallel jobs
strategy:
  matrix:
    linux:
      vmImage: 'ubuntu-latest'
    windows:
      vmImage: 'windows-latest'
    mac:
      vmImage: 'macOS-latest'

# Condition'lar ile gereksiz adımları skip et
condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
```

### Security Hardening

```yaml
# Secret masking
- script: echo "##vso[task.setvariable variable=mySecret;issecret=true]$(SecretValue)"

# Secure files
- task: DownloadSecureFile@1
  name: certificateFile
  inputs:
    secureFile: 'certificate.pfx'

# Permission kontrolü
- task: AzureCLI@2
  inputs:
    scriptType: 'bash'
    inlineScript: |
      # Verify permissions before deployment
      az account show
```

## 🐛 Troubleshooting

### Common Pipeline Issues

#### Build Fails - Dependency Not Found
```yaml
# Solution: Restore dependencies explicitly
- task: DotNetCoreCLI@2
  inputs:
    command: 'restore'
    projects: '**/*.csproj'
    feedsToUse: 'select'
    vstsFeed: 'your-feed-id'  # Private package feed
```

#### Test Timeout
```yaml
# Solution: Increase timeout
- task: DotNetCoreCLI@2
  inputs:
    command: 'test'
    arguments: '--logger trx --timeout 300000'  # 5 minutes
  timeoutInMinutes: 10
```

#### Deployment Fails - Connection Timeout
```yaml
# Solution: Verify service connection and retry
- task: AzureWebApp@1
  retryCountOnTaskFailure: 3
  inputs:
    azureSubscription: 'Azure-Production'
    # ... other settings
```

## 📖 Additional Resources

- [Azure Pipelines Documentation](https://docs.microsoft.com/en-us/azure/devops/pipelines/)
- [YAML Schema Reference](https://docs.microsoft.com/en-us/azure/devops/pipelines/yaml-schema)
- [Task Reference](https://docs.microsoft.com/en-us/azure/devops/pipelines/tasks/)
- [Best Practices](../PIPELINE_STRATEGY.md)

## 🤝 Contributing

Pipeline iyileştirmeleri için:
1. Değişikliği test edin
2. Dokümantasyonu güncelleyin
3. PR oluşturun

---

**Maintained by**: DevOps Team  
**Last Updated**: 2025-12-16  
**Version**: 1.0
