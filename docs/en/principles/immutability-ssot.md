# Immutability and Single Source of Truth (SSOT)

## Introduction

**Immutability** and **Single Source of Truth (SSOT)** are fundamental principles of safe and predictable state management in modern Android applications.

**Simple Explanation:**
- **Immutability**: Not being able to change an object after creating it
- **SSOT**: Keeping data in only one place, other places observe

**Real-life analogy:**
- **Mutable**: Whiteboard (write, erase, write again)
- **Immutable**: Paper (once you write, you can't change it. Use new paper)

> **Key Idea:** "Don't change state, create new state. Keep data in one place, observe everywhere."

**Important Note:** These principles are the cornerstones of Android Architecture Guide. They are the foundation of reactive programming and modern state management.

---

## Immutability

### `val` vs `var`

```kotlin
// var - Mutable (Değişebilir)
var name = "Ali"
name = "Veli"  // ✅ Değişir

// val - Immutable (Değiştirilemez)
val surname = "Yılmaz"
surname = "Demir"  // ❌ DERLENMEZ!
```

**`val` = Makes the reference immutable**

```kotlin
val age: Int = 25
age = 30  // ❌ DERLENMEZ

// Görsel:
// Memory: [25] ← age referansı (değiştirilemez!)
```

### ⚠️ Warning: `val` Is Not Always Immutable!

```kotlin
// ⚠️ Referans immutable, içerik mutable!
val list = mutableListOf("Ali")
list = mutableListOf("Veli")  // ❌ Referans değişemez
list.add("Veli")  // ✅ İçerik değişir!

println(list)  // [Ali, Veli]
```

**Visual:**
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

**For full immutability:**

```kotlin
// ✅ Hem referans hem içerik immutable
val list = listOf("Ali")  // Immutable List
list.add("Veli")  // ❌ DERLENMEZ (add() metodu yok!)
```

---

## Primitive Types vs Collections

### Primitive Types - Completely Immutable

```kotlin
val age: Int = 25
val name: String = "Ali"
val isActive: Boolean = true

// Hiçbiri değiştirilemez! ✅
age = 30  // ❌
name = "Veli"  // ❌
isActive = false  // ❌
```

### Collections - Be Careful!

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

## Data Class and Immutability

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

### How Does Copy Work?

```kotlin
data class User(val name: String, val age: Int)

val user1 = User("Ali", 25)
val user2 = user1.copy(age = 30)

// Farklı referanslar!
println(user1 === user2)  // false
println(user1.hashCode())  // Örnek: 123456
println(user2.hashCode())  // Örnek: 789012
```

**Visual:**
```
Memory:
┌─────────────────┐
│ user1 (Ali, 25) │  Referans: 0x123456
└─────────────────┘

┌─────────────────┐
│ user2 (Ali, 30) │  Referans: 0x789012 (YENİ NESNE!)
└─────────────────┘
```

**Advantage: Thread Safety**

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

## Advantages of Immutability

### 1. **Thread Safety** ⭐⭐⭐

**❌ Mutable (Dangerous):**

```kotlin
var count = 0

// Thread 1
count++  // count = 1

// Thread 2 (aynı anda)
count++  // count = ??? (1 mi 2 mi?)

// Race condition! ❌
```

**✅ Immutable (Safe):**

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

### 2. **Predictable State**

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

### What is SSOT?

**Single Source of Truth (SSOT)** = The principle of **keeping data in only one place**.

> **"A single source for every data, other places observe this source"**

### ❌ SSOT Violation

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

**Problems:**

1. **Synchronization problem**: Which is the current data?
2. **Memory waste**: Same data in 4 places
3. **Inconsistency**: List in Activity may be different from ViewModel
4. **Testing difficulty**: You need to test each place separately

### ✅ SSOT Implementation

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

**Now:**
- ✅ Data is kept **only in Repository** (SSOT!)
- ✅ ViewModel observes (doesn't hold data)
- ✅ Activity observes (doesn't hold data)
- ✅ Single source, no synchronization problem!

### SSOT Visual

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

## StateFlow and Immutability

### What is StateFlow?

**StateFlow** = Kotlin Coroutines' **hot stream** structure. It's the modern version of the Observable pattern.

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

### StateFlow Guarantees

1. **Thread-safe**: Multiple threads can use it safely
2. **Atomic updates**: Changes are atomic (won't be left incomplete)
3. **Conflation**: Keeps the latest value (skips old values)
4. **No initial null**: Always has a value (can be null but not empty)

---

## Immutability in StateFlow

### ❌ Wrong Approach: Direct Mutation

```kotlin
private val _articles = MutableStateFlow<List<Article>>(emptyList())

fun addArticle(article: Article) {
    // ❌ DERLENMEZ! List immutable, add() metodu yok
    _articles.value.add(article)
}
```

**Why doesn't it compile?**

```kotlin
MutableStateFlow<List<Article>>
//               ^^^^ List interface immutable!

// List interface'de add() yok:
interface List<out E> : Collection<E> {
    fun get(index: Int): E
    // add() metodu yok! ❌
}
```

### ✅ Correct Approach: Create New List

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

### Why New List?

**Reason 1: StateFlow only emits when reference changes**

```kotlin
val list = mutableListOf("Ali")
_articles.value = list

list.add("Veli")  // Referans aynı, emit ETMEZ! ❌

// StateFlow emit kontrolü:
if (newValue !== oldValue) {  // Referans karşılaştırması
    emit(newValue)
}
```

**Reason 2: Immutability guarantee**

```kotlin
// ✅ Yeni liste oluştur
_articles.value = _articles.value + article

// Her emit yeni referans:
// emit 1: List@123456 ["Ali"]
// emit 2: List@789012 ["Ali", "Veli"]  (farklı referans!)
```

### Update Function (Thread-safe!)

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

**Advantage:**

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

## Real Examples in Android

### Example 1: UI State Management (SSOT + Immutability)

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

**Advantages:**
- ✅ State only in ViewModel (SSOT)
- ✅ Activity only observing
- ✅ Immutable state (sealed class)
- ✅ Thread-safe (StateFlow)

### Example 2: List Update (Immutability)

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

### Example 3: Repository Pattern (SSOT)

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

## Immutability and SOLID Relationship

| SOLID Principle | Immutability/SSOT Equivalent |
|----------------|----------------------------------|
| **Single Responsibility** | State is kept in only one place (Repository - SSOT) |
| **Open/Closed** | State change with copy (new object, original preserved) |
| **Liskov Substitution** | StateFlow behaves the same everywhere (immutable contract) |
| **Interface Segregation** | StateFlow (read-only) and MutableStateFlow (write) are separate |
| **Dependency Inversion** | StateFlow depends on interface (implementation hidden) |

---

## When to Use?

### ✅ Use Immutability:

- When **multi-threading** exists (definitely!)
- When doing **state management** (ViewModel, Repository)
- When using **reactive programming** (StateFlow, LiveData)
- When **writing tests** (immutable objects are predictable)
- When you want to avoid **side effects**

### ✅ Use SSOT:

- In **medium/large projects** (10+ screens)
- When using **Clean Architecture** (mandatory!)
- When using **Repository Pattern**
- When you want to avoid **synchronization problems**
- When **multiple screens use the same data**

### ❌ Don't Use (Exceptions):

- **Very simple, single-screen** prototypes
- **Throw-away code** (used once)
- **Performance critical** code (rarely, measure first!)
- **Local variables** (temporary variables inside functions)

---

## Practical Rules

### Golden Rules

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

## Summary Table

| Criterion | Mutable | Immutable |
|--------|---------|-----------|
| **Modification** | ✅ Modify directly | ❌ Create new (copy) |
| **Thread Safety** | ❌ Unsafe (race condition) | ✅ Safe (immutable) |
| **Predictability** | ❌ Unexpected changes | ✅ Predictable |
| **Testing** | ❌ Hard (state changes) | ✅ Easy (state fixed) |
| **Side Effects** | ❌ Has | ✅ None |
| **Performance** | ✅ Faster (sometimes) | ⚠️ Copy overhead (usually negligible) |
| **Kotlin** | `var`, `mutableListOf` | `val`, `listOf`, `copy()` |
| **Android** | ❌ Not recommended | ✅ Best practice |

---

## Resources

- [Android Architecture Guide - UI Layer](https://developer.android.com/topic/architecture/ui-layer)
- [Kotlin Flow Documentation](https://kotlinlang.org/docs/flow.html)
- [StateFlow and SharedFlow - Official Guide](https://developer.android.com/kotlin/flow/stateflow-and-sharedflow)
- [Immutability - Effective Kotlin](https://kt.academy/article/ek-immutability)
- [Clean Architecture - Robert C. Martin](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)
