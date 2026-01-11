# Separation of Concerns (SoC)

## Giriş

**Separation of Concerns (SoC)**, yazılımı farklı sorumluluk alanlarına (concerns) göre ayırma prensibidir. Her modül veya katman sadece bir "concern"den sorumlu olmalıdır.

**Gerçek hayat analojisi:**
- **Restoran**: Garson sipariş alır, aşçı yemek yapar, kasiyer hesabı alır
- Her kişinin **tek bir görevi** var
- Aşçı hesap almaz, garson yemek yapmaz

**Android'de:**
- UI kodu → Sadece ekranda gösterim
- Business logic → Sadece iş kuralları
- Data kodu → Sadece veri getirme/kaydetme

> **Temel Fikir:** "Her modül sadece bir concern'den (kaygıdan/sorumluluktan) sorumlu olmalı"

**Önemli Not:** SoC, Clean Architecture'ın temel prensibidir. SOLID prensiplerinin mimari seviyede uygulanmış halidir.

---

## SRP (Single Responsibility) ile Farkı

| | SRP | SoC |
|---|-----|-----|
| **Seviye** | Sınıf/modül seviyesi | Mimari seviyesi |
| **Scope** | Bir sınıf tek sorumluluk | Bir katman tek concern |
| **Soru** | "Bu sınıf ne yapmalı?" | "Bu katman ne yapmalı?" |
| **Örnek** | `ArticleRepository` sadece veri yönetimi | Data katmanı sadece veri concern'i |
| **İlişki** | SoC'yi uygulamak için SRP kullanırsın | SoC, SRP'nin büyük kardeşi |

**Basitçe:**
- **SRP**: Sınıf seviyesinde tek sorumluluk
- **SoC**: Mimari seviyede katmanları ayırma

---

## Android'de Concern Türleri

Android uygulamasında farklı "concern" türleri vardır:

### 1. UI Rendering Concern (Presentation)
- Ekranda gösterme
- User input handling
- Navigation
- **Örnek:** `Activity`, `Fragment`, `Composable`, `ViewModel`

### 2. Business Logic Concern (Domain)
- İş kuralları
- Validation
- Use case orchestration
- **Örnek:** `GetArticleUseCase`, `ValidateEmailUseCase`

### 3. Data Management Concern (Data)
- Network istekleri
- Database işlemleri
- Caching stratejisi
- **Örnek:** `ArticleRepositoryImpl`, `ApiService`, `ArticleDao`

### 4. Analytics Concern (Cross-cutting)
- Event tracking
- User behavior
- Logging
- **Örnek:** `AnalyticsService`, `LogArticleViewUseCase`

### 5. Authentication Concern (Cross-cutting)
- Login/logout
- Token yönetimi
- Session handling
- **Örnek:** `AuthRepository`, `TokenManager`

### 6. Error Handling Concern (Cross-cutting)
- Exception handling
- User-friendly error messages
- Retry logic
- **Örnek:** `ErrorMapper`, `Result<T>` pattern

### 7. Configuration Concern (Cross-cutting)
- Remote config
- Feature flags
- A/B testing
- **Örnek:** `RemoteConfigService`, `FeatureFlagManager`

**Not:** Cross-cutting concerns her katmanda olabilir ama **ayrı modüller** olmalıdır.

---

## Clean Architecture'da Separation of Concerns

Clean Architecture, SoC prinsibinin mimari seviyede uygulanmış halidir.

### Katmanlar ve Sorumlulukları

```
┌─────────────────────────────────────────────────┐
│   PRESENTATION LAYER (UI Concern)               │
│   - Activity/Fragment/Composable                │
│   - ViewModel                                   │
│   - UI State                                    │
│   Sorumluluk: Kullanıcıya gösterme, input alma │
└────────────────────┬────────────────────────────┘
                     │ Uses (Dependency)
                     ▼
┌─────────────────────────────────────────────────┐
│   DOMAIN LAYER (Business Logic Concern)         │
│   - Use Cases                                   │
│   - Domain Models                               │
│   - Repository Interfaces                       │
│   Sorumluluk: "Ne yapılacak" sorusuna cevap     │
└────────────────────┬────────────────────────────┘
                     │ Implements
                     ▼
┌─────────────────────────────────────────────────┐
│   DATA LAYER (Data Access Concern)              │
│   - Repository Implementations                  │
│   - API Service (Retrofit/Ktor)                 │
│   - Database DAO (Room)                         │
│   Sorumluluk: "Nasıl yapılacak" sorusuna cevap  │
└─────────────────────────────────────────────────┘

        ┌─────────────────────────────────┐
        │   CROSS-CUTTING CONCERNS        │
        │   - Analytics                   │
        │   - Logging                     │
        │   - Authentication              │
        └─────────────────────────────────┘
```

| Katman | Concern | Sorumluluk | Örnek |
|--------|---------|------------|-------|
| **Presentation** | UI concern | Kullanıcıya gösterme, input alma | `ArticleActivity`, `ArticleViewModel` |
| **Domain** | Business logic concern | İş kuralları, validation, orchestration | `GetArticleUseCase`, `Article` (model) |
| **Data** | Data access concern | Veri getirme/kaydetme (nasıl?) | `ArticleRepositoryImpl`, `ApiService`, `ArticleDao` |

---

## ❌ Yanlış Örnek (SoC İhlali)

```kotlin
// God Object Anti-Pattern - Tüm concern'ler bir arada
class ArticleActivity : AppCompatActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_article)

        // CONCERN 1: Network Configuration (Data Layer)
        val retrofit = Retrofit.Builder()
            .baseUrl("https://api.news.com")
            .addConverterFactory(GsonConverterFactory.create())
            .build()
        val apiService = retrofit.create(ApiService::class.java)

        lifecycleScope.launch {
            // CONCERN 2: Data Fetching (Data Layer)
            val response = apiService.getArticle("123")
            val article = response.body()

            // CONCERN 3: Business Logic (Domain Layer)
            if (article?.isPremium == true && !user.isSubscribed) {
                // Premium kontrolü
                showPaywall()
                return@launch
            }

            // CONCERN 4: Analytics (Cross-cutting Concern)
            FirebaseAnalytics.getInstance(this@ArticleActivity)
                .logEvent("article_viewed", Bundle().apply {
                    putString("article_id", article?.id)
                })

            // CONCERN 5: UI Rendering (Presentation Layer)
            titleTextView.text = article?.title
            contentTextView.text = article?.content
        }
    }
}
```

**Sorunlar:**

1. **5 farklı concern bir sınıfta**
   - Network configuration
   - Data fetching
   - Business logic (premium kontrolü)
   - Analytics
   - UI rendering

2. **Test edilemez**
   - Activity mock'lanamaz
   - Her concern ayrı test edilemez

3. **Değişiklik riski**
   - Analytics servisi değişirse → Activity değişir
   - API endpoint değişirse → Activity değişir
   - Premium logic değişirse → Activity değişir

4. **Kod tekrarı**
   - Her ekranda aynı Retrofit setup
   - Her ekranda aynı analytics kodu

5. **Tight coupling**
   - Activity Retrofit'e, Firebase'e sıkı bağımlı
   - Bir şey değişince her şey değişir

---

## ✅ Doğru Örnek (Separation of Concerns)

### Görsel Katman Ayrımı

```
Activity (UI Concern)
    │
    ├─► ViewModel (UI State Concern)
    │       │
    │       ├─► GetArticleUseCase (Business Logic Concern)
    │       │       │
    │       │       └─► ArticleRepository Interface (Domain)
    │       │               │
    │       │               └─► ArticleRepositoryImpl (Data Concern)
    │       │                       │
    │       │                       ├─► NewsApiService (Network)
    │       │                       └─► ArticleDao (Database)
    │       │
    │       └─► LogArticleViewUseCase (Analytics Concern)
    │               │
    │               └─► AnalyticsService Interface (Domain)
    │                       │
    │                       └─► FirebaseAnalyticsService (Data)
```

### Kod İmplementasyonu

```kotlin
// ==========================================
// DATA LAYER - Data Concern
// ==========================================

// API Service - Sadece network işlemi
interface NewsApiService {
    @GET("articles/{id}")
    suspend fun getArticle(@Path("id") id: String): ArticleDto
}

// DTO (Data Transfer Object)
data class ArticleDto(
    val id: String,
    val title: String,
    val content: String,
    val isPremium: Boolean
)

// Repository Implementation - Sadece veri kaynağı yönetimi
class ArticleRepositoryImpl(
    private val apiService: NewsApiService,
    private val articleDao: ArticleDao
) : ArticleRepository {

    override suspend fun getArticle(id: String): Result<Article> {
        return try {
            // Remote'tan al
            val dto = apiService.getArticle(id)
            val article = dto.toDomain()

            // Local'e kaydet (caching)
            articleDao.insert(article.toEntity())

            Result.success(article)
        } catch (e: Exception) {
            // Hata olursa local'den dene
            articleDao.getArticle(id)?.let {
                Result.success(it.toDomain())
            } ?: Result.failure(e)
        }
    }
}

// ==========================================
// DOMAIN LAYER - Business Logic Concern
// ==========================================

// Repository Interface (Domain'de tanımlı!)
interface ArticleRepository {
    suspend fun getArticle(id: String): Result<Article>
}

// Domain Model (Framework'den bağımsız)
data class Article(
    val id: String,
    val title: String,
    val content: String,
    val isPremium: Boolean
)

// Use Case - Business Logic (Premium kontrolü)
class GetArticleUseCase(
    private val articleRepository: ArticleRepository,
    private val userRepository: UserRepository
) {
    suspend operator fun invoke(articleId: String): Result<ArticleViewState> {
        return articleRepository.getArticle(articleId).map { article ->
            val user = userRepository.getCurrentUser()

            // Business Logic: Premium kontrolü
            if (article.isPremium && !user.isSubscribed) {
                ArticleViewState.PaywallRequired(article)
            } else {
                ArticleViewState.Success(article)
            }
        }
    }
}

// Sealed class - Domain'de tanımlı
sealed class ArticleViewState {
    data class Success(val article: Article) : ArticleViewState()
    data class PaywallRequired(val article: Article) : ArticleViewState()
}

// ==========================================
// ANALYTICS CONCERN (Cross-cutting)
// ==========================================

// Analytics Interface (Domain'de tanımlı)
interface AnalyticsService {
    fun logEvent(eventName: String, params: Map<String, Any>)
}

// Analytics Implementation (Data'da)
class FirebaseAnalyticsService(
    private val analytics: FirebaseAnalytics
) : AnalyticsService {
    override fun logEvent(eventName: String, params: Map<String, Any>) {
        analytics.logEvent(eventName, Bundle().apply {
            params.forEach { (key, value) ->
                putString(key, value.toString())
            }
        })
    }
}

// Analytics Use Case (Domain'de)
class LogArticleViewUseCase(
    private val analyticsService: AnalyticsService
) {
    operator fun invoke(articleId: String) {
        analyticsService.logEvent(
            "article_viewed",
            mapOf("article_id" to articleId)
        )
    }
}

// ==========================================
// PRESENTATION LAYER - UI Concern
// ==========================================

// ViewModel - UI State yönetimi
class ArticleViewModel(
    private val getArticleUseCase: GetArticleUseCase,
    private val logArticleViewUseCase: LogArticleViewUseCase
) : ViewModel() {

    private val _uiState = MutableStateFlow<ArticleUiState>(ArticleUiState.Loading)
    val uiState: StateFlow<ArticleUiState> = _uiState.asStateFlow()

    fun loadArticle(articleId: String) {
        viewModelScope.launch {
            _uiState.value = ArticleUiState.Loading

            getArticleUseCase(articleId).fold(
                onSuccess = { viewState ->
                    when (viewState) {
                        is ArticleViewState.Success -> {
                            _uiState.value = ArticleUiState.Success(viewState.article)
                            logArticleViewUseCase(articleId) // Analytics
                        }
                        is ArticleViewState.PaywallRequired -> {
                            _uiState.value = ArticleUiState.PaywallRequired
                        }
                    }
                },
                onFailure = { error ->
                    _uiState.value = ArticleUiState.Error(error.message ?: "Unknown error")
                }
            )
        }
    }
}

// UI State
sealed class ArticleUiState {
    object Loading : ArticleUiState()
    data class Success(val article: Article) : ArticleUiState()
    object PaywallRequired : ArticleUiState()
    data class Error(val message: String) : ArticleUiState()
}

// Activity - Sadece UI render
class ArticleActivity : AppCompatActivity() {
    private val viewModel: ArticleViewModel by viewModels()

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_article)

        // Sadece UI concern
        lifecycleScope.launch {
            viewModel.uiState.collect { state ->
                when (state) {
                    is ArticleUiState.Loading -> showLoading()
                    is ArticleUiState.Success -> showArticle(state.article)
                    is ArticleUiState.PaywallRequired -> showPaywall()
                    is ArticleUiState.Error -> showError(state.message)
                }
            }
        }

        val articleId = intent.getStringExtra("ARTICLE_ID") ?: return
        viewModel.loadArticle(articleId)
    }

    private fun showArticle(article: Article) {
        titleTextView.text = article.title
        contentTextView.text = article.content
    }

    private fun showLoading() {
        progressBar.visibility = View.VISIBLE
    }

    private fun showPaywall() {
        // Paywall ekranına git
        startActivity(Intent(this, PaywallActivity::class.java))
    }

    private fun showError(message: String) {
        Toast.makeText(this, message, Toast.LENGTH_SHORT).show()
    }
}
```

**Artık her katmanın tek bir concern'i var:**

| Katman/Sınıf | Concern | Sorumluluk |
|--------------|---------|------------|
| `NewsApiService` | Data concern | API çağrısı |
| `ArticleRepositoryImpl` | Data concern | Veri kaynağı yönetimi |
| `ArticleRepository` (Interface) | Domain concern | Veri soyutlaması |
| `GetArticleUseCase` | Business logic concern | Premium kontrolü |
| `LogArticleViewUseCase` | Analytics concern | Event tracking |
| `ArticleViewModel` | UI state concern | UI state yönetimi |
| `ArticleActivity` | UI rendering concern | Ekranda gösterim |

---

## Avantajlar

### 1. Test Edilebilirlik
Her concern ayrı test edilir:

```kotlin
// Business logic test (Domain)
@Test
fun `premium article requires subscription`() {
    val useCase = GetArticleUseCase(mockArticleRepo, mockUserRepo)
    // ...
}

// Data layer test
@Test
fun `repository returns cached data when network fails`() {
    val repository = ArticleRepositoryImpl(mockApi, mockDao)
    // ...
}

// ViewModel test (Presentation)
@Test
fun `loadArticle shows paywall for premium content`() {
    val viewModel = ArticleViewModel(mockGetArticleUseCase, mockLogUseCase)
    // ...
}
```

### 2. Değişiklik İzolasyonu
Bir concern değiştiğinde sadece ilgili katman değişir:

- **Analytics servisi değişir** (Firebase → Mixpanel) → Sadece `FirebaseAnalyticsService` değişir
- **API endpoint değişir** → Sadece `NewsApiService` değişir
- **Premium logic değişir** → Sadece `GetArticleUseCase` değişir
- **UI tasarımı değişir** → Sadece `ArticleActivity` değişir

### 3. Kod Tekrarı Azalır (Reusability)
Her concern ayrı modül olduğu için yeniden kullanılabilir:

```kotlin
// Analytics her yerde kullanılabilir
class ProductViewModel(
    private val logProductViewUseCase: LogProductViewUseCase // Aynı analytics concern
)

class ProfileViewModel(
    private val logProfileViewUseCase: LogProfileViewUseCase // Aynı analytics concern
)
```

### 4. Parallel Development
Takım üyeleri farklı concern'lerde çalışabilir:

- **Backend Dev**: Data layer (Repository, API)
- **Android Dev 1**: Presentation layer (UI, ViewModel)
- **Android Dev 2**: Domain layer (Use Cases, Business Logic)
- **Analytics Dev**: Analytics concern

### 5. Framework Bağımsızlığı
Domain layer pure Kotlin olabilir:

- Firebase → Mixpanel geçişi kolay
- Retrofit → Ktor geçişi kolay
- Room → SQLDelight geçişi kolay

---

## Cross-cutting Concerns

Bazı concern'ler **tüm katmanlarda** kullanılır. Bunlara **cross-cutting concerns** denir.

### 1. Analytics Concern

**Sorun:** Her katmanda analytics kodu mu yazacağız?

❌ **Yanlış:**
```kotlin
class ArticleViewModel {
    fun loadArticle() {
        FirebaseAnalytics.getInstance().logEvent(...) // ❌ Direct dependency
    }
}
```

✅ **Doğru:**
```kotlin
// Analytics'i ayrı bir concern olarak tanımla
interface AnalyticsService {
    fun logEvent(eventName: String, params: Map<String, Any>)
}

// Use Case ile sar
class LogArticleViewUseCase(
    private val analyticsService: AnalyticsService
) {
    operator fun invoke(articleId: String) {
        analyticsService.logEvent("article_viewed", mapOf("article_id" to articleId))
    }
}

// ViewModel'de kullan
class ArticleViewModel(
    private val logArticleViewUseCase: LogArticleViewUseCase
) {
    fun loadArticle() {
        logArticleViewUseCase(articleId) // ✅ Clean
    }
}
```

### 2. Logging Concern

```kotlin
// Logging interface (Domain'de)
interface Logger {
    fun d(tag: String, message: String)
    fun e(tag: String, message: String, throwable: Throwable?)
}

// Implementation (Data'da)
class TimberLogger : Logger {
    override fun d(tag: String, message: String) {
        Timber.tag(tag).d(message)
    }

    override fun e(tag: String, message: String, throwable: Throwable?) {
        Timber.tag(tag).e(throwable, message)
    }
}

// Kullanım
class ArticleRepositoryImpl(
    private val apiService: NewsApiService,
    private val logger: Logger
) : ArticleRepository {
    override suspend fun getArticle(id: String): Result<Article> {
        logger.d("ArticleRepository", "Fetching article: $id")
        // ...
    }
}
```

### 3. Authentication Concern

```kotlin
// Auth interface (Domain'de)
interface AuthRepository {
    suspend fun isLoggedIn(): Boolean
    suspend fun getAccessToken(): String?
    suspend fun login(email: String, password: String): Result<User>
}

// Implementation (Data'da)
class AuthRepositoryImpl(
    private val firebaseAuth: FirebaseAuth,
    private val tokenManager: TokenManager
) : AuthRepository {
    override suspend fun isLoggedIn(): Boolean {
        return firebaseAuth.currentUser != null
    }

    override suspend fun getAccessToken(): String? {
        return tokenManager.getToken()
    }
}

// Kullanım (Use Case'te)
class GetArticleUseCase(
    private val articleRepository: ArticleRepository,
    private val authRepository: AuthRepository
) {
    suspend operator fun invoke(articleId: String): Result<Article> {
        // Auth concern
        if (!authRepository.isLoggedIn()) {
            return Result.failure(Exception("User not logged in"))
        }

        // Data concern
        return articleRepository.getArticle(articleId)
    }
}
```

---

## Ne Zaman Kullanılır?

### ✅ Kullan:

- **Orta/büyük projelerde** (10+ ekran)
- **Uzun ömürlü projelerde** (3+ yıl bakım)
- **Takım çalışmasında** (3+ developer)
- **Test yazıyorsan**
- **Clean Architecture kullanıyorsan** (zorunlu!)
- **Framework bağımsızlığı istiyorsan** (Retrofit → Ktor geçişi)

### ❌ Kullanma:

- **Tek ekranlı prototip**
- **Throw-away kod** (bir kere kullanılacak)
- **Çok basit uygulamalar** (TODO list, hesap makinesi)
- **Deadline çok sıkı** ve kod bir daha bakılmayacaksa

---

## Pratik Örnekler

### 1. Concern Karışmış Kod

```kotlin
// ❌ Yanlış - UI ve Business Logic karışmış
class LoginActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        loginButton.setOnClickListener {
            val email = emailEditText.text.toString()

            // Business Logic (Domain concern)
            if (!email.contains("@")) {
                // UI Logic (Presentation concern)
                emailEditText.error = "Invalid email"
                return@setOnClickListener
            }

            // Data Logic (Data concern)
            firebaseAuth.signInWithEmailAndPassword(email, password)
        }
    }
}
```

### 2. Concern Ayrılmış Kod

```kotlin
// ✅ Doğru - Her concern ayrı

// DOMAIN LAYER
class ValidateEmailUseCase {
    operator fun invoke(email: String): Boolean {
        return email.contains("@") && email.length > 3
    }
}

class LoginUseCase(
    private val authRepository: AuthRepository
) {
    suspend operator fun invoke(email: String, password: String): Result<User> {
        return authRepository.login(email, password)
    }
}

// PRESENTATION LAYER
class LoginViewModel(
    private val validateEmailUseCase: ValidateEmailUseCase,
    private val loginUseCase: LoginUseCase
) : ViewModel() {

    fun login(email: String, password: String) {
        // Business Logic
        if (!validateEmailUseCase(email)) {
            _uiState.value = LoginUiState.InvalidEmail
            return
        }

        viewModelScope.launch {
            // Data Logic (through use case)
            loginUseCase(email, password).fold(
                onSuccess = { _uiState.value = LoginUiState.Success },
                onFailure = { _uiState.value = LoginUiState.Error(it.message) }
            )
        }
    }
}

// UI LAYER
class LoginActivity : AppCompatActivity() {
    private val viewModel: LoginViewModel by viewModels()

    override fun onCreate(savedInstanceState: Bundle?) {
        loginButton.setOnClickListener {
            val email = emailEditText.text.toString()
            val password = passwordEditText.text.toString()

            viewModel.login(email, password) // Sadece UI concern
        }

        lifecycleScope.launch {
            viewModel.uiState.collect { state ->
                when (state) {
                    is LoginUiState.InvalidEmail -> showEmailError()
                    is LoginUiState.Success -> navigateToHome()
                    is LoginUiState.Error -> showError(state.message)
                }
            }
        }
    }
}
```

---

## SoC ve SOLID İlişkisi

Separation of Concerns, **SOLID prensiplerinin mimari seviyede uygulanmış halidir**.

| SOLID Prensibi | SoC'de Karşılığı |
|----------------|------------------|
| **Single Responsibility** | Her katman tek concern'den sorumlu |
| **Open/Closed** | Katmanlar genişletilebilir (yeni use case ekle, mevcut kodu değiştirme) |
| **Liskov Substitution** | Repository implementation'ları değiştirilebilir (Network/Cache/Mock) |
| **Interface Segregation** | Her katman sadece ihtiyacı olan interface'i kullanır |
| **Dependency Inversion** | Üst katmanlar alt katmanlara bağımlı değil, interface'e bağımlı |

---

## Özet

### Temel Prensip
> **"Her katman sadece bir concern'den sorumlu olmalı"**

### Clean Architecture Katmanları

```
Presentation → UI Concern (Ne gösterileceği)
     │
     ▼
Domain → Business Logic Concern (Ne yapılacağı)
     │
     ▼
Data → Data Access Concern (Nasıl yapılacağı)
```

### Altın Kurallar

1. **Katman sorumluluklarını karıştırma**
   - UI'da business logic yazma
   - ViewModel'de Retrofit çağrısı yapma
   - Repository'de UI logic yazma

2. **Bağımlılık yönüne dikkat et**
   - Üst katman → Alt katman (Presentation → Domain → Data)
   - Alt katman üst katmanı bilmemeli

3. **Cross-cutting concern'leri ayır**
   - Analytics, Logging, Auth ayrı modüller

4. **Test edilebilirliği düşün**
   - Her concern ayrı test edilebilmeli

---

## Kaynaklar

- [Clean Architecture - Robert C. Martin](https://www.amazon.com/Clean-Architecture-Craftsmans-Software-Structure/dp/0134494164)
- [Android Architecture Guide - Google](https://developer.android.com/topic/architecture)
- [Guide to app architecture - Android Developers](https://developer.android.com/topic/architecture)
- [Separation of Concerns - Wikipedia](https://en.wikipedia.org/wiki/Separation_of_concerns)
