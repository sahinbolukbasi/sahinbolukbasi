# Azure DevOps Branch ve Pipeline Stratejisi

## 📋 İçindekiler

Profesyonel Azure DevOps branch yönetimi, pipeline stratejisi ve CI/CD süreçleri için kapsamlı dokümantasyon.

## 📚 Dokümantasyon Yapısı

### 1. [Branch Strategy](./BRANCH_STRATEGY.md) 🌿
**Branch yönetim stratejisi ve iş akışları**

- Ana branch yapısı (main, release, sandbox)
- Feature ve hotfix branch'leri
- Merge stratejileri ve workflow'lar
- Release yönetimi ve versioning
- Best practices ve naming conventions

**Kimler için?** Tüm team members, özellikle developer'lar

### 2. [Branch Policies](./BRANCH_POLICIES.md) 🔒
**Branch koruma kuralları ve policy yapılandırması**

- Branch protection seviyelerine göre policy'ler
- Pull request gereksinimleri
- Code review standartları
- Build validation kuralları
- Branch retention ve cleanup policy'leri
- Permission modeli

**Kimler için?** DevOps engineers, project administrators

### 3. [Pipeline Strategy](./PIPELINE_STRATEGY.md) 🚀
**CI/CD pipeline mimarisi ve otomasyonlar**

- CI/CD pipeline'ları (main, release, sandbox)
- Hotfix otomatik senkronizasyon pipeline'ı
- Deployment pipeline'ları (Dev, Staging, Prod)
- Release cleanup automation
- Pipeline templates ve reusable steps
- Monitoring ve alerting

**Kimler için?** DevOps engineers, automation engineers

### 4. [Developer Workflow Guide](./DEVELOPER_WORKFLOW.md) 👨‍💻
**Günlük geliştirme iş akışları ve pratik kılavuz**

- Feature geliştirme adım adım
- Hotfix süreci
- Pull request oluşturma ve review
- Git komutları ve best practices
- Troubleshooting ve yaygın problemler
- Daily workflow örnekleri

**Kimler için?** Developer'lar, contributor'lar

### 5. [Azure DevOps Setup Guide](./AZURE_DEVOPS_SETUP.md) ⚙️
**Sıfırdan Azure DevOps kurulum kılavuzu**

- Repository oluşturma ve yapılandırma
- Branch policy kurulumu
- Pipeline yapılandırması
- Service connections ve environments
- Variable groups ve secrets yönetimi
- Permissions ve security
- Notifications ve monitoring setup

**Kimler için?** System administrators, DevOps engineers, setup yapacak kişiler

## 🎯 Hızlı Başlangıç

### Developer'lar için
1. 📖 [Developer Workflow Guide](./DEVELOPER_WORKFLOW.md) oku
2. 🌿 [Branch Strategy](./BRANCH_STRATEGY.md) genel bakış
3. 💻 Feature geliştirmeye başla

### DevOps Engineers için
1. ⚙️ [Azure DevOps Setup Guide](./AZURE_DEVOPS_SETUP.md) ile kurulum
2. 🔒 [Branch Policies](./BRANCH_POLICIES.md) yapılandır
3. 🚀 [Pipeline Strategy](./PIPELINE_STRATEGY.md) uygula

### Project Managers için
1. 🌿 [Branch Strategy](./BRANCH_STRATEGY.md) - İş akışlarını anla
2. 📊 Pipeline dashboards ve metrics gözden geçir

## 🔑 Temel Kavramlar

### Branch Yapısı
```
Production (main)
    ↑
Staging (release/*)
    ↑
Development (sandbox)
    ↑
Feature branches (feature/*)
```

### Merge Akışı
```
feature → sandbox (squash merge)
sandbox → release (merge commit)
release → main (merge commit)
main → release, sandbox (hotfix sync)
```

### Pipeline Akışı
```
Code Push → CI Validation → Code Review → PR Merge → CD Deployment → Monitoring
```

## 📊 Branch ve Pipeline Özet

| Branch | Koruma | Deploy Hedef | Merge Kaynağı | Auto-Deploy |
|--------|--------|--------------|---------------|-------------|
| `main` | 🔴 Kritik | Production | release, hotfix | ✅ |
| `release/*` | 🟠 Yüksek | Staging | sandbox, hotfix | ✅ |
| `sandbox` | 🟡 Orta | Development | feature | ✅ |
| `feature/*` | 🟢 Düşük | - | - | ❌ |
| `hotfix/*` | 🟠 Yüksek | Production | - | ✅ |

## 🎪 Önemli Özellikler

### ✨ Hotfix Otomatik Senkronizasyon
- Hotfix main'e merge olduğunda otomatik olarak release ve sandbox'a sync olur
- Conflict durumunda otomatik PR oluşturulur
- Developer'a bildirim gönderilir

### 🏷️ Release Branch Retention
- Son 5 release branch korunur
- Eski branch'ler silinmeden önce tag'lenir
- Haftalık otomatik cleanup pipeline'ı

### 🔐 Çok Katmanlı Güvenlik
- Branch-based protection policies
- Build validation requirements
- Code review enforcement
- SonarQube quality gates
- Dependency scanning

### 📈 Monitoring ve Metrics
- Pipeline performance dashboards
- Branch health monitoring
- Deployment frequency tracking
- Code quality metrics

## 🔄 Tipik İş Akışları

### Feature Geliştirme
```bash
1. sandbox'tan feature branch aç
2. Geliştirme yap ve commit et
3. sandbox'a PR aç
4. Code review al
5. Merge (squash)
6. Development'a auto-deploy
```

### Hotfix Uygulama
```bash
1. main'den hotfix branch aç
2. Bug fix uygula
3. main'e PR aç (expedited review)
4. Merge ve production'a deploy
5. Otomatik sync: release ve sandbox
6. Conflict varsa manuel resolve
```

### Release Süreci
```bash
1. sandbox → release PR
2. Staging'e deploy ve QA
3. UAT tamamla
4. release → main PR
5. Production'a deploy
6. Version tag oluştur
7. Release notes yayınla
```

## 📋 Checklist'ler

### PR Oluşturma Checklist
- [ ] Anlamlı PR title ve description
- [ ] Work item link'lendi
- [ ] Self-review yapıldı
- [ ] Test'ler eklendi/güncellendi
- [ ] Dokümantasyon güncellendi (gerekiyorsa)
- [ ] Build başarılı
- [ ] Conflict yok

### Merge Checklist
- [ ] Minimum review approval aldı
- [ ] Tüm comment'ler resolve edildi
- [ ] Build validation geçti
- [ ] Code quality gate geçti
- [ ] Work item linked

### Deployment Checklist
- [ ] Deployment approval alındı
- [ ] Environment health check OK
- [ ] Database migration hazır (varsa)
- [ ] Rollback planı var
- [ ] Monitoring alerts aktif

## 🛠️ Tools ve Teknolojiler

- **Version Control**: Git, Azure Repos
- **CI/CD**: Azure Pipelines
- **Code Quality**: SonarQube
- **Security Scanning**: WhiteSource, OWASP Dependency Check
- **Deployment**: Azure App Service, Azure CLI
- **Monitoring**: Azure Monitor, Application Insights
- **Notifications**: Email, Microsoft Teams, Slack

## 📖 Ek Kaynaklar

### İnternal Resources
- Team wiki: https://dev.azure.com/your-org/your-project/_wiki
- Training materials: /docs/training
- Video tutorials: Internal SharePoint

### External Resources
- [Azure DevOps Documentation](https://docs.microsoft.com/en-us/azure/devops/)
- [Git Best Practices](https://git-scm.com/book/en/v2)
- [Semantic Versioning](https://semver.org/)
- [Conventional Commits](https://www.conventionalcommits.org/)

## 🤝 Contribution

Bu dokümantasyonu geliştirmek için:

1. Dokümantasyon hatası/eksiklik bulduysanız
2. İyileştirme öneriniz varsa
3. Yeni section eklenmesini öneriyorsanız

**PR açın**: `docs/azure-devops` klasöründe değişiklik yapın ve PR oluşturun.

## 📞 Destek

### Teknik Destek
- **Email**: devops-team@company.com
- **Teams**: #devops-support channel
- **Office Hours**: Monday-Friday, 09:00-18:00 (GMT+3)

### Escalation
- **L1**: DevOps team member
- **L2**: DevOps lead
- **L3**: Platform architecture team

## 🔄 Doküman Güncelleme

- **Frekans**: Quarterly review
- **Sorumlular**: DevOps Team
- **Approval**: Platform Architecture Team
- **Versiyon**: Semantic versioning (Major.Minor.Patch)

## 📝 Versiyon Geçmişi

| Versiyon | Tarih | Değişiklikler | Yazar |
|----------|-------|---------------|-------|
| 1.0.0 | 2025-12-16 | İlk versiyon | DevOps Team |

## 🎓 Training Materyalleri

### Beginner Level
- [ ] Git Fundamentals
- [ ] Azure DevOps Introduction
- [ ] Branch Strategy Overview

### Intermediate Level
- [ ] Advanced Git Workflows
- [ ] Pipeline Development
- [ ] Code Review Best Practices

### Advanced Level
- [ ] Pipeline Architecture
- [ ] Security Hardening
- [ ] Performance Optimization

### Training Schedule
- **Monthly**: Git & Azure DevOps basics (yeni team members için)
- **Quarterly**: Advanced topics workshop
- **On-demand**: One-on-one coaching

## 🏆 Success Metrics

Bu strateji ile hedeflediğimiz metrikler:

- **Deployment Frequency**: Daily (Development), Weekly (Staging), Bi-weekly (Production)
- **Lead Time**: <1 week (feature → production)
- **Mean Time to Recovery (MTTR)**: <1 hour
- **Change Failure Rate**: <5%
- **Code Review Time**: <24 hours
- **Build Success Rate**: >95%
- **Code Coverage**: >80%

## ⚠️ Important Notes

1. **Security**: Asla sensitive data commit etmeyin
2. **Policies**: Branch policy bypass sadece acil durumlarda
3. **Testing**: Merge öncesi kapsamlı test yapın
4. **Documentation**: Değişiklikleri dokümante edin
5. **Communication**: Büyük değişiklikler için team'i bilgilendirin

---

**Maintained by**: DevOps Team  
**Last Review**: 2025-12-16  
**Next Review**: 2026-03-16  
**Document Owner**: Platform Architecture

**Questions?** Reach out to devops-team@company.com
