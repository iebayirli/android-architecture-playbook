### 4. Clean Architecture

#### Tanım

Clean Architecture, Uncle Bob (Robert C. Martin) tarafından önerilmiş,
katmanlar arası bağımlılığı minimize eden ve iş mantığını framework'lerden
ayıran bir mimari yaklaşımdır. Temel prensibi **Dependency Rule**'dur:
içteki katmanlar dıştaki katmanları bilmez. Clean Architecture, MVVM gibi
presentation pattern'lerle birlikte kullanılır.

#### Android'de Yapısı

Clean Architecture'da **3 ana katman** vardır:

1. **Presentation (UI) Layer**

   - Activity, Fragment, Jetpack Compose
   - ViewModel
   - UI State
   - **Bağımlılık:** Domain layer'ı bilir

2. **Domain Layer** (Business Logic)

   - Use Cases (Interactors)
   - Domain Models (Business entities)
   - Repository Interfaces
   - **Bağımlılık:** Hiçbir katmanı bilmez (Pure Kotlin/Java)

3. **Data Layer**
   - Repository Implementations
   - Data Sources (API, Database, Cache)
   - DTOs (Data Transfer Objects)
   - **Bağımlılık:** Domain layer'ı bilir (interface'leri implement eder)

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

**Önemli:** Domain layer hiçbir katmanı bilmez. Sadece interface'ler tanımlar.

#### Kod Örneği

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

#### Avantajları

- **Testability (Test Edilebilirlik):** Her katman bağımsız test edilebilir.
- **Separation of Concerns:** Her katmanın tek sorumluluğu var.
- **Independence:** Domain layer framework'den bağımsız (Pure Kotlin).
- **Flexibility:** Data source değişikliği kolay (API → Database).
- **Reusability:** Use case'ler farklı UI'larda kullanılabilir.
- **Scalability:** Büyük projelerde yönetilebilir.

#### Dezavantajları

- **Complexity:** Küçük projeler için overkill.
- **Boilerplate:** Çok fazla interface ve class.
- **Learning Curve:** Öğrenme eğrisi yüksek.
- **Initial Setup:** Başlangıç kurulumu zaman alır.
- **Over-engineering riski:** Basit işler için fazla katman.

#### Multi-Module Yapısı

Modern Clean Architecture projelerinde **Gradle modülleri** kullanılır:

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

**Gradle Bağımlılıkları:**

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

#### Kullanım Senaryoları

- **Büyük ölçekli projeler** (10+ ekran)
- **Uzun ömürlü projeler** (3+ yıl bakım)
- **Takım çalışması** (5+ developer)
- **Test edilebilirlik kritikse**
- **Framework değişikliği olasılığı varsa** (örn: Retrofit → Ktor)
- **Modern Android best practices** (Google önerir)

---
