# Separation of Concerns (SoC)

## Introduction

**Separation of Concerns (SoC)** is the principle of separating software into different areas of responsibility (concerns). Each module or layer should be responsible for only one "concern".

**Real-life analogy:**
- **Restaurant**: Waiter takes orders, chef cooks food, cashier handles billing
- Each person has **one single task**
- Chef doesn't handle billing, waiter doesn't cook

**In Android:**
- UI code → Only screen display
- Business logic → Only business rules
- Data code → Only data fetching/saving

> **Core Idea:** "Each module should be responsible for only one concern"

**Important Note:** SoC is the fundamental principle of Clean Architecture. It is the architectural-level application of SOLID principles.

---

## Difference with SRP (Single Responsibility)

| | SRP | SoC |
|---|-----|-----|
| **Level** | Class/module level | Architecture level |
| **Scope** | One class, single responsibility | One layer, single concern |
| **Question** | "What should this class do?" | "What should this layer do?" |
| **Example** | `ArticleRepository` only manages data | Data layer only handles data concern |
| **Relationship** | You use SRP to implement SoC | SoC is SRP's big brother |

**Simply:**
- **SRP**: Single responsibility at class level
- **SoC**: Separating layers at architecture level

---

## Types of Concerns in Android

In an Android application, there are different types of "concerns":

### 1. UI Rendering Concern (Presentation)
- Screen display
- User input handling
- Navigation
- **Example:** `Activity`, `Fragment`, `Composable`, `ViewModel`

### 2. Business Logic Concern (Domain)
- Business rules
- Validation
- Use case orchestration
- **Example:** `GetArticleUseCase`, `ValidateEmailUseCase`

### 3. Data Management Concern (Data)
- Network requests
- Database operations
- Caching strategy
- **Example:** `ArticleRepositoryImpl`, `ApiService`, `ArticleDao`

### 4. Analytics Concern (Cross-cutting)
- Event tracking
- User behavior
- Logging
- **Example:** `AnalyticsService`, `LogArticleViewUseCase`

### 5. Authentication Concern (Cross-cutting)
- Login/logout
- Token management
- Session handling
- **Example:** `AuthRepository`, `TokenManager`

### 6. Error Handling Concern (Cross-cutting)
- Exception handling
- User-friendly error messages
- Retry logic
- **Example:** `ErrorMapper`, `Result<T>` pattern

### 7. Configuration Concern (Cross-cutting)
- Remote config
- Feature flags
- A/B testing
- **Example:** `RemoteConfigService`, `FeatureFlagManager`

**Note:** Cross-cutting concerns can exist in every layer but should be **separate modules**.

---

## Separation of Concerns in Clean Architecture

Clean Architecture is the architectural-level application of the SoC principle.

### Layers and Their Responsibilities

```
┌─────────────────────────────────────────────────┐
│   PRESENTATION LAYER (UI Concern)               │
│   - Activity/Fragment/Composable                │
│   - ViewModel                                   │
│   - UI State                                    │
│   Responsibility: Display to user, take input   │
└────────────────────┬────────────────────────────┘
                     │ Uses (Dependency)
                     ▼
┌─────────────────────────────────────────────────┐
│   DOMAIN LAYER (Business Logic Concern)         │
│   - Use Cases                                   │
│   - Domain Models                               │
│   - Repository Interfaces                       │
│   Responsibility: Answer "what to do"           │
└────────────────────┬────────────────────────────┘
                     │ Implements
                     ▼
┌─────────────────────────────────────────────────┐
│   DATA LAYER (Data Access Concern)              │
│   - Repository Implementations                  │
│   - API Service (Retrofit/Ktor)                 │
│   - Database DAO (Room)                         │
│   Responsibility: Answer "how to do"            │
└─────────────────────────────────────────────────┘

        ┌─────────────────────────────────┐
        │   CROSS-CUTTING CONCERNS        │
        │   - Analytics                   │
        │   - Logging                     │
        │   - Authentication              │
        └─────────────────────────────────┘
```

| Layer | Concern | Responsibility | Example |
|--------|---------|------------|-------|
| **Presentation** | UI concern | Display to user, take input | `ArticleActivity`, `ArticleViewModel` |
| **Domain** | Business logic concern | Business rules, validation, orchestration | `GetArticleUseCase`, `Article` (model) |
| **Data** | Data access concern | Data fetching/saving (how?) | `ArticleRepositoryImpl`, `ApiService`, `ArticleDao` |

---

## ❌ Wrong Example (SoC Violation)

```kotlin
// God Object Anti-Pattern - All concerns together
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
                // Premium check
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

**Problems:**

1. **5 different concerns in one class**
   - Network configuration
   - Data fetching
   - Business logic (premium check)
   - Analytics
   - UI rendering

2. **Not testable**
   - Activity cannot be mocked
   - Each concern cannot be tested separately

3. **Change risk**
   - Analytics service changes → Activity changes
   - API endpoint changes → Activity changes
   - Premium logic changes → Activity changes

4. **Code duplication**
   - Same Retrofit setup on every screen
   - Same analytics code on every screen

5. **Tight coupling**
   - Activity tightly coupled to Retrofit, Firebase
   - When one thing changes, everything changes

---

## ✅ Correct Example (Separation of Concerns)

### Visual Layer Separation

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

### Code Implementation

```kotlin
// ==========================================
// DATA LAYER - Data Concern
// ==========================================

// API Service - Only network operation
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

// Repository Implementation - Only data source management
class ArticleRepositoryImpl(
    private val apiService: NewsApiService,
    private val articleDao: ArticleDao
) : ArticleRepository {

    override suspend fun getArticle(id: String): Result<Article> {
        return try {
            // Fetch from remote
            val dto = apiService.getArticle(id)
            val article = dto.toDomain()

            // Save to local (caching)
            articleDao.insert(article.toEntity())

            Result.success(article)
        } catch (e: Exception) {
            // If error, try from local
            articleDao.getArticle(id)?.let {
                Result.success(it.toDomain())
            } ?: Result.failure(e)
        }
    }
}

// ==========================================
// DOMAIN LAYER - Business Logic Concern
// ==========================================

// Repository Interface (Defined in Domain!)
interface ArticleRepository {
    suspend fun getArticle(id: String): Result<Article>
}

// Domain Model (Framework independent)
data class Article(
    val id: String,
    val title: String,
    val content: String,
    val isPremium: Boolean
)

// Use Case - Business Logic (Premium check)
class GetArticleUseCase(
    private val articleRepository: ArticleRepository,
    private val userRepository: UserRepository
) {
    suspend operator fun invoke(articleId: String): Result<ArticleViewState> {
        return articleRepository.getArticle(articleId).map { article ->
            val user = userRepository.getCurrentUser()

            // Business Logic: Premium check
            if (article.isPremium && !user.isSubscribed) {
                ArticleViewState.PaywallRequired(article)
            } else {
                ArticleViewState.Success(article)
            }
        }
    }
}

// Sealed class - Defined in Domain
sealed class ArticleViewState {
    data class Success(val article: Article) : ArticleViewState()
    data class PaywallRequired(val article: Article) : ArticleViewState()
}

// ==========================================
// ANALYTICS CONCERN (Cross-cutting)
// ==========================================

// Analytics Interface (Defined in Domain)
interface AnalyticsService {
    fun logEvent(eventName: String, params: Map<String, Any>)
}

// Analytics Implementation (In Data)
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

// Analytics Use Case (In Domain)
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

// ViewModel - UI State management
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

// Activity - Only UI render
class ArticleActivity : AppCompatActivity() {
    private val viewModel: ArticleViewModel by viewModels()

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_article)

        // Only UI concern
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
        // Navigate to paywall screen
        startActivity(Intent(this, PaywallActivity::class.java))
    }

    private fun showError(message: String) {
        Toast.makeText(this, message, Toast.LENGTH_SHORT).show()
    }
}
```

**Now each layer has a single concern:**

| Layer/Class | Concern | Responsibility |
|--------------|---------|------------|
| `NewsApiService` | Data concern | API calls |
| `ArticleRepositoryImpl` | Data concern | Data source management |
| `ArticleRepository` (Interface) | Domain concern | Data abstraction |
| `GetArticleUseCase` | Business logic concern | Premium check |
| `LogArticleViewUseCase` | Analytics concern | Event tracking |
| `ArticleViewModel` | UI state concern | UI state management |
| `ArticleActivity` | UI rendering concern | Screen display |

---

## Advantages

### 1. Testability
Each concern is tested separately:

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

### 2. Change Isolation
When a concern changes, only the relevant layer changes:

- **Analytics service changes** (Firebase → Mixpanel) → Only `FirebaseAnalyticsService` changes
- **API endpoint changes** → Only `NewsApiService` changes
- **Premium logic changes** → Only `GetArticleUseCase` changes
- **UI design changes** → Only `ArticleActivity` changes

### 3. Reduced Code Duplication (Reusability)
Since each concern is a separate module, it can be reused:

```kotlin
// Analytics can be used everywhere
class ProductViewModel(
    private val logProductViewUseCase: LogProductViewUseCase // Same analytics concern
)

class ProfileViewModel(
    private val logProfileViewUseCase: LogProfileViewUseCase // Same analytics concern
)
```

### 4. Parallel Development
Team members can work on different concerns:

- **Backend Dev**: Data layer (Repository, API)
- **Android Dev 1**: Presentation layer (UI, ViewModel)
- **Android Dev 2**: Domain layer (Use Cases, Business Logic)
- **Analytics Dev**: Analytics concern

### 5. Framework Independence
Domain layer can be pure Kotlin:

- Firebase → Mixpanel migration is easy
- Retrofit → Ktor migration is easy
- Room → SQLDelight migration is easy

---

## Cross-cutting Concerns

Some concerns are used **across all layers**. These are called **cross-cutting concerns**.

### 1. Analytics Concern

**Problem:** Do we write analytics code in every layer?

❌ **Wrong:**
```kotlin
class ArticleViewModel {
    fun loadArticle() {
        FirebaseAnalytics.getInstance().logEvent(...) // ❌ Direct dependency
    }
}
```

✅ **Correct:**
```kotlin
// Define analytics as a separate concern
interface AnalyticsService {
    fun logEvent(eventName: String, params: Map<String, Any>)
}

// Wrap with Use Case
class LogArticleViewUseCase(
    private val analyticsService: AnalyticsService
) {
    operator fun invoke(articleId: String) {
        analyticsService.logEvent("article_viewed", mapOf("article_id" to articleId))
    }
}

// Use in ViewModel
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
// Logging interface (In Domain)
interface Logger {
    fun d(tag: String, message: String)
    fun e(tag: String, message: String, throwable: Throwable?)
}

// Implementation (In Data)
class TimberLogger : Logger {
    override fun d(tag: String, message: String) {
        Timber.tag(tag).d(message)
    }

    override fun e(tag: String, message: String, throwable: Throwable?) {
        Timber.tag(tag).e(throwable, message)
    }
}

// Usage
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
// Auth interface (In Domain)
interface AuthRepository {
    suspend fun isLoggedIn(): Boolean
    suspend fun getAccessToken(): String?
    suspend fun login(email: String, password: String): Result<User>
}

// Implementation (In Data)
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

// Usage (In Use Case)
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

## When to Use?

### ✅ Use:

- **Medium/large projects** (10+ screens)
- **Long-lived projects** (3+ years maintenance)
- **Team work** (3+ developers)
- **If you're writing tests**
- **If you're using Clean Architecture** (mandatory!)
- **If you want framework independence** (Retrofit → Ktor migration)

### ❌ Don't Use:

- **Single-screen prototype**
- **Throw-away code** (one-time use)
- **Very simple applications** (TODO list, calculator)
- **Very tight deadline** and code won't be revisited

---

## Practical Examples

### 1. Mixed Concerns Code

```kotlin
// ❌ Wrong - UI and Business Logic mixed
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

### 2. Separated Concerns Code

```kotlin
// ✅ Correct - Each concern separated

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

            viewModel.login(email, password) // Only UI concern
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

## SoC and SOLID Relationship

Separation of Concerns is **SOLID principles applied at the architectural level**.

| SOLID Principle | SoC Counterpart |
|----------------|------------------|
| **Single Responsibility** | Each layer is responsible for a single concern |
| **Open/Closed** | Layers are extensible (add new use case, don't modify existing code) |
| **Liskov Substitution** | Repository implementations are interchangeable (Network/Cache/Mock) |
| **Interface Segregation** | Each layer uses only the interface it needs |
| **Dependency Inversion** | Upper layers don't depend on lower layers, they depend on interfaces |

---

## Summary

### Core Principle
> **"Each layer should be responsible for only one concern"**

### Clean Architecture Layers

```
Presentation → UI Concern (What to display)
     │
     ▼
Domain → Business Logic Concern (What to do)
     │
     ▼
Data → Data Access Concern (How to do it)
```

### Golden Rules

1. **Don't mix layer responsibilities**
   - Don't write business logic in UI
   - Don't make Retrofit calls in ViewModel
   - Don't write UI logic in Repository

2. **Pay attention to dependency direction**
   - Upper layer → Lower layer (Presentation → Domain → Data)
   - Lower layer shouldn't know upper layer

3. **Separate cross-cutting concerns**
   - Analytics, Logging, Auth as separate modules

4. **Think about testability**
   - Each concern should be testable separately

---

## Resources

- [Clean Architecture - Robert C. Martin](https://www.amazon.com/Clean-Architecture-Craftsmans-Software-Structure/dp/0134494164)
- [Android Architecture Guide - Google](https://developer.android.com/topic/architecture)
- [Guide to app architecture - Android Developers](https://developer.android.com/topic/architecture)
- [Separation of Concerns - Wikipedia](https://en.wikipedia.org/wiki/Separation_of_concerns)
