# Azure DevOps Branch Policies ve Koruma Mekanizmaları

## Genel Bakış

Bu doküman, Azure DevOps'ta her branch için uygulanması gereken branch policy'leri, koruma kurallarını ve validation ayarlarını detaylı olarak açıklar.

## Branch Policy Seviyeleri

### Koruma Seviyeleri
- **🔴 Kritik (Critical)**: main branch
- **🟠 Yüksek (High)**: release branch
- **🟡 Orta (Medium)**: sandbox branch
- **🟢 Düşük (Low)**: feature, hotfix (temporary) branch'ler

## Main Branch Policies

### 1. Require Pull Request
```yaml
Enabled: true
Settings:
  - Require a minimum number of reviewers: 2
  - Allow requestors to approve their own changes: false
  - Prohibit the most recent pusher from approving: true
  - Require at least one approval from Code Owners: true
  - When new changes are pushed: Reset all approval votes
  - Allow completion even if some reviewers vote to wait or reject: false
```

### 2. Check for Linked Work Items
```yaml
Enabled: true
Settings:
  - Required: true
  - Type: Required
  - Message: "Pull request must be linked to at least one work item"
```

### 3. Check for Comment Resolution
```yaml
Enabled: true
Settings:
  - Required: true
  - All comments must be resolved before completion
```

### 4. Build Validation
```yaml
Enabled: true
Builds:
  - Build Pipeline: CI-Main-Validation
    Policy requirement: Required
    Build expiration: Immediately when main is updated
    Display name: "Main Branch CI Validation"
    Path filter: Not specified (applies to all paths)
    
  - Build Pipeline: Security-Scan
    Policy requirement: Required
    Build expiration: After 12 hours
    Display name: "Security and Compliance Scan"
    
  - Build Pipeline: Integration-Tests
    Policy requirement: Required
    Build expiration: After 1 hour
    Display name: "Full Integration Test Suite"
    
  - Build Pipeline: Performance-Tests
    Policy requirement: Optional (but recommended)
    Build expiration: After 2 hours
    Display name: "Performance Regression Tests"
```

### 5. Status Checks
```yaml
Required Status Checks:
  - SonarQube Quality Gate: Required
  - Code Coverage (minimum 80%): Required
  - Dependency Vulnerability Scan: Required
  - License Compliance Check: Required
  - Container Image Scan: Required
```

### 6. Limit Merge Types
```yaml
Allowed Merge Types:
  - Merge (no fast-forward): ✅ Enabled
  - Squash merge: ❌ Disabled
  - Rebase and fast-forward: ❌ Disabled
  - Rebase with merge commit: ❌ Disabled

Settings:
  - Default merge type: Merge (no fast-forward)
```

### 7. Branch Locking
```yaml
Locking Settings:
  - Allow lock: true
  - Lock reason required: true
  - Lock duration maximum: 7 days
  - Auto-unlock on expiration: true
  - Lock bypass permission: Project Administrators only
```

### 8. Additional Settings
```yaml
Settings:
  - Require branches to be up to date before merging: true
  - Automatically include reviewers: 
    - DevOps Team: Required
    - Security Team (for infrastructure changes): Required when paths match */infrastructure/*
  - Enforce validation on every commit: true
  - Prevent direct pushes: true
```

## Release Branch Policies

### 1. Require Pull Request
```yaml
Enabled: true
Settings:
  - Require a minimum number of reviewers: 1
  - Allow requestors to approve their own changes: false
  - Prohibit the most recent pusher from approving: true
  - Require at least one approval from Code Owners: false
  - When new changes are pushed: Reset all approval votes
  - Allow completion even if some reviewers vote to wait or reject: false
```

### 2. Check for Linked Work Items
```yaml
Enabled: true
Settings:
  - Required: true
  - Type: Required
  - Message: "PR must link to release work item or feature"
```

### 3. Source Branch Restrictions
```yaml
Enabled: true
Settings:
  - Allowed source branches:
    - sandbox (for regular releases)
    - hotfix/* (for hotfix propagation only via automated pipeline)
  - Blocked source branches:
    - feature/* (must go through sandbox first)
    - All other branches
  - Enforcement: Strict
```

### 4. Build Validation
```yaml
Enabled: true
Builds:
  - Build Pipeline: CI-Release-Validation
    Policy requirement: Required
    Build expiration: Immediately
    
  - Build Pipeline: UAT-Tests
    Policy requirement: Required
    Build expiration: After 2 hours
    
  - Build Pipeline: Regression-Tests
    Policy requirement: Required
    Build expiration: After 1 hour
    
  - Build Pipeline: Database-Migration-Validation
    Policy requirement: Required (if DB changes detected)
    Path filter: */migrations/*, */database/*
```

### 5. Status Checks
```yaml
Required Status Checks:
  - SonarQube Quality Gate: Required
  - Code Coverage (minimum 75%): Required
  - Staging Deployment Validation: Required
```

### 6. Limit Merge Types
```yaml
Allowed Merge Types:
  - Merge (no fast-forward): ✅ Enabled
  - Squash merge: ❌ Disabled
  - Rebase and fast-forward: ❌ Disabled
  - Rebase with merge commit: ❌ Disabled
```

### 7. Branch Retention Policy
```yaml
Retention:
  - Keep latest: 5 branches
  - Branch pattern: release/v*
  - Grace period before deletion: 90 days after replacement
  - Archive before delete: true
  - Tag releases before archiving: true
  - Notification before deletion: 14 days
  - Notification recipients: DevOps Team, Release Managers
```

### 8. Release Freeze Policy
```yaml
Freeze Settings:
  - Enable freeze mode: true (configurable)
  - Freeze bypass permission: Release Managers only
  - Freeze notification: All team members
  - Default freeze periods:
    - Year-end: December 20 - January 5
    - Pre-major releases: Configurable per release
```

## Sandbox Branch Policies

### 1. Require Pull Request
```yaml
Enabled: true
Settings:
  - Require a minimum number of reviewers: 1
  - Allow requestors to approve their own changes: false
  - Prohibit the most recent pusher from approving: false
  - Require at least one approval from Code Owners: false
  - When new changes are pushed: Retain all approval votes
  - Allow completion even if some reviewers vote to wait or reject: true
```

### 2. Check for Linked Work Items
```yaml
Enabled: true
Settings:
  - Required: false (recommended but not enforced)
  - Type: Optional
```

### 3. Source Branch Restrictions
```yaml
Enabled: true
Settings:
  - Allowed source branches:
    - feature/*
    - bugfix/*
    - hotfix/* (for hotfix propagation)
  - Blocked source branches:
    - main
    - release/*
  - Enforcement: Strict
```

### 4. Build Validation
```yaml
Enabled: true
Builds:
  - Build Pipeline: CI-Sandbox-Validation
    Policy requirement: Required
    Build expiration: After 4 hours
    
  - Build Pipeline: Unit-Tests
    Policy requirement: Required
    Build expiration: After 1 hour
    
  - Build Pipeline: Code-Quality-Check
    Policy requirement: Optional
    Build expiration: After 2 hours
```

### 5. Status Checks
```yaml
Required Status Checks:
  - Build Success: Required
  - Unit Tests Pass: Required
  - Code Quality (SonarQube): Recommended
  - Code Coverage (minimum 60%): Recommended
```

### 6. Limit Merge Types
```yaml
Allowed Merge Types:
  - Merge (no fast-forward): ✅ Enabled
  - Squash merge: ✅ Enabled (default)
  - Rebase and fast-forward: ❌ Disabled
  - Rebase with merge commit: ❌ Disabled

Settings:
  - Default merge type: Squash merge
```

### 7. Auto-Complete
```yaml
Auto-Complete Settings:
  - Enable auto-complete: true
  - Auto-complete conditions:
    - All required reviewers approved: true
    - All build validations passed: true
    - All comments resolved: true
    - Work item linked: false
  - Auto-delete source branch: true
  - Squash when auto-completing: true
```

## Feature Branch Policies

### 1. Naming Convention Enforcement
```yaml
Branch Pattern: feature/<ticket-id>-<description>
Regex: ^feature\/[A-Z]+-\d+-[a-z0-9-]+$
Examples:
  - ✅ feature/JIRA-123-user-authentication
  - ✅ feature/ADO-456-payment-gateway
  - ❌ feature/user-auth (no ticket ID)
  - ❌ feature/JIRA-123 (no description)

Enforcement:
  - Validation on branch creation: true
  - Automatic branch deletion on invalid pattern: false
  - Warning message: true
  - Block PR if pattern invalid: true
```

### 2. Lifecycle Management
```yaml
Settings:
  - Maximum branch age: 30 days
  - Stale branch warning: After 14 days
  - Auto-deletion of merged branches: true
  - Auto-deletion delay: 1 day after merge
  - Notification before deletion: 1 day
```

### 3. Required Updates
```yaml
Settings:
  - Require branch to be up-to-date with sandbox: Recommended
  - Maximum commits behind sandbox: 20 (warning threshold)
  - Force update notification: After 10 days without sync
```

## Hotfix Branch Policies

### 1. Naming Convention
```yaml
Branch Pattern: hotfix/<version>-<description>
Regex: ^hotfix\/v?\d+\.\d+\.\d+-[a-z0-9-]+$
Examples:
  - ✅ hotfix/1.2.1-security-patch
  - ✅ hotfix/v2.0.1-critical-bug
  - ❌ hotfix/fix-bug (no version)

Enforcement:
  - Strict validation: true
  - Block non-compliant branches: true
```

### 2. Source Branch Validation
```yaml
Settings:
  - Must branch from: main
  - Validation on creation: true
  - Block if not from main: true
  - Exception override: Project Administrators only
```

### 3. Expedited Review Process
```yaml
Settings:
  - Priority label: Critical
  - Required reviewers: 1 (reduced from normal)
  - Review SLA: 2 hours
  - Auto-assign reviewers: On-call DevOps engineer
  - Escalation: After 2 hours, notify team lead
```

### 4. Lifecycle
```yaml
Settings:
  - Auto-delete after merge: true
  - Auto-delete delay: Immediate (after sync completion)
  - Require completion checklist:
    - ✅ Merged to main
    - ✅ Synced to release (automated)
    - ✅ Synced to sandbox (automated)
    - ✅ Post-deployment verification
```

## Global Policy Settings

### 1. Code Owners (CODEOWNERS file)
```
# Global Code Owners
*                                   @devops-team

# Specific Path Owners
/infrastructure/*                   @infrastructure-team @security-team
/database/migrations/*              @database-team @backend-team
/.azure-pipelines/*                 @devops-team
/src/security/*                     @security-team
/docs/*                             @tech-writers @devops-team

# Branch Specific
release/*                           @release-managers
hotfix/*                            @on-call-engineer @devops-lead
```

### 2. Required Reviewers by Path
```yaml
Infrastructure Changes:
  Path: */infrastructure/*, *.tf, *.bicep
  Required Reviewers: @infrastructure-team
  Required Approvals: 2

Database Changes:
  Path: */migrations/*, */database/*
  Required Reviewers: @database-team
  Required Approvals: 1

Security Changes:
  Path: */security/*, */auth/*
  Required Reviewers: @security-team
  Required Approvals: 1

Pipeline Changes:
  Path: .azure-pipelines/*, azure-pipelines.yml
  Required Reviewers: @devops-team
  Required Approvals: 1
```

### 3. Repository Settings
```yaml
Security:
  - Require signed commits: false (optional recommendation)
  - Require two-factor authentication: true (organization-level)
  - Disable force push: true (all protected branches)
  - Disable branch deletion: true (main, release/*)

Pull Requests:
  - Allow draft PRs: true
  - Auto-link work items: true
  - Auto-complete enabled: true (with restrictions)
  - Limit number of active PRs per user: 5 (recommended)

Merge:
  - Delete source branch on merge: true (default)
  - Require merge commit: true (for main, release)
  - Allow squash merge: true (for sandbox)
```

## Branch Permission Model

### Main Branch Permissions
```yaml
Administrators:
  - Allow: All operations
  - Members: Project Administrators

DevOps Team:
  - Allow: Create tag, Contribute to pull requests
  - Deny: Force push, Bypass policies, Delete branch

Developers:
  - Allow: Contribute to pull requests
  - Deny: Force push, Delete branch, Bypass policies

Readers:
  - Allow: Read only
```

### Release Branch Permissions
```yaml
Release Managers:
  - Allow: All operations (except force push)
  - Members: Release Manager Group

DevOps Team:
  - Allow: Create branch, Contribute to pull requests
  - Deny: Force push, Delete branch (without approval)

Developers:
  - Allow: Contribute to pull requests
  - Deny: Direct push, Delete branch
```

### Sandbox Branch Permissions
```yaml
Developers:
  - Allow: Contribute to pull requests
  - Deny: Force push, Delete branch, Bypass policies

All Users:
  - Allow: Read, Create feature branches from sandbox
```

## Bypass Permissions

### Policy Bypass
```yaml
Allowed to Bypass:
  - Project Administrators: All policies
  - Build Service Account: Build validation policies only
  - Automated Sync Pipeline: Source branch restrictions (hotfix sync)

Conditions:
  - Bypass must be logged
  - Bypass reason required
  - Notification to security team
  - Audit retention: 1 year
```

## Monitoring ve Compliance

### Policy Compliance Dashboard
```yaml
Metrics:
  - Policy bypass frequency
  - Average PR review time
  - Policy violation attempts
  - Branch age distribution
  - Merge frequency per branch
  - Failed build validation rate

Alerts:
  - Policy bypass notification: Immediate
  - Stale branch alert: After 14 days
  - Large PR alert: >500 lines changed
  - Missing work item link: Daily summary
```

### Audit Trail
```yaml
Tracked Events:
  - Policy changes
  - Policy bypass events
  - Branch protection modifications
  - Permission changes
  - Force push attempts
  - Branch deletions

Retention:
  - Audit logs: 1 year
  - Security events: 2 years
```

## Uygulama Adımları

### Azure DevOps'ta Policy Kurulumu

1. **Branch Policies Ekranına Erişim**
   ```
   Project Settings → Repositories → [Your Repo] → Policies
   ```

2. **Main Branch Policy Oluşturma**
   ```
   - Branch name: main
   - Add policy → Require a minimum number of reviewers
   - Configure diğer policy'leri (yukarıdaki ayarlara göre)
   ```

3. **Pattern-based Policies (release/* için)**
   ```
   - Branch name pattern: release/*
   - Configure retention ve diğer policy'leri
   ```

4. **Build Validation Policy Ekleme**
   ```
   - Add policy → Build validation
   - Select build pipeline
   - Configure trigger ve expiration
   ```

5. **Status Check Policy**
   ```
   - Add policy → Status check
   - Configure external status checks (SonarQube, etc.)
   ```

## Best Practices

### Policy Management
1. ✅ Policy değişikliklerini version control'de takip edin
2. ✅ Policy değişikliklerini dokümante edin
3. ✅ Major policy değişikliklerini team'e duyurun
4. ✅ Policy bypass'ları düzenli olarak audit edin
5. ✅ Policy effectiveness'ı quarterly review edin

### Review Process
1. ✅ Review checklist kullanın
2. ✅ Constructive feedback verin
3. ✅ Security ve performance açısından değerlendirin
4. ✅ Yeterli context olmadan approve etmeyin
5. ✅ Review SLA'larına uyun

## Troubleshooting

### Common Issues

**Problem**: PR policy bypass gerekiyor ama permission yok
**Çözüm**: Project Administrator'dan onay alın, bypass reason belgelendirin

**Problem**: Build validation sürekli fail oluyor
**Çözüm**: Build pipeline'ı debug edin, gerekirse policy'yi temporarily disable edin (admin approval ile)

**Problem**: Review SLA'sına uymakta zorluk
**Çözüm**: Reviewer pool'u genişletin, auto-assign rules optimize edin

## Ek Kaynaklar

- [Branch Strategy](./BRANCH_STRATEGY.md)
- [Pipeline Strategy](./PIPELINE_STRATEGY.md)
- [Azure DevOps Setup](./AZURE_DEVOPS_SETUP.md)

---

**Son Güncelleme**: 2025-12-16  
**Doküman Versiyonu**: 1.0  
**Onaylayan**: DevOps Team & Security Team
