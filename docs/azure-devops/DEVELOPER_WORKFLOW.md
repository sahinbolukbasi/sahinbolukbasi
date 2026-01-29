# Developer Workflow Guide

## Genel Bakış

Bu kılavuz, geliştiricilerin günlük iş akışlarında kullanacakları branch yönetimi, PR süreci ve best practice'leri adım adım açıklar.

## Quick Start

### İlk Kurulum

```bash
# 1. Repository'yi clone edin
git clone https://dev.azure.com/your-org/your-project/_git/your-repo
cd your-repo

# 2. Git yapılandırması
git config user.name "Your Name"
git config user.email "your.email@company.com"

# 3. Ana branch'leri fetch edin
git fetch origin
git checkout main
git checkout sandbox
git checkout release  # En güncel release branch
```

### Branch Yapısını Anlama

```
main (Production)
├── release/v1.4 (Staging)
│   └── sandbox (Development)
│       ├── feature/JIRA-123-user-auth
│       ├── feature/JIRA-124-payment-integration
│       └── feature/JIRA-125-email-service
└── hotfix/1.4.1-critical-bug (Emergency fixes)
```

## Feature Geliştirme İş Akışı

### 1. Yeni Feature Başlatma

```bash
# Sandbox branch'ini güncel hale getirin
git checkout sandbox
git pull origin sandbox

# Feature branch oluşturun (naming convention'a uygun)
git checkout -b feature/JIRA-123-user-authentication

# İlk push (remote'a branch'i gönderin)
git push -u origin feature/JIRA-123-user-authentication
```

#### Naming Convention Örnekleri
```
✅ feature/JIRA-123-user-authentication
✅ feature/ADO-456-payment-gateway-integration
✅ feature/GH-789-email-notification-service

❌ feature/user-auth (Ticket ID yok)
❌ feature/JIRA-123 (Açıklama yok)
❌ feature-user-auth (Format yanlış)
❌ user-auth (Prefix yok)
```

### 2. Geliştirme Yapma

```bash
# Değişikliklerinizi yapın
# Dosyaları düzenleyin...

# Değişiklikleri gözden geçirin
git status
git diff

# Commit hazırlığı
git add .
# veya spesifik dosyalar
git add src/Authentication/UserService.cs

# Commit (Conventional Commits formatında)
git commit -m "feat(auth): implement JWT token validation"
```

#### Conventional Commits Örnekleri

```bash
# Feature ekleme
git commit -m "feat(payment): add Stripe payment integration"

# Bug fix
git commit -m "fix(auth): resolve token expiration issue"

# Dokümantasyon
git commit -m "docs(api): update authentication endpoints"

# Refactoring
git commit -m "refactor(services): extract common validation logic"

# Test ekleme
git commit -m "test(auth): add unit tests for token service"

# Performance iyileştirme
git commit -m "perf(database): optimize user query with indexing"

# Breaking change
git commit -m "feat(api)!: change authentication response format

BREAKING CHANGE: API response structure changed from { token } to { accessToken, refreshToken }"
```

### 3. Branch'i Güncel Tutma

```bash
# Sandbox'tan son değişiklikleri alın (düzenli olarak yapın)
git checkout sandbox
git pull origin sandbox

git checkout feature/JIRA-123-user-authentication
git merge sandbox

# Conflict varsa çözün
# Dosyaları düzenleyin...
git add .
git commit -m "merge: resolve conflicts with sandbox"

# Güncel branch'i push edin
git push origin feature/JIRA-123-user-authentication
```

#### Alternatif: Rebase Kullanma (Temiz history için)

```bash
# Sandbox'ı güncel al
git checkout sandbox
git pull origin sandbox

# Feature branch'e dön ve rebase yap
git checkout feature/JIRA-123-user-authentication
git rebase sandbox

# Conflict varsa:
# 1. Dosyaları düzenle
# 2. git add <resolved-files>
# 3. git rebase --continue
# İptal etmek için: git rebase --abort

# Rebase sonrası force push gerekli (dikkatli kullanın!)
git push --force-with-lease origin feature/JIRA-123-user-authentication
```

### 4. Pull Request Oluşturma

#### Azure DevOps'ta PR Açma

1. **Web UI'dan PR Oluşturma**
```
Repos → Pull Requests → New Pull Request

Source Branch: feature/JIRA-123-user-authentication
Target Branch: sandbox
```

2. **PR Şablonunu Doldurun**
```markdown
## Description
Implemented JWT-based user authentication system with token refresh mechanism.

## Type of Change
- [x] New feature
- [ ] Bug fix
- [ ] Breaking change
- [ ] Documentation update

## Related Work Items
- Closes #JIRA-123
- Related to #JIRA-100

## Changes Made
- Added JwtTokenService for token generation
- Implemented token validation middleware
- Created refresh token endpoint
- Added unit tests (85% coverage)

## Testing Performed
- [x] Unit tests added/updated
- [x] Integration tests passed
- [x] Manual testing completed
- [x] Local development environment tested

## Screenshots (if applicable)
[Add screenshots if UI changes]

## Checklist
- [x] Code follows team coding standards
- [x] Self-review performed
- [x] Comments added for complex logic
- [x] Documentation updated
- [x] No merge conflicts
- [x] All tests passing
- [x] Build successful
```

3. **Reviewer'ları Atayın**
- Minimum 1 reviewer (sandbox için)
- Code owners otomatik atanır
- Specific expertise için manuel atama

#### CLI ile PR Oluşturma (Azure CLI)

```bash
# Azure DevOps CLI kurulumu (tek seferlik)
az extension add --name azure-devops

# Login
az login
az devops configure --defaults organization=https://dev.azure.com/your-org project=your-project

# PR oluştur
az repos pr create \
  --source-branch feature/JIRA-123-user-authentication \
  --target-branch sandbox \
  --title "feat(auth): implement JWT authentication" \
  --description "Implements JWT-based authentication system. Closes #JIRA-123" \
  --work-items JIRA-123 \
  --reviewers "user1@company.com" "user2@company.com"
```

### 5. Code Review Süreci

#### PR Sahibi Sorumlulukları

```bash
# Review feedback sonrası değişiklik yapma
git checkout feature/JIRA-123-user-authentication

# Değişiklikleri yap
# Dosyaları düzenle...

# Commit ve push
git add .
git commit -m "fix: address code review feedback"
git push origin feature/JIRA-123-user-authentication

# PR otomatik güncellenir
```

#### Self-Review Checklist

PR göndermeden önce kendiniz kontrol edin:

```markdown
## Code Quality
- [ ] Kod okunabilir ve anlaşılır mı?
- [ ] Gereksiz comment'ler var mı?
- [ ] Console.log, debugger statements kaldırıldı mı?
- [ ] Dead code temizlendi mi?

## Functionality
- [ ] Kod çalışıyor mu?
- [ ] Edge case'ler handle ediliyor mu?
- [ ] Error handling uygun mu?
- [ ] Validation'lar eksiksiz mi?

## Testing
- [ ] Unit test'ler eklendi mi?
- [ ] Test coverage yeterli mi?
- [ ] Tüm test'ler geçiyor mu?
- [ ] Manuel test yapıldı mı?

## Security
- [ ] Input validation var mı?
- [ ] Sensitive data expose edilmiyor mu?
- [ ] SQL injection risk'i yok mu?
- [ ] XSS vulnerability yok mu?

## Performance
- [ ] Gereksiz loop veya query var mı?
- [ ] Memory leak riski yok mu?
- [ ] Database query'leri optimize mi?

## Documentation
- [ ] README güncellendi mi (gerekiyorsa)?
- [ ] API dokümantasyonu güncel mi?
- [ ] Complex logic açıklandı mı?
```

### 6. Merge ve Cleanup

```bash
# PR merge edildikten sonra (Azure DevOps UI'dan complete edilir)

# Yerel cleanup
git checkout sandbox
git pull origin sandbox  # Merge edilmiş değişiklikleri al

# Feature branch'i sil
git branch -d feature/JIRA-123-user-authentication

# Remote'tan da silinir (auto-delete enabled ise)
# Manuel silmek için:
git push origin --delete feature/JIRA-123-user-authentication

# Stale branch'leri temizle
git remote prune origin
```

## Hotfix İş Akışı

### 1. Hotfix Başlatma

```bash
# Production'da kritik bug tespit edildi!

# Main branch'ten hotfix oluştur
git checkout main
git pull origin main

# Hotfix branch oluştur (version numarasına dikkat!)
# Mevcut version 1.4.0 ise, patch version artır: 1.4.1
git checkout -b hotfix/1.4.1-payment-processing-error

# Push to remote
git push -u origin hotfix/1.4.1-payment-processing-error
```

### 2. Hızlı Fix Uygulama

```bash
# Bug'ı fix et (minimal changes!)
# Sadece gerekli değişiklikleri yap

# Test et (critical!)
# Lokalde kapsamlı test yap

# Commit
git add .
git commit -m "fix(payment): resolve processing error for international cards"

# Push
git push origin hotfix/1.4.1-payment-processing-error
```

### 3. Hotfix PR ve Merge

```bash
# Azure DevOps'ta PR oluştur
# Source: hotfix/1.4.1-payment-processing-error
# Target: main
# Priority: Critical

# PR Template
"""
## HOTFIX - CRITICAL

### Issue
Production payment processing failing for international cards.
Error rate: 45%
Impact: 10,000+ users affected

### Root Cause
Currency conversion service timeout (30s → need 5s)

### Fix Applied
- Reduced timeout to 5s with retry logic
- Added fallback to cached rates
- Improved error handling

### Testing
- [x] Unit tests (95% coverage)
- [x] Integration tests passed
- [x] Load tested with 1000 concurrent requests
- [x] Tested with real international cards (staging)

### Rollback Plan
If deployment fails, rollback to v1.4.0 via Azure DevOps release.

### Post-Deployment Verification
1. Monitor payment success rate
2. Check error logs for 30 minutes
3. Verify international card processing

Closes #INCIDENT-789
"""

# Expedited review (max 2 hours)
# Merge to main
# Auto-deploy to production triggers
# Hotfix sync pipeline automatically runs (merge to release & sandbox)
```

### 4. Post-Hotfix Verification

```bash
# Production'da verify et
# Monitoring dashboard'u kontrol et

# Sandbox ve release'e otomatik sync yapıldığını kontrol et
git checkout sandbox
git pull origin sandbox
git log --oneline -5  # Hotfix commit'i görünmeli

git checkout release
git pull origin release
git log --oneline -5  # Hotfix commit'i görünmeli

# Conflict varsa, Azure DevOps'ta otomatik oluşturulan work item'a bak
# Manuel resolve gerekebilir
```

## Release İş Akışı

### 1. Release Hazırlığı

```bash
# Sandbox'ta yeterli feature birikti mi kontrol et
git checkout sandbox
git log --oneline origin/release..HEAD  # Release'den sonraki commit'ler

# Release notes hazırla
# CHANGELOG.md güncelle
```

### 2. Release Branch'e PR

```bash
# Release branch'e merge için PR oluştur
# Source: sandbox
# Target: release

# PR Template
"""
## Release v1.5.0

### New Features
- User authentication with JWT (#JIRA-123)
- Payment gateway integration (#JIRA-124)
- Email notification service (#JIRA-125)

### Bug Fixes
- Fixed login timeout issue (#JIRA-130)
- Resolved payment display bug (#JIRA-131)

### Improvements
- Database query optimization
- API response time improvement (30% faster)

### Breaking Changes
None

### Migration Notes
- Run database migrations: `dotnet ef database update`
- Update configuration: Add SMTP settings

### Testing Checklist
- [x] All unit tests passing
- [x] Integration tests passing
- [x] UAT completed
- [ ] Performance testing pending
- [ ] Security scan completed

Closes Sprint 15
"""
```

### 3. QA ve UAT

```bash
# Release branch'e merge sonrası staging'e deploy olur

# QA Testing
# - Functional testing
# - Regression testing
# - Performance testing
# - Security testing

# UAT
# - Business validation
# - User acceptance
```

### 4. Production Release

```bash
# QA ve UAT başarılı ise

# Release → Main PR oluştur
# Final approval (Release Manager)
# Merge to main
# Production deployment triggers

# Tag oluştur (manual veya pipeline)
git checkout main
git pull origin main
git tag -a v1.5.0 -m "Release version 1.5.0"
git push origin v1.5.0
```

## Daily Workflow Örnekleri

### Sabah Rutini

```bash
# 1. Son değişiklikleri al
git checkout sandbox
git pull origin sandbox

git checkout main
git pull origin main

# 2. Aktif feature branch'ini güncelle
git checkout feature/JIRA-123-user-authentication
git merge sandbox

# 3. Değişiklikleri kontrol et
git log --oneline -10
git status

# 4. Günlük planlamaya başla
```

### Akşam Rutini

```bash
# 1. Tüm değişiklikleri commit et
git status
git add .
git commit -m "feat: add user profile validation"

# 2. Push et
git push origin feature/JIRA-123-user-authentication

# 3. PR durumunu kontrol et
# Azure DevOps → Pull Requests
# Review feedback var mı?

# 4. Yarın için TODO
# - Review comments address et
# - Integration test ekle
```

## Troubleshooting

### Problem: Merge Conflict

```bash
# Conflict oluştu
git merge sandbox
# CONFLICT in src/file.cs

# Solution 1: Manuel resolve
# 1. Dosyayı aç ve conflict marker'ları bul
# <<<<<<< HEAD
# Your changes
# =======
# Incoming changes
# >>>>>>> sandbox

# 2. Conflict'i resolve et (doğru kodu seç)
# 3. Marker'ları kaldır

git add src/file.cs
git commit -m "merge: resolve conflict in file.cs"

# Solution 2: Visual Studio / VS Code merge tool kullan
# Otomatik conflict resolution UI'ı
```

### Problem: Yanlış Branch'te Commit

```bash
# Yanlışlıkla main'de değişiklik yaptın!

# Solution 1: Commit yapmadıysan
git stash
git checkout feature/JIRA-123-user-authentication
git stash pop

# Solution 2: Commit yaptıysan ama push etmedin
git log --oneline -1  # Commit hash'i al: abc123

git checkout feature/JIRA-123-user-authentication
git cherry-pick abc123

git checkout main
git reset --hard HEAD~1  # Son commit'i geri al

# Solution 3: Push ettiysen
# Admin'e danış, force push gerekebilir
```

### Problem: Branch Silindi Ama Henüz Merge Edilmedi

```bash
# Branch yanlışlıkla silindi

# Local'de hala varsa
git checkout feature/JIRA-123-user-authentication
git push origin feature/JIRA-123-user-authentication

# Local'de de yoksa, reflog kullan
git reflog
# Branch'in son commit'ini bul
git checkout -b feature/JIRA-123-user-authentication <commit-hash>
git push origin feature/JIRA-123-user-authentication
```

### Problem: Large File Accidentally Committed

```bash
# Büyük dosya commit edildi (örn: video, build artifacts)

# Henüz push edilmediyse
git reset HEAD~1  # Son commit'i unstage et
# .gitignore'a ekle
echo "*.mp4" >> .gitignore
git add .gitignore
git commit -m "chore: add video files to gitignore"

# Push edildiyse
# BFG Repo-Cleaner veya git filter-branch kullan
# https://rtyley.github.io/bfg-repo-cleaner/
```

## Best Practices Özet

### Do's ✅

1. **Küçük, Focused PR'lar**
   - Single responsibility
   - Max 400 satır değişiklik (ideal)
   - Bir feature/bug per PR

2. **Sık Commit**
   - Logical units halinde
   - Meaningful commit messages
   - Work-in-progress işaretlemesi

3. **Branch Güncel Tutma**
   - Günde en az 1 kez sandbox'tan merge
   - Conflict'leri erkenden çöz

4. **Comprehensive Testing**
   - Unit test coverage >80%
   - Integration tests
   - Manuel test

5. **Documentation**
   - README güncel
   - Code comments (complex logic)
   - API documentation

### Don'ts ❌

1. **Force Push (Genelde)**
   - Shared branch'lerde asla
   - Kendi feature branch'inde bile dikkatli

2. **Long-Running Feature Branches**
   - Max 2 hafta
   - Daha uzunsa feature'ı böl

3. **Direct Push to Protected Branches**
   - Main, release, sandbox: sadece PR ile

4. **Commit Sensitive Data**
   - Passwords, API keys, secrets
   - .gitignore doğru yapılandır

5. **Ignore CI/CD Failures**
   - Build fail: immediate fix
   - Test fail: debug ve fix

## Useful Commands

### Git Aliases (Setup Once)

```bash
# ~/.gitconfig veya git config --global

[alias]
    st = status
    co = checkout
    br = branch
    ci = commit
    unstage = reset HEAD --
    last = log -1 HEAD
    visual = log --graph --oneline --all
    sync = !git checkout sandbox && git pull origin sandbox
    cleanup = !git branch --merged | grep -v \"\\*\" | xargs -n 1 git branch -d
```

### Useful Git Commands

```bash
# Branch'ler arası diff
git diff main..sandbox

# Commit history görselleştirme
git log --graph --oneline --all --decorate

# Specific file history
git log --follow -- path/to/file.cs

# Who changed what
git blame path/to/file.cs

# Stash with message
git stash save "WIP: authentication refactoring"

# List all stashes
git stash list

# Apply specific stash
git stash apply stash@{2}

# Interactive rebase (son 3 commit düzenle)
git rebase -i HEAD~3

# Cherry-pick multiple commits
git cherry-pick abc123 def456 ghi789

# Find commit by message
git log --grep="authentication"

# Undo last commit (keep changes)
git reset --soft HEAD~1

# Undo last commit (discard changes)
git reset --hard HEAD~1
```

## Support ve Resources

### İletişim

- **DevOps Team**: devops-team@company.com
- **Azure DevOps**: https://dev.azure.com/your-org
- **Team Chat**: Microsoft Teams - #devops-support

### Documentation

- [Branch Strategy](./BRANCH_STRATEGY.md)
- [Branch Policies](./BRANCH_POLICIES.md)
- [Pipeline Strategy](./PIPELINE_STRATEGY.md)
- [Azure DevOps Setup](./AZURE_DEVOPS_SETUP.md)

### Training

- Azure DevOps Fundamentals (internal)
- Git Best Practices Workshop (quarterly)
- Security Guidelines (required)

---

**Son Güncelleme**: 2025-12-16  
**Doküman Versiyonu**: 1.0  
**Hazırlayan**: DevOps Team
