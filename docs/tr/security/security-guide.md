# Android Security — Kapsamlı Rehber

> **Kapsam:** Tehdit modelleme, statik/dinamik analiz, ağ güvenliği, kriptografi, uygulama sertleştirme, platform güvenliği, saldırı teknikleri ve savunma stratejileri. Güncel (2024–2025) teknikler ve araçlar.

> **⚡ Hızlı Başlangıç:** Production-ready uygulamalar için temel güvenlik gereksinimleri ve kod örnekleri için [Core Security Practices](./core-security-practices.md) dokümanına bakın.

---

## Öncelik & Kullanım Haritası

Bu döküman **detaylı güvenlik analizi, pentest teknikleri ve ileri seviye konular** içerir. Öncelik sırasına göre düzenlenmiştir:

| Etiket           | Anlam                                                                          |
| ---------------- | ------------------------------------------------------------------------------ |
| 🔴 **CORE**      | Production gereksinimleri → [Core Security Practices](./core-security-practices.md) |
| 🟠 **IMPORTANT** | Önemli ama daha az sık; derinlemesine bilinmesi beklenir                       |
| 🟡 **USEFUL**    | Bilmek seni farklılaştırır; pentest/audit senaryolarında gerekli               |
| ⚪ **REFERENCE** | Başvuru kaynağı; her şeyi ezberlemek gerekmez, nerede bulacağını bil           |

> **Not:** 🔴 CORE işaretli bölümler artık ayrı bir dokümanda. Production uygulamaları için kod örnekleri ve best practice'ler için [Core Security Practices](./core-security-practices.md) dokümanını inceleyin.

---

## İçindekiler

### 🔴 CORE — Her Zaman, Her Projede

1. [Android Güvenlik Mimarisi](#1-android-güvenlik-mimarisi) — Temeller, sandbox, permission modeli
2. [Ağ Güvenliği & Man-in-the-Middle (MitM)](#5-ağ-güvenliği--man-in-the-middle-mitm) — SSL Pinning, MitM kurulumu, bypass teknikleri
3. [IPC Güvenliği (Binder, Intent, Content Provider)](#9-ipc-güvenliği) — Exported components, intent injection, WebView
4. [Kriptografi](#6-kriptografi) — Doğru AES/GCM, Android Keystore, yaygın hatalar
5. [Data Storage Güvenliği](#8-data-storage-güvenliği) — EncryptedSharedPreferences, SQLCipher, log sızıntısı
6. [CI/CD & Güvenli Geliştirme Pratiği (SSDLC)](#14-güvenli-geliştirme-pratiği-ssdlc) — Semgrep, Dependency Check, kod review checklist

### 🟠 IMPORTANT — Senior Seviyede Beklenen

7. [Dinamik Analiz / DAST — Frida & Objection](#4-dinamik-analiz-dast) — Runtime hook, bypass teknikleri
8. [Root Detection & Integrity Checks](#11-root-detection--integrity-checks) — Play Integrity API, emülatör tespiti
9. [Kimlik Doğrulama & Yetkilendirme](#7-kimlik-doğrulama--yetkilendirme) — BiometricPrompt, OAuth/PKCE, token yönetimi
10. [Supply Chain & Dependency Security](#12-supply-chain--dependency-security) — CVE tarama, SDK güvenliği

### 🟡 USEFUL — Seni Farklılaştıran

11. [Statik Analiz / SAST](#3-statik-analiz-sast) — APK anatomisi, jadx/apktool, MobSF
12. [Penetration Testing Metodolojisi](#15-penetration-testing-metodolojisi) — 5 fazlı pentest, Drozer, bulgu raporlama
13. [Kod Gizleme & Tersine Mühendislik Karşıtı](#10-kod-gizleme--tersine-mühendislik-karşıtı) — R8 limitleri, DexGuard, anti-tamper

### ⚪ REFERENCE — Gerektiğinde Bak

14. [Tehdit Modelleme](#2-tehdit-modelleme) — STRIDE, DREAD, OWASP Top 10
15. [Platform Güvenlik Özellikleri](#13-platform-güvenlik-özellikleri) — Android 12–15 değişiklikleri, pKVM
16. [Araç Seti Referansı](#16-araç-seti-referansı) — Tüm araçlar, kurulum scripti

---

## 🔴 CORE 1 — Android Güvenlik Mimarisi

### 1.1 Katmanlı Savunma Modeli

Android güvenliği "defense-in-depth" prensibine dayanır. Her katman bağımsız olarak güvenliği sağlamaya çalışır.

```
┌─────────────────────────────────┐
│         Uygulama Katmanı        │  ← Geliştiricinin kontrolü
├─────────────────────────────────┤
│     Android Framework (Java)    │  ← Permission, Intent filtresi
├─────────────────────────────────┤
│    Native Libraries / NDK       │  ← C/C++, BIONIC libc
├─────────────────────────────────┤
│     Android Runtime (ART)       │  ← Bytecode doğrulama, DEX
├─────────────────────────────────┤
│        HAL (Vendor)             │  ← SELinux ile izole
├─────────────────────────────────┤
│       Linux Kernel              │  ← UID izolasyonu, Seccomp
├─────────────────────────────────┤
│     Donanım (TEE / StrongBox)   │  ← Keystore, Secure Enclave
└─────────────────────────────────┘
```

### 1.2 Sandbox Mekanizması

Her APK farklı bir Linux UID ile çalışır. Bu temel izolasyon mekanizmasıdır:

- UID `u0_a<N>` — Her uygulama benzersiz bir UID alır.
- Uygulama dosyaları `/data/data/<package>/` altında yalnızca o UID'e aittir.
- `sharedUserId` (deprecated Android 10+) ile iki uygulama aynı UID'i paylaşabilirdi.

**SELinux (Mandatory Access Control):** Android 5.0'dan itibaren enforcing mode. Süreçler etiketlenir (`domain`), dosyalar etiketlenir (`type`). `avc: denied` logları güvenlik ihlallerini gösterir.

```bash
# SELinux context görme
adb shell ls -Z /data/data/com.example.app/
# Process context
adb shell ps -Z | grep com.example
```

### 1.3 Permission Modeli

| Kategori   | Örnek                     | Davranış                              |
| ---------- | ------------------------- | ------------------------------------- |
| Normal     | `INTERNET`, `VIBRATE`     | Install-time, otomatik grant          |
| Dangerous  | `CAMERA`, `READ_CONTACTS` | Runtime prompt (Android 6+)           |
| Signature  | Özel sistem perms         | Sadece aynı sertifikayla sign edilmiş |
| Privileged | `INSTALL_PACKAGES`        | System/priv-app + allowlist           |
| AppOps     | `MANAGE_APPOPS`           | Framework seviyesi ince kontrol       |

**Android 12+ — Permission Timeline:**

- Tek seferlik izinler (one-time permissions) mikrofonun arka planda kullanımını kısıtladı.
- `BLUETOOTH_SCAN`, `BLUETOOTH_CONNECT` ayrıldı.
- Yaklaşık konum (`COARSE_LOCATION`) tek başına seçilebilir.

**Android 14+ — Photo Picker & Health Connect:**

- Medya erişimi için `READ_MEDIA_IMAGES` granüler hale geldi.
- Photo Picker sistemi, `READ_EXTERNAL_STORAGE` ihtiyacını ortadan kaldırdı.

---

## ⚪ REFERENCE 2 — Tehdit Modelleme

### 2.1 STRIDE Modeli (Android Bağlamında)

| Tehdit                     | Android Örneği                         | Karşı Önlem                               |
| -------------------------- | -------------------------------------- | ----------------------------------------- |
| **S**poofing               | Intent hijacking, sahte broadcast      | `exported=false`, explicit intent         |
| **T**ampering              | APK yeniden paketleme, kod enjeksiyonu | Signature check, Play Integrity API       |
| **R**epudiation            | Log silme, işlem inkarı                | Server-side audit log, attestation        |
| **I**nformation Disclosure | Log sızıntısı, backup                  | `android:allowBackup=false`, ProGuard     |
| **D**enial of Service      | Broadcast flood, WakeLock abuse        | Rate limiting, BroadcastReceiver koruması |
| **E**levation of Privilege | Root exploit, dirty COW                | Kernel patch seviyesi kontrolü            |

### 2.2 DREAD Skoring

Her tehdit için: **D**amage + **R**eproducibility + **E**xploitability + **A**ffected users + **D**iscoverability → 1–10 arası puanlama.

### 2.3 Attack Surface Analizi

```
┌─────── Saldırı Yüzeyleri ─────────────────────────────────────────┐
│                                                                     │
│  Dış Yüzeyler:                                                      │
│  ├── Network (HTTP/S endpoints, WebSocket, gRPC)                   │
│  ├── IPC (exported Activity/Service/BroadcastReceiver/Provider)    │
│  ├── Deep Links / App Links                                         │
│  ├── Clipboard                                                      │
│  └── NFC / Bluetooth / WiFi-Direct                                 │
│                                                                     │
│  İç Yüzeyler:                                                       │
│  ├── Shared Storage (/sdcard, MediaStore)                          │
│  ├── WebView (JavaScript bridge)                                   │
│  ├── Native (JNI) arayüzleri                                       │
│  └── Third-party SDK'lar                                           │
└─────────────────────────────────────────────────────────────────────┘
```

### 2.4 OWASP Mobile Top 10 (2024)

1. M1 — Improper Credential Usage
2. M2 — Inadequate Supply Chain Security
3. M3 — Insecure Authentication/Authorization
4. M4 — Insufficient Input/Output Validation
5. M5 — Insecure Communication
6. M6 — Inadequate Privacy Controls
7. M7 — Insufficient Binary Protections
8. M8 — Security Misconfiguration
9. M9 — Insecure Data Storage
10. M10 — Insufficient Cryptography

---

## 🟡 USEFUL 3 — Statik Analiz (SAST)

### 3.1 APK Anatomisi

```
app.apk (ZIP)
├── AndroidManifest.xml      ← Binary XML → AAPT2 ile decode
├── classes.dex              ← Dalvik bytecode
├── classes2.dex             ← MultiDex
├── resources.arsc           ← Binary resource table
├── res/                     ← XML + drawable
├── assets/                  ← Raw assets
├── lib/
│   ├── arm64-v8a/           ← .so dosyaları
│   └── x86_64/
└── META-INF/
    ├── MANIFEST.MF
    ├── CERT.SF
    └── CERT.RSA/DSA/EC      ← İmza bloğu (v1)
```

### 3.2 APK İmza Şemaları

| Şema     | Android Sürümü | Konum                   | Not                                          |
| -------- | -------------- | ----------------------- | -------------------------------------------- |
| v1 (JAR) | Tümü           | META-INF/               | ZIP dosyasını değiştirme açığına karşı zayıf |
| v2       | Android 7.0+   | APK imza bloğu          | Tüm APK hash'lenir                           |
| v3       | Android 9+     | APK imza bloğu          | Key rotation destekler                       |
| v3.1     | Android 13+    | APK imza bloğu          | Rotated key'i yeni cihazlara zorlar          |
| v4       | Android 11+    | `.apk.idsig` ayrı dosya | ADB incremental install için                 |

**İmza doğrulama (komut satırı):**

```bash
# v1-v3 bilgilerini göster
apksigner verify --verbose --print-certs app.apk

# apktool ile decode
apktool d app.apk -o app_decoded/

# jadx ile Java'ya çevir
jadx -d jadx_output/ app.apk

# dex2jar
d2j-dex2jar.sh classes.dex -o classes.jar
```

### 3.3 AndroidManifest Güvenlik Kontrolleri

```xml
<!-- TEHLİKELİ: Backup açık, debug açık, tüm trafiğe izin var -->
<application
    android:allowBackup="true"          <!-- Kötü: ADB backup ile veri çekilebilir -->
    android:debuggable="true"           <!-- Kötü: Prod'da kesinlikle false olmalı -->
    android:usesCleartextTraffic="true" <!-- Kötü: HTTP'e izin veriyor -->
    android:networkSecurityConfig="@xml/network_security_config">

<!-- TEHLİKELİ: exported Activity, permission yok -->
<activity android:name=".AdminActivity"
    android:exported="true"/>           <!-- Herhangi bir uygulama açabilir! -->

<!-- TEHLİKELİ: exported BroadcastReceiver -->
<receiver android:name=".PaymentReceiver"
    android:exported="true"/>           <!-- Intent injection açığı -->
```

**Güvenli Manifest Checklist:**

```xml
<application
    android:allowBackup="false"
    android:debuggable="false"
    android:usesCleartextTraffic="false"
    android:networkSecurityConfig="@xml/network_security_config">

<activity android:name=".InternalActivity"
    android:exported="false"/>          <!-- Sadece aynı uygulama açabilir -->

<activity android:name=".DeepLinkActivity"
    android:exported="true">
    <intent-filter android:autoVerify="true">  <!-- App Links doğrulaması -->
        <action android:name="android.intent.action.VIEW"/>
        <category android:name="android.intent.category.BROWSABLE"/>
        <data android:scheme="https" android:host="example.com"/>
    </intent-filter>
</activity>
```

### 3.4 Otomatik SAST Araçları

**MobSF (Mobile Security Framework):**

```bash
# Docker ile çalıştır
docker run -it --rm -p 8000:8000 opensecurity/mobile-security-framework-mobsf

# API ile tarama
curl -F "file=@app.apk" http://localhost:8000/api/v1/upload \
     -H "Authorization: <apikey>"
```

**semgrep (Custom Rules):**

```bash
# Android güvenlik kuralları
semgrep --config "p/android" --output results.json .

# Özel kural örneği: Log'a hassas veri yazdırma tespiti
# rule: android-log-sensitive
# pattern: Log.$METHOD(..., $DATA, ...) where $DATA matches /password|token|secret/i
```

**Jadx + Manuel İnceleme için kritik aranacak noktalar:**

```
✗ Log.d/v/i ile şifre, token, PII yazdırma
✗ BuildConfig.DEBUG kontrolü olmayan debug kodları
✗ Hardcoded API key / secret
✗ SQLite raw query (SQL injection riski)
✗ WebView.loadUrl ile user input kullanımı
✗ Reflection ile dinamik class yükleme
✗ Runtime.exec() / ProcessBuilder ile shell komutu
✗ ObjectInputStream (Java deserialization)
✗ Random yerine SecureRandom kullanımı
✗ MD5 / SHA1 ile şifre hash'leme
```

---

## 🟠 IMPORTANT 4 — Dinamik Analiz (DAST)

### 4.1 Test Ortamı Kurulumu

**Emülatör (Tercihli — root erişimi için):**

```bash
# Google Play olmayan (AOSP) imaj — root için ideal
emulator -avd Pixel_7_API_33 -writable-system -no-snapshot

# Root erişimi
adb root
adb remount
```

**Fiziksel Cihaz (Gerçekçi test için):**

```bash
# USB debug aç, geliştirici modunu etkinleştir
adb devices
adb shell su   # Root gerektirir (Magisk vb.)
```

### 4.2 Frida — Dinamik Enstrümantasyon

Frida, çalışan bir uygulamaya JavaScript enjekte ederek metodları hook'lamayı sağlar.

**Kurulum:**

```bash
pip install frida-tools
# Cihaza frida-server yükle (ABI'a uygun)
adb push frida-server-16.x.x-android-arm64 /data/local/tmp/frida-server
adb shell chmod 755 /data/local/tmp/frida-server
adb shell /data/local/tmp/frida-server &
```

**Temel Hook Örnekleri:**

```javascript
// 1. SSL Pinning Bypass (OkHttp)
Java.perform(() => {
  const OkHttpClient = Java.use("okhttp3.OkHttpClient$Builder");
  OkHttpClient.certificatePinner.overload(
    "okhttp3.CertificatePinner",
  ).implementation = function (pinner) {
    console.log("[*] CertificatePinner bypass");
    return this;
  };
});

// 2. Root Detection Bypass (örnek: RootBeer)
Java.perform(() => {
  const RootBeer = Java.use("com.scottyab.rootbeer.RootBeer");
  RootBeer.isRooted.implementation = function () {
    console.log("[*] isRooted → false döndürüldü");
    return false;
  };
});

// 3. Şifreleme anahtarı yakalama (javax.crypto)
Java.perform(() => {
  const SecretKeySpec = Java.use("javax.crypto.spec.SecretKeySpec");
  SecretKeySpec.$init.overload("[B", "java.lang.String").implementation =
    function (keyBytes, algorithm) {
      console.log(`[*] Key (${algorithm}): ${bytesToHex(keyBytes)}`);
      return this.$init(keyBytes, algorithm);
    };
});

// 4. Biyometrik bypass
Java.perform(() => {
  const BiometricPrompt = Java.use(
    "android.hardware.biometrics.BiometricPrompt",
  );
  BiometricPrompt.authenticate.overload(
    "android.hardware.biometrics.BiometricPrompt$CryptoObject",
    "android.os.CancellationSignal",
    "java.util.concurrent.Executor",
    "android.hardware.biometrics.BiometricPrompt$AuthenticationCallback",
  ).implementation = function (crypto, cancel, executor, callback) {
    console.log("[*] BiometricPrompt.authenticate intercepted");
    // AuthenticationResult oluştur ve başarılı say
    const AuthResult = Java.use(
      "android.hardware.biometrics.BiometricPrompt$AuthenticationResult",
    );
    callback.onAuthenticationSucceeded(AuthResult.$new(crypto));
  };
});

// 5. Log tüm HTTP trafiği (OkHttp interceptor)
Java.perform(() => {
  const Buffer = Java.use("okio.Buffer");
  const RequestBody = Java.use("okhttp3.RequestBody");
  // Response body'yi intercept et
  const ResponseBody = Java.use("okhttp3.ResponseBody");
  ResponseBody.string.implementation = function () {
    const result = this.string();
    console.log("[HTTP Response]:", result);
    return result;
  };
});
```

**Frida ile Process Spawn:**

```bash
frida -U -f com.example.app --no-pause -l hook.js

# Attach (çalışan process)
frida -U com.example.app -l hook.js

# Trace
frida-trace -U -i "SSL_*" com.example.app
```

### 4.3 Objection — Frida Tabanlı Otomatik Test

```bash
# Kurulum
pip install objection

# Launch
objection -g com.example.app explore

# SSL pinning bypass (tüm yaygın kütüphaneler için)
android sslpinning disable

# Root detection bypass
android root disable

# Bellekteki string'leri dump et
memory search "password" --string

# SharedPreferences oku
android shared_preferences get

# Keystore listele
android keystore list

# Activity başlat
android intent launch_activity com.example.app.AdminActivity

# Dosya sistemi
file download /data/data/com.example.app/databases/app.db local_app.db
```

### 4.4 Drozer — IPC Saldırı Yüzeyi Testi

```bash
# Agent APK cihaza yükle, forward
adb forward tcp:31415 tcp:31415
drozer console connect

# Attack surface
run app.package.attacksurface com.example.app

# Exported Activity'leri listele
run app.activity.info -a com.example.app

# Activity'yi zorla başlat
run app.activity.start --component com.example.app com.example.app.AdminActivity

# Content Provider sorgula
run app.provider.query content://com.example.app.provider/users

# SQL Injection testi
run app.provider.query content://com.example.app.provider/users \
    --projection "* FROM users--"

# Intent fuzzing
run app.broadcast.send --component com.example.app com.example.app.PaymentReceiver \
    --extra string action "REFUND" --extra int amount 99999
```

### 4.5 Runtime Memory Analizi

```bash
# PID bul
adb shell ps | grep com.example

# /proc/<pid>/maps — bellek haritası
adb shell cat /proc/<pid>/maps

# Heap dump (Android Studio veya)
adb shell am dumpheap com.example.app /data/local/tmp/heap.hprof
adb pull /data/local/tmp/heap.hprof

# Memory içinde arama (r2frida)
# r2 frida://com.example.app
# /iz password   ← string search
```

---

## 🔴 CORE 2 — Ağ Güvenliği & Man-in-the-Middle (MitM)

### 5.1 MitM Test Ortamı Kurma

**Yöntem A: HTTP Proxy (Burp Suite / mitmproxy)**

```
[Android Cihaz] ──WiFi──> [Proxy (PC)] ──> [İnternet]
                  ^
            Proxy IP:Port ayarlı
            Test CA sertifikası yüklü
```

**Adım 1: Proxy CA sertifikasını sisteme yükleme (Android 7+)**

Android 7.0'dan itibaren kullanıcı CA'ları yalnızca `user` trust store'a eklenir ve uygulamalar varsayılan olarak buna güvenmez.

```bash
# Burp CA'yı DER formatından PEM'e çevir
openssl x509 -inform DER -in cacert.der -out cacert.pem

# Hash değeri al (Android sistem CA formatı: <hash>.0)
HASH=$(openssl x509 -subject_hash_old -in cacert.pem | head -1)
cp cacert.pem ${HASH}.0

# Sisteme yükle (root + writable-system gerekli)
adb root
adb remount
adb push ${HASH}.0 /system/etc/security/cacerts/
adb shell chmod 644 /system/etc/security/cacerts/${HASH}.0
adb reboot
```

**Adım 2: Network Security Config bypass (debug build):**

```xml
<!-- res/xml/network_security_config.xml -->
<network-security-config>
    <debug-overrides>
        <trust-anchors>
            <certificates src="user"/>      <!-- Debug'da user CA'ya güven -->
            <certificates src="system"/>
        </trust-anchors>
    </debug-overrides>
    <base-config cleartextTrafficPermitted="false">
        <trust-anchors>
            <certificates src="system"/>    <!-- Prod'da sadece system CA -->
        </trust-anchors>
    </base-config>
</network-security-config>
```

**Yöntem B: iptables ile transparan proxy (root gerekir)**

```bash
# Tüm HTTP/S trafiğini proxy'e yönlendir
adb shell iptables -t nat -A OUTPUT -p tcp --dport 443 \
    -j DNAT --to-destination 192.168.1.100:8080
```

**mitmproxy ile trafik yakalama:**

```bash
# Transparan mod
mitmproxy --mode transparent --showhost

# Intercept + otomatik kaydet
mitmdump -w traffic.dump

# Python script ile filtrele
mitmdump -s filter_script.py
```

### 5.2 SSL/TLS Güvenliği

**TLS Konfigürasyon Sorunları:**

```kotlin
// ❌ Tehlikeli: Tüm sertifikalara güven (Development hatası Prod'a taşınan)
val trustAll = object : X509TrustManager {
    override fun checkClientTrusted(chain: Array<X509Certificate>, authType: String) {}
    override fun checkServerTrusted(chain: Array<X509Certificate>, authType: String) {}
    override fun getAcceptedIssuers(): Array<X509Certificate> = emptyArray()
}

// ❌ Tehlikeli: Hostname doğrulamasını devre dışı bırakma
HttpsURLConnection.setDefaultHostnameVerifier { _, _ -> true }

// ✅ Doğru: OkHttp ile standart TLS
val client = OkHttpClient.Builder()
    .build() // Varsayılan TrustManager yeterli

// ✅ Doğru: Güçlü TLS versiyonu zorla
val spec = ConnectionSpec.Builder(ConnectionSpec.MODERN_TLS)
    .tlsVersions(TlsVersion.TLS_1_3, TlsVersion.TLS_1_2)
    .cipherSuites(
        CipherSuite.TLS_AES_128_GCM_SHA256,
        CipherSuite.TLS_AES_256_GCM_SHA384,
        CipherSuite.TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256
    )
    .build()
```

### 5.3 SSL Pinning

SSL Pinning; sunucunun kimliğini sertifika zinciri yerine belirli bir sertifika veya public key ile doğrular.

**Yöntem 1: OkHttp CertificatePinner (Hash bazlı)**

```kotlin
// Public key hash'ini önceden al:
// openssl s_client -connect api.example.com:443 | openssl x509 -pubkey -noout |
// openssl rsa -pubin -outform der | openssl dgst -sha256 -binary | base64

val client = OkHttpClient.Builder()
    .certificatePinner(
        CertificatePinner.Builder()
            // Birincil sertifika
            .add("api.example.com", "sha256/AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=")
            // Backup pin (sertifika rotasyonu için kritik!)
            .add("api.example.com", "sha256/BBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBB=")
            .build()
    )
    .build()
```

**Yöntem 2: Network Security Config (XML)**

```xml
<network-security-config>
    <domain-config>
        <domain includeSubdomains="true">api.example.com</domain>
        <pin-set expiration="2026-01-01">
            <pin digest="SHA-256">AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=</pin>
            <pin digest="SHA-256">BBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBB=</pin>
        </pin-set>
    </domain-config>
</network-security-config>
```

**Yöntem 3: TrustKit / CustomTrustManager (Fine-grained kontrol)**

```kotlin
// OkHttp interceptor tabanlı custom pin kontrolü
class PinningInterceptor(
    private val expectedPins: Set<String>
) : Interceptor {
    override fun intercept(chain: Interceptor.Chain): Response {
        val request = chain.request()
        val response = chain.proceed(request)

        // Sertifikayı al ve hash'i karşılaştır
        val certs = (response.handshake?.peerCertificates ?: emptyList())
        val valid = certs.any { cert ->
            val pin = cert.publicKey.encoded.sha256().base64()
            pin in expectedPins
        }

        if (!valid) throw SSLPeerUnverifiedException("Pin mismatch!")
        return response
    }
}
```

**SSL Pinning Bypass Teknikleri (Pentest):**

```javascript
// 1. Frida ile OkHttp CertificatePinner bypass
Java.perform(() => {
  const CertificatePinner = Java.use("okhttp3.CertificatePinner");
  CertificatePinner.check.overload(
    "java.lang.String",
    "java.util.List",
  ).implementation = function (host, certs) {
    console.log(`[SSL Pin Bypass] host: ${host}`);
    // Exception fırlatma → bypass
  };
  CertificatePinner.check.overload(
    "java.lang.String",
    "[Ljava.security.cert.Certificate;",
  ).implementation = function (host, certs) {
    console.log(`[SSL Pin Bypass v2] host: ${host}`);
  };
});

// 2. TrustManager bypass
Java.perform(() => {
  const X509TrustManager = Java.use("javax.net.ssl.X509TrustManager");
  const SSLContext = Java.use("javax.net.ssl.SSLContext");

  const TrustManagerImpl = Java.use(
    "com.android.org.conscrypt.TrustManagerImpl",
  );
  TrustManagerImpl.verifyChain.implementation = function (
    untrustedChain,
    trustAnchorChain,
    host,
    clientAuth,
    ocspData,
    tlsSctData,
  ) {
    return untrustedChain;
  };
});

// 3. Objection (tek komut)
// android sslpinning disable

// 4. apk-mitm (APK yeniden paketleme ile)
// apk-mitm app.apk → patched app'e proxy sertifikası trust ekler
```

### 5.4 Certificate Transparency (CT)

Modern Android güvenliğinde SSL pinning'in yanında CT log kontrolü:

```kotlin
// Conscrypt ile CT doğrulama
val provider = Conscrypt.newProvider()
Security.insertProviderAt(provider, 1)
// Android 9+ varsayılan olarak CT gerektiriyor (publiclyTrusted CA'lar için)
```

### 5.5 Network Security Config — Üretim Yapılandırması

```xml
<?xml version="1.0" encoding="utf-8"?>
<network-security-config>
    <!-- Cleartext yasak (varsayılan Android 9+) -->
    <base-config cleartextTrafficPermitted="false">
        <trust-anchors>
            <certificates src="system"/>
            <!-- Özel CA varsa -->
            <!-- <certificates src="@raw/custom_ca"/> -->
        </trust-anchors>
    </base-config>

    <!-- İstisna: Yalnızca local debug sunucu -->
    <domain-config cleartextTrafficPermitted="true">
        <domain includeSubdomains="false">10.0.2.2</domain>
    </domain-config>

    <!-- Belirli domain için pin -->
    <domain-config>
        <domain includeSubdomains="true">api.production.com</domain>
        <pin-set expiration="2026-06-01">
            <pin digest="SHA-256">...</pin>
            <pin digest="SHA-256">...</pin>  <!-- Backup pin zorunlu! -->
        </pin-set>
    </domain-config>
</network-security-config>
```

### 5.6 API Güvenliği

```kotlin
// ✅ Doğru: API Key header'da (body değil) ve obfuscated
// ✅ Doğru: Request imzalama (HMAC-SHA256)
fun signRequest(url: String, body: String, secret: ByteArray): String {
    val timestamp = System.currentTimeMillis() / 1000
    val payload = "$timestamp\n$url\n${body.sha256()}"
    val mac = Mac.getInstance("HmacSHA256")
    mac.init(SecretKeySpec(secret, "HmacSHA256"))
    return mac.doFinal(payload.toByteArray()).base64()
}

// ✅ Doğru: Certificate Transparency loglara doğrulama
// ✅ Doğru: HSTS (HTTP Strict Transport Security) — server tarafı
// ✅ Doğru: HTTP/2 ile TLS 1.3 zorunlu
```

---

## 🔴 CORE 4 — Kriptografi

### 6.1 Kriptografi Hataları — Top Sorunlar

```kotlin
// ❌ ECB modu (blok paterni sızdırır)
Cipher.getInstance("AES/ECB/PKCS5Padding")

// ❌ Sabit IV (nonce reuse — semantic security yok)
val iv = ByteArray(12) { 0 }
Cipher.getInstance("AES/GCM/NoPadding").init(Cipher.ENCRYPT_MODE, key, GCMParameterSpec(128, iv))

// ❌ MD5 / SHA-1 ile şifre hash (collision-prone)
MessageDigest.getInstance("MD5").digest(password.toByteArray())

// ❌ Zayıf PRNG
Random().nextInt()  // Kriptografik değil!

// ❌ Hardcoded key
val key = SecretKeySpec("1234567890abcdef".toByteArray(), "AES")
```

### 6.2 Doğru Kriptografi Uygulaması

```kotlin
// ✅ AES-256-GCM (authenticated encryption)
fun encrypt(plaintext: ByteArray, key: SecretKey): Pair<ByteArray, ByteArray> {
    val cipher = Cipher.getInstance("AES/GCM/NoPadding")
    val iv = ByteArray(12).also { SecureRandom().nextBytes(it) }  // Her seferinde yeni IV
    cipher.init(Cipher.ENCRYPT_MODE, key, GCMParameterSpec(128, iv))
    return iv to cipher.doFinal(plaintext)
}

fun decrypt(iv: ByteArray, ciphertext: ByteArray, key: SecretKey): ByteArray {
    val cipher = Cipher.getInstance("AES/GCM/NoPadding")
    cipher.init(Cipher.DECRYPT_MODE, key, GCMParameterSpec(128, iv))
    return cipher.doFinal(ciphertext)  // GCM otomatik integrity kontrolü yapar
}

// ✅ Şifre hash — Argon2 (Android 10+) veya PBKDF2
fun hashPassword(password: CharArray, salt: ByteArray): ByteArray {
    val factory = SecretKeyFactory.getInstance("PBKDF2WithHmacSHA256")
    val spec = PBEKeySpec(
        password,
        salt,
        600_000,  // OWASP 2024 minimum iterasyon
        256
    )
    return factory.generateSecret(spec).encoded
}

// ✅ Güvenli rastgele salt üretimi
val salt = ByteArray(32).also { SecureRandom().nextBytes(it) }

// ✅ ECDH anahtar değişimi
val keyPairGen = KeyPairGenerator.getInstance("EC")
keyPairGen.initialize(ECGenParameterSpec("secp256r1"))
val keyPair = keyPairGen.generateKeyPair()

// ✅ ECDSA imzalama
val signature = Signature.getInstance("SHA256withECDSA")
signature.initSign(keyPair.private)
signature.update(data)
val sig = signature.sign()
```

### 6.3 Android Keystore

Keystore, kriptografik anahtarları TEE (Trusted Execution Environment) veya StrongBox (HSM) içinde saklar. Anahtar, keystore dışına asla çıkmaz.

```kotlin
// ✅ Keystore'da anahtar üretme
val keyGenerator = KeyGenerator.getInstance(KeyProperties.KEY_ALGORITHM_AES, "AndroidKeyStore")
keyGenerator.init(
    KeyGenParameterSpec.Builder(
        "my_secret_key",
        KeyProperties.PURPOSE_ENCRYPT or KeyProperties.PURPOSE_DECRYPT
    )
    .setKeySize(256)
    .setBlockModes(KeyProperties.BLOCK_MODE_GCM)
    .setEncryptionPaddings(KeyProperties.ENCRYPTION_PADDING_NONE)
    .setUserAuthenticationRequired(true)              // Biyometrik / PIN gerektirir
    .setUserAuthenticationParameters(
        0,                                            // 0 = her kullanımda auth gerekir
        KeyProperties.AUTH_BIOMETRIC_STRONG or KeyProperties.AUTH_DEVICE_CREDENTIAL
    )
    .setInvalidatedByBiometricEnrollment(true)        // Yeni parmak izi eklenince geçersiz
    .setUnlockedDeviceRequired(true)                  // Ekran kilitli iken kullanılamaz
    .setIsStrongBoxBacked(true)                       // StrongBox (HSM) kullan (varsa)
    .build()
)
keyGenerator.generateKey()

// ✅ Keystore'dan anahtar okuma ve kullanma
val keyStore = KeyStore.getInstance("AndroidKeyStore").apply { load(null) }
val key = keyStore.getKey("my_secret_key", null) as SecretKey

// ✅ RSA asimetrik anahtar (Keystore'da private key asla çıkmaz)
val keyPairGenerator = KeyPairGenerator.getInstance(
    KeyProperties.KEY_ALGORITHM_RSA, "AndroidKeyStore"
)
keyPairGenerator.initialize(
    KeyGenParameterSpec.Builder("rsa_key", KeyProperties.PURPOSE_SIGN)
        .setKeySize(2048)
        .setDigests(KeyProperties.DIGEST_SHA256)
        .setSignaturePaddings(KeyProperties.SIGNATURE_PADDING_RSA_PSS)
        .setUserAuthenticationRequired(true)
        .build()
)
```

### 6.4 EncryptedSharedPreferences & EncryptedFile

```kotlin
// Jetpack Security Crypto kütüphanesi
implementation("androidx.security:security-crypto:1.1.0-alpha06")

// ✅ EncryptedSharedPreferences
val masterKey = MasterKey.Builder(context)
    .setKeyScheme(MasterKey.KeyScheme.AES256_GCM)
    .setUserAuthenticationRequired(true, 30)  // 30 saniyede bir re-auth
    .build()

val encryptedPrefs = EncryptedSharedPreferences.create(
    context,
    "secure_prefs",
    masterKey,
    EncryptedSharedPreferences.PrefKeyEncryptionScheme.AES256_SIV,
    EncryptedSharedPreferences.PrefValueEncryptionScheme.AES256_GCM
)

// ✅ EncryptedFile
val encryptedFile = EncryptedFile.Builder(
    context,
    File(context.filesDir, "secret.txt"),
    masterKey,
    EncryptedFile.FileEncryptionScheme.AES256_GCM_HKDF_4KB
).build()
```

---

## 🟠 IMPORTANT 3 — Kimlik Doğrulama & Yetkilendirme

### 7.1 Biyometrik Kimlik Doğrulama

```kotlin
// ✅ BiometricPrompt — Crypto bağlı (güçlü)
class BiometricAuthManager(private val activity: FragmentActivity) {

    fun authenticate(
        keyName: String,
        onSuccess: (Cipher) -> Unit,
        onError: (String) -> Unit
    ) {
        val keyStore = KeyStore.getInstance("AndroidKeyStore").apply { load(null) }
        val cipher = Cipher.getInstance("AES/GCM/NoPadding")
        cipher.init(Cipher.ENCRYPT_MODE, keyStore.getKey(keyName, null) as SecretKey)

        val cryptoObject = BiometricPrompt.CryptoObject(cipher)

        val executor = ContextCompat.getMainExecutor(activity)
        val prompt = BiometricPrompt(activity, executor, object : BiometricPrompt.AuthenticationCallback() {
            override fun onAuthenticationSucceeded(result: BiometricPrompt.AuthenticationResult) {
                // Cipher artık unlock, gerçek şifreleme yapılabilir
                result.cryptoObject?.cipher?.let { onSuccess(it) }
            }
            override fun onAuthenticationError(errorCode: Int, errString: CharSequence) {
                onError(errString.toString())
            }
        })

        val promptInfo = BiometricPrompt.PromptInfo.Builder()
            .setTitle("Kimliğini Doğrula")
            .setSubtitle("Devam etmek için biyometrik kullan")
            .setAllowedAuthenticators(
                BiometricManager.Authenticators.BIOMETRIC_STRONG  // Sadece strong (Class 3)
            )
            .build()

        prompt.authenticate(promptInfo, cryptoObject)
    }
}
```

### 7.2 Token Yönetimi

```kotlin
// ✅ Access token: Bellekte tut (kısa ömürlü, asla diske yazma)
// ✅ Refresh token: EncryptedSharedPreferences (Keystore backed)
// ✅ Token expiry kontrolü (JWT decode — verify etmeden)

fun isTokenExpired(jwt: String): Boolean {
    val payload = jwt.split(".")[1]
    val decoded = Base64.decode(payload, Base64.URL_SAFE or Base64.NO_PADDING)
    val json = JSONObject(String(decoded))
    val exp = json.getLong("exp")
    return System.currentTimeMillis() / 1000 >= exp - 30  // 30s buffer
}

// ✅ Logout: Token'ı hem bellekten hem depodan sil
fun logout() {
    tokenStore.clear()
    encryptedPrefs.edit().remove("refresh_token").apply()
    // Server-side da token'ı revoke et!
    apiClient.revokeToken()
}
```

### 7.3 OAuth 2.0 / PKCE (Mobil için zorunlu)

```kotlin
// ✅ AppAuth kütüphanesi ile OAuth 2.0 + PKCE
implementation("net.openid:appauth:0.11.1")

// Code Verifier üret (PKCE)
val codeVerifier = CodeVerifierUtil.generateRandomCodeVerifier()
val codeChallenge = CodeVerifierUtil.deriveCodeVerifierChallenge(codeVerifier)

val authRequest = AuthorizationRequest.Builder(
    serviceConfig,
    clientId,
    ResponseTypeValues.CODE,
    redirectUri
)
    .setCodeVerifier(codeVerifier, codeChallenge, "S256")
    .setScope("openid profile")
    .build()

// Custom Tab ile (WebView değil! — phishing koruması)
val customTabIntent = authService.getCustomTabsIntentBuilder().build()
authService.performAuthorizationRequest(authRequest, pendingIntent, customTabIntent)
```

**PKCE Neden Önemli?** Authorization code interception saldırısına karşı korur. Code verifier yalnızca client'ta biliniyor, ağda yakalanmış code tek başına kullanılamaz.

---

## 🔴 CORE 5 — Data Storage Güvenliği

### 8.1 Depolama Seçenekleri ve Güvenlik Profilleri

| Depolama                     | Erişim       | Şifreleme                    | Uygun Kullanım          |
| ---------------------------- | ------------ | ---------------------------- | ----------------------- |
| `SharedPreferences`          | Uygulama     | ❌ (Düz metin)               | Şifresiz tercihler      |
| `EncryptedSharedPreferences` | Uygulama     | ✅ AES-256                   | Token, küçük gizli veri |
| `Internal Storage`           | Uygulama     | ❌ (FDE/FBE ile OS şifreler) | Uygulama dosyaları      |
| `EncryptedFile`              | Uygulama     | ✅ AES-256-GCM               | Hassas dosyalar         |
| `Room + SQLCipher`           | Uygulama     | ✅ AES-256                   | Şifreli yerel DB        |
| `External Storage`           | Herkese açık | ❌                           | Paylaşılabilir medya    |
| `Keystore`                   | TEE          | ✅ Donanım                   | Kriptografik anahtarlar |

### 8.2 SQLite Güvenliği

```kotlin
// ❌ SQL Injection açığı
db.rawQuery("SELECT * FROM users WHERE id = $userId", null)

// ✅ Parametreli sorgu
db.rawQuery("SELECT * FROM users WHERE id = ?", arrayOf(userId.toString()))

// ✅ Room ile (otomatik parametreli)
@Query("SELECT * FROM users WHERE id = :userId")
fun getUserById(userId: Int): User

// ✅ SQLCipher ile şifreli veritabanı
implementation("net.zetetic:android-database-sqlcipher:4.5.4")

val passphrase = SQLiteDatabase.getBytes(masterPassword)
val factory = SupportFactory(passphrase)
val db = Room.databaseBuilder(context, AppDatabase::class.java, "app.db")
    .openHelperFactory(factory)
    .build()
```

### 8.3 Log Güvenliği

```kotlin
// ❌ Production'da log sızıntısı
Log.d("Payment", "Card number: $cardNumber, CVV: $cvv")

// ✅ BuildConfig kontrolü
if (BuildConfig.DEBUG) {
    Log.d(TAG, "Debug only: $safeInfo")
}

// ✅ Timber — production'da log'ları filtrele
class ReleaseTree : Timber.Tree() {
    override fun log(priority: Int, tag: String?, message: String, t: Throwable?) {
        if (priority == Log.ERROR || priority == Log.WARN) {
            // Yalnızca error/warn → Crashlytics'e gönder (PII olmadan!)
            FirebaseCrashlytics.getInstance().log(message)
        }
        // Debug/Verbose → hiçbir şey yapma
    }
}
```

### 8.4 Clipboard Güvenliği (Android 13+)

```kotlin
// Android 13+ — clipboard okuma uyarısı sistem tarafından gösterilir
// Hassas alanlar için clipboard'a erişimi engelle
binding.passwordField.apply {
    setOnLongClickListener { true }  // Context menu'yü engelle
    // Jetpack Compose
}

// Compose
TextField(
    value = password,
    onValueChange = { password = it },
    visualTransformation = PasswordVisualTransformation(),
    keyboardOptions = KeyboardOptions(keyboardType = KeyboardType.Password)
    // NOT: Compose TextField kopyalamayı otomatik engeller (PasswordVisualTransformation ile)
)

// Otomatik clipboard temizleme (Android 13+)
// Sistem otomatik temizliyor, ek işlem gerekmez
// Android 12 ve altı için:
val clipboard = getSystemService(ClipboardManager::class.java)
Handler(Looper.getMainLooper()).postDelayed({
    if (clipboard.hasPrimaryClip()) {
        clipboard.setPrimaryClip(ClipData.newPlainText("", ""))
    }
}, 60_000)  // 1 dk sonra temizle
```

---

## 🔴 CORE 3 — IPC Güvenliği (Binder, Intent, Content Provider, WebView)

### 9.1 Intent Güvenliği

```kotlin
// ❌ Implicit intent ile hassas data gönderme
val intent = Intent("com.example.ACTION_PAYMENT")
intent.putExtra("amount", 100.0)
sendBroadcast(intent)  // Herhangi bir uygulama yakalayabilir!

// ✅ Explicit intent (tek alıcı)
val intent = Intent(context, PaymentService::class.java)
intent.putExtra("amount", 100.0)
startService(intent)

// ✅ Permission ile protected broadcast
sendBroadcast(intent, "com.example.permission.PAYMENT_RECEIVER")

// ✅ LocalBroadcastManager (aynı process, IPC yok)
LocalBroadcastManager.getInstance(context).sendBroadcast(intent)

// Intent injection tespiti — gelen intent'i doğrula
override fun onNewIntent(intent: Intent?) {
    super.onNewIntent(intent)
    // Kaynak doğrulama
    val action = intent?.action ?: return
    val allowedActions = setOf("com.example.SAFE_ACTION")
    if (action !in allowedActions) return

    // Extra'ları doğrula
    val amount = intent.getDoubleExtra("amount", -1.0)
    require(amount > 0 && amount < 10_000) { "Invalid amount" }
}
```

### 9.2 Content Provider Güvenliği

```kotlin
// AndroidManifest — sadece gerekli erişimi ver
<provider
    android:name=".SecureProvider"
    android:exported="false"                          <!-- Dışa kapalı -->
    android:grantUriPermissions="false"               <!-- URI permission grant yok -->
    android:readPermission="com.example.READ_DATA"    <!-- Okuma için custom permission -->
    android:writePermission="com.example.WRITE_DATA"/>

// ✅ Projection saldırısına karşı whitelist
override fun query(uri: Uri, projection: Array<String>?, ...): Cursor? {
    val allowedColumns = setOf("id", "name", "email")
    val safeProjection = projection?.filter { it in allowedColumns }?.toTypedArray()
        ?: arrayOf("id", "name", "email")

    val safeSelection = DatabaseUtils.sqlEscapeString(selection ?: "")
    return db.query(TABLE, safeProjection, safeSelection, selectionArgs, ...)
}

// ✅ FileProvider ile güvenli dosya paylaşımı (doğrudan URI yerine)
val uri = FileProvider.getUriForFile(
    context,
    "${context.packageName}.fileprovider",
    file
)
intent.addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION)  // Geçici izin ver
```

### 9.3 Binder IPC & AIDL Güvenliği

```kotlin
// ✅ Service'te çağıran caller doğrulama
class SecureService : Service() {
    override fun onBind(intent: Intent): IBinder = binder

    private val binder = object : ISecureService.Stub() {
        override fun sensitiveOperation(data: String): String {
            // Caller'ın iznini kontrol et
            if (checkCallingPermission("com.example.SECURE_PERM")
                    != PackageManager.PERMISSION_GRANTED) {
                throw SecurityException("Caller lacks required permission")
            }

            // UID doğrulama (belirli bir uygulamaya kısıtla)
            val callingUid = Binder.getCallingUid()
            val callingPackage = packageManager.getNameForUid(callingUid)
            if (callingPackage != "com.trusted.app") {
                throw SecurityException("Unauthorized caller: $callingPackage")
            }

            return processData(data)
        }
    }
}
```

### 9.4 Deep Link & App Link Güvenliği

```kotlin
// ❌ Tehlikeli: URL scheme (phishing'e açık)
<intent-filter>
    <data android:scheme="myapp"/>  <!-- myapp:// herhangi bir uygulama kopyalayabilir -->
</intent-filter>

// ✅ App Links (HTTPS + .well-known/assetlinks.json doğrulaması)
<intent-filter android:autoVerify="true">
    <action android:name="android.intent.action.VIEW"/>
    <category android:name="android.intent.category.DEFAULT"/>
    <category android:name="android.intent.category.BROWSABLE"/>
    <data android:scheme="https" android:host="app.example.com"/>
</intent-filter>

// .well-known/assetlinks.json (sunucuda barındır)
// [{ "relation": ["delegate_permission/common.handle_all_urls"],
//    "target": { "namespace": "android_app",
//                "package_name": "com.example.app",
//                "sha256_cert_fingerprints": ["AB:CD:EF:..."] } }]

// ✅ Deep link alındığında gelen URL'yi doğrula
override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    handleIntent(intent)
}

private fun handleIntent(intent: Intent) {
    val data = intent.data ?: return

    // Host whitelist
    val allowedHosts = setOf("app.example.com")
    if (data.host !in allowedHosts) return

    // Path whitelist
    val allowedPaths = Regex("^/(product|category)/[a-zA-Z0-9-]+$")
    if (!allowedPaths.matches(data.path ?: "")) return

    // Parametreleri sanitize et
    val id = data.getQueryParameter("id")?.toLongOrNull() ?: return
    navigateToProduct(id)
}
```

### 9.5 WebView Güvenliği

```kotlin
// ❌ Tehlikeli WebView konfigürasyonu
webView.settings.apply {
    javaScriptEnabled = true
    allowFileAccess = true              // file:// URL'lere erişim
    allowUniversalAccessFromFileURLs = true  // Cross-origin erişim
    allowFileAccessFromFileURLs = true
}
webView.addJavascriptInterface(dangerousObject, "NativeBridge")  // XSS → RCE

// ✅ Güvenli WebView konfigürasyonu
webView.settings.apply {
    javaScriptEnabled = true            // Gerekiyorsa
    allowFileAccess = false
    allowContentAccess = false
    allowUniversalAccessFromFileURLs = false
    allowFileAccessFromFileURLs = false
    setSupportZoom(false)
    databaseEnabled = false
    domStorageEnabled = false           // Gerekmiyorsa
    cacheMode = WebSettings.LOAD_NO_CACHE  // Hassas içerik için
    mixedContentMode = WebSettings.MIXED_CONTENT_NEVER_ALLOW
}

// ✅ URL whitelist kontrolü
webView.webViewClient = object : WebViewClient() {
    override fun shouldOverrideUrlLoading(view: WebView, request: WebResourceRequest): Boolean {
        val uri = request.url
        val allowedDomains = setOf("app.example.com", "cdn.example.com")

        return if (uri.host in allowedDomains && uri.scheme == "https") {
            false  // WebView'e yüklet
        } else {
            true   // Engelle
        }
    }

    override fun onReceivedSslError(view: WebView, handler: SslErrorHandler, error: SslError) {
        handler.cancel()  // ❌ handler.proceed() ASLA çağrılmamalı!
    }
}

// ✅ JavaScript Interface — input validation
class SafeJsBridge(private val context: Context) {
    @JavascriptInterface
    fun getToken(): String {
        // Thread kontrolü — ana thread'den çağrılmamalı
        check(Looper.myLooper() != Looper.getMainLooper())
        return tokenManager.getToken() ?: ""
    }

    @JavascriptInterface
    fun navigateTo(path: String) {
        val safePath = path.filter { it.isLetterOrDigit() || it == '/' || it == '-' }
        if (safePath.length > 100) return
        Handler(Looper.getMainLooper()).post {
            router.navigate(safePath)
        }
    }
}
```

---

## 🟡 USEFUL 3 — Kod Gizleme & Tersine Mühendislik Karşıtı

> **Gerçekçi Not:** R8/ProGuard bir **optimizasyon** aracıdır, güvenlik aracı değil. Yan etkisi olarak isim obfuscation yapar ama logic tamamen okunabilir kalır. jadx ile decompile edince `a.b.c()` isimleri görürsün, mantık aynıdır. Deneyimli bir reverse engineer için R8 bypass'ı 30 dakika. Gerçek koruma için DexGuard/iXGuard (ücretli) gerekir. **Sonuç:** Doğru `-keep` kurallarını yazmayı bil, limitleri anla — derinlemesine öğrenmeye gerek yok.

### 10.1 ProGuard / R8 Konfigürasyonu

```pro
# build.gradle
buildTypes {
    release {
        isMinifyEnabled = true
        isShrinkResources = true
        proguardFiles(
            getDefaultProguardFile("proguard-android-optimize.txt"),
            "proguard-rules.pro"
        )
    }
}

# proguard-rules.pro
# Model sınıflarını koru (Gson/Retrofit deserialize eder)
-keep class com.example.data.model.** { *; }

# Enum'ları koru
-keepclassmembers enum * { public static **[] values(); public static ** valueOf(java.lang.String); }

# Crash reporting için satır numaralarını koru
-keepattributes SourceFile,LineNumberTable
-renamesourcefileattribute SourceFile

# Native metodları koru
-keepclasseswithmembernames class * {
    native <methods>;
}

# Custom view'ları koru
-keep public class * extends android.view.View

# Reflection ile erişilen sınıflar
-keep class com.example.plugin.** { *; }
```

### 10.2 İleri Obfuscation — DexGuard / Guardsquare

ProGuard'ın ötesinde:

- String encryption (hardcoded string'leri şifreler)
- Class encryption (DEX sınıflarını şifreler, runtime'da çözer)
- Control flow obfuscation (decompiler'ları zorlaştırır)
- Reflection hardening

```kotlin
// DexGuard annotation (sınıfı şifrele)
@EncryptStrings
class ApiConfig {
    val baseUrl = "https://api.example.com"  // Encrypted at build time
    val apiKey = "sk-xxxx"                   // Encrypted at build time
}
```

### 10.3 Anti-Tamper

```kotlin
// ✅ APK imza doğrulama (runtime)
fun isSignatureValid(context: Context): Boolean {
    return try {
        val packageInfo = if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.P) {
            context.packageManager.getPackageInfo(
                context.packageName,
                PackageManager.GET_SIGNING_CERTIFICATES
            )
        } else {
            @Suppress("DEPRECATION")
            context.packageManager.getPackageInfo(
                context.packageName,
                PackageManager.GET_SIGNATURES
            )
        }

        val signatures = if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.P) {
            packageInfo.signingInfo.apkContentsSigners
        } else {
            @Suppress("DEPRECATION")
            packageInfo.signatures
        }

        val expectedHash = "AB:CD:EF:..." // Build zamanında sabit — DexGuard ile şifreli
        val actualHash = MessageDigest.getInstance("SHA-256")
            .digest(signatures.first().toByteArray())
            .joinToString(":") { "%02X".format(it) }

        actualHash == expectedHash
    } catch (e: Exception) {
        false
    }
}

// ✅ APK yolu kontrolü (yeniden paketlenmiş APK'lar farklı path'te olabilir)
fun isInstalledFromTrustedStore(context: Context): Boolean {
    val installer = context.packageManager
        .getInstallerPackageName(context.packageName)
    val trustedInstallers = setOf(
        "com.android.vending",           // Google Play
        "com.amazon.venezia",            // Amazon AppStore
        null                             // ADB install (debug için)
    )
    return installer in trustedInstallers
}
```

### 10.4 Anti-Debug

```kotlin
// ✅ Debug detection
fun isDebuggerAttached(): Boolean {
    return android.os.Debug.isDebuggerConnected() ||
           android.os.Debug.waitingForDebugger()
}

// ✅ Build tipi kontrolü
fun isDebuggableBuild(context: Context): Boolean {
    return context.applicationInfo.flags and ApplicationInfo.FLAG_DEBUGGABLE != 0
}

// ✅ TracerPid kontrolü (/proc/self/status)
fun isBeingTraced(): Boolean {
    return try {
        File("/proc/self/status").readLines()
            .find { it.startsWith("TracerPid:") }
            ?.substringAfter("TracerPid:")
            ?.trim()
            ?.toIntOrNull()
            ?.let { it != 0 }
            ?: false
    } catch (e: Exception) { false }
}

// Native (JNI) anti-debug — daha güvenilir
// ptrace(PTRACE_TRACEME, 0, 0, 0) → Zaten trace ediliyor mu?
```

---

## 🟠 IMPORTANT 2 — Root Detection & Integrity Checks

### 11.1 Root Detection Stratejileri

```kotlin
class RootDetector(private val context: Context) {

    fun isRooted(): Boolean {
        return checkSuBinary() ||
               checkBusyBox() ||
               checkDangerousApps() ||
               checkBuildTags() ||
               checkWritableSystemPaths() ||
               checkRWMount()
    }

    private fun checkSuBinary(): Boolean {
        val paths = arrayOf(
            "/system/bin/su", "/system/xbin/su",
            "/sbin/su", "/data/local/xbin/su",
            "/data/local/bin/su", "/data/local/su"
        )
        return paths.any { File(it).exists() }
    }

    private fun checkDangerousApps(): Boolean {
        val packages = listOf(
            "com.topjohnwu.magisk",          // Magisk
            "com.koushikdutta.superuser",
            "eu.chainfire.supersu",
            "com.noshufou.android.su",
            "com.thirdparty.superuser",
            "com.yellowes.su",
            "com.kingroot.kinguser",
            "com.kingo.root"
        )
        return packages.any {
            try {
                context.packageManager.getPackageInfo(it, 0)
                true
            } catch (e: PackageManager.NameNotFoundException) { false }
        }
    }

    private fun checkBuildTags(): Boolean {
        val tags = Build.TAGS
        return tags != null && tags.contains("test-keys")
    }

    private fun checkWritableSystemPaths(): Boolean {
        val dirs = arrayOf("/system", "/system/bin", "/vendor/bin", "/sys/fs")
        return dirs.any { path ->
            val mount = Runtime.getRuntime().exec("mount")
            mount.inputStream.bufferedReader().readLines()
                .any { line -> line.contains(path) && line.contains("rw,") }
        }
    }
}
```

### 11.2 Play Integrity API (Google'ın Tasdik Sistemi)

Play Integrity API, Google Play Protect tarafından imzalanmış bir verdict döndürür. Eski SafetyNet API'nin yerine geçti (deprecated 2024).

```kotlin
// ✅ Play Integrity API
implementation("com.google.android.play:integrity:1.3.0")

suspend fun getIntegrityVerdict(nonce: String): String {
    val integrityManager = IntegrityManagerFactory.create(context)

    val request = StandardIntegrityManager.StandardIntegrityTokenRequest.builder()
        .setRequestHash(nonce.sha256())  // Server'dan alınan nonce
        .build()

    val tokenResponse = integrityManager
        .requestStandardIntegrityToken(
            StandardIntegrityManager.PrepareIntegrityTokenRequest.builder()
                .setCloudProjectNumber(CLOUD_PROJECT_NUMBER)
                .build()
        )
        .await()

    val token = tokenResponse.token()
    // Token'ı server'a gönder, server Google'a doğrulat
    return token
}

// Server tarafı (Kotlin/Backend)
// verdict içeriği:
// - requestDetails.requestPackageName
// - appIntegrity.appRecognitionVerdict: PLAY_RECOGNIZED / UNRECOGNIZED_VERSION / UNEVALUATED
// - deviceIntegrity.deviceRecognitionVerdict: MEETS_DEVICE_INTEGRITY / MEETS_STRONG_INTEGRITY
// - accountDetails.appLicensingVerdict: LICENSED / UNLICENSED / UNEVALUATED
```

**Verdict Değerlendirmesi:**

| Verdict                  | Anlam                            | Önerilen Aksiyon                |
| ------------------------ | -------------------------------- | ------------------------------- |
| `MEETS_STRONG_INTEGRITY` | Sertifikalı cihaz, Play korumalı | Tam erişim                      |
| `MEETS_DEVICE_INTEGRITY` | Root'lu olabilir                 | Uyar, kısıtlı erişim            |
| `MEETS_BASIC_INTEGRITY`  | Emülatör/modifiye                | Yüksek riskli işlemleri engelle |
| `PLAY_RECOGNIZED`        | Google Play'den kurulmuş         | Normal akış                     |

### 11.3 Emülatör Tespiti

```kotlin
fun isEmulator(): Boolean {
    return (Build.FINGERPRINT.startsWith("generic") ||
            Build.FINGERPRINT.startsWith("unknown") ||
            Build.MODEL.contains("google_sdk") ||
            Build.MODEL.contains("Emulator") ||
            Build.MODEL.contains("Android SDK built for x86") ||
            Build.MANUFACTURER.contains("Genymotion") ||
            Build.BRAND.startsWith("generic") ||
            Build.DEVICE.startsWith("generic") ||
            "google_sdk" == Build.PRODUCT ||
            // BlueStacks
            Build.BOARD == "QC_Reference_Phone" ||
            // NOX
            Build.HARDWARE == "vbox86" ||
            // Genymotion
            Build.HARDWARE.contains("goldfish") ||
            Build.HARDWARE.contains("ranchu") ||
            // Sensor yok (emülatörler genelde sensörsüz)
            checkSensors())
}

private fun checkSensors(): Boolean {
    val sensorManager = context.getSystemService(SensorManager::class.java)
    return sensorManager.getSensorList(Sensor.TYPE_ALL).isEmpty()
}
```

---

## 🟠 IMPORTANT 4 — Supply Chain & Dependency Security

### 12.1 Dependency Vulnerability Scanning

```bash
# OWASP Dependency Check
./gradlew dependencyCheckAnalyze

# build.gradle.kts
plugins {
    id("org.owasp.dependencycheck") version "9.0.0"
}

dependencyCheck {
    failBuildOnCVSS = 7.0f  // CVSS 7+ ise build'i durdur
    formats = listOf("HTML", "JSON")
    nvd {
        apiKey = System.getenv("NVD_API_KEY")
    }
}

# Snyk
snyk test --file=build.gradle
snyk monitor

# GitHub Dependabot (otomatik PR açar)
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: "gradle"
    directory: "/"
    schedule:
      interval: "weekly"
    open-pull-requests-limit: 10
```

### 12.2 SDK Güvenliği

Third-party SDK'lar uygulama ile aynı UID'de çalışır → Tüm izinlere sahip!

```kotlin
// ✅ SDK izolasyonu için SdkSandbox (Android 13+)
// Privacy Sandbox — SDK, ayrı bir process'te çalışır
// Henüz erken aşamada, yakında zorunlu olabilir

// ✅ SDK yetkilendirmesi sınırlama (build.gradle)
// Her SDK için minimum gerekli izinleri listele
// SDK ProGuard kurallarını incele — ne keep ediyor?

// ✅ SDK güvenlik checklist
// □ SDK kaynağı güvenilir mi? (Official repo, verified publisher)
// □ Son güncelleme tarihi?
// □ CVE geçmişi var mı?
// □ İzin gereksinimleri makul mü?
// □ Sertifikası imzalı mı?
// □ Obfuscated source mu? (Reverse edilebilir mi?)
```

### 12.3 Version Pinning & Checksum Doğrulama

```kotlin
// gradle/libs.versions.toml
[versions]
okhttp = "4.12.0"
retrofit = "2.9.0"

[libraries]
okhttp = { module = "com.squareup.okhttp3:okhttp", version.ref = "okhttp" }

# gradle.properties — checksum doğrulama (Gradle 6.2+)
# Verification metadata
# ./gradlew --write-verification-metadata sha256 help
# → gradle/verification-metadata.xml oluşturur, tüm bağımlılık hash'lerini içerir
```

---

## ⚪ REFERENCE 2 — Platform Güvenlik Özellikleri

### 13.1 Android 12-15 Güvenlik Değişiklikleri

**Android 12:**

- `exported` artık `<intent-filter>` olan tüm bileşenler için zorunlu.
- Bluetooth izinleri ayrıldı: `BLUETOOTH_SCAN`, `BLUETOOTH_CONNECT`, `BLUETOOTH_ADVERTISE`.
- PendingIntent `FLAG_MUTABLE` / `FLAG_IMMUTABLE` zorunlu.
- Clipboard okuma → kullanıcıya bildirim.
- `foregroundServiceType` → kamera/mikrofon kullanan FGS için zorunlu.

```kotlin
// Android 12+ PendingIntent
val pendingIntent = PendingIntent.getActivity(
    context, 0, intent,
    PendingIntent.FLAG_IMMUTABLE or PendingIntent.FLAG_UPDATE_CURRENT
)
```

**Android 13:**

- `POST_NOTIFICATIONS` izni zorunlu (runtime permission).
- `NEARBY_WIFI_DEVICES` izni (WiFi scan için artık konum gerekmez).
- `READ_MEDIA_IMAGES`, `READ_MEDIA_VIDEO`, `READ_MEDIA_AUDIO` — granüler medya.
- Arka plan Activity başlatma ek kısıtlamalar.
- Play SDK Runtime (SDK Sandbox) erken erişim.

**Android 14:**

- `SCHEDULE_EXACT_ALARM` → kullanıcı ayarlarından açık olmalı.
- Partial Photo & Video access.
- Health Connect izinleri.
- Dinamik kod yükleme kısıtlamaları (`.dex` sideload).
- `minSdkVersion` 23 altı APK'lar kurulamaz.

**Android 15:**

- Ekran yakalama koruması güçlendirildi.
- Sahte erişilebilirlik hizmeti kısıtlamaları.
- PDF viewer built-in (WebView saldırı yüzeyi azaltıldı).
- Edge-to-edge zorunlu.

### 13.2 pKVM (Protected KVM) & Confidential Computing

Android 13+ bazı cihazlarda pKVM desteği:

- Hypervisor Level izolasyon → pVM'ler host'tan bile gizli belleğe sahip.
- Attestation: pVM imzası doğrulanabilir.
- Fintech / DRM uygulamaları için kritik platform güvenliği.

### 13.3 Scoped Storage

```kotlin
// Android 10+ Scoped Storage
// /sdcard'a doğrudan erişim yok, MediaStore API kullanılmalı

// Fotoğraf kaydetme
val values = ContentValues().apply {
    put(MediaStore.Images.Media.DISPLAY_NAME, "photo.jpg")
    put(MediaStore.Images.Media.MIME_TYPE, "image/jpeg")
    put(MediaStore.Images.Media.RELATIVE_PATH, Environment.DIRECTORY_PICTURES)
}
val uri = contentResolver.insert(MediaStore.Images.Media.EXTERNAL_CONTENT_URI, values)

// Kendi uygulama dosyaları — getExternalFilesDir (izin gerekmez)
val dir = context.getExternalFilesDir(Environment.DIRECTORY_DOCUMENTS)
```

---

## 🔴 CORE 6 — CI/CD & Güvenli Geliştirme Pratiği (SSDLC)

> **Neden CORE?** Diğer bölümler "açığı nasıl bulursun" sorusunu cevaplar. Bu bölüm "açığın production'a gitmesini nasıl engellersin" sorusunu cevaplar. Google gibi şirketler ikincisini daha çok önemser. **Minimum yapılacak:** Semgrep + OWASP Dependency Check + Gitleaks üçlüsünü bir kez gerçek projede kur, nasıl çalıştığını gör.

### 14.1 CI/CD Entegrasyonu

```yaml
# .github/workflows/security.yml
name: Security Scan

on: [push, pull_request]

jobs:
  sast:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Semgrep SAST
        uses: returntocorp/semgrep-action@v1
        with:
          config: p/android

      - name: MobSF Scan
        run: |
          docker run -d -p 8000:8000 opensecurity/mobile-security-framework-mobsf
          sleep 30
          UPLOAD=$(curl -s -F "file=@app/build/outputs/apk/release/app-release.apk" \
                   http://localhost:8000/api/v1/upload \
                   -H "Authorization: ${{ secrets.MOBSF_API_KEY }}")
          HASH=$(echo $UPLOAD | jq -r .hash)
          curl -s http://localhost:8000/api/v1/scan -d "hash=$HASH" \
               -H "Authorization: ${{ secrets.MOBSF_API_KEY }}"

      - name: OWASP Dependency Check
        run: ./gradlew dependencyCheckAnalyze

      - name: Secret Scanning (Gitleaks)
        uses: gitleaks/gitleaks-action@v2
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

### 14.2 Güvenlik Test Seviyeleri

```
Unit Test       → Input validation, crypto fonksiyonları
Integration     → IPC, API contract
Instrumented    → Runtime permission, biometric
Security Test   → Pentest, SAST/DAST otomatik
Penetration     → Manuel red-team, Frida/Drozer ile
```

### 14.3 Güvenli Kod Review Checklist

```markdown
## PR Güvenlik Kontrol Listesi

### Veri Güvenliği

- [ ] Yeni SharedPreferences → EncryptedSharedPreferences mi?
- [ ] Log'larda PII/token yok mu?
- [ ] Dosyalar internal storage'a mı yazılıyor?
- [ ] Kullanıcı verisi backup'tan hariç tutulmuş mu?

### Ağ

- [ ] HTTP URL kullanımı var mı?
- [ ] Yeni endpoint için SSL pinning eklendi mi?
- [ ] API anahtarı hardcoded değil mi?

### IPC

- [ ] Yeni Activity/Service/Receiver → exported=false mu?
- [ ] Deep link parametreleri doğrulanıyor mu?
- [ ] Content Provider projection whitelist'i var mı?

### Kriptografi

- [ ] Güvenli Random kullanılıyor mu?
- [ ] IV/Nonce her şifrelemede yeni üretiliyor mu?
- [ ] Zayıf algoritma (MD5, SHA1, DES, ECB) kullanılmıyor mu?

### Üçüncü Taraf

- [ ] Yeni dependency CVE taramasından geçti mi?
- [ ] SDK izin gereksinimleri incelendi mi?
```

---

## 🟡 USEFUL 2 — Penetration Testing Metodolojisi

### 15.1 OWASP MSTG / MASVS Çerçevesi

MASVS (Mobile Application Security Verification Standard) v2.0 üç seviye tanımlar:

| Seviye | Açıklama                     | Hedef Uygulama                  |
| ------ | ---------------------------- | ------------------------------- |
| L1     | Temel güvenlik               | Tüm uygulamalar                 |
| L2     | Derinlemesine savunma        | Hassas veri işleyen uygulamalar |
| R      | Tersine mühendislik koruması | DRM, fintech, sağlık            |

### 15.2 Pentest Aşamaları

**Faz 1: Keşif & Statik Analiz**

```bash
# APK toplama
adb shell pm list packages | grep target
adb shell pm path com.target.app
adb pull /data/app/~~xxxx==/com.target.app-xxxx==/base.apk

# Decode
apktool d base.apk -o decoded/
jadx -d java_output/ base.apk

# Gizli bilgi tespiti
grep -r "password\|secret\|key\|token\|api" decoded/ --include="*.xml"
grep -r "http://" decoded/ --include="*.xml"
strings base.apk | grep -E "(http|api|key|password|secret)"

# Sertifika bilgileri
apksigner verify --verbose --print-certs base.apk
keytool -printcert -jarfile base.apk
```

**Faz 2: Dinamik Analiz**

```bash
# Uygulama başlat, proxy'e bağla
adb shell am start -n com.target.app/.MainActivity

# Trafik yakala (Burp Suite)
# 1. Proxy ayarla
# 2. CA yükle
# 3. Tüm endpoint'leri tara

# Frida ile hook
frida -U -f com.target.app -l universal_ssl_bypass.js --no-pause

# Uygulama logları izle
adb logcat -s com.target.app:V *:E

# Dosya sistemi değişiklikleri izle
adb shell inotifywait -m -r /data/data/com.target.app/
```

**Faz 3: IPC Saldırıları**

```bash
# Drozer attack surface
drozer console connect
run app.package.attacksurface com.target.app
run app.activity.info -a com.target.app -u  # unexported dahil
run app.provider.info -a com.target.app
run scanner.provider.sqltables -a com.target.app
run scanner.provider.injection -a com.target.app
run scanner.activity.browsable -a com.target.app

# ADB ile activity başlat
adb shell am start -n com.target.app/.AdminActivity \
    --es "action" "delete_user" \
    --ei "user_id" "1"
```

**Faz 4: Binary Analiz**

```bash
# .so dosyası analizi
file lib/arm64-v8a/libnative.so
strings lib/arm64-v8a/libnative.so | grep -i "key\|secret\|pin"

# Ghidra ile decompile
# radare2 ile analiz
r2 -A lib/arm64-v8a/libnative.so
> afl    # Fonksiyonları listele
> pdf @ sym.check_license  # Fonksiyonu decompile et

# objdump
objdump -d lib/arm64-v8a/libnative.so | grep -A20 "verify"
```

**Faz 5: Raporlama**

````markdown
# Bulgu Rapor Şablonu

## Başlık: [CWE-ID] — Açıklama

### Risk Seviyesi: Kritik / Yüksek / Orta / Düşük

### CVSS v3.1 Skoru: X.X

Vector: CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:H

### Açıklama

...

### Teknik Detay

`kod veya adımlar`

### İstismar PoC

`adımlar`

### Etki

...

### Öneri

...

### Referans

- OWASP MSTG: MSTG-STORAGE-1
- CWE: CWE-312
````

---

## ⚪ REFERENCE 3 — Araç Seti Referansı

### 16.1 Statik Analiz Araçları

| Araç                | Kullanım                  | Lisans     |
| ------------------- | ------------------------- | ---------- |
| **apktool**         | APK decode/encode         | Apache 2.0 |
| **jadx**            | DEX → Java decompiler     | Apache 2.0 |
| **dex2jar**         | DEX → JAR                 | Apache 2.0 |
| **MobSF**           | Otomatik SAST/DAST        | GPL-3.0    |
| **semgrep**         | Custom SAST kuralları     | LGPL-2.1   |
| **APKiD**           | Packer/obfuscator tespiti | GPL-3.0    |
| **Ghidra**          | Reverse engineering       | Apache 2.0 |
| **radare2**         | Binary analiz             | LGPL-3.0   |
| **bytecode-viewer** | DEX bytecode viewer       | GPL-3.0    |

### 16.2 Dinamik Analiz Araçları

| Araç           | Kullanım                        | Lisans             |
| -------------- | ------------------------------- | ------------------ |
| **Frida**      | Runtime instrumentation         | wxWindows          |
| **Objection**  | Frida wrapper, otomatik test    | Apache 2.0         |
| **Drozer**     | IPC/component test              | BSD                |
| **Burp Suite** | HTTP proxy, MITM                | Ticari / Community |
| **mitmproxy**  | HTTP/S proxy, Python scriptable | MIT                |
| **r2frida**    | radare2 + Frida entegrasyon     | LGPL               |
| **Medusa**     | Frida-tabanlı modüler framework | MIT                |

### 16.3 Network Araçları

| Araç              | Kullanım                        |
| ----------------- | ------------------------------- |
| **Wireshark**     | Paket analizi (tüm protokoller) |
| **tcpdump**       | CLI paket yakalama              |
| **Charles Proxy** | Burp alternatifi, macOS odaklı  |
| **HTTP Toolkit**  | Modern HTTP proxy               |
| **SSLstrip2**     | HSTS bypass (lab ortamı)        |

### 16.4 Hızlı Kurulum Scripti

```bash
#!/bin/bash
# Android Security Toolkit Kurulumu

# Python araçları
pip3 install frida-tools objection androguard

# Node araçları
npm install -g apk-mitm

# jadx (latest)
wget https://github.com/skylot/jadx/releases/latest/download/jadx-1.5.0.zip
unzip jadx-1.5.0.zip -d ~/tools/jadx/

# apktool
wget https://raw.githubusercontent.com/iBotPeaches/Apktool/master/scripts/linux/apktool
wget https://bitbucket.org/iBotPeaches/apktool/downloads/apktool_2.9.3.jar
chmod +x apktool
sudo mv apktool apktool_2.9.3.jar /usr/local/bin/

# MobSF
docker pull opensecurity/mobile-security-framework-mobsf:latest

# Frida server (cihaz için)
# https://github.com/frida/frida/releases
# arch=$(adb shell getprop ro.product.cpu.abi)
# wget https://github.com/frida/frida/releases/download/16.x.x/frida-server-16.x.x-android-${arch}.xz
```

---

## Ekler

### Hızlı Referans: Yaygın Güvenlik Açıkları → Çözüm

| Açık                     | CWE     | Hızlı Çözüm                                |
| ------------------------ | ------- | ------------------------------------------ |
| Cleartext depolama       | CWE-312 | EncryptedSharedPreferences / EncryptedFile |
| Cleartext iletişim       | CWE-319 | HTTPS only + networkSecurityConfig         |
| SSL validation bypass    | CWE-295 | Varsayılan TrustManager kullan             |
| Hardcoded credentials    | CWE-798 | Keystore / server-side secret management   |
| SQL Injection            | CWE-89  | Room + parametreli sorgu                   |
| Debug log sızıntısı      | CWE-532 | Timber Release tree                        |
| Exported component       | CWE-926 | exported=false veya permission ekle        |
| Intent injection         | CWE-927 | Input validation + explicit intent         |
| Weak crypto              | CWE-327 | AES-256-GCM + Argon2/PBKDF2                |
| Insecure deserialization | CWE-502 | JSON yerine (Moshi/kotlinx.serialization)  |
| Path traversal           | CWE-22  | File path normalization + whitelist        |
| WebView XSS → RCE        | CWE-79  | URL whitelist + JS interface validation    |

### Kritik Güvenlik Standartları & Kaynaklar

- **OWASP MASVS v2.0** — https://mas.owasp.org/MASVS/
- **OWASP MSTG** — https://mas.owasp.org/MASTG/
- **Android Security Bulletins** — https://source.android.com/docs/security/bulletin
- **CWE Mobile Top 25** — https://cwe.mitre.org/top25/
- **NIST Mobile Security Guidelines** — SP 800-163r1
- **Google Play Policy Center** — https://play.google.com/about/developer-content-policy/

---

_Son güncelleme: Mart 2025 | Android 15 / API 35 baz alınmıştır_
