# Immutability ve Single Source of Truth (SSOT)

## Giriş

**Immutability (Değişmezlik)** ve **Single Source of Truth (Tek Gerçeklik Kaynağı)**, modern Android uygulamalarında güvenli ve öngörülebilir state yönetiminin temel prensipleridir.

**Basit Anlatım:**
- **Immutability**: Bir nesneyi oluşturduktan sonra değiştirememe
- **SSOT**: Veriyi sadece tek bir yerde tutma, diğer yerler gözlemler

**Gerçek hayat analojisi:**
- **Mutable (Değişebilir)**: Tahta (üzerine yaz, sil, tekrar yaz)
- **Immutable (Değişmez)**: Kağıt (bir kere yazdın, değiştiremezsin. Yeni kağıt kullan)

> **Temel Fikir:** "State'i değiştirme, yeni state oluştur. Veriyi bir yerde tut, her yerde gözlemle."

**Önemli Not:** Bu prensipler Android Architecture Guide'ın temel taşlarıdır. Reactive programming ve modern state management'ın temelidir.

---

## Immutability (Değişmezlik)

### `val` vs `var`

```kotlin
// var - Mutable (Değişebilir)
var name = "Ali"
name = "Veli"  // ✅ Değişir

// val - Immutable (Değiştirilemez)
val surname = "Yılmaz"
surname = "Demir"  // ❌ DERLENMEZ!
```

**`val` = Referansı immutable yapar**

```kotlin
val age: Int = 25
age = 30  // ❌ DERLENMEZ

// Görsel:
// Memory: [25] ← age referansı (değiştirilemez!)
```

### ⚠️ Dikkat: `val` Her Zaman Immutable Değildir!

```kotlin
// ⚠️ Referans immutable, içerik mutable!
val list = mutableListOf("Ali")
list = mutableListOf("Veli")  // ❌ Referans değişemez
list.add("Veli")  // ✅ İçerik değişir!

println(list)  // [Ali, Veli]
```

**Görsel:**
```
Memory:
┌─────────────────────┐
│ list (referans)     │ ← val (değiştirilemez!)
└──────────┬──────────┘
           │
           ▼
    ┌──────────────┐
    │ ["Ali"]      │ ← MutableList (değiştirilebilir!)
    │ ["Ali", "Veli"] ← add() ile değişti
    └──────────────┘
```

**Tam immutability için:**

```kotlin
// ✅ Hem referans hem içerik immutable
val list = listOf("Ali")  // Immutable List
list.add("Veli")  // ❌ DERLENMEZ (add() metodu yok!)
```

---

## Primitive Types vs Collections

### Primitive Types - Tamamen Immutable

```kotlin
val age: Int = 25
val name: String = "Ali"
val isActive: Boolean = true

// Hiçbiri değiştirilemez! ✅
age = 30  // ❌
name = "Veli"  // ❌
isActive = false  // ❌
```

### Collections - Dikkatli Ol!

```kotlin
// ❌ Mutable collection
val mutableList = mutableListOf("Ali")
mutableList.add("Veli")  // ✅ Değişir!

// ✅ Immutable collection
val immutableList = listOf("Ali")
immutableList.add("Veli")  // ❌ DERLENMEZ!
```

**Comparison Table:**

| Type | Mutable | Immutable |
|------|---------|-----------|
| **List** | `mutableListOf()`, `ArrayList()` | `listOf()`, `emptyList()` |
| **Set** | `mutableSetOf()`, `HashSet()` | `setOf()`, `emptySet()` |
| **Map** | `mutableMapOf()`, `HashMap()` | `mapOf()`, `emptyMap()` |

---

## Data Class ve Immutability

### Data Class Copy Pattern

```kotlin
// ✅ Immutable data class
data class User(
    val name: String,
    val age: Int
)

val user1 = User("Ali", 25)
user1.age = 30  // ❌ DERLENMEZ (val ile tanımlı)

// ✅ Yeni nesne oluştur (copy)
val user2 = user1.copy(age = 30)

println(user1)  // User(name=Ali, age=25) - Değişmedi! ✅
println(user2)  // User(name=Ali, age=30) - Yeni nesne! ✅
```

### Copy Nasıl Çalışır?

```kotlin
data class User(val name: String, val age: Int)

val user1 = User("Ali", 25)
val user2 = user1.copy(age = 30)

// Farklı referanslar!
println(user1 === user2)  // false
println(user1.hashCode())  // Örnek: 123456
println(user2.hashCode())  // Örnek: 789012
```

**Görsel:**
```
Memory:
┌─────────────────┐
│ user1 (Ali, 25) │  Referans: 0x123456
└─────────────────┘

┌─────────────────┐
│ user2 (Ali, 30) │  Referans: 0x789012 (YENİ NESNE!)
└─────────────────┘
```

**Avantaj: Thread Safety**

```kotlin
// Thread 1
val user1 = User("Ali", 25)

// Thread 2 (aynı anda)
val user2 = user1.copy(age = 30)

// user1 değişmedi! Thread-safe! ✅
println(user1.age)  // 25
println(user2.age)  // 30
```

---

## Immutability'nin Avantajları

### 1. **Thread Safety** ⭐⭐⭐

**❌ Mutable (Tehlikeli):**

```kotlin
var count = 0

// Thread 1
count++  // count = 1

// Thread 2 (aynı anda)
count++  // count = ??? (1 mi 2 mi?)

// Race condition! ❌
```

**✅ Immutable (Güvenli):**

```kotlin
val count1 = 0

// Thread 1
val count2 = count1 + 1  // Yeni değer

// Thread 2 (aynı anda)
val count3 = count1 + 1  // Yeni değer

println(count1)  // 0 (değişmedi!)
println(count2)  // 1
println(count3)  // 1

// Thread-safe! ✅
```

### 2. **Predictable State** (Öngörülebilir Durum)

**❌ Mutable:**

```kotlin
data class Article(
    var title: String,
    var isRead: Boolean
)

val article = Article("SOLID Prensipleri", false)

fun markAsRead(article: Article) {
    article.isRead = true  // Orijinal değişti! ❌
}

markAsRead(article)
println(article.isRead)  // true - Beklenmeyen değişiklik!
```

**✅ Immutable:**

```kotlin
data class Article(
    val title: String,
    val isRead: Boolean
)

val article = Article("SOLID Prensipleri", false)

fun markAsRead(article: Article): Article {
    return article.copy(isRead = true)  // Yeni nesne döndür ✅
}

val updatedArticle = markAsRead(article)
println(article.isRead)  // false - Orijinal değişmedi! ✅
println(updatedArticle.isRead)  // true - Yeni nesne! ✅
```

### 3. **Easier Testing**

```kotlin
// ✅ Immutable - Test kolay
data class User(val name: String, val age: Int)

@Test
fun `copy creates new instance`() {
    val user1 = User("Ali", 25)
    val user2 = user1.copy(age = 30)

    assertEquals(25, user1.age)  // Orijinal değişmedi ✅
    assertEquals(30, user2.age)  // Yeni değer ✅
    assertNotSame(user1, user2)  // Farklı nesneler ✅
}
```

### 4. **No Side Effects**

```kotlin
// ❌ Mutable - Side effect var
fun processUser(user: User) {
    user.age = user.age + 1  // Orijinali değiştiriyor! ❌
}

val user = User("Ali", 25)
processUser(user)
println(user.age)  // 26 - Beklenmeyen değişiklik!

// ✅ Immutable - Side effect yok
fun processUser(user: User): User {
    return user.copy(age = user.age + 1)  // Yeni nesne döndür ✅
}

val user = User("Ali", 25)
val newUser = processUser(user)
println(user.age)  // 25 - Orijinal korundu! ✅
println(newUser.age)  // 26 - Yeni nesne! ✅
```

---

## Single Source of Truth (SSOT)

### SSOT Nedir?

**Single Source of Truth (SSOT)** = **Veriyi sadece tek bir yerde tutma** prensibidir.

> **"Her veri için tek bir kaynak, diğer yerler bu kaynağı gözlemler"**

### ❌ SSOT İhlali

```kotlin
// ❌ Aynı veri 3 yerde tutulmuş!
class ArticleActivity : AppCompatActivity() {
    private var articles: List<Article> = emptyList()  // 1. yer

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        lifecycleScope.launch {
            articles = api.getArticles()  // Activity'de veri
            updateUI()
        }
    }
}

class ArticleViewModel : ViewModel() {
    private var articles: List<Article> = emptyList()  // 2. yer

    fun loadArticles() {
        viewModelScope.launch {
            articles = api.getArticles()  // ViewModel'de veri
        }
    }
}

class ArticleRepository {
    private var articles: List<Article> = emptyList()  // 3. yer
    private var cachedArticles: List<Article> = emptyList()  // 4. yer (!)

    suspend fun getArticles(): List<Article> {
        // Hangisi doğru? articles mı cachedArticles mi? 🤔
        return articles
    }
}
```

**Sorunlar:**

1. **Senkronizasyon problemi**: Hangisi güncel veri?
2. **Bellek israfı**: Aynı veri 4 yerde
3. **Tutarsızlık**: Activity'deki liste ViewModel'den farklı olabilir
4. **Test zorluğu**: Her yeri ayrı test etmen gerekir

### ✅ SSOT Uygulaması

```kotlin
// ==========================================
// DATA LAYER - SSOT (Tek Kaynak!)
// ==========================================

class ArticleRepository(
    private val apiService: NewsApiService,
    private val articleDao: ArticleDao
) {
    // ✅ Veri SADECE burada tutulur (SSOT!)
    private val _articles = MutableStateFlow<List<Article>>(emptyList())
    val articles: StateFlow<List<Article>> = _articles.asStateFlow()

    suspend fun refreshArticles() {
        try {
            // API'den al
            val apiArticles = apiService.getArticles()

            // Local'e kaydet
            articleDao.insertAll(apiArticles.map { it.toEntity() })

            // SSOT'u güncelle
            _articles.value = apiArticles
        } catch (e: Exception) {
            // Hata olursa local'den al
            val localArticles = articleDao.getAll().map { it.toDomain() }
            _articles.value = localArticles
        }
    }
}

// ==========================================
// DOMAIN LAYER - Use Case
// ==========================================

class GetArticlesUseCase(
    private val repository: ArticleRepository
) {
    // ❌ Veri tutmuyor, sadece Repository'yi expose ediyor
    operator fun invoke(): StateFlow<List<Article>> {
        return repository.articles  // SSOT'tan al
    }
}

// ==========================================
// PRESENTATION LAYER - ViewModel
// ==========================================

class ArticleViewModel(
    private val getArticlesUseCase: GetArticlesUseCase,
    private val refreshArticlesUseCase: RefreshArticlesUseCase
) : ViewModel() {

    // ❌ Veri tutmuyor, sadece gözlemliyor!
    val articles: StateFlow<List<Article>> = getArticlesUseCase()

    fun refresh() {
        viewModelScope.launch {
            refreshArticlesUseCase()  // Repository'yi güncelle
        }
    }
}

// ==========================================
// UI LAYER - Activity
// ==========================================

class ArticleActivity : AppCompatActivity() {
    private val viewModel: ArticleViewModel by viewModels()

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        // ❌ Veri tutmuyor, sadece gözlemliyor!
        lifecycleScope.launch {
            viewModel.articles.collect { articles ->
                updateUI(articles)  // UI'ı güncelle
            }
        }

        swipeRefresh.setOnRefreshListener {
            viewModel.refresh()  // Refresh tetikle
        }
    }
}
```

**Artık:**
- ✅ Veri **sadece Repository'de** tutulur (SSOT!)
- ✅ ViewModel gözlemler (veri tutmaz)
- ✅ Activity gözlemler (veri tutmaz)
- ✅ Tek kaynak, senkronizasyon problemi yok!

### SSOT Görsel

```
┌──────────────────────────────────┐
│   Repository (SSOT)              │
│   private val _articles = ...    │ ← Veri SADECE burada!
│   val articles: StateFlow        │
└────────────┬─────────────────────┘
             │
             ├──► UseCase (gözlemler)
             │        │
             │        └──► ViewModel (gözlemler)
             │                 │
             │                 └──► Activity (gözlemler)
             │
             └──► Fragment (gözlemler)

Veri akışı: Repository → UseCase → ViewModel → UI
Her katman sadece gözlemler, veri tutmaz! ✅
```

---

## StateFlow ve Immutability

### StateFlow Nedir?

**StateFlow** = Kotlin Coroutines'in **hot stream** yapısıdır. Observable pattern'ın modern versiyonudur.

```kotlin
// MutableStateFlow - Değiştirilebilir (private)
private val _articles = MutableStateFlow<List<Article>>(emptyList())

// StateFlow - Read-only (public)
val articles: StateFlow<List<Article>> = _articles.asStateFlow()
```

### Encapsulation Pattern

```kotlin
class ArticleViewModel : ViewModel() {
    // Private - Sadece ViewModel değiştirebilir
    private val _uiState = MutableStateFlow<UiState>(UiState.Loading)

    // Public - Herkes gözlemleyebilir ama değiştiremez!
    val uiState: StateFlow<UiState> = _uiState.asStateFlow()

    fun loadArticles() {
        viewModelScope.launch {
            _uiState.value = UiState.Loading

            val result = repository.getArticles()
            _uiState.value = if (result.isSuccess) {
                UiState.Success(result.getOrNull()!!)
            } else {
                UiState.Error(result.exceptionOrNull()?.message ?: "Unknown error")
            }
        }
    }
}

// Activity - Sadece okur
lifecycleScope.launch {
    viewModel.uiState.collect { state ->
        when (state) {
            is UiState.Loading -> showLoading()
            is UiState.Success -> showArticles(state.articles)
            is UiState.Error -> showError(state.message)
        }
    }
}

// ❌ Activity ViewModel'i değiştiremez!
viewModel._uiState.value = UiState.Loading  // DERLENMEZ (_uiState private!)
```

### StateFlow Garantileri

1. **Thread-safe**: Birden fazla thread güvenli şekilde kullanabilir
2. **Atomic updates**: Değişim atomik (yarım kalmaz)
3. **Conflation**: En son değeri tutar (eski değerleri atlar)
4. **No initial null**: Her zaman bir değer var (null olabilir ama empty değil)

---

## StateFlow'da Immutability

### ❌ Yanlış Yaklaşım: Direct Mutation

```kotlin
private val _articles = MutableStateFlow<List<Article>>(emptyList())

fun addArticle(article: Article) {
    // ❌ DERLENMEZ! List immutable, add() metodu yok
    _articles.value.add(article)
}
```

**Neden derlenmez?**

```kotlin
MutableStateFlow<List<Article>>
//               ^^^^ List interface immutable!

// List interface'de add() yok:
interface List<out E> : Collection<E> {
    fun get(index: Int): E
    // add() metodu yok! ❌
}
```

### ✅ Doğru Yaklaşım: Yeni Liste Oluştur

```kotlin
private val _articles = MutableStateFlow<List<Article>>(emptyList())

// ✅ Yöntem 1: Plus operator
fun addArticle(article: Article) {
    _articles.value = _articles.value + article
}

// ✅ Yöntem 2: update {} fonksiyonu
fun addArticle(article: Article) {
    _articles.update { currentList ->
        currentList + article
    }
}

// ✅ Yöntem 3: buildList {}
fun addArticle(article: Article) {
    _articles.value = buildList {
        addAll(_articles.value)
        add(article)
    }
}

// ✅ Yöntem 4: toMutableList() + toList()
fun addArticle(article: Article) {
    _articles.value = _articles.value.toMutableList().apply {
        add(article)
    }.toList()
}
```

### Neden Yeni Liste?

**Sebep 1: StateFlow sadece referans değişince emit eder**

```kotlin
val list = mutableListOf("Ali")
_articles.value = list

list.add("Veli")  // Referans aynı, emit ETMEZ! ❌

// StateFlow emit kontrolü:
if (newValue !== oldValue) {  // Referans karşılaştırması
    emit(newValue)
}
```

**Sebep 2: Immutability garantisi**

```kotlin
// ✅ Yeni liste oluştur
_articles.value = _articles.value + article

// Her emit yeni referans:
// emit 1: List@123456 ["Ali"]
// emit 2: List@789012 ["Ali", "Veli"]  (farklı referans!)
```

### Update Fonksiyonu (Thread-safe!)

```kotlin
// ✅ update {} thread-safe
_articles.update { currentList ->
    currentList + article  // Thread-safe! ✅
}

// update {} implementation (basitleştirilmiş):
fun <T> MutableStateFlow<T>.update(function: (T) -> T) {
    while (true) {
        val prevValue = value
        val nextValue = function(prevValue)
        if (compareAndSet(prevValue, nextValue)) {
            return
        }
    }
}
```

**Avantaj:**

```kotlin
// Thread 1
_articles.update { it + article1 }

// Thread 2 (aynı anda)
_articles.update { it + article2 }

// update {} garantisi:
// Sonuç: [article1, article2] veya [article2, article1]
// Hiçbir update kaybolmaz! ✅
```

---

## Android'de Gerçek Örnekler

### Örnek 1: UI State Yönetimi (SSOT + Immutability)

```kotlin
// UI State - Immutable sealed class
sealed class ArticleUiState {
    object Loading : ArticleUiState()
    data class Success(val articles: List<Article>) : ArticleUiState()
    data class Error(val message: String) : ArticleUiState()
}

// ViewModel - SSOT
class ArticleViewModel(
    private val repository: ArticleRepository
) : ViewModel() {

    // Private mutable state (SSOT)
    private val _uiState = MutableStateFlow<ArticleUiState>(ArticleUiState.Loading)

    // Public immutable state
    val uiState: StateFlow<ArticleUiState> = _uiState.asStateFlow()

    init {
        loadArticles()
    }

    fun loadArticles() {
        viewModelScope.launch {
            _uiState.value = ArticleUiState.Loading

            repository.getArticles().fold(
                onSuccess = { articles ->
                    _uiState.value = ArticleUiState.Success(articles)
                },
                onFailure = { error ->
                    _uiState.value = ArticleUiState.Error(error.message ?: "Unknown error")
                }
            )
        }
    }

    fun refresh() {
        loadArticles()
    }
}

// Activity - Observer
class ArticleActivity : AppCompatActivity() {
    private val viewModel: ArticleViewModel by viewModels()

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_article)

        // ❌ Activity'de veri tutmuyoruz!
        lifecycleScope.launch {
            viewModel.uiState.collect { state ->
                when (state) {
                    is ArticleUiState.Loading -> {
                        progressBar.visibility = View.VISIBLE
                        recyclerView.visibility = View.GONE
                    }
                    is ArticleUiState.Success -> {
                        progressBar.visibility = View.GONE
                        recyclerView.visibility = View.VISIBLE
                        adapter.submitList(state.articles)
                    }
                    is ArticleUiState.Error -> {
                        progressBar.visibility = View.GONE
                        Toast.makeText(this@ArticleActivity, state.message, Toast.LENGTH_SHORT).show()
                    }
                }
            }
        }

        swipeRefresh.setOnRefreshListener {
            viewModel.refresh()
            swipeRefresh.isRefreshing = false
        }
    }
}
```

**Avantajlar:**
- ✅ State sadece ViewModel'de (SSOT)
- ✅ Activity sadece gözlemliyor
- ✅ Immutable state (sealed class)
- ✅ Thread-safe (StateFlow)

### Örnek 2: Liste Güncelleme (Immutability)

```kotlin
class ArticleViewModel(
    private val repository: ArticleRepository
) : ViewModel() {

    private val _articles = MutableStateFlow<List<Article>>(emptyList())
    val articles: StateFlow<List<Article>> = _articles.asStateFlow()

    // ✅ Yeni makale ekle (immutable)
    fun addArticle(article: Article) {
        _articles.update { currentList ->
            currentList + article  // Yeni liste
        }
    }

    // ✅ Makale sil (immutable)
    fun removeArticle(articleId: String) {
        _articles.update { currentList ->
            currentList.filter { it.id != articleId }  // Yeni liste
        }
    }

    // ✅ Makale güncelle (immutable)
    fun updateArticle(updatedArticle: Article) {
        _articles.update { currentList ->
            currentList.map { article ->
                if (article.id == updatedArticle.id) {
                    updatedArticle  // Güncellenen makale
                } else {
                    article  // Orijinal makale
                }
            }
        }
    }

    // ✅ Makaleyi favori yap (immutable)
    fun toggleFavorite(articleId: String) {
        _articles.update { currentList ->
            currentList.map { article ->
                if (article.id == articleId) {
                    article.copy(isFavorite = !article.isFavorite)  // Copy kullan!
                } else {
                    article
                }
            }
        }
    }
}
```

### Örnek 3: Repository Pattern (SSOT)

```kotlin
class ArticleRepository(
    private val apiService: NewsApiService,
    private val articleDao: ArticleDao,
    private val ioDispatcher: CoroutineDispatcher = Dispatchers.IO
) {
    // ✅ SSOT - Veri sadece burada
    private val _articles = MutableStateFlow<List<Article>>(emptyList())
    val articles: StateFlow<List<Article>> = _articles.asStateFlow()

    // İlk yükleme
    init {
        // Local'den başlat
        CoroutineScope(ioDispatcher).launch {
            val localArticles = articleDao.getAll().map { it.toDomain() }
            _articles.value = localArticles
        }
    }

    // Network'ten refresh
    suspend fun refresh(): Result<Unit> = withContext(ioDispatcher) {
        try {
            // API'den al
            val apiArticles = apiService.getArticles()
            val articles = apiArticles.map { it.toDomain() }

            // Local'e kaydet
            articleDao.deleteAll()
            articleDao.insertAll(articles.map { it.toEntity() })

            // SSOT'u güncelle
            _articles.value = articles

            Result.success(Unit)
        } catch (e: Exception) {
            Result.failure(e)
        }
    }

    // Single article al
    suspend fun getArticle(id: String): Result<Article> = withContext(ioDispatcher) {
        try {
            val apiArticle = apiService.getArticle(id)
            val article = apiArticle.toDomain()

            // SSOT'u güncelle (tek makaleyi güncelle)
            _articles.update { currentList ->
                currentList.map {
                    if (it.id == id) article else it
                }
            }

            Result.success(article)
        } catch (e: Exception) {
            // Local'den dene
            articleDao.getById(id)?.let {
                Result.success(it.toDomain())
            } ?: Result.failure(e)
        }
    }
}
```

---

## Immutability ve SOLID İlişkisi

| SOLID Prensibi | Immutability/SSOT'daki Karşılığı |
|----------------|----------------------------------|
| **Single Responsibility** | State sadece bir yerde tutulur (Repository - SSOT) |
| **Open/Closed** | State değişimi copy ile (yeni nesne, orijinal korunur) |
| **Liskov Substitution** | StateFlow her yerde aynı şekilde davranır (immutable contract) |
| **Interface Segregation** | StateFlow (read-only) ve MutableStateFlow (write) ayrı |
| **Dependency Inversion** | StateFlow interface'e bağımlı (implementation gizli) |

---

## Ne Zaman Kullanılır?

### ✅ Immutability Kullan:

- **Multi-threading** varsa (kesinlikle!)
- **State management** yapıyorsan (ViewModel, Repository)
- **Reactive programming** kullanıyorsan (StateFlow, LiveData)
- **Test yazıyorsan** (immutable nesneler tahmin edilebilir)
- **Side effect'lerden** kaçınmak istiyorsan

### ✅ SSOT Kullan:

- **Orta/büyük projelerde** (10+ ekran)
- **Clean Architecture** kullanıyorsan (zorunlu!)
- **Repository Pattern** kullanıyorsan
- **Senkronizasyon problemlerinden** kaçınmak istiyorsan
- **Birden fazla screen aynı veriyi** kullanıyorsa

### ❌ Kullanma (İstisnalar):

- **Çok basit, tek ekranlı** prototipler
- **Throw-away kod** (bir kere kullanılacak)
- **Performance critical** kod (nadiren, ölç önce!)
- **Local değişkenler** (fonksiyon içi temporary değişkenler)

---

## Pratik Kurallar

### Altın Kurallar

1. **State = Immutable**
   ```kotlin
   // ✅ Doğru
   data class User(val name: String, val age: Int)

   // ❌ Yanlış
   data class User(var name: String, var age: Int)
   ```

2. **SSOT = Repository**
   ```kotlin
   // ✅ Repository'de tut
   class Repository {
       private val _data = MutableStateFlow<List<Item>>(emptyList())
       val data: StateFlow<List<Item>> = _data.asStateFlow()
   }

   // ❌ ViewModel'de tutma
   class ViewModel {
       private var data: List<Item> = emptyList()  // SSOT ihlali!
   }
   ```

3. **Public = Read-only**
   ```kotlin
   // ✅ Doğru
   private val _state = MutableStateFlow<State>(...)
   val state: StateFlow<State> = _state.asStateFlow()

   // ❌ Yanlış
   val state = MutableStateFlow<State>(...)  // Herkes değiştirebilir!
   ```

4. **Update = Copy**
   ```kotlin
   // ✅ Doğru
   _state.update { it.copy(isLoading = false) }

   // ❌ Yanlış
   _state.value.isLoading = false  // Mutation!
   ```

---

## Özet Tablo

| Kriter | Mutable | Immutable |
|--------|---------|-----------|
| **Değiştirme** | ✅ Direkt değiştir | ❌ Yeni oluştur (copy) |
| **Thread Safety** | ❌ Unsafe (race condition) | ✅ Safe (değişmez) |
| **Predictability** | ❌ Beklenmeyen değişiklik | ✅ Tahmin edilebilir |
| **Testing** | ❌ Zor (state değişir) | ✅ Kolay (state sabit) |
| **Side Effects** | ❌ Var | ✅ Yok |
| **Performance** | ✅ Daha hızlı (bazen) | ⚠️ Copy overhead (genelde ihmal edilebilir) |
| **Kotlin** | `var`, `mutableListOf` | `val`, `listOf`, `copy()` |
| **Android** | ❌ Tavsiye edilmez | ✅ Best practice |

---

## Kaynaklar

- [Android Architecture Guide - UI Layer](https://developer.android.com/topic/architecture/ui-layer)
- [Kotlin Flow Documentation](https://kotlinlang.org/docs/flow.html)
- [StateFlow and SharedFlow - Official Guide](https://developer.android.com/kotlin/flow/stateflow-and-sharedflow)
- [Immutability - Effective Kotlin](https://kt.academy/article/ek-immutability)
- [Clean Architecture - Robert C. Martin](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)
