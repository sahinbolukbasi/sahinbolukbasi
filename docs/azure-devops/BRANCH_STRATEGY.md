# Azure DevOps Branch Strategy

## Genel Bakış

Bu doküman, projemizin Azure DevOps üzerindeki branch yönetim stratejisini, kurallarını ve iş akışlarını tanımlar.

## Branch Yapısı

### Ana Branch'ler

#### 1. **main** (Production)
- **Amaç**: Production ortamına deploy edilen kararlı kod
- **Koruma Seviyesi**: En Yüksek
- **Güncelleme Yöntemi**: Sadece PR ile
- **Deploy Hedefi**: Production

#### 2. **release** (Staging/Pre-Production)
- **Amaç**: Production'a geçmeden önce test edilen sürümler
- **Koruma Seviyesi**: Yüksek
- **Güncelleme Yöntemi**: Sadece sandbox branch'inden gelen PR'lar
- **Deploy Hedefi**: Staging/Pre-Production
- **Özel Kural**: Son 5 release branch korunmalı (örn: release/v1.0, release/v1.1, vb.)

#### 3. **sandbox** (Development/Integration)
- **Amaç**: Feature'ların entegre edildiği development ortamı
- **Koruma Seviyesi**: Orta
- **Güncelleme Yöntemi**: Feature branch'lerinden gelen PR'lar
- **Deploy Hedefi**: Development

### Yardımcı Branch'ler

#### 4. **feature/** (Feature Branches)
- **Naming Convention**: `feature/<ticket-id>-<short-description>`
  - Örnek: `feature/JIRA-123-user-authentication`
  - Örnek: `feature/ADO-456-payment-integration`
- **Kaynak**: sandbox branch'inden açılır
- **Hedef**: sandbox branch'ine merge edilir
- **Yaşam Döngüsü**: Merge sonrası silinir

#### 5. **hotfix/** (Hotfix Branches)
- **Naming Convention**: `hotfix/<version>-<description>`
  - Örnek: `hotfix/1.2.1-critical-security-fix`
  - Örnek: `hotfix/2.0.1-payment-bug`
- **Kaynak**: main branch'inden açılır
- **Hedef**: main branch'ine merge edilir, ardından otomatik olarak release ve sandbox'a senkronize edilir
- **Yaşam Döngüsü**: Merge ve senkronizasyon sonrası silinir

## Branch İş Akışları

### Feature Geliştirme Akışı

```
1. Developer sandbox branch'inden feature branch açar
   git checkout sandbox
   git pull origin sandbox
   git checkout -b feature/JIRA-123-user-auth

2. Geliştirme yapılır ve commit'lenir
   git add .
   git commit -m "feat: implement user authentication"

3. Feature branch push edilir
   git push origin feature/JIRA-123-user-auth

4. Azure DevOps'ta sandbox'a PR açılır
   - PR template'i doldurulur
   - Code review istenir
   - Build ve test pipeline'ları otomatik çalışır

5. Review approve edildikten ve tüm check'ler geçtikten sonra merge
   - Squash merge kullanılır (temiz history için)
   - Feature branch otomatik silinir

6. Sandbox'ta CI/CD pipeline tetiklenir
   - Development ortamına deploy edilir
```

### Release Akışı

```
1. Sandbox'ta yeterli feature biriktiğinde release hazırlanır
   - Release notları hazırlanır
   - Version numarası belirlenir (Semantic Versioning)

2. Sandbox'tan release branch'ine PR açılır
   git checkout release
   git pull origin release
   
   # Azure DevOps'ta PR oluştur: sandbox → release

3. Release branch'e merge sonrası
   - Staging ortamına deploy edilir
   - QA testleri yapılır
   - UAT (User Acceptance Testing) gerçekleştirilir

4. Release başarılı ise, release → main PR'ı açılır
   git checkout main
   git pull origin main
   
   # Azure DevOps'ta PR oluştur: release → main

5. Main'e merge sonrası
   - Version tag'i oluşturulur (örn: v1.2.0)
   - Production'a deploy edilir
   - Release branch arşivlenir (son 5 korunur)
```

### Hotfix Akışı

```
1. Production'da kritik bug tespit edilir
   - Incident ticket oluşturulur
   - Hotfix branch açılır

   git checkout main
   git pull origin main
   git checkout -b hotfix/1.2.1-security-fix

2. Bug fix uygulanır
   git add .
   git commit -m "fix: critical security vulnerability in auth module"

3. Hotfix branch push edilir
   git push origin hotfix/1.2.1-security-fix

4. Main branch'e PR açılır
   - Priority: Critical
   - Expedited review process
   - Automated tests çalıştırılır

5. PR approve ve merge sonrası
   - Production'a deploy edilir
   - Version tag oluşturulur (patch version artar)
   
6. Otomatik Senkronizasyon Pipeline tetiklenir
   - Hotfix değişiklikleri release branch'ine merge edilir
   - Hotfix değişiklikleri sandbox branch'ine merge edilir
   - Conflict durumunda otomatik PR oluşturulur ve developer'a bildirim gider
```

## Branch Merge Stratejisi

### Feature → Sandbox
- **Strateji**: Squash Merge
- **Neden**: Temiz commit history, feature başına tek commit
- **Commit Mesajı**: Conventional Commits standardı

### Sandbox → Release
- **Strateji**: Merge Commit (No Fast-Forward)
- **Neden**: Release history'sini korumak, hangi feature'ların release'e dahil olduğunu görmek

### Release → Main
- **Strateji**: Merge Commit (No Fast-Forward)
- **Neden**: Production deploy noktalarını açıkça işaretlemek

### Hotfix → Main → Release/Sandbox
- **Strateji**: Cherry-pick veya Merge Commit
- **Neden**: Hotfix'i tüm branch'lere yaymak, conflict minimizasyonu

## Release Branch Yönetimi

### Release Branch Naming Convention
```
release/v<major>.<minor>
Örnekler:
- release/v1.0
- release/v1.1
- release/v2.0
```

### Release Branch Retention Policy

Azure DevOps'ta branch retention policy tanımlanmalı:

```yaml
# Son 5 release branch korunur
# Örnek senaryo:
Active Releases:
  - release/v1.4 (aktif)
  - release/v1.3 (korunur)
  - release/v1.2 (korunur)
  - release/v1.1 (korunur)
  - release/v1.0 (korunur)

Deleted Releases:
  - release/v0.9 (silinir)
  - release/v0.8 (silinir)
```

**Uygulama**:
1. Azure DevOps → Repos → Branches → Branch Policies
2. Release branch pattern için retention ayarı: Keep last 5 branches
3. Tag'ler her zaman korunur (git tag'leri silinmez)

## Branch Protection ve Validation

### Branch Lock/Freeze Durumları

**Production Freeze** (Örn: Tatil öncesi):
- Main branch'e merge geçici olarak durdurulur
- Sadece critical hotfix'ler için exception
- Azure DevOps branch policy ile enforce edilir

**Release Freeze**:
- Release branch'e yeni feature merge'i durdurulur
- Bug fix'ler devam edebilir
- QA ve UAT sürecinde uygulanır

## Conflict Yönetimi

### Feature Branch Conflict'leri
1. Developer kendi feature branch'inde sandbox'ı merge eder
2. Conflict'leri çözer
3. PR'ı günceller

### Hotfix Senkronizasyon Conflict'leri
1. Otomatik pipeline conflict tespit eder
2. PR otomatik oluşturulur (auto-merge disabled)
3. Developer'a atama ve bildirim gönderilir
4. Developer conflict'i çözer ve PR'ı tamamlar

## Best Practices

### Geliştirici Sorumlulukları
1. ✅ Feature branch'leri güncel tutun (düzenli olarak sandbox'tan merge)
2. ✅ Küçük ve odaklı PR'lar oluşturun
3. ✅ Anlamlı commit mesajları yazın (Conventional Commits)
4. ✅ PR description'ı eksiksiz doldurun
5. ✅ Self-review yapın (PR açmadan önce kendi kodunuzu gözden geçirin)

### Code Review Standartları
1. ✅ En az 1 approval gerekli
2. ✅ Kritik değişiklikler için 2 approval
3. ✅ Review süresini 24 saat içinde tamamlayın
4. ✅ Constructive feedback verin
5. ✅ "Approve with comments" yerine "Request changes" kullanın

### Naming Conventions
```
Feature Branch:  feature/<ticket-id>-<description>
Hotfix Branch:   hotfix/<version>-<description>
Release Branch:  release/v<major>.<minor>
Bugfix Branch:   bugfix/<ticket-id>-<description>
```

## Version Tagging

### Semantic Versioning
```
<major>.<minor>.<patch>

Major: Breaking changes (v2.0.0)
Minor: New features (v1.1.0)
Patch: Bug fixes (v1.0.1)
```

### Tag Oluşturma
```bash
# Main branch'te release sonrası
git tag -a v1.2.0 -m "Release version 1.2.0"
git push origin v1.2.0

# Hotfix sonrası
git tag -a v1.2.1 -m "Hotfix: Critical security patch"
git push origin v1.2.1
```

## Monitoring ve Raporlama

### Branch Health Metrics
- Açık PR sayısı ve yaşı
- Feature branch yaşam süresi ortalaması
- Merge time to production
- Hotfix frequency ve response time
- Code review turnaround time

### Azure DevOps Dashboards
1. **Branch Overview Dashboard**: Aktif branch'ler ve durumları
2. **PR Metrics Dashboard**: PR throughput ve review metrikleri
3. **Release Dashboard**: Release history ve deployment status
4. **Hotfix Dashboard**: Hotfix tracking ve metrics

## Acil Durum Prosedürleri

### Critical Production Issue
1. Hotfix branch oluştur (main'den)
2. Hızlı fix uygula ve test et
3. Expedited review süreci
4. Main'e merge ve immediate deploy
5. Otomatik senkronizasyon tetikle

### Rollback Senaryosu
1. Main branch'i önceki stable tag'e revert et
2. Production'a redeploy
3. Incident analysis yap
4. Fix planı oluştur

## Ek Kaynaklar

- [Branch Policy Configuration](./BRANCH_POLICIES.md)
- [Pipeline Configuration](./PIPELINE_STRATEGY.md)
- [Developer Workflow Guide](./DEVELOPER_WORKFLOW.md)
- [Azure DevOps Setup Guide](./AZURE_DEVOPS_SETUP.md)

---

**Son Güncelleme**: 2025-12-16  
**Doküman Versiyonu**: 1.0  
**Onaylayan**: DevOps Team
