# Android Core Security Practices — Production Checklist

> **Hedef Kitle:** Her Android developer'ın bilmesi ve her PR'da uygulaması gereken temel güvenlik prensipleri. Bu doküman production-ready uygulamalar için minimum güvenlik gereksinimlerini içerir.

> **İlişkili Doküman:** Detaylı güvenlik analizi, pentest teknikleri ve ileri seviye konular için [Security Guide](./security-guide.md) dokümanına bakın.

---

## İçindekiler

1. [Android Güvenlik Mimarisi Temelleri](#1-android-güvenlik-mimarisi-temelleri)
2. [Ağ Güvenliği](#2-ağ-güvenliği)
3. [Kriptografi](#3-kriptografi)
4. [Data Storage Güvenliği](#4-data-storage-güvenliği)
5. [IPC Güvenliği](#5-ipc-güvenliği)
6. [CI/CD & Güvenli Geliştirme](#6-cicd--güvenli-geliştirme)
7. [Production Deployment](#7-production-deployment)
8. [Monitoring & Incident Response](#8-monitoring--incident-response)

---

## 1. Android Güvenlik Mimarisi Temelleri

### 1.1 Katmanlı Savunma Modeli

Android güvenliği "defense-in-depth" prensibine dayanır:

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

### 1.2 Sandbox & UID İzolasyonu

Her APK farklı bir Linux UID ile çalışır:

- UID `u0_a<N>` — Her uygulama benzersiz bir UID alır
- Uygulama dosyaları `/data/data/<package>/` altında yalnızca o UID'e aittir
- SELinux (Mandatory Access Control) enforcing mode

```bash
# SELinux context görme
adb shell ls -Z /data/data/com.example.app/

# Process context
adb shell ps -Z | grep com.example
```

### 1.3 Permission Modeli

| Kategori   | Örnek                     | Davranış                    |
| ---------- | ------------------------- | --------------------------- |
| Normal     | `INTERNET`, `VIBRATE`     | Install-time, otomatik      |
| Dangerous  | `CAMERA`, `READ_CONTACTS` | Runtime prompt (Android 6+) |
| Signature  | Özel sistem perms         | Aynı sertifika ile sign     |
| Privileged | `INSTALL_PACKAGES`        | System/priv-app + allowlist |

**Android 12+ Kritik Değişiklikler:**

- `exported` zorunlu (intent-filter olan tüm component'ler için)
- PendingIntent `FLAG_IMMUTABLE` / `FLAG_MUTABLE` zorunlu
- Bluetooth izinleri ayrıldı: `BLUETOOTH_SCAN`, `BLUETOOTH_CONNECT`

```kotlin
// ✅ Android 12+ PendingIntent
val pendingIntent = PendingIntent.getActivity(
    context, 0, intent,
    PendingIntent.FLAG_IMMUTABLE or PendingIntent.FLAG_UPDATE_CURRENT
)
```

---

## 2. Ağ Güvenliği

### 2.1 HTTPS Only — Cleartext Traffic Yasağı

```xml
<!-- AndroidManifest.xml -->
<application
    android:usesCleartextTraffic="false"
    android:networkSecurityConfig="@xml/network_security_config">
```

```xml
<!-- res/xml/network_security_config.xml -->
<network-security-config>
    <base-config cleartextTrafficPermitted="false">
        <trust-anchors>
            <certificates src="system"/>
        </trust-anchors>
    </base-config>

    <!-- Debug ortamı için exception -->
    <domain-config cleartextTrafficPermitted="true">
        <domain includeSubdomains="false">10.0.2.2</domain>
        <domain includeSubdomains="false">localhost</domain>
    </domain-config>
</network-security-config>
```

### 2.2 SSL/TLS Konfigürasyonu

```kotlin
// ✅ Doğru: OkHttp ile güvenli TLS
val client = OkHttpClient.Builder()
    .connectionSpecs(
        listOf(
            ConnectionSpec.Builder(ConnectionSpec.MODERN_TLS)
                .tlsVersions(TlsVersion.TLS_1_3, TlsVersion.TLS_1_2)
                .build()
        )
    )
    .build()

// ❌ Tehlikeli: Tüm sertifikalara güven
val trustAll = object : X509TrustManager {
    override fun checkClientTrusted(chain: Array<X509Certificate>, authType: String) {}
    override fun checkServerTrusted(chain: Array<X509Certificate>, authType: String) {}
    override fun getAcceptedIssuers(): Array<X509Certificate> = emptyArray()
}
// ASLA production'da kullanma!
```

### 2.3 SSL Pinning (Production Zorunlu)

**Yöntem 1: OkHttp CertificatePinner**

```kotlin
// Public key hash'ini al:
// openssl s_client -connect api.example.com:443 | openssl x509 -pubkey -noout |
// openssl rsa -pubin -outform der | openssl dgst -sha256 -binary | base64

val client = OkHttpClient.Builder()
    .certificatePinner(
        CertificatePinner.Builder()
            // Birincil sertifika
            .add("api.example.com", "sha256/AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=")
            // Backup pin (sertifika rotasyonu için ZORUNLU!)
            .add("api.example.com", "sha256/BBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBB=")
            .build()
    )
    .build()
```

**Yöntem 2: Network Security Config**

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

**⚠️ Pin Rotation Stratejisi:**

```kotlin
// Remote config ile pin güncellemesi
class DynamicPinningManager(
    private val remoteConfig: FirebaseRemoteConfig
) {
    fun getCurrentPins(): Set<String> {
        return remoteConfig.getString("ssl_pins")
            .split(",")
            .toSet()
    }

    // Uygulama başlangıcında pin'leri güncelle
    suspend fun updatePins() {
        remoteConfig.fetchAndActivate().await()
        val newPins = getCurrentPins()
        // CertificatePinner'ı güncelle
    }
}
```

### 2.4 API Güvenliği

```kotlin
// ✅ Request imzalama (HMAC-SHA256)
class ApiRequestSigner(private val secretKey: ByteArray) {

    fun signRequest(url: String, body: String): String {
        val timestamp = System.currentTimeMillis() / 1000
        val payload = "$timestamp\n$url\n${body.sha256()}"

        val mac = Mac.getInstance("HmacSHA256")
        mac.init(SecretKeySpec(secretKey, "HmacSHA256"))

        return mac.doFinal(payload.toByteArray()).base64()
    }
}

// ✅ OkHttp Interceptor ile otomatik imzalama
class SignatureInterceptor(private val signer: ApiRequestSigner) : Interceptor {
    override fun intercept(chain: Interceptor.Chain): Response {
        val originalRequest = chain.request()
        val body = originalRequest.body?.let {
            val buffer = Buffer()
            it.writeTo(buffer)
            buffer.readUtf8()
        } ?: ""

        val signature = signer.signRequest(originalRequest.url.toString(), body)
        val timestamp = System.currentTimeMillis() / 1000

        val signedRequest = originalRequest.newBuilder()
            .header("X-Timestamp", timestamp.toString())
            .header("X-Signature", signature)
            .build()

        return chain.proceed(signedRequest)
    }
}
```

### 2.5 Certificate Transparency

```kotlin
// Android 9+ varsayılan olarak CT gerektiriyor (publicly trusted CA'lar için)
// Conscrypt ile CT doğrulama
val provider = Conscrypt.newProvider()
Security.insertProviderAt(provider, 1)
```

---

## 3. Kriptografi

### 3.1 Yaygın Kriptografi Hataları

```kotlin
// ❌ ECB modu (blok paterni sızdırır)
Cipher.getInstance("AES/ECB/PKCS5Padding")

// ❌ Sabit IV (nonce reuse)
val iv = ByteArray(12) { 0 }

// ❌ MD5 / SHA-1 ile şifre hash
MessageDigest.getInstance("MD5").digest(password.toByteArray())

// ❌ Zayıf PRNG
Random().nextInt()

// ❌ Hardcoded key
val key = SecretKeySpec("1234567890abcdef".toByteArray(), "AES")
```

### 3.2 Doğru Şifreleme (AES-256-GCM)

```kotlin
// ✅ AES-256-GCM (authenticated encryption)
class SecureEncryption {

    fun encrypt(plaintext: ByteArray, key: SecretKey): EncryptedData {
        val cipher = Cipher.getInstance("AES/GCM/NoPadding")

        // Her seferinde yeni IV
        val iv = ByteArray(12).also { SecureRandom().nextBytes(it) }

        cipher.init(Cipher.ENCRYPT_MODE, key, GCMParameterSpec(128, iv))
        val ciphertext = cipher.doFinal(plaintext)

        return EncryptedData(iv, ciphertext)
    }

    fun decrypt(data: EncryptedData, key: SecretKey): ByteArray {
        val cipher = Cipher.getInstance("AES/GCM/NoPadding")
        cipher.init(Cipher.DECRYPT_MODE, key, GCMParameterSpec(128, data.iv))
        return cipher.doFinal(data.ciphertext)  // GCM otomatik integrity kontrolü yapar
    }
}

data class EncryptedData(val iv: ByteArray, val ciphertext: ByteArray)
```

### 3.3 Şifre Hash (PBKDF2 / Argon2)

```kotlin
// ✅ PBKDF2 ile güvenli şifre hash
class PasswordHasher {

    fun hashPassword(password: CharArray, salt: ByteArray = generateSalt()): HashedPassword {
        val factory = SecretKeyFactory.getInstance("PBKDF2WithHmacSHA256")
        val spec = PBEKeySpec(
            password,
            salt,
            600_000,  // OWASP 2024 minimum iterasyon
            256
        )

        val hash = factory.generateSecret(spec).encoded
        return HashedPassword(salt, hash)
    }

    fun verifyPassword(password: CharArray, stored: HashedPassword): Boolean {
        val computed = hashPassword(password, stored.salt)
        return computed.hash.contentEquals(stored.hash)
    }

    private fun generateSalt(): ByteArray {
        return ByteArray(32).also { SecureRandom().nextBytes(it) }
    }
}

data class HashedPassword(val salt: ByteArray, val hash: ByteArray)
```

### 3.4 Android Keystore

Keystore, kriptografik anahtarları TEE (Trusted Execution Environment) veya StrongBox içinde saklar.

```kotlin
// ✅ Keystore'da anahtar üretme
class KeystoreManager(private val context: Context) {

    fun generateKey(alias: String, requireBiometric: Boolean = true) {
        val keyGenerator = KeyGenerator.getInstance(
            KeyProperties.KEY_ALGORITHM_AES,
            "AndroidKeyStore"
        )

        keyGenerator.init(
            KeyGenParameterSpec.Builder(
                alias,
                KeyProperties.PURPOSE_ENCRYPT or KeyProperties.PURPOSE_DECRYPT
            )
                .setKeySize(256)
                .setBlockModes(KeyProperties.BLOCK_MODE_GCM)
                .setEncryptionPaddings(KeyProperties.ENCRYPTION_PADDING_NONE)
                .setUserAuthenticationRequired(requireBiometric)
                .setUserAuthenticationParameters(
                    0,  // 0 = her kullanımda auth gerekir
                    KeyProperties.AUTH_BIOMETRIC_STRONG or KeyProperties.AUTH_DEVICE_CREDENTIAL
                )
                .setInvalidatedByBiometricEnrollment(true)
                .setUnlockedDeviceRequired(true)
                .setIsStrongBoxBacked(true)  // HSM kullan (varsa)
                .build()
        )

        keyGenerator.generateKey()
    }

    fun getKey(alias: String): SecretKey {
        val keyStore = KeyStore.getInstance("AndroidKeyStore").apply { load(null) }
        return keyStore.getKey(alias, null) as SecretKey
    }
}
```

### 3.5 EncryptedSharedPreferences

```kotlin
// ✅ Jetpack Security Crypto
implementation("androidx.security:security-crypto:1.1.0-alpha06")

class SecurePreferences(private val context: Context) {

    private val masterKey = MasterKey.Builder(context)
        .setKeyScheme(MasterKey.KeyScheme.AES256_GCM)
        .setUserAuthenticationRequired(true, 30)  // 30 saniyede bir re-auth
        .build()

    private val encryptedPrefs = EncryptedSharedPreferences.create(
        context,
        "secure_prefs",
        masterKey,
        EncryptedSharedPreferences.PrefKeyEncryptionScheme.AES256_SIV,
        EncryptedSharedPreferences.PrefValueEncryptionScheme.AES256_GCM
    )

    fun saveToken(token: String) {
        encryptedPrefs.edit().putString("auth_token", token).apply()
    }

    fun getToken(): String? {
        return encryptedPrefs.getString("auth_token", null)
    }
}
```

---

## 4. Data Storage Güvenliği

### 4.1 Depolama Güvenlik Matrisi

| Depolama                     | Erişim   | Şifreleme | Uygun Kullanım          |
| ---------------------------- | -------- | --------- | ----------------------- |
| `SharedPreferences`          | Uygulama | ❌         | Şifresiz tercihler      |
| `EncryptedSharedPreferences` | Uygulama | ✅ AES-256 | Token, küçük gizli veri |
| `Internal Storage`           | Uygulama | ❌ (FBE)   | Uygulama dosyaları      |
| `EncryptedFile`              | Uygulama | ✅ AES-256 | Hassas dosyalar         |
| `Room + SQLCipher`           | Uygulama | ✅ AES-256 | Şifreli yerel DB        |
| `External Storage`           | Herkese  | ❌         | Paylaşılabilir medya    |
| `Keystore`                   | TEE      | ✅ Donanım | Kriptografik anahtarlar |

### 4.2 SQLite Güvenliği

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

### 4.3 Log Güvenliği

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

// Application.onCreate()
if (BuildConfig.DEBUG) {
    Timber.plant(Timber.DebugTree())
} else {
    Timber.plant(ReleaseTree())
}
```

### 4.4 Backup Güvenliği

```xml
<!-- AndroidManifest.xml -->
<application
    android:allowBackup="false"
    android:fullBackupContent="@xml/backup_rules">
```

```xml
<!-- res/xml/backup_rules.xml -->
<full-backup-content>
    <!-- Internal storage'daki dosyaları dahil et -->
    <include domain="file" path="."/>

    <!-- Hassas dosyaları hariç tut -->
    <exclude domain="sharedpref" path="secure_prefs"/>
    <exclude domain="database" path="app.db"/>
    <exclude domain="file" path="secret_keys/"/>
</full-backup-content>
```

### 4.5 Clipboard Güvenliği

```kotlin
// Hassas alanlar için clipboard'a erişimi engelle
binding.passwordField.apply {
    setOnLongClickListener { true }  // Context menu'yü engelle
    customSelectionActionModeCallback = object : ActionMode.Callback {
        override fun onCreateActionMode(mode: ActionMode?, menu: Menu?) = false
        override fun onPrepareActionMode(mode: ActionMode?, menu: Menu?) = false
        override fun onActionItemClicked(mode: ActionMode?, item: MenuItem?) = false
        override fun onDestroyActionMode(mode: ActionMode?) {}
    }
}

// Compose
TextField(
    value = password,
    onValueChange = { password = it },
    visualTransformation = PasswordVisualTransformation(),
    keyboardOptions = KeyboardOptions(keyboardType = KeyboardType.Password)
)
```

---

## 5. IPC Güvenliği

### 5.1 Intent Güvenliği

```kotlin
// ❌ Implicit intent ile hassas data
val intent = Intent("com.example.ACTION_PAYMENT")
intent.putExtra("amount", 100.0)
sendBroadcast(intent)  // Herhangi bir uygulama yakalayabilir!

// ✅ Explicit intent
val intent = Intent(context, PaymentService::class.java)
intent.putExtra("amount", 100.0)
startService(intent)

// ✅ Permission ile protected broadcast
sendBroadcast(intent, "com.example.permission.PAYMENT_RECEIVER")

// ✅ LocalBroadcastManager (aynı process)
LocalBroadcastManager.getInstance(context).sendBroadcast(intent)

// ✅ Intent doğrulama
override fun onNewIntent(intent: Intent?) {
    super.onNewIntent(intent)

    val action = intent?.action ?: return
    val allowedActions = setOf("com.example.SAFE_ACTION")
    if (action !in allowedActions) return

    val amount = intent.getDoubleExtra("amount", -1.0)
    require(amount > 0 && amount < 10_000) { "Invalid amount" }
}
```

### 5.2 Component Export Kontrolü

```xml
<!-- ✅ Güvenli Manifest -->
<application>
    <!-- Internal Activity -->
    <activity
        android:name=".InternalActivity"
        android:exported="false"/>

    <!-- Deep Link Activity — doğrulama ile -->
    <activity
        android:name=".DeepLinkActivity"
        android:exported="true">
        <intent-filter android:autoVerify="true">
            <action android:name="android.intent.action.VIEW"/>
            <category android:name="android.intent.category.BROWSABLE"/>
            <data android:scheme="https" android:host="example.com"/>
        </intent-filter>
    </activity>

    <!-- Protected Service -->
    <service
        android:name=".PaymentService"
        android:exported="true"
        android:permission="com.example.permission.USE_PAYMENT_SERVICE"/>

    <!-- Protected Receiver -->
    <receiver
        android:name=".PaymentReceiver"
        android:exported="true"
        android:permission="com.example.permission.SEND_PAYMENT"/>
</application>
```

### 5.3 Content Provider Güvenliği

```kotlin
class SecureProvider : ContentProvider() {

    private val allowedColumns = setOf("id", "name", "email")

    override fun query(
        uri: Uri,
        projection: Array<String>?,
        selection: String?,
        selectionArgs: Array<String>?,
        sortOrder: String?
    ): Cursor? {
        // Projection whitelist
        val safeProjection = projection?.filter { it in allowedColumns }?.toTypedArray()
            ?: allowedColumns.toTypedArray()

        // SQL injection koruması
        val safeSelection = DatabaseUtils.sqlEscapeString(selection ?: "")

        return db.query(TABLE, safeProjection, safeSelection, selectionArgs, null, null, sortOrder)
    }
}
```

```xml
<!-- AndroidManifest.xml -->
<provider
    android:name=".SecureProvider"
    android:exported="false"
    android:grantUriPermissions="false"
    android:readPermission="com.example.READ_DATA"
    android:writePermission="com.example.WRITE_DATA"/>
```

### 5.4 Deep Link & App Link Güvenliği

```kotlin
// ✅ App Links (HTTPS + domain doğrulaması)
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

```json
// .well-known/assetlinks.json (sunucuda barındır)
[{
  "relation": ["delegate_permission/common.handle_all_urls"],
  "target": {
    "namespace": "android_app",
    "package_name": "com.example.app",
    "sha256_cert_fingerprints": [
      "AB:CD:EF:..."
    ]
  }
}]
```

### 5.5 WebView Güvenliği

```kotlin
// ✅ Güvenli WebView konfigürasyonu
webView.settings.apply {
    javaScriptEnabled = true  // Gerekiyorsa
    allowFileAccess = false
    allowContentAccess = false
    allowUniversalAccessFromFileURLs = false
    allowFileAccessFromFileURLs = false
    setSupportZoom(false)
    databaseEnabled = false
    domStorageEnabled = false
    cacheMode = WebSettings.LOAD_NO_CACHE
    mixedContentMode = WebSettings.MIXED_CONTENT_NEVER_ALLOW
}

// URL whitelist kontrolü
webView.webViewClient = object : WebViewClient() {
    override fun shouldOverrideUrlLoading(
        view: WebView,
        request: WebResourceRequest
    ): Boolean {
        val uri = request.url
        val allowedDomains = setOf("app.example.com", "cdn.example.com")

        return if (uri.host in allowedDomains && uri.scheme == "https") {
            false  // WebView'e yüklet
        } else {
            true   // Engelle
        }
    }

    override fun onReceivedSslError(
        view: WebView,
        handler: SslErrorHandler,
        error: SslError
    ) {
        handler.cancel()  // ❌ ASLA handler.proceed() çağrılmamalı!
    }
}

// JavaScript Interface — input validation
class SafeJsBridge(private val context: Context) {
    @JavascriptInterface
    fun getToken(): String {
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

## 6. CI/CD & Güvenli Geliştirme

### 6.1 Otomatik Güvenlik Taraması

```yaml
# .github/workflows/security.yml
name: Security Scan

on: [push, pull_request]

jobs:
  security-checks:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      # SAST — Semgrep
      - name: Semgrep SAST
        uses: returntocorp/semgrep-action@v1
        with:
          config: p/android

      # Dependency Check
      - name: OWASP Dependency Check
        run: ./gradlew dependencyCheckAnalyze

      # Secret Scanning
      - name: Gitleaks
        uses: gitleaks/gitleaks-action@v2
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

      # Lint Security Issues
      - name: Android Lint
        run: ./gradlew lintRelease

      # Build APK
      - name: Build Release APK
        run: ./gradlew assembleRelease

      # Upload APK for manual testing
      - name: Upload APK
        uses: actions/upload-artifact@v3
        with:
          name: app-release
          path: app/build/outputs/apk/release/app-release.apk
```

### 6.2 Güvenli Kod Review Checklist

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
- [ ] Certificate transparency kontrolü var mı?

### IPC
- [ ] Yeni Activity/Service/Receiver → exported=false mu?
- [ ] Deep link parametreleri doğrulanıyor mu?
- [ ] Content Provider projection whitelist'i var mı?
- [ ] WebView URL whitelist kontrolü var mı?

### Kriptografi
- [ ] SecureRandom kullanılıyor mu?
- [ ] IV/Nonce her şifrelemede yeni üretiliyor mu?
- [ ] Zayıf algoritma (MD5, SHA1, DES, ECB) yok mu?
- [ ] Keystore kullanılıyor mu?

### Üçüncü Taraf
- [ ] Yeni dependency CVE taramasından geçti mi?
- [ ] SDK izin gereksinimleri incelendi mi?
- [ ] Version pinning yapıldı mı?

### Build & Deploy
- [ ] ProGuard/R8 enabled mi?
- [ ] debuggable=false mi (release)?
- [ ] Signing config güvenli mi?
```

### 6.3 Semgrep Custom Rules

```yaml
# .semgrep/android-security.yml
rules:
  - id: hardcoded-secret
    pattern: |
      val $VAR = "$SECRET"
    message: Hardcoded secret detected
    severity: ERROR
    languages: [kotlin]

  - id: cleartext-traffic
    pattern: |
      android:usesCleartextTraffic="true"
    message: Cleartext traffic is enabled
    severity: WARNING
    languages: [xml]

  - id: weak-crypto
    patterns:
      - pattern-either:
          - pattern: Cipher.getInstance("AES/ECB/$PADDING")
          - pattern: MessageDigest.getInstance("MD5")
          - pattern: MessageDigest.getInstance("SHA1")
    message: Weak cryptography algorithm
    severity: ERROR
    languages: [kotlin, java]

  - id: log-sensitive-data
    pattern-either:
      - pattern: Log.$LEVEL(..., $DATA, ...)
        metavariable-regex:
          metavariable: $DATA
          regex: (password|token|secret|key|credit_card)
    message: Sensitive data in logs
    severity: ERROR
    languages: [kotlin, java]
```

### 6.4 Dependency Security

```kotlin
// build.gradle.kts
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

// gradle/libs.versions.toml — Version Catalog
[versions]
okhttp = "4.12.0"
retrofit = "2.9.0"
androidx-security = "1.1.0-alpha06"

[libraries]
okhttp = { module = "com.squareup.okhttp3:okhttp", version.ref = "okhttp" }
retrofit = { module = "com.squareup.retrofit2:retrofit", version.ref = "retrofit" }
androidx-security-crypto = { module = "androidx.security:security-crypto", version.ref = "androidx-security" }

# Verification metadata ile checksum kontrolü
# ./gradlew --write-verification-metadata sha256 help
```

---

## 7. Production Deployment

### 7.1 APK Signing Best Practices

```bash
# Keystore oluşturma (ilk kez)
keytool -genkeypair -v \
  -keystore release.keystore \
  -alias release \
  -keyalg RSA \
  -keysize 4096 \
  -validity 10000 \
  -storepass $KEYSTORE_PASSWORD \
  -keypass $KEY_PASSWORD

# Keystore backup (GÜVENLİ YERDE SAKLA!)
# - Cloud KMS (Google Cloud, AWS, Azure)
# - Hardware Security Module (HSM)
# - Encrypted backup + offline storage
```

```kotlin
// build.gradle.kts — Signing config
android {
    signingConfigs {
        create("release") {
            storeFile = file(System.getenv("KEYSTORE_FILE") ?: "release.keystore")
            storePassword = System.getenv("KEYSTORE_PASSWORD")
            keyAlias = System.getenv("KEY_ALIAS")
            keyPassword = System.getenv("KEY_PASSWORD")

            // v3.1 signature scheme (Android 13+)
            enableV3Signing = true
            enableV4Signing = true
        }
    }

    buildTypes {
        release {
            signingConfig = signingConfigs.getByName("release")
            isMinifyEnabled = true
            isShrinkResources = true
            proguardFiles(
                getDefaultProguardFile("proguard-android-optimize.txt"),
                "proguard-rules.pro"
            )
        }
    }
}
```

### 7.2 ProGuard/R8 Production Rules

```pro
# proguard-rules.pro

# Keep line numbers for crash reporting
-keepattributes SourceFile,LineNumberTable
-renamesourcefileattribute SourceFile

# Keep custom exceptions
-keep public class * extends java.lang.Exception

# Keep native methods
-keepclasseswithmembernames class * {
    native <methods>;
}

# Model classes (Gson/Moshi/kotlinx.serialization)
-keep class com.example.data.model.** { *; }

# Enum'ları koru
-keepclassmembers enum * {
    public static **[] values();
    public static ** valueOf(java.lang.String);
}

# Parcelable
-keep class * implements android.os.Parcelable {
    public static final android.os.Parcelable$Creator *;
}

# Remove debug logs
-assumenosideeffects class android.util.Log {
    public static *** d(...);
    public static *** v(...);
}

# Security: Obfuscate package names
-repackageclasses 'o'
```

### 7.3 Build Variants & Security Configs

```kotlin
// build.gradle.kts
android {
    buildTypes {
        debug {
            applicationIdSuffix = ".debug"
            isDebuggable = true
            // Debug ortamı için user CA'lara güven
            manifestPlaceholders["networkSecurityConfig"] = "@xml/network_security_config_debug"
        }

        release {
            isDebuggable = false
            isMinifyEnabled = true
            isShrinkResources = true
            manifestPlaceholders["networkSecurityConfig"] = "@xml/network_security_config"
        }

        create("staging") {
            initWith(getByName("release"))
            applicationIdSuffix = ".staging"
            // Staging ortamı için farklı API endpoints
        }
    }

    flavorDimensions += "environment"
    productFlavors {
        create("production") {
            dimension = "environment"
            buildConfigField("String", "API_BASE_URL", "\"https://api.example.com\"")
        }

        create("development") {
            dimension = "environment"
            buildConfigField("String", "API_BASE_URL", "\"https://api-dev.example.com\"")
        }
    }
}
```

### 7.4 Remote Config ile Feature Flags

```kotlin
// ✅ Security-critical features için feature flags
class SecurityFeatureFlags(
    private val remoteConfig: FirebaseRemoteConfig
) {

    suspend fun initialize() {
        remoteConfig.setConfigSettingsAsync(
            remoteConfigSettings {
                minimumFetchIntervalInSeconds = 3600  // 1 saat
            }
        )

        remoteConfig.setDefaultsAsync(
            mapOf(
                "ssl_pinning_enabled" to true,
                "root_detection_enabled" to true,
                "max_api_retry_count" to 3,
                "force_update_min_version" to 100
            )
        )

        remoteConfig.fetchAndActivate().await()
    }

    fun isSslPinningEnabled(): Boolean {
        return remoteConfig.getBoolean("ssl_pinning_enabled")
    }

    fun isRootDetectionEnabled(): Boolean {
        return remoteConfig.getBoolean("root_detection_enabled")
    }

    fun shouldForceUpdate(currentVersion: Int): Boolean {
        val minVersion = remoteConfig.getLong("force_update_min_version").toInt()
        return currentVersion < minVersion
    }
}
```

---

## 8. Monitoring & Incident Response

### 8.1 Crash Reporting

```kotlin
// ✅ Firebase Crashlytics
class CrashReportingManager(
    private val crashlytics: FirebaseCrashlytics
) {

    fun initialize(userId: String?) {
        crashlytics.setUserId(userId ?: "anonymous")

        // Custom keys for debugging
        crashlytics.setCustomKey("environment", BuildConfig.BUILD_TYPE)
        crashlytics.setCustomKey("api_version", BuildConfig.VERSION_CODE)
    }

    fun logSecurityEvent(event: SecurityEvent) {
        crashlytics.log("Security: ${event.type} - ${event.message}")

        if (event.severity == Severity.CRITICAL) {
            crashlytics.recordException(
                SecurityException("${event.type}: ${event.message}")
            )
        }
    }

    fun logNonFatal(throwable: Throwable) {
        crashlytics.recordException(throwable)
    }
}

data class SecurityEvent(
    val type: String,
    val message: String,
    val severity: Severity
)

enum class Severity {
    INFO, WARNING, CRITICAL
}
```

### 8.2 Security Analytics

```kotlin
// ✅ Güvenlik olaylarını izleme
class SecurityAnalytics(
    private val analytics: FirebaseAnalytics,
    private val crashReporting: CrashReportingManager
) {

    fun logSslPinningFailure(host: String) {
        analytics.logEvent("security_ssl_pinning_failure") {
            param("host", host)
            param("timestamp", System.currentTimeMillis())
        }

        crashReporting.logSecurityEvent(
            SecurityEvent(
                type = "SSL_PINNING_FAILURE",
                message = "Host: $host",
                severity = Severity.CRITICAL
            )
        )
    }

    fun logRootDetected() {
        analytics.logEvent("security_root_detected") {
            param("device_model", Build.MODEL)
            param("os_version", Build.VERSION.SDK_INT.toString())
        }
    }

    fun logApiAuthFailure(statusCode: Int) {
        analytics.logEvent("security_api_auth_failure") {
            param("status_code", statusCode.toString())
        }
    }

    fun logSuspiciousActivity(activity: String) {
        crashReporting.logSecurityEvent(
            SecurityEvent(
                type = "SUSPICIOUS_ACTIVITY",
                message = activity,
                severity = Severity.WARNING
            )
        )
    }
}
```

### 8.3 Rate Limiting & Throttling

```kotlin
// ✅ Client-side rate limiting
class ApiRateLimiter {

    private val requestTimestamps = mutableMapOf<String, MutableList<Long>>()
    private val maxRequestsPerMinute = 60

    fun shouldAllowRequest(endpoint: String): Boolean {
        val now = System.currentTimeMillis()
        val timestamps = requestTimestamps.getOrPut(endpoint) { mutableListOf() }

        // Son 1 dakika içindeki request'leri filtrele
        timestamps.removeAll { now - it > 60_000 }

        return if (timestamps.size < maxRequestsPerMinute) {
            timestamps.add(now)
            true
        } else {
            false
        }
    }
}

// OkHttp Interceptor ile entegrasyon
class RateLimitInterceptor(
    private val rateLimiter: ApiRateLimiter
) : Interceptor {
    override fun intercept(chain: Interceptor.Chain): Response {
        val request = chain.request()
        val endpoint = request.url.encodedPath

        if (!rateLimiter.shouldAllowRequest(endpoint)) {
            throw RateLimitException("Too many requests to $endpoint")
        }

        return chain.proceed(request)
    }
}
```

### 8.4 Incident Response Plan

```kotlin
// ✅ Emergency kill switch
class EmergencyKillSwitch(
    private val remoteConfig: FirebaseRemoteConfig
) {

    suspend fun checkKillSwitch(): KillSwitchStatus {
        remoteConfig.fetchAndActivate().await()

        val isKilled = remoteConfig.getBoolean("emergency_kill_switch")
        val message = remoteConfig.getString("kill_switch_message")
        val minVersion = remoteConfig.getLong("min_supported_version").toInt()

        return when {
            isKilled -> KillSwitchStatus.Killed(message)
            BuildConfig.VERSION_CODE < minVersion -> KillSwitchStatus.ForceUpdate(minVersion)
            else -> KillSwitchStatus.Active
        }
    }
}

sealed class KillSwitchStatus {
    object Active : KillSwitchStatus()
    data class Killed(val message: String) : KillSwitchStatus()
    data class ForceUpdate(val minVersion: Int) : KillSwitchStatus()
}

// Application.onCreate() içinde kontrol
class MyApplication : Application() {
    override fun onCreate() {
        super.onCreate()

        lifecycleScope.launch {
            when (val status = killSwitch.checkKillSwitch()) {
                is KillSwitchStatus.Killed -> {
                    // Uygulamayı kapat ve kullanıcıyı bilgilendir
                    showKillSwitchDialog(status.message)
                }
                is KillSwitchStatus.ForceUpdate -> {
                    // Play Store'a yönlendir
                    showForceUpdateDialog(status.minVersion)
                }
                KillSwitchStatus.Active -> {
                    // Normal başlatma
                }
            }
        }
    }
}
```

### 8.5 Security Breach Response

```markdown
# Güvenlik İhlali Müdahale Planı

## Faz 1: Tespit (0-1 saat)
- [ ] İhlal kapsamını belirle (hangi veriler, kaç kullanıcı)
- [ ] İhlal kaynağını tespit et (API, client-side, third-party)
- [ ] Log'ları topla ve analiz et
- [ ] Incident response team'i bilgilendir

## Faz 2: Kontrol Altına Alma (1-4 saat)
- [ ] Etkilenen endpoint'leri kapat (kill switch)
- [ ] Etkilenen kullanıcıları logout et (token revoke)
- [ ] API rate limit'i sıfırla
- [ ] Emergency patch hazırla

## Faz 3: Eradikasyon (4-24 saat)
- [ ] Root cause'u düzelt
- [ ] Güvenlik yamalarını uygula
- [ ] Penetration test ile doğrula
- [ ] Emergency release yayınla

## Faz 4: Recovery (24-72 saat)
- [ ] Etkilenen kullanıcıları bilgilendir
- [ ] Şifre reset zorunluluğu (gerekirse)
- [ ] Güvenlik güncellemesini dağıt
- [ ] Monitoring'i artır

## Faz 5: Post-Mortem (72+ saat)
- [ ] Incident report hazırla
- [ ] Root cause analysis
- [ ] Preventive measures planla
- [ ] Team training düzenle
```

---

## Hızlı Referans: Common Vulnerabilities

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
| Insecure deserialization | CWE-502 | JSON (Moshi/kotlinx.serialization)         |
| Path traversal           | CWE-22  | File path normalization + whitelist        |
| WebView XSS → RCE        | CWE-79  | URL whitelist + JS interface validation    |

---

## Production Deployment Checklist

```markdown
## Release Öncesi Güvenlik Kontrolü

### Build Configuration
- [ ] `debuggable=false` (AndroidManifest.xml)
- [ ] `android:allowBackup="false"` veya custom backup rules
- [ ] `android:usesCleartextTraffic="false"`
- [ ] ProGuard/R8 enabled ve optimized
- [ ] Signing config production keystore ile
- [ ] Version code ve name güncel

### Network Security
- [ ] SSL pinning production endpoints için aktif
- [ ] Network security config production profili kullanıyor
- [ ] Certificate transparency kontrolü aktif
- [ ] API request signing uygulanmış
- [ ] Timeout ve retry stratejileri tanımlı

### Data Security
- [ ] Hassas veriler EncryptedSharedPreferences'da
- [ ] SQLite encryption (SQLCipher) aktif
- [ ] Keystore kullanımı doğrulanmış
- [ ] Log'larda PII yok
- [ ] Clipboard koruması hassas alanlarda

### IPC Security
- [ ] Tüm component'lerin export durumu gözden geçirilmiş
- [ ] Deep link validation aktif
- [ ] App Links doğrulaması yapılmış
- [ ] WebView güvenlik ayarları production profili
- [ ] FileProvider doğru konfigüre edilmiş

### Third-Party
- [ ] Dependency CVE taraması yapılmış
- [ ] SDK'lar güncel versiyonda
- [ ] Third-party SDK izinleri gözden geçirilmiş
- [ ] Analytics PII scrubbing aktif

### Monitoring
- [ ] Crashlytics entegre ve test edilmiş
- [ ] Security analytics event'leri tanımlı
- [ ] Kill switch mekanizması test edilmiş
- [ ] Rate limiting aktif
- [ ] Performance monitoring aktif

### Testing
- [ ] SAST (Semgrep) passed
- [ ] DAST test edilmiş
- [ ] Manual pentest tamamlanmış
- [ ] Regression test suite passed
- [ ] Beta testing tamamlanmış

### Compliance
- [ ] GDPR uyumluluğu (gerekirse)
- [ ] OWASP MASVS-L1 minimum compliance
- [ ] Play Store policy uyumluluğu
- [ ] Privacy policy güncel
- [ ] Terms of service güncel
```

---

## Kaynaklar

- **OWASP MASVS v2.0** — https://mas.owasp.org/MASVS/
- **OWASP MSTG** — https://mas.owasp.org/MASTG/
- **Android Security Best Practices** — https://developer.android.com/topic/security/best-practices
- **Google Play Security Policy** — https://play.google.com/about/developer-content-policy/
- **CWE Mobile Top 25** — https://cwe.mitre.org/top25/

---

_Son güncelleme: Mart 2025 | Android 15 / API 35 baz alınmıştır_
