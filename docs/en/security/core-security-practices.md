# Android Core Security Practices — Production Checklist

> **Target Audience:** Essential security principles that every Android developer should know and apply in every PR. This document contains minimum security requirements for production-ready applications.

> **Related Document:** For detailed security analysis, pentest techniques, and advanced topics, see the [Security Guide](./security-guide.md) document.

---

## Table of Contents

1. [Android Security Architecture Fundamentals](#1-android-security-architecture-fundamentals)
2. [Network Security](#2-network-security)
3. [Cryptography](#3-cryptography)
4. [Data Storage Security](#4-data-storage-security)
5. [IPC Security](#5-ipc-security)
6. [CI/CD & Secure Development](#6-cicd--secure-development)
7. [Production Deployment](#7-production-deployment)
8. [Monitoring & Incident Response](#8-monitoring--incident-response)

---

## 1. Android Security Architecture Fundamentals

### 1.1 Defense-in-Depth Model

Android security is based on the "defense-in-depth" principle:

```
┌─────────────────────────────────┐
│      Application Layer          │  ← Developer's control
├─────────────────────────────────┤
│   Android Framework (Java)      │  ← Permission, Intent filtering
├─────────────────────────────────┤
│   Native Libraries / NDK        │  ← C/C++, BIONIC libc
├─────────────────────────────────┤
│   Android Runtime (ART)         │  ← Bytecode verification, DEX
├─────────────────────────────────┤
│       HAL (Vendor)              │  ← Isolated by SELinux
├─────────────────────────────────┤
│      Linux Kernel               │  ← UID isolation, Seccomp
├─────────────────────────────────┤
│   Hardware (TEE / StrongBox)    │  ← Keystore, Secure Enclave
└─────────────────────────────────┘
```

### 1.2 Sandbox & UID Isolation

Each APK runs with a different Linux UID:

- UID `u0_a<N>` — Each app gets a unique UID
- App files under `/data/data/<package>/` belong only to that UID
- SELinux (Mandatory Access Control) in enforcing mode

```bash
# View SELinux context
adb shell ls -Z /data/data/com.example.app/

# Process context
adb shell ps -Z | grep com.example
```

### 1.3 Permission Model

| Category   | Example                   | Behavior                        |
| ---------- | ------------------------- | ------------------------------- |
| Normal     | `INTERNET`, `VIBRATE`     | Install-time, automatic grant   |
| Dangerous  | `CAMERA`, `READ_CONTACTS` | Runtime prompt (Android 6+)     |
| Signature  | Custom system perms       | Same signature required         |
| Privileged | `INSTALL_PACKAGES`        | System/priv-app + allowlist     |

**Android 12+ Critical Changes:**

- `exported` mandatory (for all components with intent-filter)
- PendingIntent `FLAG_IMMUTABLE` / `FLAG_MUTABLE` required
- Bluetooth permissions split: `BLUETOOTH_SCAN`, `BLUETOOTH_CONNECT`

```kotlin
// ✅ Android 12+ PendingIntent
val pendingIntent = PendingIntent.getActivity(
    context, 0, intent,
    PendingIntent.FLAG_IMMUTABLE or PendingIntent.FLAG_UPDATE_CURRENT
)
```

---

## 2. Network Security

### 2.1 HTTPS Only — Cleartext Traffic Ban

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

    <!-- Exception for debug environment -->
    <domain-config cleartextTrafficPermitted="true">
        <domain includeSubdomains="false">10.0.2.2</domain>
        <domain includeSubdomains="false">localhost</domain>
    </domain-config>
</network-security-config>
```

### 2.2 SSL/TLS Configuration

```kotlin
// ✅ Correct: Secure TLS with OkHttp
val client = OkHttpClient.Builder()
    .connectionSpecs(
        listOf(
            ConnectionSpec.Builder(ConnectionSpec.MODERN_TLS)
                .tlsVersions(TlsVersion.TLS_1_3, TlsVersion.TLS_1_2)
                .build()
        )
    )
    .build()

// ❌ Dangerous: Trust all certificates
val trustAll = object : X509TrustManager {
    override fun checkClientTrusted(chain: Array<X509Certificate>, authType: String) {}
    override fun checkServerTrusted(chain: Array<X509Certificate>, authType: String) {}
    override fun getAcceptedIssuers(): Array<X509Certificate> = emptyArray()
}
// NEVER use in production!
```

### 2.3 SSL Pinning (Production Mandatory)

**Method 1: OkHttp CertificatePinner**

```kotlin
// Get public key hash:
// openssl s_client -connect api.example.com:443 | openssl x509 -pubkey -noout |
// openssl rsa -pubin -outform der | openssl dgst -sha256 -binary | base64

val client = OkHttpClient.Builder()
    .certificatePinner(
        CertificatePinner.Builder()
            // Primary certificate
            .add("api.example.com", "sha256/AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=")
            // Backup pin (MANDATORY for certificate rotation!)
            .add("api.example.com", "sha256/BBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBB=")
            .build()
    )
    .build()
```

**Method 2: Network Security Config**

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

**⚠️ Pin Rotation Strategy:**

```kotlin
// Update pins via remote config
class DynamicPinningManager(
    private val remoteConfig: FirebaseRemoteConfig
) {
    fun getCurrentPins(): Set<String> {
        return remoteConfig.getString("ssl_pins")
            .split(",")
            .toSet()
    }

    // Update pins at app startup
    suspend fun updatePins() {
        remoteConfig.fetchAndActivate().await()
        val newPins = getCurrentPins()
        // Update CertificatePinner
    }
}
```

### 2.4 API Security

```kotlin
// ✅ Request signing (HMAC-SHA256)
class ApiRequestSigner(private val secretKey: ByteArray) {

    fun signRequest(url: String, body: String): String {
        val timestamp = System.currentTimeMillis() / 1000
        val payload = "$timestamp\n$url\n${body.sha256()}"

        val mac = Mac.getInstance("HmacSHA256")
        mac.init(SecretKeySpec(secretKey, "HmacSHA256"))

        return mac.doFinal(payload.toByteArray()).base64()
    }
}

// ✅ Automatic signing with OkHttp Interceptor
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
// Android 9+ requires CT by default (for publicly trusted CAs)
// CT verification with Conscrypt
val provider = Conscrypt.newProvider()
Security.insertProviderAt(provider, 1)
```

---

## 3. Cryptography

### 3.1 Common Cryptography Mistakes

```kotlin
// ❌ ECB mode (leaks block patterns)
Cipher.getInstance("AES/ECB/PKCS5Padding")

// ❌ Fixed IV (nonce reuse)
val iv = ByteArray(12) { 0 }

// ❌ Password hashing with MD5 / SHA-1
MessageDigest.getInstance("MD5").digest(password.toByteArray())

// ❌ Weak PRNG
Random().nextInt()

// ❌ Hardcoded key
val key = SecretKeySpec("1234567890abcdef".toByteArray(), "AES")
```

### 3.2 Correct Encryption (AES-256-GCM)

```kotlin
// ✅ AES-256-GCM (authenticated encryption)
class SecureEncryption {

    fun encrypt(plaintext: ByteArray, key: SecretKey): EncryptedData {
        val cipher = Cipher.getInstance("AES/GCM/NoPadding")

        // New IV each time
        val iv = ByteArray(12).also { SecureRandom().nextBytes(it) }

        cipher.init(Cipher.ENCRYPT_MODE, key, GCMParameterSpec(128, iv))
        val ciphertext = cipher.doFinal(plaintext)

        return EncryptedData(iv, ciphertext)
    }

    fun decrypt(data: EncryptedData, key: SecretKey): ByteArray {
        val cipher = Cipher.getInstance("AES/GCM/NoPadding")
        cipher.init(Cipher.DECRYPT_MODE, key, GCMParameterSpec(128, data.iv))
        return cipher.doFinal(data.ciphertext)  // GCM performs automatic integrity check
    }
}

data class EncryptedData(val iv: ByteArray, val ciphertext: ByteArray)
```

### 3.3 Password Hashing (PBKDF2 / Argon2)

```kotlin
// ✅ Secure password hashing with PBKDF2
class PasswordHasher {

    fun hashPassword(password: CharArray, salt: ByteArray = generateSalt()): HashedPassword {
        val factory = SecretKeyFactory.getInstance("PBKDF2WithHmacSHA256")
        val spec = PBEKeySpec(
            password,
            salt,
            600_000,  // OWASP 2024 minimum iterations
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

Keystore stores cryptographic keys in TEE (Trusted Execution Environment) or StrongBox.

```kotlin
// ✅ Generate key in Keystore
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
                    0,  // 0 = auth required for each use
                    KeyProperties.AUTH_BIOMETRIC_STRONG or KeyProperties.AUTH_DEVICE_CREDENTIAL
                )
                .setInvalidatedByBiometricEnrollment(true)
                .setUnlockedDeviceRequired(true)
                .setIsStrongBoxBacked(true)  // Use HSM if available
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
        .setUserAuthenticationRequired(true, 30)  // Re-auth every 30 seconds
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

## 4. Data Storage Security

### 4.1 Storage Security Matrix

| Storage                      | Access | Encryption | Appropriate Use         |
| ---------------------------- | ------ | ---------- | ----------------------- |
| `SharedPreferences`          | App    | ❌          | Unencrypted preferences |
| `EncryptedSharedPreferences` | App    | ✅ AES-256  | Tokens, small secrets   |
| `Internal Storage`           | App    | ❌ (FBE)    | App files               |
| `EncryptedFile`              | App    | ✅ AES-256  | Sensitive files         |
| `Room + SQLCipher`           | App    | ✅ AES-256  | Encrypted local DB      |
| `External Storage`           | Public | ❌          | Shareable media         |
| `Keystore`                   | TEE    | ✅ Hardware | Cryptographic keys      |

### 4.2 SQLite Security

```kotlin
// ❌ SQL Injection vulnerability
db.rawQuery("SELECT * FROM users WHERE id = $userId", null)

// ✅ Parameterized query
db.rawQuery("SELECT * FROM users WHERE id = ?", arrayOf(userId.toString()))

// ✅ With Room (automatic parameterization)
@Query("SELECT * FROM users WHERE id = :userId")
fun getUserById(userId: Int): User

// ✅ Encrypted database with SQLCipher
implementation("net.zetetic:android-database-sqlcipher:4.5.4")

val passphrase = SQLiteDatabase.getBytes(masterPassword)
val factory = SupportFactory(passphrase)
val db = Room.databaseBuilder(context, AppDatabase::class.java, "app.db")
    .openHelperFactory(factory)
    .build()
```

### 4.3 Log Security

```kotlin
// ❌ Log leakage in production
Log.d("Payment", "Card number: $cardNumber, CVV: $cvv")

// ✅ BuildConfig check
if (BuildConfig.DEBUG) {
    Log.d(TAG, "Debug only: $safeInfo")
}

// ✅ Timber — filter logs in production
class ReleaseTree : Timber.Tree() {
    override fun log(priority: Int, tag: String?, message: String, t: Throwable?) {
        if (priority == Log.ERROR || priority == Log.WARN) {
            // Only error/warn → send to Crashlytics (without PII!)
            FirebaseCrashlytics.getInstance().log(message)
        }
        // Debug/Verbose → do nothing
    }
}

// Application.onCreate()
if (BuildConfig.DEBUG) {
    Timber.plant(Timber.DebugTree())
} else {
    Timber.plant(ReleaseTree())
}
```

### 4.4 Backup Security

```xml
<!-- AndroidManifest.xml -->
<application
    android:allowBackup="false"
    android:fullBackupContent="@xml/backup_rules">
```

```xml
<!-- res/xml/backup_rules.xml -->
<full-backup-content>
    <!-- Include files from internal storage -->
    <include domain="file" path="."/>

    <!-- Exclude sensitive files -->
    <exclude domain="sharedpref" path="secure_prefs"/>
    <exclude domain="database" path="app.db"/>
    <exclude domain="file" path="secret_keys/"/>
</full-backup-content>
```

### 4.5 Clipboard Security

```kotlin
// Disable clipboard access for sensitive fields
binding.passwordField.apply {
    setOnLongClickListener { true }  // Disable context menu
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

## 5. IPC Security

### 5.1 Intent Security

```kotlin
// ❌ Implicit intent with sensitive data
val intent = Intent("com.example.ACTION_PAYMENT")
intent.putExtra("amount", 100.0)
sendBroadcast(intent)  // Any app can receive it!

// ✅ Explicit intent
val intent = Intent(context, PaymentService::class.java)
intent.putExtra("amount", 100.0)
startService(intent)

// ✅ Protected broadcast with permission
sendBroadcast(intent, "com.example.permission.PAYMENT_RECEIVER")

// ✅ LocalBroadcastManager (same process)
LocalBroadcastManager.getInstance(context).sendBroadcast(intent)

// ✅ Intent validation
override fun onNewIntent(intent: Intent?) {
    super.onNewIntent(intent)

    val action = intent?.action ?: return
    val allowedActions = setOf("com.example.SAFE_ACTION")
    if (action !in allowedActions) return

    val amount = intent.getDoubleExtra("amount", -1.0)
    require(amount > 0 && amount < 10_000) { "Invalid amount" }
}
```

### 5.2 Component Export Control

```xml
<!-- ✅ Secure Manifest -->
<application>
    <!-- Internal Activity -->
    <activity
        android:name=".InternalActivity"
        android:exported="false"/>

    <!-- Deep Link Activity — with validation -->
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

### 5.3 Content Provider Security

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

        // SQL injection protection
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

### 5.4 Deep Link & App Link Security

```kotlin
// ✅ App Links (HTTPS + domain verification)
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

    // Sanitize parameters
    val id = data.getQueryParameter("id")?.toLongOrNull() ?: return
    navigateToProduct(id)
}
```

```json
// .well-known/assetlinks.json (host on server)
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

### 5.5 WebView Security

```kotlin
// ✅ Secure WebView configuration
webView.settings.apply {
    javaScriptEnabled = true  // If needed
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

// URL whitelist check
webView.webViewClient = object : WebViewClient() {
    override fun shouldOverrideUrlLoading(
        view: WebView,
        request: WebResourceRequest
    ): Boolean {
        val uri = request.url
        val allowedDomains = setOf("app.example.com", "cdn.example.com")

        return if (uri.host in allowedDomains && uri.scheme == "https") {
            false  // Load in WebView
        } else {
            true   // Block
        }
    }

    override fun onReceivedSslError(
        view: WebView,
        handler: SslErrorHandler,
        error: SslError
    ) {
        handler.cancel()  // ❌ NEVER call handler.proceed()!
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

## 6. CI/CD & Secure Development

### 6.1 Automated Security Scanning

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

### 6.2 Secure Code Review Checklist

```markdown
## PR Security Checklist

### Data Security
- [ ] New SharedPreferences → using EncryptedSharedPreferences?
- [ ] No PII/tokens in logs?
- [ ] Files written to internal storage?
- [ ] User data excluded from backup?

### Network
- [ ] Any HTTP URL usage?
- [ ] SSL pinning added for new endpoints?
- [ ] No hardcoded API keys?
- [ ] Certificate transparency check in place?

### IPC
- [ ] New Activity/Service/Receiver → exported=false?
- [ ] Deep link parameters validated?
- [ ] Content Provider has projection whitelist?
- [ ] WebView has URL whitelist check?

### Cryptography
- [ ] Using SecureRandom?
- [ ] New IV/Nonce generated for each encryption?
- [ ] No weak algorithms (MD5, SHA1, DES, ECB)?
- [ ] Using Keystore?

### Third-Party
- [ ] New dependency passed CVE scan?
- [ ] SDK permission requirements reviewed?
- [ ] Version pinning applied?

### Build & Deploy
- [ ] ProGuard/R8 enabled?
- [ ] debuggable=false (release)?
- [ ] Signing config secure?
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
    failBuildOnCVSS = 7.0f  // Fail build if CVSS 7+
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

# Checksum verification with verification metadata
# ./gradlew --write-verification-metadata sha256 help
```

---

## 7. Production Deployment

### 7.1 APK Signing Best Practices

```bash
# Create keystore (first time)
keytool -genkeypair -v \
  -keystore release.keystore \
  -alias release \
  -keyalg RSA \
  -keysize 4096 \
  -validity 10000 \
  -storepass $KEYSTORE_PASSWORD \
  -keypass $KEY_PASSWORD

# Keystore backup (STORE SECURELY!)
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

# Keep enums
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
            // Trust user CAs for debug environment
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
            // Different API endpoints for staging
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

### 7.4 Remote Config with Feature Flags

```kotlin
// ✅ Feature flags for security-critical features
class SecurityFeatureFlags(
    private val remoteConfig: FirebaseRemoteConfig
) {

    suspend fun initialize() {
        remoteConfig.setConfigSettingsAsync(
            remoteConfigSettings {
                minimumFetchIntervalInSeconds = 3600  // 1 hour
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
// ✅ Monitor security events
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

        // Filter requests from last minute
        timestamps.removeAll { now - it > 60_000 }

        return if (timestamps.size < maxRequestsPerMinute) {
            timestamps.add(now)
            true
        } else {
            false
        }
    }
}

// Integration with OkHttp Interceptor
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

// Check in Application.onCreate()
class MyApplication : Application() {
    override fun onCreate() {
        super.onCreate()

        lifecycleScope.launch {
            when (val status = killSwitch.checkKillSwitch()) {
                is KillSwitchStatus.Killed -> {
                    // Close app and inform user
                    showKillSwitchDialog(status.message)
                }
                is KillSwitchStatus.ForceUpdate -> {
                    // Redirect to Play Store
                    showForceUpdateDialog(status.minVersion)
                }
                KillSwitchStatus.Active -> {
                    // Normal startup
                }
            }
        }
    }
}
```

### 8.5 Security Breach Response

```markdown
# Security Breach Response Plan

## Phase 1: Detection (0-1 hour)
- [ ] Determine breach scope (which data, how many users)
- [ ] Identify breach source (API, client-side, third-party)
- [ ] Collect and analyze logs
- [ ] Notify incident response team

## Phase 2: Containment (1-4 hours)
- [ ] Close affected endpoints (kill switch)
- [ ] Logout affected users (token revoke)
- [ ] Reset API rate limits
- [ ] Prepare emergency patch

## Phase 3: Eradication (4-24 hours)
- [ ] Fix root cause
- [ ] Apply security patches
- [ ] Validate with penetration test
- [ ] Release emergency update

## Phase 4: Recovery (24-72 hours)
- [ ] Notify affected users
- [ ] Force password reset (if needed)
- [ ] Distribute security update
- [ ] Increase monitoring

## Phase 5: Post-Mortem (72+ hours)
- [ ] Prepare incident report
- [ ] Root cause analysis
- [ ] Plan preventive measures
- [ ] Conduct team training
```

---

## Quick Reference: Common Vulnerabilities

| Vulnerability                | CWE     | Quick Fix                                  |
| ---------------------------- | ------- | ------------------------------------------ |
| Cleartext storage            | CWE-312 | EncryptedSharedPreferences / EncryptedFile |
| Cleartext communication      | CWE-319 | HTTPS only + networkSecurityConfig         |
| SSL validation bypass        | CWE-295 | Use default TrustManager                   |
| Hardcoded credentials        | CWE-798 | Keystore / server-side secret management   |
| SQL Injection                | CWE-89  | Room + parameterized queries               |
| Debug log leakage            | CWE-532 | Timber Release tree                        |
| Exported component           | CWE-926 | exported=false or add permission           |
| Intent injection             | CWE-927 | Input validation + explicit intent         |
| Weak crypto                  | CWE-327 | AES-256-GCM + Argon2/PBKDF2                |
| Insecure deserialization     | CWE-502 | JSON (Moshi/kotlinx.serialization)         |
| Path traversal               | CWE-22  | File path normalization + whitelist        |
| WebView XSS → RCE            | CWE-79  | URL whitelist + JS interface validation    |

---

## Production Deployment Checklist

```markdown
## Pre-Release Security Check

### Build Configuration
- [ ] `debuggable=false` (AndroidManifest.xml)
- [ ] `android:allowBackup="false"` or custom backup rules
- [ ] `android:usesCleartextTraffic="false"`
- [ ] ProGuard/R8 enabled and optimized
- [ ] Signing config with production keystore
- [ ] Version code and name updated

### Network Security
- [ ] SSL pinning active for production endpoints
- [ ] Network security config using production profile
- [ ] Certificate transparency check active
- [ ] API request signing implemented
- [ ] Timeout and retry strategies defined

### Data Security
- [ ] Sensitive data in EncryptedSharedPreferences
- [ ] SQLite encryption (SQLCipher) active
- [ ] Keystore usage verified
- [ ] No PII in logs
- [ ] Clipboard protection on sensitive fields

### IPC Security
- [ ] All components' export status reviewed
- [ ] Deep link validation active
- [ ] App Links verification completed
- [ ] WebView security settings in production profile
- [ ] FileProvider correctly configured

### Third-Party
- [ ] Dependency CVE scan completed
- [ ] SDKs on latest versions
- [ ] Third-party SDK permissions reviewed
- [ ] Analytics PII scrubbing active

### Monitoring
- [ ] Crashlytics integrated and tested
- [ ] Security analytics events defined
- [ ] Kill switch mechanism tested
- [ ] Rate limiting active
- [ ] Performance monitoring active

### Testing
- [ ] SAST (Semgrep) passed
- [ ] DAST tested
- [ ] Manual pentest completed
- [ ] Regression test suite passed
- [ ] Beta testing completed

### Compliance
- [ ] GDPR compliance (if applicable)
- [ ] OWASP MASVS-L1 minimum compliance
- [ ] Play Store policy compliance
- [ ] Privacy policy updated
- [ ] Terms of service updated
```

---

## Resources

- **OWASP MASVS v2.0** — https://mas.owasp.org/MASVS/
- **OWASP MSTG** — https://mas.owasp.org/MASTG/
- **Android Security Best Practices** — https://developer.android.com/topic/security/best-practices
- **Google Play Security Policy** — https://play.google.com/about/developer-content-policy/
- **CWE Mobile Top 25** — https://cwe.mitre.org/top25/

---

_Last updated: March 2025 | Based on Android 15 / API 35_
