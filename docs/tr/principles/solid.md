# SOLID Prensipleri

## Giriş

SOLID, nesne yönelimli programlamada kod kalitesini artırmak için kullanılan **5 temel tasarım prensibidir**. Robert C. Martin (Uncle Bob) tarafından önerilmiştir.

SOLID prensipleri:

- Kodun **bakımını kolaylaştırır**
- **Test edilebilirliği** artırır
- **Değişikliklere karşı esnek** hale getirir
- **Okunabilir ve anlaşılır** kod yazmanızı sağlar

**Önemli Not:** SOLID prensipleri **araçtır**, dogma değildir. Her yerde körü körüne uygulamak **over-engineering**'e yol açar. Doğru yerde, doğru zamanda kullanılmalıdır.

---

## İçindekiler

1. **S** - Single Responsibility Principle (Tek Sorumluluk Prensibi)
2. **O** - Open/Closed Principle (Açık/Kapalı Prensibi)
3. **L** - Liskov Substitution Principle (Liskov Yerine Geçme Prensibi)
4. **I** - Interface Segregation Principle (Arayüz Ayrımı Prensibi)
5. **D** - Dependency Inversion Principle (Bağımlılık Tersine Çevirme Prensibi)

---

## 1. Single Responsibility Principle (SRP)

### Tanım

> **"Bir sınıf veya modül, sadece bir değişme sebebine sahip olmalıdır"**

Başka bir deyişle: **Bir sınıf sadece bir işten sorumlu olmalı.**

### "Değişme Sebebi" Ne Demek?

**Değişme sebebi**, bir sınıfın değiştirilmesini gerektirecek farklı iş gereksinimleridir.

Örneğin aşağıdaki yanlış örnekteki `ArticleActivity` sınıfının **5 farklı değişme sebebi** var:

- API endpoint değişirse → Network kodunu değiştirmen gerekir
- Analytics servisi değişirse (Firebase → Mixpanel) → Analytics kodunu değiştirmen gerekir
- Database şeması değişirse → Database kodunu değiştirmen gerekir
- UI tasarımı değişirse → UI kodunu değiştirmen gerekir
- Veri formatı değişirse → Parsing kodunu değiştirmen gerekir

**Sorun:** Her değişiklik aynı sınıfa dokunuyor, bu da kod çakışmalarına ve test zorluğuna yol açıyor.

**Çözüm:** Her sorumluluk için ayrı sınıf → Her sınıfın tek bir değişme sebebi olur.

### ❌ Yanlış Örnek (News App)

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

**Sorunlar:**

- Activity **5 farklı işten sorumlu** (network, veri işleme, analytics, database, UI)
- Test edilemez (Activity mock'lanamaz)
- Bir iş değiştiğinde (örn: analytics servisi değişir) Activity'yi değiştirmen gerekir
- Kod tekrarı (her ekranda aynı işlemler)

### ✅ Doğru Örnek (Clean Architecture + MVVM)

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

**Artık her sınıfın tek sorumluluğu var:**

- `NewsApiService`: Sadece API çağrısı
- `ArticleDao`: Sadece database işlemi
- `ArticleRepository`: Sadece veri kaynağı koordinasyonu
- `GetArticleUseCase`: Sadece makale getirme business logic
- `LogArticleViewUseCase`: Sadece analytics
- `ArticleViewModel`: Sadece UI state yönetimi
- `ArticleActivity`: Sadece UI render

### Avantajlar

- Her sınıf **bağımsız test** edilebilir
- Değişiklik yapılacağında **sadece ilgili sınıf** değişir
- Kod tekrarı azalır (**reusability**)
- Okunabilirlik artar

### Ne Zaman Kullanılır?

- **Her zaman!** SRP en temel prensiptir
- Özellikle **orta/büyük projelerde** kritik
- Clean Architecture ve MVVM bu prensibe dayanır

### Ne Zaman Kullanılmaz?

- Çok basit, tek ekranlı prototipler
- Throw-away kod (bir kere kullanılacak)

---

## 2. Open/Closed Principle (OCP)

### Tanım

> **"Sınıflar genişlemeye açık, değişikliğe kapalı olmalıdır"**

Yeni özellik eklerken **mevcut kodu değiştirmemeli**, sadece **yeni kod eklemeliyiz**.

### ❌ Yanlış Örnek (E-commerce - Ödeme Sistemi)

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

**Sorun:** Yeni ödeme yöntemi eklendiğinde (örn: Apple Pay), **`PaymentProcessor` sınıfını değiştirmen** gerekiyor.

```kotlin
// Apple Pay eklemek için mevcut kodu değiştiriyorsun ❌
"APPLE_PAY" -> {
    println("Processing Apple Pay payment: $amount")
    Result.success(Unit)
}
```

### ✅ Doğru Örnek (Polymorphism ile)

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
object PaymentMethodFactory {
    fun getPaymentMethod(type: String): PaymentMethod {
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

**Artık yeni ödeme yöntemi eklerken:**

- Yeni bir `ApplePayPayment` class'ı yazarsın
- Factory'e eklersin
- **`PaymentProcessor` sınıfını değiştirmezsin!** ✅

### Avantajlar

- Yeni özellik eklerken **mevcut kodu bozmak riski yok**
- Test edilmiş kod değişmediği için **regression riski azalır**
- Extensible (genişletilebilir) sistem

### Ne Zaman Kullanılır?

- Sık sık yeni tipler ekleniyorsa (ödeme yöntemleri, notification tipleri)
- Plugin sistemi yapıyorsan
- 3rd party entegrasyonlar (her müşteri farklı implementation)
- Kodun başkaları tarafından genişletilecekse

### Ne Zaman Kullanılmaz?

- Tipler **sabitse** ve nadiren değişiyorsa → `when` kullan (daha pratik)
- Çok basit durumlar (2-3 tip var ve değişmeyecek)
- **Over-engineering'den kaçın!**

**Örnek:** Android'de plug type'lar (Type2, CCS, CHAdeMO) neredeyse hiç değişmiyor → `when` kullanmak daha mantıklı.

---

## 3. Liskov Substitution Principle (LSP)

### Tanım

> **"Alt sınıflar (child), üst sınıfın (parent) yerine kullanıldığında beklenmedik davranış göstermemeli"**

Başka bir deyişle: **Parent interface/class beklendiği yerde child class kullanabilmeliyim, kod patlamadan.**

### ❌ Yanlış Örnek (Data Source - Cache vs Network)

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

**Sorunlar:**

1. **Interface güvenli görünüyor ama değil**: `getUser()` her zaman `User` döneceğini vaat ediyor ama exception fırlatabilir
2. **Beklenmedik davranış**: Parent interface yerine child class koyunca kod patlar
3. **Try-catch zorunluluğu**: Her yerde exception handling yapmak zorundasın
4. **Test edilemez**: Mock yaparken exception fırlatıp fırlatmayacağını tahmin edemezsin

### ✅ Doğru Örnek (Result Pattern ile)

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

**Artık:**

- ✅ Her `UserDataSource` implementation'ı **aynı şekilde davranıyor**
- ✅ Hiçbiri **exception fırlatmıyor**
- ✅ Her yerde **güvenli şekilde** kullanabilirsin
- ✅ **Test edilebilir** (Mock kolayca yazılır)

### Avantajlar

- **Polymorphism güvenli** şekilde kullanılır (her implementation aynı şekilde davranır)
- **Kod tahmin edilebilir** olur (hangi implementation gelirse gelsin, kod patlamaz)
- **Runtime exception'ları** ortadan kalkar
- **Test edilebilirlik** artar (her implementation aynı contract'a uyar)
- **Result pattern** ile hata yönetimi açık ve net

### Ne Zaman Kullanılır?

- Interface veya inheritance kullanıyorsan **her zaman!**
- Birden fazla data source varsa (Network, Cache, Database, Mock)
- Repository Pattern kullanıyorsan
- Farklı implementation'lar aynı şekilde davranmalıysa

### Pratik Kurallar (Android için)

1. **Exception fırlatma yasak:** `Result<T>`, `sealed class`, veya nullable kullan

   ```kotlin
   // ❌ Yanlış
   fun getData(): User // Exception fırlatabilir

   // ✅ Doğru
   suspend fun getData(): Result<User> // Hata olabileceği açık
   ```

2. **Return type tutarlı olmalı:** Parent `Result<T>` dönüyorsa, child da `Result<T>` dönmeli

   ```kotlin
   interface Repository {
       suspend fun getData(): Result<User>
   }

   class NetworkRepo : Repository {
       // ✅ Doğru - Aynı return type
       override suspend fun getData(): Result<User> { ... }
   }
   ```

3. **Preconditions (Ön koşullar):** Child class, parent'tan **daha fazla** koşul istememeli

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

4. **Postconditions (Son koşullar):** Child class, parent'ın garantilerini korumalı

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

### Tanım

> **"Bir class, kullanmadığı methodları implement etmeye zorlanmamalı"**

Başka bir deyişle: **Büyük, şişkin interface'ler yerine küçük, spesifik interface'ler kullan.**

### ❌ Yanlış Örnek (E-commerce - Repository)

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

### ✅ Doğru Örnek (Interface Segregation)

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

### Avantajlar

- **Gereksiz bağımlılıklar** olmaz
- Test edilebilirlik artar (**sadece ihtiyacın olanı mock'larsın**)
- Kod daha **okunabilir** olur (ViewModel ne kullanıyor belli)
- **Separation of concerns** artar

### Ne Zaman Kullanılır?

- Büyük, şişkin interface'leriniz varsa
- Bazı class'lar interface'in sadece bir kısmını kullanıyorsa
- Test ederken çok fazla mock yapmak zorunda kalıyorsanız
- **Clean Architecture** ve **Repository Pattern** kullanıyorsanız (kesinlikle!)

### Ne Zaman Kullanılmaz?

- Interface zaten küçükse (3-5 method)
- Her method gerçekten her yerde kullanılıyorsa
- Çok basit projelerde (over-engineering olabilir)

---

## 5. Dependency Inversion Principle (DIP)

### Tanım

> **"Üst seviye modüller, alt seviye modüllere bağımlı olmamalı. İkisi de soyutlamalara (abstraction) bağımlı olmalı."**

Başka bir deyişle: **Concrete class yerine interface/abstract class'a bağımlı ol.**

### ❌ Yanlış Örnek (News App)

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

**Sorunlar:**

1. ViewModel, **Retrofit'e sıkı bağımlı** (yarın Ktor'a geçemezsin)
2. Test edilemez (Retrofit'i mock'lamak zor)
3. **Tight coupling** (sıkı bağımlılık)
4. ViewModel, alt seviye detayları biliyor (Retrofit, DTO, vb.)

### ✅ Doğru Örnek (Dependency Inversion)

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

### Bağımlılık Yönü (Dependency Flow)

```
❌ Yanlış (Dependency Inversion YOK):
┌─────────────────┐
│   ViewModel     │
│  (Üst Seviye)   │
└────────┬────────┘
         │ Depends on (Bağımlı)
         ▼
┌─────────────────┐
│ RetrofitService │
│  (Alt Seviye)   │
└─────────────────┘
Sorun: Üst seviye, alt seviyeye bağımlı!


✅ Doğru (Dependency Inversion VAR):
┌─────────────────┐         ┌──────────────────┐
│   ViewModel     │────────▶│ ArticleRepository│
│  (Üst Seviye)   │         │   (Interface)    │
└─────────────────┘         └────────▲─────────┘
                                     │ Implements
                            ┌────────┴─────────┐
                            │RepositoryImpl    │
                            │  (Alt Seviye)    │
                            └──────────────────┘

Artık: Her iki modül de interface'e bağımlı!
```

### Avantajlar

- **Framework bağımsızlığı** (Retrofit → Ktor geçişi kolay)
- **Test edilebilirlik** (Repository'yi mock'layabilirsin)
- **Loose coupling** (gevşek bağımlılık)
- **Domain layer pure Kotlin** olabilir (Android/Framework detaylarından bağımsız)

### Ne Zaman Kullanılır?

- **Her zaman!** (özellikle orta/büyük projelerde)
- Clean Architecture kullanıyorsan **zorunlu**
- Repository Pattern kullanıyorsan
- Test yazıyorsan

### Ne Zaman Kullanılmaz?

- Çok basit, tek ekranlı uygulamalar
- Prototype/throw-away kod

### Pratik Örnekler

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

## SOLID ve Clean Architecture İlişkisi

Clean Architecture, **SOLID prensiplerinin mimariye uygulanmış halidir**.

| SOLID Prensibi            | Clean Architecture'da Karşılığı                                            |
| ------------------------- | -------------------------------------------------------------------------- |
| **Single Responsibility** | Her katman (Presentation, Domain, Data) tek sorumluluk                     |
| **Open/Closed**           | Use Case'ler genişletilebilir (yeni use case ekle, mevcut kodu değiştirme) |
| **Liskov Substitution**   | Repository interface'leri değiştirilebilir (Network/Cache/Mock)            |
| **Interface Segregation** | Repository'ler küçük, spesifik interface'ler (Read/Write ayrı)             |
| **Dependency Inversion**  | **Dependency Rule:** İçteki katmanlar dıştaki katmanları bilmez            |

---

## Over-Engineering Uyarısı

SOLID prensiplerini **her yerde körü körüne uygulamak** over-engineering'e yol açar.

### Ne Zaman SOLID Kullanmalısın?

✅ **Kullan:**

- Orta/büyük projelerde (10+ ekran)
- Uzun ömürlü projelerde (3+ yıl bakım)
- Takım çalışmasında (5+ developer)
- Test yazıyorsan
- Kod tekrar kullanılacaksa

❌ **Kullanma:**

- Tek ekranlı prototip
- Throw-away kod
- Çok basit işlemler (2-3 satır kod için interface açma!)
- Deadline çok sıkı ve kod bir daha bakılmayacaksa

### Altın Kural

> **"Basit tut, ama ihtiyaç duyduğunda genişletebilir yap"**

Kod yazarken kendin sor:

- Bu abstraction gerçekten lazım mı?
- Yarın değişme ihtimali var mı?
- Test edilmesi gerekiyor mu?

**Eğer cevaplar "Hayır" ise, SOLID'e gerek yok. Basit yaz!**

---

## Özet Tablo

| Prensip                   | Ne Zaman Kullan            | Ne Zaman Kullanma             |
| ------------------------- | -------------------------- | ----------------------------- |
| **Single Responsibility** | Her zaman                  | Çok basit prototipler         |
| **Open/Closed**           | Sık değişen tipler         | Sabit, nadiren değişen tipler |
| **Liskov Substitution**   | Inheritance kullanıyorsan  | Inheritance yoksa             |
| **Interface Segregation** | Büyük interface'ler        | Zaten küçükse (3-5 method)    |
| **Dependency Inversion**  | Her zaman (orta+ projeler) | Tek ekranlı basit uygulamalar |

---

## Kaynaklar

- [Clean Code - Robert C. Martin](https://www.amazon.com/Clean-Code-Handbook-Software-Craftsmanship/dp/0132350882)
- [SOLID Principles - Uncle Bob](https://blog.cleancoder.com/uncle-bob/2020/10/18/Solid-Relevance.html)
- [Android Architecture Guide - Google](https://developer.android.com/topic/architecture)
