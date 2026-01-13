# SOLID Principles

## Introduction

SOLID is **5 fundamental design principles** used to improve code quality in object-oriented programming. It was proposed by Robert C. Martin (Uncle Bob).

SOLID principles:

- **Make maintenance easier**
- Increase **testability**
- Make code **flexible to changes**
- Enable writing **readable and understandable** code

**Important Note:** SOLID principles are **tools**, not dogma. Blindly applying them everywhere leads to **over-engineering**. They should be used in the right place, at the right time.

---

## Table of Contents

1. **S** - Single Responsibility Principle
2. **O** - Open/Closed Principle
3. **L** - Liskov Substitution Principle
4. **I** - Interface Segregation Principle
5. **D** - Dependency Inversion Principle

---

## 1. Single Responsibility Principle (SRP)

### Definition

> **"A class or module should have only one reason to change"**

In other words: **A class should be responsible for only one thing.**

### What Does "Reason to Change" Mean?

**Reason to change** refers to different business requirements that would necessitate modifying a class.

For example, the `ArticleActivity` class in the wrong example below has **5 different reasons to change**:

- If the API endpoint changes → You need to change the network code
- If the analytics service changes (Firebase → Mixpanel) → You need to change the analytics code
- If the database schema changes → You need to change the database code
- If the UI design changes → You need to change the UI code
- If the data format changes → You need to change the parsing code

**Problem:** Every change touches the same class, leading to code conflicts and testing difficulties.

**Solution:** Separate class for each responsibility → Each class has only one reason to change.

### ❌ Wrong Example (News App)

```kotlin
// God Object Anti-Pattern - Her şeyi yapan bir sınıf
class ArticleActivity : AppCompatActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_article)

        // 1. Network isteği yapıyor
        val retrofit = Retrofit.Builder()
            .baseUrl("https://api.news.com")
            .build()
        val apiService = retrofit.create(ApiService::class.java)

        // 2. Veriyi işliyor
        lifecycleScope.launch {
            val response = apiService.getArticle("123")
            val article = response.body()

            // 3. Analytics gönderiyor
            FirebaseAnalytics.getInstance(this@ArticleActivity)
                .logEvent("article_viewed", Bundle().apply {
                    putString("article_id", article?.id)
                })

            // 4. Database'e kaydediyor
            val db = Room.databaseBuilder(
                applicationContext,
                AppDatabase::class.java,
                "news-db"
            ).build()
            db.articleDao().insert(article)

            // 5. UI güncelliyor
            titleTextView.text = article?.title
            contentTextView.text = article?.content
        }
    }
}
```

**Problems:**

- Activity is **responsible for 5 different tasks** (network, data processing, analytics, database, UI)
- Not testable (Activity cannot be mocked)
- When one task changes (e.g., analytics service changes), you need to change the Activity
- Code duplication (same operations on every screen)

### ✅ Correct Example (Clean Architecture + MVVM)

```kotlin
// ==========================================
// DATA LAYER - Sadece veri kaynağı yönetimi
// ==========================================

// API Service - Sadece network işlemi
interface NewsApiService {
    @GET("articles/{id}")
    suspend fun getArticle(@Path("id") id: String): ArticleDto
}

// Database DAO - Sadece veritabanı işlemi
@Dao
interface ArticleDao {
    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun insert(article: ArticleEntity)

    @Query("SELECT * FROM articles WHERE id = :id")
    suspend fun getArticle(id: String): ArticleEntity?
}

// Repository - Sadece veri kaynağı koordinasyonu
class ArticleRepositoryImpl(
    private val apiService: NewsApiService,
    private val articleDao: ArticleDao
) : ArticleRepository {
    override suspend fun getArticle(id: String): Result<Article> {
        return try {
            // Remote'tan al
            val dto = apiService.getArticle(id)
            val article = dto.toDomain()

            // Local'e kaydet
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
// DOMAIN LAYER - Sadece business logic
// ==========================================

// Use Case - Sadece bir iş mantığı
class GetArticleUseCase(
    private val repository: ArticleRepository
) {
    suspend operator fun invoke(articleId: String): Result<Article> {
        return repository.getArticle(articleId)
    }
}

// Analytics Use Case - Sadece analytics
class LogArticleViewUseCase(
    private val analyticsService: AnalyticsService
) {
    operator fun invoke(articleId: String) {
        analyticsService.logEvent("article_viewed", mapOf("article_id" to articleId))
    }
}

// ==========================================
// PRESENTATION LAYER - Sadece UI logic
// ==========================================

// ViewModel - Sadece UI state yönetimi
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
                onSuccess = { article ->
                    _uiState.value = ArticleUiState.Success(article)
                    logArticleViewUseCase(articleId)
                },
                onFailure = { error ->
                    _uiState.value = ArticleUiState.Error(error.message ?: "Unknown error")
                }
            )
        }
    }
}

// Activity - Sadece UI render
class ArticleActivity : AppCompatActivity() {
    private val viewModel: ArticleViewModel by viewModels()

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_article)

        lifecycleScope.launch {
            viewModel.uiState.collect { state ->
                when (state) {
                    is ArticleUiState.Loading -> showLoading()
                    is ArticleUiState.Success -> showArticle(state.article)
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
}
```

**Now each class has a single responsibility:**

- `NewsApiService`: Only API calls
- `ArticleDao`: Only database operations
- `ArticleRepository`: Only data source coordination
- `GetArticleUseCase`: Only article fetching business logic
- `LogArticleViewUseCase`: Only analytics
- `ArticleViewModel`: Only UI state management
- `ArticleActivity`: Only UI rendering

### Advantages

- Each class can be **independently tested**
- When changes are needed, **only the relevant class** changes
- Code duplication decreases (**reusability**)
- Readability increases

### When to Use?

- **Always!** SRP is the most fundamental principle
- Especially **critical in medium/large projects**
- Clean Architecture and MVVM are based on this principle

### When Not to Use?

- Very simple, single-screen prototypes
- Throw-away code (one-time use)

---

## 2. Open/Closed Principle (OCP)

### Definition

> **"Classes should be open for extension, closed for modification"**

When adding new features, we should **not modify existing code**, only **add new code**.

### ❌ Wrong Example (E-commerce - Payment System)

```kotlin
class PaymentProcessor {

    fun processPayment(paymentType: String, amount: Double): Result<Unit> {
        return when (paymentType) {
            "CREDIT_CARD" -> {
                // Kredi kartı işlemi
                println("Processing credit card payment: $amount")
                Result.success(Unit)
            }
            "PAYPAL" -> {
                // PayPal işlemi
                println("Processing PayPal payment: $amount")
                Result.success(Unit)
            }
            "GOOGLE_PAY" -> {
                // Google Pay işlemi
                println("Processing Google Pay payment: $amount")
                Result.success(Unit)
            }
            else -> {
                Result.failure(Exception("Unknown payment type"))
            }
        }
    }
}

// Kullanım
val processor = PaymentProcessor()
processor.processPayment("CREDIT_CARD", 100.0)
```

**Problem:** When a new payment method is added (e.g., Apple Pay), **you need to modify the `PaymentProcessor` class**.

```kotlin
// Apple Pay eklemek için mevcut kodu değiştiriyorsun ❌
"APPLE_PAY" -> {
    println("Processing Apple Pay payment: $amount")
    Result.success(Unit)
}
```

### ✅ Correct Example (With Polymorphism)

```kotlin
// 1. Interface tanımla (Soyutlama)
interface PaymentMethod {
    fun processPayment(amount: Double): Result<Unit>
}

// 2. Her ödeme tipi için ayrı implementation
class CreditCardPayment : PaymentMethod {
    override fun processPayment(amount: Double): Result<Unit> {
        println("Processing credit card payment: $amount")
        // Kredi kartı logic
        return Result.success(Unit)
    }
}

class PayPalPayment : PaymentMethod {
    override fun processPayment(amount: Double): Result<Unit> {
        println("Processing PayPal payment: $amount")
        // PayPal logic
        return Result.success(Unit)
    }
}

class GooglePayPayment : PaymentMethod {
    override fun processPayment(amount: Double): Result<Unit> {
        println("Processing Google Pay payment: $amount")
        // Google Pay logic
        return Result.success(Unit)
    }
}

// 3. Yeni ödeme yöntemi ekle (Mevcut kodu DEĞİŞTİRMEDEN)
class ApplePayPayment : PaymentMethod {
    override fun processPayment(amount: Double): Result<Unit> {
        println("Processing Apple Pay payment: $amount")
        // Apple Pay logic
        return Result.success(Unit)
    }
}

// 4. PaymentProcessor artık değişmiyor
class PaymentProcessor {
    fun processPayment(paymentMethod: PaymentMethod, amount: Double): Result<Unit> {
        return paymentMethod.processPayment(amount)
    }
}

// 5. Factory pattern ile ödeme metodunu al
// Factory Pattern: String tip alıp doğru implementation'ı döndürür
object PaymentMethodFactory {
    fun getPaymentMethod(type: String): PaymentMethod {
        // Not: Factory'de when kullanmak sorun değil!
        // Çünkü sadece BURASI değişiyor, PaymentProcessor değişmiyor ✅
        return when (type) {
            "CREDIT_CARD" -> CreditCardPayment()
            "PAYPAL" -> PayPalPayment()
            "GOOGLE_PAY" -> GooglePayPayment()
            "APPLE_PAY" -> ApplePayPayment() // Sadece buraya ekliyoruz
            else -> throw IllegalArgumentException("Unknown payment type")
        }
    }
}

// Kullanım
val paymentMethod = PaymentMethodFactory.getPaymentMethod("APPLE_PAY")
val processor = PaymentProcessor()
processor.processPayment(paymentMethod, 100.0)
```

**Now when adding a new payment method:**

- You write a new `ApplePayPayment` class
- You add it to the Factory
- **You don't modify the `PaymentProcessor` class!** ✅

### Advantages

- No risk of **breaking existing code** when adding new features
- **Regression risk decreases** since tested code doesn't change
- Extensible system

### When to Use?

- When new types are frequently added (payment methods, notification types)
- When building a plugin system
- 3rd party integrations (each customer has different implementation)
- When code will be extended by others

### When Not to Use?

- Types are **fixed** and rarely change → Use `when` (more practical)
- Very simple cases (2-3 types that won't change)
- **Avoid over-engineering!**

**Example:** Plug types in Android (Type2, CCS, CHAdeMO) almost never change → Using `when` makes more sense.

---

## 3. Liskov Substitution Principle (LSP)

### Definition

> **"Subclasses (child) should not exhibit unexpected behavior when used in place of the parent class"**

In other words: **I should be able to use a child class where a parent interface/class is expected, without the code breaking.**

### Simple Explanation

**Real-life example:**
- You defined a `drive()` method for cars
- Both Tesla and gasoline cars implement the `Car` interface
- Whichever car you use, `drive()` should work (shouldn't crash in one but work in the other)
- If `drive()` throws an exception in Tesla but not in gasoline cars → **LSP violation!**

**Android example:**
- There's a `UserDataSource` interface
- `NetworkDataSource` and `CacheDataSource` implement it
- I should be able to use both in the same place (one shouldn't throw an exception while the other doesn't)

### ❌ Wrong Example (Data Source - Cache vs Network)

```kotlin
// Interface tanımı
interface UserDataSource {
    fun getUser(id: String): User // Her zaman User dönüyor gibi görünüyor
}

// Network implementation
class NetworkUserDataSource(
    private val apiService: ApiService
) : UserDataSource {
    override fun getUser(id: String): User {
        // Network hatası olabilir ama exception fırlatıyor ❌
        return apiService.getUser(id) // Network hatası -> Exception!
    }
}

// Cache implementation
class CacheUserDataSource(
    private val cache: MutableMap<String, User>
) : UserDataSource {
    override fun getUser(id: String): User {
        // Cache boşsa exception fırlatıyor ❌
        return cache[id] ?: throw Exception("User not in cache!")
    }
}

// Kullanım - LSP İhlali!
fun displayUser(dataSource: UserDataSource, id: String) {
    try {
        val user = dataSource.getUser(id) // ❌ Her implementation ile PATLAMA riski!
        println("User: ${user.name}")
    } catch (e: Exception) {
        // Exception handling yapmak zorunda kalıyoruz
        println("Error: ${e.message}")
    }
}

// Kullanım
val networkSource = NetworkUserDataSource(apiService)
displayUser(networkSource, "123") // Network hatası -> PATLAR!

val cacheSource = CacheUserDataSource(mutableMapOf())
displayUser(cacheSource, "123") // Cache boş -> PATLAR!
```

**Problems:**

1. **Interface looks safe but isn't**: `getUser()` promises to always return `User` but can throw an exception
2. **Unexpected behavior**: Code crashes when you put a child class in place of the parent interface
3. **Try-catch mandatory**: You must do exception handling everywhere
4. **Not testable**: When mocking, you can't predict whether it will throw an exception

### ✅ Correct Example (With Result Pattern)

```kotlin
// Interface tanımı - Hata olabileceği açık ✅
interface UserDataSource {
    suspend fun getUser(id: String): Result<User> // Başarılı veya hatalı
}

// Network implementation
class NetworkUserDataSource(
    private val apiService: ApiService
) : UserDataSource {
    override suspend fun getUser(id: String): Result<User> {
        return try {
            val user = apiService.getUser(id)
            Result.success(user) // ✅ Başarılı
        } catch (e: Exception) {
            Result.failure(e) // ✅ Hatalı (exception fırlatmıyor!)
        }
    }
}

// Cache implementation
class CacheUserDataSource(
    private val cache: MutableMap<String, User>
) : UserDataSource {
    override suspend fun getUser(id: String): Result<User> {
        return cache[id]?.let {
            Result.success(it) // ✅ Cache'de var
        } ?: Result.failure(Exception("User not in cache")) // ✅ Cache'de yok
    }
}

// Mock implementation (Test için)
class MockUserDataSource(
    private val mockUser: User?
) : UserDataSource {
    override suspend fun getUser(id: String): Result<User> {
        return mockUser?.let {
            Result.success(it)
        } ?: Result.failure(Exception("Mock user not found"))
    }
}

// Kullanım - Artık her implementation ile güvenli çalışır ✅
suspend fun displayUser(dataSource: UserDataSource, id: String) {
    dataSource.getUser(id).fold(
        onSuccess = { user -> println("User: ${user.name}") },
        onFailure = { error -> println("Error: ${error.message}") }
    )
}

// Kullanım - HİÇ PATLAMAZ!
val networkSource = NetworkUserDataSource(apiService)
displayUser(networkSource, "123") // Network hatası -> Error mesajı yazar ✅

val cacheSource = CacheUserDataSource(mutableMapOf())
displayUser(cacheSource, "123") // Cache boş -> Error mesajı yazar ✅

val mockSource = MockUserDataSource(null)
displayUser(mockSource, "123") // Mock boş -> Error mesajı yazar ✅
```

**Now:**

- ✅ Every `UserDataSource` implementation **behaves the same way**
- ✅ None **throw exceptions**
- ✅ Can be used **safely** everywhere
- ✅ **Testable** (Mocks can be easily written)

### Advantages

- **Polymorphism is used safely** (every implementation behaves the same way)
- **Code is predictable** (whichever implementation comes, code doesn't crash)
- **Runtime exceptions** are eliminated
- **Testability** increases (every implementation adheres to the same contract)
- **Result pattern** makes error management clear and explicit

### When to Use?

- **Always!** when using interfaces or inheritance
- When there are multiple data sources (Network, Cache, Database, Mock)
- When using Repository Pattern
- When different implementations should behave the same way

### Practical Rules (For Android)

1. **No throwing exceptions:** Use `Result<T>`, `sealed class`, or nullable

   ```kotlin
   // ❌ Yanlış
   fun getData(): User // Exception fırlatabilir

   // ✅ Doğru
   suspend fun getData(): Result<User> // Hata olabileceği açık
   ```

2. **Return type must be consistent:** If parent returns `Result<T>`, child must also return `Result<T>`

   ```kotlin
   interface Repository {
       suspend fun getData(): Result<User>
   }

   class NetworkRepo : Repository {
       // ✅ Doğru - Aynı return type
       override suspend fun getData(): Result<User> { ... }
   }
   ```

3. **Preconditions:** Child class should not require **more** conditions than parent

   ```kotlin
   // ❌ Yanlış
   interface DataSource {
       fun getData(id: String): User
   }

   class StrictDataSource : DataSource {
       override fun getData(id: String): User {
           require(id.length > 10) // ❌ Parent böyle bir koşul yok!
           ...
       }
   }
   ```

4. **Postconditions:** Child class must preserve parent's guarantees

   ```kotlin
   // ✅ Doğru - Her implementation Result döndürüyor
   interface Repository {
       suspend fun getData(): Result<User> // Garanti: Result dönecek
   }

   class CacheRepo : Repository {
       override suspend fun getData(): Result<User> {
           return Result.success(...) // ✅ Garanti korundu
       }
   }
   ```

---

## 4. Interface Segregation Principle (ISP)

### Definition

> **"A class should not be forced to implement methods it doesn't use"**

In other words: **Use small, specific interfaces instead of large, bloated interfaces.**

### ❌ Wrong Example (E-commerce - Repository)

```kotlin
// Şişkin interface - Her şey bir arada
interface ProductRepository {
    // Read operations
    suspend fun getProducts(): List<Product>
    suspend fun getProductDetails(id: String): Product
    suspend fun searchProducts(query: String): List<Product>

    // Write operations
    suspend fun addProduct(product: Product)
    suspend fun updateProduct(product: Product)
    suspend fun deleteProduct(id: String)

    // Cart operations
    suspend fun addToCart(productId: String)
    suspend fun removeFromCart(productId: String)

    // Favorite operations
    suspend fun addToFavorites(productId: String)
    suspend fun removeFromFavorites(productId: String)
}

// ❌ Sorun: Product listesi ekranında sadece getProducts() lazım
// ama 10 tane method implement etmek zorunda!
class ProductListViewModel(
    private val repository: ProductRepository // 10 method var ama sadece 1 kullanıyoruz!
) : ViewModel() {

    fun loadProducts() {
        viewModelScope.launch {
            val products = repository.getProducts() // Sadece bunu kullanıyoruz
        }
    }
}

// Test ederken de gereksiz mock'lamalar yapman gerekir
@Test
fun testLoadProducts() {
    val mockRepo = mock<ProductRepository>()
    // 10 tane method var, hepsini mock'laman gerekir mi? Hayır! Sadece 1 tanesi lazım.
}
```

### ✅ Correct Example (Interface Segregation)

**Visual:**

```
❌ Şişkin Interface:
┌────────────────────────────┐
│  ProductRepository         │
│  (10 method)               │
└────────────────────────────┘
         │
         │ Hepsini implement et
         ▼
┌────────────────────────────┐
│ ProductListViewModel       │
│ (Sadece 1 method kullanıyor)│
└────────────────────────────┘
Sorun: 9 gereksiz method!

✅ Ayrılmış Interface'ler:
┌────────────────┐  ┌────────────────┐  ┌────────────────┐
│ ReadRepository │  │WriteRepository │  │ CartRepository │
│  (3 method)    │  │  (3 method)    │  │  (3 method)    │
└────────────────┘  └────────────────┘  └────────────────┘
        │
        │ Sadece ihtiyacın olanı al
        ▼
┌────────────────────────────┐
│ ProductListViewModel       │
│ (ReadRepository alıyor)    │
└────────────────────────────┘
```

```kotlin
// 1. Küçük, spesifik interface'ler
interface ProductReadRepository {
    suspend fun getProducts(): List<Product>
    suspend fun getProductDetails(id: String): Product
    suspend fun searchProducts(query: String): List<Product>
}

interface ProductWriteRepository {
    suspend fun addProduct(product: Product)
    suspend fun updateProduct(product: Product)
    suspend fun deleteProduct(id: String)
}

interface CartRepository {
    suspend fun addToCart(productId: String)
    suspend fun removeFromCart(productId: String)
    suspend fun getCartItems(): List<CartItem>
}

interface FavoriteRepository {
    suspend fun addToFavorites(productId: String)
    suspend fun removeFromFavorites(productId: String)
    suspend fun getFavorites(): List<Product>
}

// 2. Implementation tüm interface'leri implement eder (gerekirse)
class ProductRepositoryImpl(
    private val apiService: ApiService,
    private val database: ProductDao
) : ProductReadRepository, ProductWriteRepository {

    override suspend fun getProducts(): List<Product> {
        return apiService.getProducts().map { it.toDomain() }
    }

    override suspend fun getProductDetails(id: String): Product {
        return apiService.getProductDetails(id).toDomain()
    }

    override suspend fun searchProducts(query: String): List<Product> {
        return apiService.searchProducts(query).map { it.toDomain() }
    }

    override suspend fun addProduct(product: Product) {
        apiService.addProduct(product.toDto())
    }

    override suspend fun updateProduct(product: Product) {
        apiService.updateProduct(product.toDto())
    }

    override suspend fun deleteProduct(id: String) {
        apiService.deleteProduct(id)
    }
}

// 3. ViewModel'ler sadece ihtiyacı olan interface'i alır
class ProductListViewModel(
    private val readRepository: ProductReadRepository // Sadece read lazım ✅
) : ViewModel() {

    fun loadProducts() {
        viewModelScope.launch {
            val products = readRepository.getProducts()
        }
    }
}

class ProductEditViewModel(
    private val writeRepository: ProductWriteRepository // Sadece write lazım ✅
) : ViewModel() {

    fun updateProduct(product: Product) {
        viewModelScope.launch {
            writeRepository.updateProduct(product)
        }
    }
}

class CartViewModel(
    private val cartRepository: CartRepository // Sadece cart lazım ✅
) : ViewModel() {

    fun addToCart(productId: String) {
        viewModelScope.launch {
            cartRepository.addToCart(productId)
        }
    }
}

// 4. Test ederken sadece ihtiyacın olanı mock'larsın
@Test
fun testLoadProducts() {
    val mockReadRepo = mock<ProductReadRepository>() // Sadece read interface'i
    val viewModel = ProductListViewModel(mockReadRepo)

    // Sadece getProducts() mock'lanır
    whenever(mockReadRepo.getProducts()).thenReturn(listOf(mockProduct))

    viewModel.loadProducts()

    verify(mockReadRepo).getProducts()
}
```

### Advantages

- **No unnecessary dependencies**
- Testability increases (**you only mock what you need**)
- Code becomes more **readable** (it's clear what the ViewModel uses)
- **Separation of concerns** increases

### When to Use?

- When you have large, bloated interfaces
- When some classes only use a portion of an interface
- When you're forced to do too much mocking in tests
- When using **Clean Architecture** and **Repository Pattern** (definitely!)

### When Not to Use?

- When the interface is already small (3-5 methods)
- When every method is actually used everywhere
- In very simple projects (can be over-engineering)

---

## 5. Dependency Inversion Principle (DIP)

### Definition

> **"High-level modules should not depend on low-level modules. Both should depend on abstractions."**

In other words: **Depend on interfaces/abstract classes instead of concrete classes.**

### What Do "High-Level" and "Low-Level" Mean?

- **High-level**: Modules containing business logic
  - Examples: ViewModel, UseCase, Repository Interface
  - Layers far from the framework (can be pure Kotlin)
  - Answers the question **"What to do"**

- **Low-level**: Modules containing technical details
  - Examples: Retrofit, Room, Firebase, SharedPreferences
  - Layers close to the framework (dependent on Android/3rd party libraries)
  - Answers the question **"How to do it"**

**Rule:** High-level should not depend on low-level. Both should depend on interfaces.

**Example:**
- ❌ Wrong: `ViewModel` → `Retrofit` (High-level depends on low-level)
- ✅ Correct: `ViewModel` → `Repository Interface` ← `RepositoryImpl (Retrofit)` (Both depend on interface)

### ❌ Wrong Example (News App)

```kotlin
// Alt seviye modül (Concrete class)
class RetrofitNewsApiService {
    fun getArticles(): List<ArticleDto> {
        // Retrofit ile API çağrısı
        return retrofit.create(NewsApi::class.java).getArticles()
    }
}

// Üst seviye modül - Direkt concrete class'a bağımlı ❌
class ArticleViewModel(
    private val apiService: RetrofitNewsApiService // Concrete class!
) : ViewModel() {

    fun loadArticles() {
        viewModelScope.launch {
            val articles = apiService.getArticles()
            // ...
        }
    }
}
```

**Problems:**

1. ViewModel is **tightly coupled to Retrofit** (you can't switch to Ktor tomorrow)
2. Not testable (mocking Retrofit is difficult)
3. **Tight coupling**
4. ViewModel knows low-level details (Retrofit, DTO, etc.)

### ✅ Correct Example (Dependency Inversion)

```kotlin
// ==========================================
// DOMAIN LAYER (Üst Seviye - Soyutlama)
// ==========================================

// Interface (Soyutlama) - Domain katmanında tanımlanır
interface ArticleRepository {
    suspend fun getArticles(): Result<List<Article>>
}

// Domain Model (Framework'den bağımsız)
data class Article(
    val id: String,
    val title: String,
    val content: String
)

// Use Case (Üst seviye modül)
class GetArticlesUseCase(
    private val repository: ArticleRepository // Interface'e bağımlı ✅
) {
    suspend operator fun invoke(): Result<List<Article>> {
        return repository.getArticles()
    }
}

// ==========================================
// DATA LAYER (Alt Seviye - Implementation)
// ==========================================

// DTO (Data Transfer Object - API response)
data class ArticleDto(
    val id: String,
    val title: String,
    val content: String
)

// Retrofit Service (Alt seviye detay)
interface NewsApiService {
    @GET("articles")
    suspend fun getArticles(): List<ArticleDto>
}

// Repository Implementation (Interface'i implement eder)
class ArticleRepositoryImpl(
    private val apiService: NewsApiService // Alt seviye detay
) : ArticleRepository {

    override suspend fun getArticles(): Result<List<Article>> {
        return try {
            val dtos = apiService.getArticles()
            val articles = dtos.map { it.toDomain() }
            Result.success(articles)
        } catch (e: Exception) {
            Result.failure(e)
        }
    }
}

// ==========================================
// PRESENTATION LAYER (Üst Seviye)
// ==========================================

// ViewModel (Üst seviye modül) - Interface'e bağımlı ✅
class ArticleViewModel(
    private val getArticlesUseCase: GetArticlesUseCase // Use Case'e bağımlı
) : ViewModel() {

    private val _articles = MutableStateFlow<List<Article>>(emptyList())
    val articles: StateFlow<List<Article>> = _articles

    fun loadArticles() {
        viewModelScope.launch {
            getArticlesUseCase().fold(
                onSuccess = { _articles.value = it },
                onFailure = { /* handle error */ }
            )
        }
    }
}

// ==========================================
// DEPENDENCY INJECTION (Hilt)
// ==========================================

@Module
@InstallIn(SingletonComponent::class)
object DataModule {

    @Provides
    @Singleton
    fun provideNewsApiService(): NewsApiService {
        return Retrofit.Builder()
            .baseUrl("https://api.news.com")
            .addConverterFactory(GsonConverterFactory.create())
            .build()
            .create(NewsApiService::class.java)
    }

    @Provides
    @Singleton
    fun provideArticleRepository(
        apiService: NewsApiService
    ): ArticleRepository { // Interface döndür ✅
        return ArticleRepositoryImpl(apiService)
    }
}

@Module
@InstallIn(ViewModelComponent::class)
object DomainModule {

    @Provides
    fun provideGetArticlesUseCase(
        repository: ArticleRepository // Interface inject et ✅
    ): GetArticlesUseCase {
        return GetArticlesUseCase(repository)
    }
}
```

### Dependency Flow

```
❌ Yanlış (Dependency Inversion YOK):

    Üst Seviye
┌─────────────────┐
│   ViewModel     │
│ (Business Logic)│
└────────┬────────┘
         │ Depends on (Bağımlı)
         ▼
    Alt Seviye
┌─────────────────┐
│ RetrofitService │
│(Teknik Detay)   │
└─────────────────┘

Sorun: Üst seviye, alt seviyeye bağımlı!
- ViewModel Retrofit'e bağımlı → Yarın Ktor'a geçemezsin
- Test edilemez → Retrofit mock'lamak zor
- Framework değişikliği ViewModel'i etkiler


✅ Doğru (Dependency Inversion VAR):

    Üst Seviye
┌─────────────────┐         ┌──────────────────┐
│   ViewModel     │────────▶│ ArticleRepository│
│ (Business Logic)│         │   (Interface)    │◀─┐
└─────────────────┘         └──────────────────┘  │
                                                   │ Implements
    Alt Seviye                                     │
┌──────────────────┐                               │
│RepositoryImpl    │───────────────────────────────┘
│(Retrofit + Room) │
└──────────────────┘

Artık: Her iki modül de interface'e bağımlı!
- ViewModel → Repository Interface (Abstraction'a bağımlı)
- RepositoryImpl → Repository Interface (Abstraction'ı implement eder)
- Bağımlılık yönü tersine çevrildi! (Dependency Inversion)
```

### Advantages

- **Framework independence** (easy to switch from Retrofit → Ktor)
- **Testability** (you can mock the Repository)
- **Loose coupling**
- **Domain layer can be pure Kotlin** (independent of Android/Framework details)

### When to Use?

- **Always!** (especially in medium/large projects)
- **Mandatory** when using Clean Architecture
- When using Repository Pattern
- When writing tests

### When Not to Use?

- Very simple, single-screen applications
- Prototype/throw-away code

### Practical Examples

```kotlin
// ❌ Yanlış - Concrete class'a bağımlı
class MyViewModel(private val firebaseAuth: FirebaseAuth)

// ✅ Doğru - Interface'e bağımlı
class MyViewModel(private val authRepository: AuthRepository)

// ❌ Yanlış - Concrete class'a bağımlı
class MyUseCase(private val roomDatabase: AppDatabase)

// ✅ Doğru - Interface'e bağımlı
class MyUseCase(private val userRepository: UserRepository)
```

---

## SOLID and Clean Architecture Relationship

Clean Architecture is **SOLID principles applied to architecture**.

| SOLID Principle               | Clean Architecture Equivalent                                             |
| ----------------------------- | ------------------------------------------------------------------------- |
| **Single Responsibility**     | Each layer (Presentation, Domain, Data) has a single responsibility      |
| **Open/Closed**               | Use Cases are extensible (add new use case, don't modify existing code)  |
| **Liskov Substitution**       | Repository interfaces are interchangeable (Network/Cache/Mock)           |
| **Interface Segregation**     | Repositories use small, specific interfaces (Read/Write separate)        |
| **Dependency Inversion**      | **Dependency Rule:** Inner layers don't know about outer layers          |

---

## Over-Engineering Warning

Blindly applying SOLID principles **everywhere** leads to over-engineering.

### When Should You Use SOLID?

✅ **Use:**

- In medium/large projects (10+ screens)
- In long-lived projects (3+ years maintenance)
- In team work (5+ developers)
- When writing tests
- When code will be reused

❌ **Don't Use:**

- Single-screen prototype
- Throw-away code
- Very simple operations (don't create interfaces for 2-3 lines of code!)
- When the deadline is very tight and the code won't be maintained

### Golden Rule

> **"Keep it simple, but make it extensible when needed"**

When writing code, ask yourself:

- Is this abstraction really necessary?
- Is there a chance it will change tomorrow?
- Does it need to be tested?

**If the answers are "No", you don't need SOLID. Write it simple!**

---

## Summary Table

| Principle                     | One-Sentence Summary                             | When to Use                | When Not to Use               |
| ----------------------------- | ------------------------------------------------ | -------------------------- | ----------------------------- |
| **Single Responsibility**     | One class = one responsibility                   | Always                     | Very simple prototypes        |
| **Open/Closed**               | New feature = new class (existing code unchanged)| Frequently changing types  | Fixed, rarely changing types  |
| **Liskov Substitution**       | Code doesn't crash when child replaces parent    | When using inheritance     | When no inheritance           |
| **Interface Segregation**     | Small interfaces > Bloated interfaces            | Large interfaces           | Already small (3-5 methods)   |
| **Dependency Inversion**      | Depend on interfaces, not concrete classes       | Always (medium+ projects)  | Single-screen simple apps     |

---

## Resources

- [Clean Code - Robert C. Martin](https://www.amazon.com/Clean-Code-Handbook-Software-Craftsmanship/dp/0132350882)
- [SOLID Principles - Uncle Bob](https://blog.cleancoder.com/uncle-bob/2020/10/18/Solid-Relevance.html)
- [Android Architecture Guide - Google](https://developer.android.com/topic/architecture)
