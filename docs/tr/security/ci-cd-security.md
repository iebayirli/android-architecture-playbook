# Android Security — CI/CD & Güvenli Geliştirme Rehberi

> **Ana fikir:** Diğer security konuları "açığı nasıl bulursun" sorusunu cevaplar.  
> Bu döküman "açığın production'a gitmesini nasıl **engellersin**" sorusunu cevaplar.

---

## İçindekiler

1. [Büyük Resim — Security Pipeline Nedir?](#1-büyük-resim)
2. [Araç Üçlüsü — Minimum Viable Security Pipeline](#2-araç-üçlüsü)
3. [Semgrep — SAST Kurulumu ve Kullanımı](#3-semgrep)
4. [OWASP Dependency Check — Bağımlılık Güvenliği](#4-owasp-dependency-check)
5. [Gitleaks — Secret Scanning](#5-gitleaks)
6. [GitHub Actions — Tam Pipeline Entegrasyonu](#6-github-actions-tam-pipeline)
7. [MobSF — APK Otomatik Tarama](#7-mobsf-apk-tarama)
8. [Gradle Güvenlik Konfigürasyonu](#8-gradle-güvenlik-konfigürasyonu)
9. [PR Güvenlik Review Checklist](#9-pr-güvenlik-review-checklist)
10. [Güvenlik Test Piramidi](#10-güvenlik-test-piramidi)
11. [Ölçüm & Raporlama](#11-ölçüm--raporlama)

---

## 1. Büyük Resim

### Neden CI/CD'ye Security Entegre Edilmeli?

Manuel security review'ların iki temel sorunu vardır: ölçeklenmez ve geç kalır. Bir açık ne kadar erken bulunursa maliyeti o kadar düşer:

```
Geliştirme aşaması  →  Düzeltme maliyeti: 1x
Code review         →  Düzeltme maliyeti: 6x
QA/Test             →  Düzeltme maliyeti: 15x
Production          →  Düzeltme maliyeti: 100x
```

### Security Pipeline Katmanları

```
┌─────────────────────────────────────────────────────────┐
│                   DEVELOPER MACHINE                     │
│  Pre-commit hook: Gitleaks (secret scan)                │
│  IDE: Android Studio + lint security rules              │
└────────────────────┬────────────────────────────────────┘
                     │ git push
┌────────────────────▼────────────────────────────────────┐
│                   PULL REQUEST                          │
│  1. Semgrep SAST          → kod kalitesi & güvenlik     │
│  2. Gitleaks CI           → hardcoded secret kontrolü   │
│  3. Dependency Check      → CVE taraması                │
│  4. Lint Security Rules   → Android özel kurallar       │
└────────────────────┬────────────────────────────────────┘
                     │ merge → main
┌────────────────────▼────────────────────────────────────┐
│                   RELEASE BUILD                         │
│  5. MobSF APK Scan        → binary güvenlik analizi     │
│  6. APK imza doğrulama    → signing şema kontrolü       │
│  7. ProGuard/R8 raporu    → obfuscation doğrulama       │
└─────────────────────────────────────────────────────────┘
```

### "Shift Left" Prensibi

Security kontrollerini geliştirme sürecinin soluna (erken aşamaya) taşımak demektir. Pre-commit hook'lar, IDE uyarıları ve PR kontrolleri — hepsi "shift left" araçlarıdır.

---

## 2. Araç Üçlüsü

Başlamak için gereken minimum set. Bu üçünü kur, gerisini sonra ekle.

| Araç                       | Ne Yapar                      | Ne Zaman Çalışır | Kurulum Zorluğu |
| -------------------------- | ----------------------------- | ---------------- | --------------- |
| **Semgrep**                | Kod güvenlik açıklarını bulur | Her PR           | ⭐ Kolay        |
| **OWASP Dependency Check** | CVE'li bağımlılıkları bulur   | Her PR           | ⭐⭐ Orta       |
| **Gitleaks**               | Hardcoded secret bulur        | Pre-commit + PR  | ⭐ Kolay        |

Bu üçünü kurduktan sonra isteğe bağlı eklentiler:

| Araç             | Ne Ekler                | Öneri                                |
| ---------------- | ----------------------- | ------------------------------------ |
| **MobSF**        | Binary APK analizi      | Release branch'te çalıştır           |
| **Android Lint** | Android'e özel kurallar | Zaten Gradle'da var, sadece aktif et |
| **Detekt**       | Kotlin statik analiz    | Semgrep'e ek olarak                  |

---

## 3. Semgrep

### Semgrep Nedir?

Semgrep, kod örüntülerini AST (Abstract Syntax Tree) seviyesinde arayan bir statik analiz aracıdır. Regex'ten çok daha akıllı çünkü kodun yapısını anlar.

```
# Regex ile: her "MD5" geçen yeri bulur (yorum satırları dahil, yanlış pozitif çok)
# Semgrep ile: gerçekten MessageDigest.getInstance("MD5") çağrısı olan satırları bulur
```

### Kurulum

```bash
# pip ile
pip install semgrep

# Homebrew (macOS)
brew install semgrep

# Versiyon kontrolü
semgrep --version
```

### Temel Kullanım

```bash
# Android için hazır kural seti ile tara (en hızlı başlangıç)
semgrep --config "p/android" --output results.json .

# Belirli bir dizini tara
semgrep --config "p/android" app/src/

# Sonuçları SARIF formatında çıkar (GitHub Code Scanning ile uyumlu)
semgrep --config "p/android" --sarif --output results.sarif .

# Verbose çıktı — hangi kuralın neden tetiklendiğini görmek için
semgrep --config "p/android" --verbose .
```

### Android İçin Kritik Kural Setleri

```bash
# Temel Android güvenlik kuralları
semgrep --config "p/android"

# Kotlin güvenlik kuralları
semgrep --config "p/kotlin"

# Gizli bilgi tespiti
semgrep --config "p/secrets"

# OWASP Top 10
semgrep --config "p/owasp-top-ten"

# Hepsini birden
semgrep --config "p/android" --config "p/secrets" --config "p/kotlin" .
```

### Özel Kural Yazma

Hazır kurallar yeterli gelmediğinde proje özelinde kurallar yazılabilir. Kural dosyaları YAML formatındadır.

```yaml
# rules/android-security.yml

rules:
  # Kural 1: Log'a şifre/token yazdırma tespiti
  - id: android-log-sensitive-data
    patterns:
      - pattern: Log.$LEVEL($TAG, $MSG, ...)
      - metavariable-regex:
          metavariable: $MSG
          regex: .*(password|token|secret|key|pin|cvv|ssn).*
    message: >
      Log çağrısında hassas veri olabilir: $MSG.
      Production'da bu bilgi logcat'e yazılır.
    languages: [kotlin, java]
    severity: ERROR
    metadata:
      cwe: CWE-532
      owasp: M9

  # Kural 2: Sabit IV ile şifreleme
  - id: android-static-iv
    pattern: |
      val $IV = ByteArray($SIZE) { 0 }
      ...
      $CIPHER.init(..., GCMParameterSpec(..., $IV))
    message: >
      Sabit (sıfır dolu) IV kullanılıyor. Her şifrelemede SecureRandom ile
      yeni IV üretilmeli, aksi halde semantic security sağlanamaz.
    languages: [kotlin]
    severity: ERROR
    metadata:
      cwe: CWE-330

  # Kural 3: WebView'de allowUniversalAccessFromFileURLs
  - id: android-webview-universal-access
    pattern: $WV.settings.allowUniversalAccessFromFileURLs = true
    message: >
      allowUniversalAccessFromFileURLs=true XSS saldırılarında
      cross-origin veri okumaya izin verir. false olmalı.
    languages: [kotlin, java]
    severity: ERROR
    metadata:
      cwe: CWE-79

  # Kural 4: Handler.proceed() ile SSL hatası geçme
  - id: android-ssl-error-proceed
    pattern: $HANDLER.proceed()
    message: >
      onReceivedSslError içinde handler.proceed() çağrısı SSL sertifika
      hatalarını sessizce geçer. MitM saldırısına kapı açar.
    languages: [kotlin, java]
    severity: ERROR
    metadata:
      cwe: CWE-295

  # Kural 5: Hardcoded API key
  - id: android-hardcoded-api-key
    patterns:
      - pattern: val $VAR = "$VALUE"
      - metavariable-regex:
          metavariable: $VAR
          regex: .*(api_key|apiKey|API_KEY|secret|SECRET|token|TOKEN).*
      - metavariable-regex:
          metavariable: $VALUE
          regex: ^[A-Za-z0-9_\-]{20,}$ # 20+ char string = muhtemelen key
    message: >
      Hardcoded credential olabilir: $VAR = "$VALUE".
      Anahtarlar BuildConfig veya server-side'da yönetilmeli.
    languages: [kotlin]
    severity: WARNING
    metadata:
      cwe: CWE-798

  # Kural 6: Random yerine SecureRandom kontrolü
  - id: android-insecure-random
    pattern: java.util.Random()
    message: >
      Kriptografik işlemlerde java.util.Random kullanılmamalı.
      java.security.SecureRandom kullan.
    languages: [kotlin, java]
    severity: WARNING
    metadata:
      cwe: CWE-330

  # Kural 7: ECB modu
  - id: android-aes-ecb-mode
    pattern: Cipher.getInstance("AES/ECB/...")
    message: >
      AES/ECB modu blok örüntülerini sızdırır.
      AES/GCM/NoPadding kullan.
    languages: [kotlin, java]
    severity: ERROR
    metadata:
      cwe: CWE-327
```

```bash
# Özel kuralları çalıştır
semgrep --config rules/android-security.yml app/src/
```

### Semgrep Baseline (Yanlış Pozitif Yönetimi)

Büyük projelerde ilk taramada çok sayıda bulgu çıkabilir. Baseline ile mevcut sorunları kaydet, sadece yeni girenleri engelle:

```bash
# Mevcut durumu baseline olarak kaydet
semgrep --config "p/android" --json > .semgrep-baseline.json

# Sonraki taramalarda yalnızca yeni bulguları göster
semgrep --config "p/android" --baseline-commit HEAD~1
```

### .semgrepignore

```
# .semgrepignore
build/
.gradle/
**/test/**          # Test kodlarında false positive çok olabilir
**/androidTest/**
vendor/
```

---

## 4. OWASP Dependency Check

### Neden Önemli?

Kullandığın her kütüphane bir saldırı yüzeyidir. Log4Shell (CVE-2021-44228) gibi kritik açıklar, güncellenmemiş bağımlılıklar aracılığıyla aylarca istismar edildi. OWASP Dependency Check, bağımlılıklarındaki bilinen CVE'leri NVD (National Vulnerability Database) üzerinden kontrol eder.

### Gradle Plugin Kurulumu

```kotlin
// build.gradle.kts (root)
plugins {
    id("org.owasp.dependencycheck") version "9.2.0"
}

dependencyCheck {
    // CVSS skoru 7.0 ve üzeri bulunan bağımlılık varsa build'i durdur
    // CVSS: 0-3.9 Low | 4.0-6.9 Medium | 7.0-8.9 High | 9.0-10.0 Critical
    failBuildOnCVSS = 7.0f

    // Rapor formatları
    formats = listOf("HTML", "JSON", "SARIF")

    // Rapor çıktı dizini
    outputDirectory = "${project.buildDir}/reports/dependency-check"

    // NVD API key (rate limit için — ücretsiz kayıt: nvd.nist.gov)
    nvd {
        apiKey = System.getenv("NVD_API_KEY") ?: ""
        delay = 4000  // API rate limit için bekleme (ms)
    }

    // Belirli bağımlılıkları atla (yanlış pozitif veya accept edilmiş risk)
    suppressionFile = "dependency-check-suppressions.xml"

    // Analiz edilecek konfigürasyonlar
    analyzers {
        assemblyEnabled = false     // .NET assembly analizi — Android'de gerekmez
        nuspecEnabled = false
        nugetconfEnabled = false
    }
}
```

### Çalıştırma

```bash
# Tam analiz
./gradlew dependencyCheckAnalyze

# Raporu aç (HTML)
open build/reports/dependency-check/dependency-check-report.html

# Sadece belirli modül
./gradlew :app:dependencyCheckAnalyze

# Cache'i sıfırla ve taze analiz
./gradlew dependencyCheckPurge dependencyCheckAnalyze
```

### Suppression Dosyası (Kabul Edilen Riskler)

Bazı CVE'ler uygulamanı etkilemiyor olabilir (örneğin server-side bir açık, Android'de ilgisiz). Bu durumda suppression dosyasıyla geçici olarak bastırılabilir.

```xml
<!-- dependency-check-suppressions.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<suppressions xmlns="https://jeremylong.github.io/DependencyCheck/dependency-suppression.1.3.xsd">

    <!-- Örnek: CVE sadece server-side etkiliyor, Android client'ta ilgisiz -->
    <suppress>
        <notes>
            CVE-2023-XXXX: Bu açık yalnızca server-side LDAP işlemlerini etkiliyor.
            Android client'ta bu kod path'i kullanılmıyor.
            Son inceleme: 2025-03-10, incelenen: Ismail
        </notes>
        <cve>CVE-2023-XXXX</cve>
    </suppress>

    <!-- Belirli bir bağımlılık için tüm düşük CVE'leri bastır -->
    <suppress>
        <notes>Retrofit test bağımlılığı, production build'e dahil değil</notes>
        <gav regex="true">com\.squareup\.retrofit2:.*:.*</gav>
        <cvssBelow>4</cvssBelow>  <!-- CVSS 4 altındakiler için -->
    </suppress>

</suppressions>
```

> ⚠️ **Önemli:** Suppression dosyasına `notes` + tarih + kişi bilgisi eklemek zorunlu alışkanlık olmalı. "Bu neden bastırıldı?" sorusuna 6 ay sonra da cevap verilebilmeli.

### Version Pinning & Checksum Doğrulama

```bash
# Gradle verification metadata oluştur
# Tüm bağımlılıkların SHA-256 hash'lerini kaydeder
./gradlew --write-verification-metadata sha256 help

# → gradle/verification-metadata.xml oluşur
# Bu dosyayı git'e commit et!
```

```xml
<!-- gradle/verification-metadata.xml (otomatik oluşturulur, örnek) -->
<verification-metadata>
    <configuration>
        <verify-metadata>true</verify-metadata>
        <verify-signatures>false</verify-signatures>
    </configuration>
    <components>
        <component group="com.squareup.okhttp3" name="okhttp" version="4.12.0">
            <artifact name="okhttp-4.12.0.jar">
                <sha256 value="abc123..." origin="Generated by Gradle"/>
            </artifact>
        </component>
    </components>
</verification-metadata>
```

Artık bağımlılık hash'i değişirse (supply chain saldırısı dahil) Gradle build fail eder.

---

## 5. Gitleaks

### Neden Önemli?

API key, AWS secret, private key, JWT secret gibi bilgilerin commit'e girmesi son derece yaygın bir hatadır. GitHub'ın kendi istatistiklerine göre her gün milyonlarca secret commit'e giriyor. Gitleaks bu bilgileri hem commit öncesi (pre-commit hook) hem CI'da yakalar.

### Kurulum

```bash
# macOS
brew install gitleaks

# Linux
curl -sL https://github.com/gitleaks/gitleaks/releases/latest/download/gitleaks_linux_x64.tar.gz \
  | tar xz -C /usr/local/bin

# pip (cross-platform)
pip install gitleaks

# Versiyon
gitleaks version
```

### Temel Kullanım

```bash
# Tüm git history'yi tara (ilk kurulumda)
gitleaks detect --source . --report-format json --report-path gitleaks-report.json

# Sadece staged değişiklikleri tara (pre-commit için)
gitleaks protect --staged

# Belirli commit aralığını tara
gitleaks detect --log-opts="HEAD~10..HEAD"

# Verbose (hangi kuralın tetiklendiğini görmek için)
gitleaks detect --verbose
```

### Pre-commit Hook Kurulumu

```bash
# pre-commit framework ile (önerilen yöntem)
pip install pre-commit

# .pre-commit-config.yaml dosyası oluştur
cat > .pre-commit-config.yaml << 'EOF'
repos:
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.18.4
    hooks:
      - id: gitleaks
EOF

# Hook'u kur
pre-commit install

# Manuel test
pre-commit run gitleaks --all-files
```

Artık her `git commit` öncesinde Gitleaks otomatik çalışır. Secret tespit ederse commit engellenir.

### .gitleaksignore (İstisna Yönetimi)

```bash
# Belirli bir bulguyu bastır (false positive)
# Önce hash'i al
gitleaks detect --report-format json | jq '.[].Fingerprint'

# .gitleaksignore dosyasına ekle
echo "abc123fingerprint" >> .gitleaksignore
```

---

## 6. GitHub Actions — Tam Pipeline

Bu YAML dosyası yukarıdaki üç aracı + MobSF'i entegre eden eksiksiz bir security pipeline'dır.

```yaml
# .github/workflows/android-security.yml
name: Android Security Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

# Paralel job'lar için izinler
permissions:
  contents: read
  security-events: write # GitHub Code Scanning için
  pull-requests: write # PR'a yorum yazabilmek için

jobs:
  # ─────────────────────────────────────────────
  # JOB 1: Secret Scanning (en hızlı, önce çalışsın)
  # ─────────────────────────────────────────────
  secret-scan:
    name: 🔑 Secret Scanning (Gitleaks)
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0 # Tüm git history için

      - name: Run Gitleaks
        uses: gitleaks/gitleaks-action@v2
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        with:
          args: detect --source . --log-opts="HEAD~1..HEAD" --report-format sarif --report-path gitleaks.sarif

      - name: Upload SARIF (GitHub Security Tab)
        uses: github/codeql-action/upload-sarif@v3
        if: always() # Fail etse bile yükle
        with:
          sarif_file: gitleaks.sarif
          category: gitleaks

  # ─────────────────────────────────────────────
  # JOB 2: SAST (Semgrep)
  # ─────────────────────────────────────────────
  sast:
    name: 🔍 SAST (Semgrep)
    runs-on: ubuntu-latest
    container:
      image: semgrep/semgrep
    steps:
      - uses: actions/checkout@v4

      - name: Run Semgrep (Android + Kotlin + Secrets)
        run: |
          semgrep \
            --config "p/android" \
            --config "p/kotlin" \
            --config "p/secrets" \
            --config "rules/" \
            --sarif \
            --output semgrep.sarif \
            --error \
            app/src/
        env:
          SEMGREP_APP_TOKEN: ${{ secrets.SEMGREP_APP_TOKEN }} # Opsiyonel: cloud dashboard için

      - name: Upload SARIF
        uses: github/codeql-action/upload-sarif@v3
        if: always()
        with:
          sarif_file: semgrep.sarif
          category: semgrep

  # ─────────────────────────────────────────────
  # JOB 3: Dependency Check
  # ─────────────────────────────────────────────
  dependency-check:
    name: 📦 Dependency Vulnerability Scan
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: 17

      - name: Setup Gradle
        uses: gradle/actions/setup-gradle@v3

      - name: Cache NVD Database
        uses: actions/cache@v4
        with:
          path: ~/.gradle/dependency-check-data
          key: nvd-${{ runner.os }}-${{ hashFiles('**/*.gradle.kts') }}
          restore-keys: nvd-${{ runner.os }}-

      - name: Run OWASP Dependency Check
        run: ./gradlew dependencyCheckAnalyze
        env:
          NVD_API_KEY: ${{ secrets.NVD_API_KEY }}

      - name: Upload HTML Report
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: dependency-check-report
          path: build/reports/dependency-check/
          retention-days: 30

      - name: Upload SARIF
        uses: github/codeql-action/upload-sarif@v3
        if: always()
        with:
          sarif_file: build/reports/dependency-check/dependency-check-report.sarif
          category: dependency-check

  # ─────────────────────────────────────────────
  # JOB 4: Android Lint Security Rules
  # ─────────────────────────────────────────────
  android-lint:
    name: 🤖 Android Lint
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: 17

      - name: Setup Gradle
        uses: gradle/actions/setup-gradle@v3

      - name: Run Lint
        run: ./gradlew lintRelease

      - name: Upload Lint Report
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: lint-report
          path: app/build/reports/lint-results-release.html

  # ─────────────────────────────────────────────
  # JOB 5: MobSF APK Scan (sadece main branch)
  # ─────────────────────────────────────────────
  mobsf-scan:
    name: 📱 MobSF APK Scan
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main' # Sadece main'de çalış
    needs: [secret-scan, sast, dependency-check] # Önceki job'lar geçince
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: 17

      - name: Setup Gradle
        uses: gradle/actions/setup-gradle@v3

      - name: Build Release APK
        run: ./gradlew assembleRelease
        env:
          KEYSTORE_PATH: ${{ secrets.KEYSTORE_PATH }}
          KEY_ALIAS: ${{ secrets.KEY_ALIAS }}
          KEY_PASSWORD: ${{ secrets.KEY_PASSWORD }}
          STORE_PASSWORD: ${{ secrets.STORE_PASSWORD }}

      - name: Start MobSF
        run: |
          docker run -d \
            --name mobsf \
            -p 8000:8000 \
            -e MOBSF_API_KEY_FILE=/tmp/api_key \
            opensecurity/mobile-security-framework-mobsf:latest
          echo "${{ secrets.MOBSF_API_KEY }}" > /tmp/api_key
          sleep 45   # MobSF başlaması için bekle

      - name: Upload APK & Scan
        id: mobsf_scan
        run: |
          APK_PATH=$(find app/build/outputs/apk/release -name "*.apk" | head -1)

          # Upload
          UPLOAD=$(curl -s \
            -F "file=@${APK_PATH}" \
            http://localhost:8000/api/v1/upload \
            -H "Authorization: ${{ secrets.MOBSF_API_KEY }}")
          echo "Upload response: $UPLOAD"

          HASH=$(echo $UPLOAD | jq -r '.hash')
          echo "hash=$HASH" >> $GITHUB_OUTPUT

          # Scan başlat
          curl -s \
            http://localhost:8000/api/v1/scan \
            -d "hash=$HASH" \
            -H "Authorization: ${{ secrets.MOBSF_API_KEY }}"

          # Skor al
          SCORE=$(curl -s \
            "http://localhost:8000/api/v1/scorecard?hash=$HASH" \
            -H "Authorization: ${{ secrets.MOBSF_API_KEY }}")
          echo "Security Score: $SCORE"

          # Raporu indir
          curl -s \
            "http://localhost:8000/api/v1/download_pdf?hash=$HASH" \
            -H "Authorization: ${{ secrets.MOBSF_API_KEY }}" \
            -o mobsf-report.pdf

      - name: Upload MobSF Report
        uses: actions/upload-artifact@v4
        with:
          name: mobsf-report
          path: mobsf-report.pdf

  # ─────────────────────────────────────────────
  # JOB 6: APK İmza Doğrulama
  # ─────────────────────────────────────────────
  apk-signing-check:
    name: ✍️ APK Signing Verification
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: 17

      - name: Build Release APK
        run: ./gradlew assembleRelease

      - name: Verify APK Signature
        run: |
          APK=$(find app/build/outputs/apk/release -name "*.apk" | head -1)

          # İmza şemalarını kontrol et
          apksigner verify --verbose --print-certs "$APK"

          # v2 ve v3 imzaların mevcut olduğunu doğrula
          apksigner verify --verbose "$APK" 2>&1 | grep -E "Verified using v[23]"

          echo "✅ APK signing verification passed"
```

### Secrets Yönetimi (GitHub)

```bash
# GitHub CLI ile secret ekle
gh secret set NVD_API_KEY --body "your-nvd-api-key"
gh secret set MOBSF_API_KEY --body "your-mobsf-key"
gh secret set SEMGREP_APP_TOKEN --body "your-semgrep-token"

# Keystore için (base64 encode edip kaydet)
base64 -i keystore.jks | gh secret set KEYSTORE_BASE64
```

```yaml
# build.gradle.kts — keystore'u CI'da kullan
android {
    signingConfigs {
        release {
            storeFile = file(System.getenv("KEYSTORE_PATH") ?: "keystore.jks")
            storePassword = System.getenv("STORE_PASSWORD") ?: ""
            keyAlias = System.getenv("KEY_ALIAS") ?: ""
            keyPassword = System.getenv("KEY_PASSWORD") ?: ""
        }
    }
}
```

---

## 7. MobSF APK Tarama

MobSF, APK'yı binary seviyede analiz eden bir platformdur. CI'a entegre edilebilir, ayrıca lokal web arayüzü de vardır.

### Lokal Çalıştırma (Manuel İnceleme)

```bash
# Docker ile başlat
docker run -it --rm \
  -p 8000:8000 \
  -v "$(pwd)/uploads:/home/mobsf/.MobSF/uploads" \
  opensecurity/mobile-security-framework-mobsf:latest

# Tarayıcıdan aç
open http://localhost:8000
# API Key: değiştirmek için Settings > API Key
```

### MobSF'in Kontrol Ettiği Başlıca Şeyler

```
AndroidManifest.xml Analizi:
  ✓ debuggable=true tespiti
  ✓ allowBackup=true tespiti
  ✓ Exported component'lar
  ✓ Permission listesi

Kaynak Kod Analizi (jadx ile):
  ✓ Hardcoded secret/key
  ✓ SQLite raw query (SQL injection)
  ✓ WebView güvensiz konfigürasyonu
  ✓ Kriptografi hataları

Binary Analizi:
  ✓ .so dosyalarında string tarama
  ✓ Güvensiz fonksiyon çağrıları (strcpy, gets)
  ✓ Stack canary / PIE / NX bit kontrolü
  ✓ Obfuscation tespiti

Ağ Analizi:
  ✓ URL'lerin HTTP/HTTPS dağılımı
  ✓ IP adresi tespiti
  ✓ Firebase URL'leri

Sertifika Analizi:
  ✓ İmza algoritması (SHA1 kullanımı uyarısı)
  ✓ Sertifika geçerlilik tarihi
  ✓ Self-signed sertifika tespiti
```

### MobSF Skoru

MobSF analiz sonunda 0-100 arası güvenlik skoru verir. Genel referans:

| Skor   | Değerlendirme                              |
| ------ | ------------------------------------------ |
| 80-100 | İyi — temel güvenlik kuralları uygulanmış  |
| 60-79  | Orta — dikkat edilmesi gereken açıklar var |
| 40-59  | Zayıf — kritik sorunlar mevcut             |
| 0-39   | Kritik — acil müdahale gerekli             |

---

## 8. Gradle Güvenlik Konfigürasyonu

### Android Lint Security Kuralları

```kotlin
// app/build.gradle.kts
android {
    lint {
        // Security ile ilgili lint kurallarını error seviyesine çek
        error.addAll(listOf(
            "HardcodedDebugMode",        // debuggable=true tespiti
            "AllowBackup",               // allowBackup=true uyarısı
            "ExportedPreferenceActivity",// Exported PreferenceActivity
            "SetJavaScriptEnabled",      // WebView JS enabled uyarısı
            "TrustAllX509TrustManager",  // Tüm sertifikalara güvenme
            "SSLCertificateSocketFactoryCreateSocket", // Güvensiz socket
            "PackageManagerGetSignatures" // Eski imza API kullanımı
        ))

        // Build'i durdur
        abortOnError = true

        // Baseline — mevcut sorunları kaydet, yeni girenleri engelle
        baseline = file("lint-baseline.xml")

        // HTML + XML rapor
        htmlReport = true
        xmlReport = true
        htmlOutput = file("${project.buildDir}/reports/lint/lint-report.html")
        xmlOutput = file("${project.buildDir}/reports/lint/lint-report.xml")
    }
}
```

### Lint Baseline Oluşturma

```bash
# İlk kurulumda: mevcut uyarıları baseline'a al
./gradlew lintRelease -Dlint.baselines.continue=true

# → lint-baseline.xml oluşur, git'e commit et
# Yeni kod'a sadece yeni uyarılar eklenir
```

### ProGuard / R8 — Minimum Güvenlik Konfigürasyonu

> **Hatırlatma:** R8 bir güvenlik aracı değil, optimizasyon aracıdır. Yan etkisi olarak isim obfuscation yapar. Log satırlarını, çoğu mantığı hâlâ görebilirsin. Gerçek binary koruma için DexGuard gerekir.

```kotlin
// build.gradle.kts
buildTypes {
    release {
        isMinifyEnabled = true          // R8 aktif
        isShrinkResources = true        // Kullanılmayan kaynak sil
        proguardFiles(
            getDefaultProguardFile("proguard-android-optimize.txt"),
            "proguard-rules.pro"
        )
        // Logging'i production'da kapat
        buildConfigField("boolean", "ENABLE_LOGGING", "false")
    }
    debug {
        isMinifyEnabled = false
        buildConfigField("boolean", "ENABLE_LOGGING", "true")
    }
}
```

```pro
# proguard-rules.pro

# Crash raporları için kaynak satır bilgisi koru
-keepattributes SourceFile, LineNumberTable
-renamesourcefileattribute SourceFile

# Retrofit/Moshi model sınıfları — reflection için koru
-keep class com.example.data.model.** { *; }

# Enum'lar
-keepclassmembers enum * {
    public static **[] values();
    public static ** valueOf(java.lang.String);
}

# JNI native metodlar
-keepclasseswithmembernames class * {
    native <methods>;
}

# Serializable sınıflar
-keepclassmembers class * implements java.io.Serializable {
    static final long serialVersionUID;
    private static final java.io.ObjectStreamField[] serialPersistentFields;
    private void writeObject(java.io.ObjectOutputStream);
    private void readObject(java.io.ObjectInputStream);
    java.lang.Object writeReplace();
    java.lang.Object readResolve();
}

# Custom view'lar (XML'den inflate için)
-keep public class * extends android.view.View {
    public <init>(android.content.Context);
    public <init>(android.content.Context, android.util.AttributeSet);
}

# WebView JavaScript interface (reflection ile çağrılır)
-keepclassmembers class * {
    @android.webkit.JavascriptInterface <methods>;
}
```

---

## 9. PR Güvenlik Review Checklist

Her PR'da kontrol edilmesi gereken noktalar. Bunu GitHub PR template'ine ekle.

```markdown
<!-- .github/pull_request_template.md -->

## Güvenlik Kontrol Listesi

> Bu checklist yalnızca güvenlikle ilgili değişiklikler için zorunludur.
> Eğer PR'ın güvenlikle ilgisi yoksa "N/A" yaz.

### 🌐 Ağ & İletişim

- [ ] HTTP URL yok (cleartext traffic yok)
- [ ] Yeni API endpoint için SSL pinning eklendi / güncellendi
- [ ] API anahtarı / secret hardcoded değil
- [ ] Network Security Config güncellendi (gerekliyse)

### 💾 Veri Depolama

- [ ] Yeni SharedPreferences → EncryptedSharedPreferences kullanıldı
- [ ] Hassas veri disk yerine bellekte tutuluyor (mümkünse)
- [ ] Backup'tan hariç tutulması gereken veri `android:allowBackup` ile kontrol altında
- [ ] Dosyalar internal storage'a yazılıyor (external değil)

### 🔐 Kriptografi

- [ ] SecureRandom kullanılıyor (java.util.Random değil)
- [ ] Zayıf algoritma yok: MD5, SHA1, DES, AES/ECB
- [ ] IV/Nonce her şifrelemede SecureRandom ile yeni üretiliyor
- [ ] Anahtar Keystore'da saklanıyor

### 📡 IPC & Bileşen Güvenliği

- [ ] Yeni Activity/Service/BroadcastReceiver → `exported=false` (veya gerekçeli `true`)
- [ ] Deep link parametreleri whitelist/validation'dan geçiyor
- [ ] Gelen Intent'ler doğrulanıyor

### 🌍 WebView

- [ ] `allowFileAccess=false`
- [ ] `allowUniversalAccessFromFileURLs=false`
- [ ] `addJavascriptInterface` kullanımı gerekçeli ve input validate ediliyor
- [ ] URL whitelist kontrolü var

### 📝 Log & Debug

- [ ] Log satırlarında şifre/token/PII yok
- [ ] `BuildConfig.DEBUG` kontrolü olmayan debug kodu yok
- [ ] `android:debuggable=true` production manifest'te yok

### 📦 Bağımlılıklar

- [ ] Yeni dependency için CVE kontrolü yapıldı
- [ ] Dependency versiyonu sabitlendi (range değil exact version)
- [ ] SDK izin gereksinimleri incelendi

### 🔑 Kimlik Doğrulama

- [ ] Biyometrik kullanımı `BIOMETRIC_STRONG` sınıfıyla
- [ ] Token'lar session sonunda temizleniyor
- [ ] Re-authentication kritik işlemlerde zorunlu

---

**Reviewer notu:** Yukarıdaki maddelerden birini gözden kaçırdıysan PR'ı onaylama.
```

---

## 10. Güvenlik Test Piramidi

```
                    ╔══════════════╗
                    ║   Pentest    ║  ← Nadir, pahalı, elle yapılır
                    ║  (Red Team)  ║    Frida / Drozer / Burp ile
                    ╚══════════════╝
                 ╔══════════════════╗
                 ║  Security E2E   ║  ← Release öncesi; MobSF, DAST
                 ║   Automation    ║
                 ╚══════════════════╝
            ╔══════════════════════════╗
            ║   Instrumented Tests    ║  ← Her CI build; permission,
            ║ (Security Integration)  ║    biometric, IPC kontrolleri
            ╚══════════════════════════╝
       ╔══════════════════════════════════╗
       ║       SAST + Secret Scan        ║  ← Her PR; Semgrep, Gitleaks
       ║    + Dependency Check           ║    Dependency Check
       ╚══════════════════════════════════╝
  ╔══════════════════════════════════════════╗
  ║          Unit Tests (Security)          ║  ← Her commit; crypto
  ║   Input validation, Crypto functions    ║    fonksiyonları, validation
  ╚══════════════════════════════════════════╝
```

### Seviye 1 — Unit Tests (Security)

```kotlin
// Kriptografi fonksiyonları için unit test örneği
class CryptoTest {

    @Test
    fun `encrypt should produce different ciphertext for same plaintext`() {
        val key = generateAesKey()
        val plaintext = "test-data".toByteArray()

        val (iv1, cipher1) = encrypt(plaintext, key)
        val (iv2, cipher2) = encrypt(plaintext, key)

        // Aynı plaintext → farklı IV → farklı ciphertext (semantic security)
        assertThat(iv1).isNotEqualTo(iv2)
        assertThat(cipher1).isNotEqualTo(cipher2)
    }

    @Test
    fun `decrypt should restore original plaintext`() {
        val key = generateAesKey()
        val original = "sensitive-data".toByteArray()
        val (iv, ciphertext) = encrypt(original, key)
        val decrypted = decrypt(iv, ciphertext, key)
        assertThat(decrypted).isEqualTo(original)
    }

    @Test
    fun `tampered ciphertext should throw exception`() {
        val key = generateAesKey()
        val (iv, ciphertext) = encrypt("data".toByteArray(), key)
        val tampered = ciphertext.copyOf().also { it[0] = it[0].xor(0xFF.toByte()) }

        // GCM integrity check — tampered data throws
        assertThrows<AEADBadTagException> {
            decrypt(iv, tampered, key)
        }
    }

    @Test
    fun `password hash should use PBKDF2 with sufficient iterations`() {
        val password = "user-password".toCharArray()
        val salt = ByteArray(32).also { SecureRandom().nextBytes(it) }
        val hash = hashPassword(password, salt)

        assertThat(hash.size).isEqualTo(32)   // 256-bit output
        assertThat(hash).isNotEqualTo(ByteArray(32))  // Sıfır değil
    }
}
```

### Seviye 2 — Instrumented Security Tests

```kotlin
// AndroidTest — Runtime permission ve bileşen güvenliği
@RunWith(AndroidJUnit4::class)
class SecurityInstrumentedTest {

    @get:Rule
    val activityRule = ActivityScenarioRule(MainActivity::class.java)

    @Test
    fun exported_admin_activity_should_not_be_accessible_without_permission() {
        val context = ApplicationProvider.getApplicationContext<Context>()

        val intent = Intent().apply {
            setClassName(context.packageName, "com.example.AdminActivity")
        }

        // SecurityException bekleniyor (exported=false ise)
        try {
            context.startActivity(intent)
            fail("AdminActivity açılmamalıydı")
        } catch (e: ActivityNotFoundException) {
            // Beklenen davranış — exported=false
        } catch (e: SecurityException) {
            // Beklenen davranış — permission gerekiyor
        }
    }

    @Test
    fun content_provider_should_reject_sql_injection() {
        val context = ApplicationProvider.getApplicationContext<Context>()
        val uri = Uri.parse("content://${context.packageName}.provider/users")

        // SQL injection denemesi
        val maliciousProjection = arrayOf("* FROM users--")
        val cursor = context.contentResolver.query(uri, maliciousProjection, null, null, null)

        // Ya null dönmeli ya da injection içermeyen güvenli sonuç
        cursor?.use {
            assertThat(it.columnNames).doesNotContain("* FROM users--")
        }
    }
}
```

---

## 11. Ölçüm & Raporlama

### Security Metrics — Takip Edilmesi Gereken

```
DAST Coverage:    Toplam endpoint sayısı / test edilen endpoint sayısı
Vuln. Lead Time:  Açığın tespit edildiği tarih - commit tarihi
MTTR:             Açığın kapatılma süresi (Mean Time To Remediate)
Dep. Freshness:   Outdated dependency sayısı / toplam dependency sayısı
Secret Leak Rate: Engellenen secret commit sayısı / toplam commit
```

### GitHub Security Dashboard

GitHub repo → Security tab → Code scanning alerts altında tüm SARIF raporları görünür. Semgrep, Gitleaks, Dependency Check bulguları buraya akıyor.

```bash
# GitHub CLI ile mevcut security alert'leri listele
gh api /repos/{owner}/{repo}/code-scanning/alerts \
  --jq '.[] | {rule: .rule.id, severity: .rule.severity, file: .most_recent_instance.location.path}'
```

### Haftalık Security Raporu Otomasyonu

```yaml
# .github/workflows/weekly-security-report.yml
name: Weekly Security Report

on:
  schedule:
    - cron: "0 9 * * 1" # Her Pazartesi 09:00

jobs:
  report:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Full Dependency Scan
        run: ./gradlew dependencyCheckAnalyze
        env:
          NVD_API_KEY: ${{ secrets.NVD_API_KEY }}

      - name: Full Semgrep Scan (tüm kod tabanı)
        run: semgrep --config "p/android" --config "p/secrets" --json > semgrep-full.json
        continue-on-error: true

      - name: Send Slack Notification
        uses: slackapi/slack-github-action@v1
        with:
          payload: |
            {
              "text": "📊 Haftalık Android Security Raporu",
              "attachments": [{
                "color": "warning",
                "text": "Dependency Check ve Semgrep raporları hazır. Artifacts'ten indir."
              }]
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}

      - uses: actions/upload-artifact@v4
        with:
          name: weekly-security-report-${{ github.run_number }}
          path: |
            build/reports/dependency-check/
            semgrep-full.json
          retention-days: 90
```

---

## Hızlı Başlangıç — Adım Adım

Mevcut bir projeye sıfırdan security CI/CD eklemek için minimum yol:

```bash
# 1. Gitleaks pre-commit hook (5 dakika)
pip install pre-commit
cat > .pre-commit-config.yaml << 'EOF'
repos:
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.18.4
    hooks:
      - id: gitleaks
EOF
pre-commit install

# 2. Semgrep GitHub Action (10 dakika)
# → .github/workflows/semgrep.yml oluştur (yukarıdaki Job 2'yi al)

# 3. OWASP Dependency Check Gradle plugin (15 dakika)
# → build.gradle.kts'e ekle (Bölüm 4)
# → ./gradlew dependencyCheckAnalyze ile test et
# → NVD_API_KEY al: https://nvd.nist.gov/developers/request-an-api-key

# 4. PR template (5 dakika)
# → .github/pull_request_template.md oluştur (Bölüm 9'daki şablonu kullan)

# Toplam süre: ~35 dakika
# Kapsam: her PR'da secret scan + SAST + dependency CVE kontrolü
```

---

_Son güncelleme: Mart 2025 | GitHub Actions, Semgrep 1.x, OWASP Dependency Check 9.x baz alınmıştır_
