### 4. Clean Architecture

#### Definition

Clean Architecture is an architectural approach proposed by Uncle Bob (Robert C. Martin) that minimizes dependencies between layers and separates business logic from frameworks. Its fundamental principle is the **Dependency Rule**: inner layers don't know about outer layers. Clean Architecture is used together with presentation patterns like MVVM.

#### Structure in Android

Clean Architecture has **3 main layers**:

1. **Presentation (UI) Layer**

   - Activity, Fragment, Jetpack Compose
   - ViewModel
   - UI State
   - **Dependency:** Knows the Domain layer

2. **Domain Layer** (Business Logic)

   - Use Cases (Interactors)
   - Domain Models (Business entities)
   - Repository Interfaces
   - **Dependency:** Knows no layer (Pure Kotlin/Java)

3. **Data Layer**
   - Repository Implementations
   - Data Sources (API, Database, Cache)
   - DTOs (Data Transfer Objects)
   - **Dependency:** Knows the Domain layer (implements interfaces)

#### Dependency Rule

```
┌─────────────────────────────────────┐
│   Presentation (UI/App Module)     │
│   ├─ ViewModel                      │
│   ├─ Activity/Fragment              │
│   └─ Compose UI                     │
└──────────────┬──────────────────────┘
               │ Depends on (→)
               ▼
┌─────────────────────────────────────┐
│   Domain (Business Logic)           │
│   ├─ Use Cases                      │
│   ├─ Domain Models                  │
│   └─ Repository Interfaces          │
└─────────────────────────────────────┘
               ▲
               │ Implements (←)
               │
┌──────────────┴──────────────────────┐
│   Data (Data Sources)               │
│   ├─ Repository Implementations     │
│   ├─ API (Retrofit)                 │
│   └─ Database (Room)                │
└─────────────────────────────────────┘
```

**Important:** The Domain layer knows no layer. It only defines interfaces.

#### Code Example

```kotlin
// ====================================
// DOMAIN LAYER (Pure Kotlin)
// ====================================

// Domain Model
data class User(
    val id: String,
    val name: String,
    val email: String
)

// Repository Interface (Domain'de tanımlanır)
interface UserRepository {
    suspend fun getUser(id: String): Result<User>
}

// Use Case
class GetUserUseCase(
    private val repository: UserRepository // Interface'e bağımlı
) {
    suspend operator fun invoke(userId: String): Result<User> {
        // Business logic burada
        return repository.getUser(userId)
    }
}

// ====================================
// DATA LAYER
// ====================================

// DTO (API response)
data class UserDto(
    val id: String,
    val name: String,
    val email: String
)

// API Service
interface ApiService {
    @GET("users/{id}")
    suspend fun getUser(@Path("id") id: String): UserDto
}

// Repository Implementation (Domain'deki interface'i implement eder)
class UserRepositoryImpl(
    private val apiService: ApiService
) : UserRepository {
    override suspend fun getUser(id: String): Result<User> {
        return try {
            val dto = apiService.getUser(id)
            // DTO → Domain Model mapping
            val user = User(
                id = dto.id,
                name = dto.name,
                email = dto.email
            )
            Result.success(user)
        } catch (e: Exception) {
            Result.failure(e)
        }
    }
}

// ====================================
// PRESENTATION LAYER
// ====================================

// UI State
data class UserUiState(
    val isLoading: Boolean = false,
    val user: User? = null,
    val error: String? = null
)

// ViewModel
class UserViewModel(
    private val getUserUseCase: GetUserUseCase // Use Case'e bağımlı
) : ViewModel() {

    private val _uiState = MutableStateFlow(UserUiState())
    val uiState: StateFlow<UserUiState> = _uiState.asStateFlow()

    fun loadUser(userId: String) {
        viewModelScope.launch {
            _uiState.value = UserUiState(isLoading = true)

            getUserUseCase(userId).fold(
                onSuccess = { user ->
                    _uiState.value = UserUiState(user = user)
                },
                onFailure = { error ->
                    _uiState.value = UserUiState(error = error.message)
                }
            )
        }
    }
}

// Activity
class UserActivity : AppCompatActivity() {
    private val viewModel: UserViewModel by viewModels()

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        lifecycleScope.launch {
            viewModel.uiState.collect { state ->
                when {
                    state.isLoading -> showLoading()
                    state.user != null -> showUser(state.user)
                    state.error != null -> showError(state.error)
                }
            }
        }

        viewModel.loadUser("123")
    }
}

// ====================================
// DEPENDENCY INJECTION (Hilt)
// ====================================

@Module
@InstallIn(SingletonComponent::class)
object DataModule {

    @Provides
    @Singleton
    fun provideApiService(): ApiService {
        return Retrofit.Builder()
            .baseUrl("https://api.example.com")
            .build()
            .create(ApiService::class.java)
    }

    @Provides
    @Singleton
    fun provideUserRepository(
        apiService: ApiService
    ): UserRepository {
        return UserRepositoryImpl(apiService)
    }
}

@Module
@InstallIn(ViewModelComponent::class)
object DomainModule {

    @Provides
    fun provideGetUserUseCase(
        repository: UserRepository
    ): GetUserUseCase {
        return GetUserUseCase(repository)
    }
}
```

#### Advantages

- **Testability:** Each layer can be tested independently.
- **Separation of Concerns:** Each layer has a single responsibility.
- **Independence:** Domain layer is framework-independent (Pure Kotlin).
- **Flexibility:** Data source changes are easy (API → Database).
- **Reusability:** Use cases can be used in different UIs.
- **Scalability:** Manageable in large projects.

#### Disadvantages

- **Complexity:** Overkill for small projects.
- **Boilerplate:** Too many interfaces and classes.
- **Learning Curve:** High learning curve.
- **Initial Setup:** Initial setup takes time.
- **Over-engineering risk:** Too many layers for simple tasks.

#### Multi-Module Structure

Modern Clean Architecture projects use **Gradle modules**:

```
project/
├── app/ (Presentation)
│   ├── Activity, Fragment
│   ├── ViewModel
│   └── DI Modules
├── domain/ (Business Logic)
│   ├── Use Cases
│   ├── Models
│   └── Repository Interfaces
└── data/ (Data Sources)
    ├── Repository Implementations
    ├── API
    └── Database
```

**Gradle Dependencies:**

```gradle
// app/build.gradle
dependencies {
    implementation(project(":domain"))
    // app, data'yı direkt bilmez, sadece domain'i bilir
}

// data/build.gradle
dependencies {
    implementation(project(":domain"))
    // data, domain'deki interface'leri implement eder
}

// domain/build.gradle
dependencies {
    // Domain hiçbir modüle bağımlı değil (Pure Kotlin)
}
```

#### Use Cases

- **Large-scale projects** (10+ screens)
- **Long-lived projects** (3+ years maintenance)
- **Team work** (5+ developers)
- **When testability is critical**
- **When framework changes are likely** (e.g., Retrofit → Ktor)
- **Modern Android best practices** (Recommended by Google)

---
