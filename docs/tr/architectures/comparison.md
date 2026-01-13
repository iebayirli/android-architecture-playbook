# Mimari Karşılaştırması

## Giriş

Bu doküman, Android'de yaygın kullanılan mimari pattern'leri (MVC, MVP, MVVM, Clean Architecture) karşılaştırır ve her birinin güçlü/zayıf yönlerini objektif olarak inceler.

---

## İçindekiler

1. [Detaylı Karşılaştırma](#detaylı-karşılaştırma)
2. [Veri Akışı Karşılaştırması](#veri-akışı-karşılaştırması)
3. [Mimari Bağımsız Yaygın Pratikler](#mimari-bağımsız-yaygın-pratikler)

---

## Detaylı Karşılaştırma

### 1. Mimari Yapı

#### MVC

```
┌─────────────────────────────────────┐
│   Controller (Activity/Fragment)   │ ← God Object!
│   ├─ UI Logic                       │
│   ├─ Business Logic                 │
│   ├─ Data Access                    │
│   └─ Lifecycle Management           │
└──────────┬──────────────────────────┘
           │
           ├──► Model (Data)
           └──► View (XML Layout)

Özellikler:
- Controller (Activity/Fragment) her şeyi bilir
- Model ve View'e doğrudan erişim
- Tight coupling (sıkı bağımlılık)
- Separation of concerns zayıf
```

#### MVP

```
┌──────────────────┐      ┌──────────────────┐
│  View (Activity) │◄────►│    Presenter     │
│  ├─ UI Rendering │      │  ├─ Business     │
│  └─ User Events  │      │  └─ Logic        │
└──────────────────┘      └────────┬─────────┘
                                   │
                                   ▼
                          ┌────────────────┐
                          │  Model (Data)  │
                          └────────────────┘

Özellikler:
- Presenter, View referansını tutar (interface üzerinden)
- İki yönlü iletişim (View ↔ Presenter)
- View, Presenter'a bağımlı
- Logic ayrıldı ancak View leak riski var
```

#### MVVM

```
┌──────────────────┐      ┌──────────────────┐
│  View (Activity) │      │    ViewModel     │
│  ├─ UI Rendering │      │  ├─ UI State     │
│  └─ Observe      │◄─────│  └─ Logic        │
└──────────────────┘      └────────┬─────────┘
                                   │
                                   ▼
                          ┌────────────────┐
                          │ Repository     │
                          └────────────────┘

Özellikler:
- ViewModel, View referansını tutmaz
- Tek yönlü iletişim (ViewModel → View)
- View, ViewModel'i gözlemler (observe)
- Reactive ve Lifecycle-aware
- Memory leak riski düşük
```

#### Clean Architecture

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
┌──────────────┴──────────────────────┐
│   Data (Data Sources)               │
│   ├─ Repository Implementations     │
│   ├─ API (Retrofit)                 │
│   └─ Database (Room)                │
└─────────────────────────────────────┘

Özellikler:
- Katmanlar arası net sınırlar (Dependency Rule)
- İç katmanlar dış katmanları bilmez
- Domain layer framework bağımsız (Pure Kotlin)
- Her katman izole edilebilir ve test edilebilir
- Dependency Inversion prensibi uygulanır
```

---

### 2. Test Edilebilirlik

#### MVC - Test Edilemez ❌

```kotlin
class UserActivity : AppCompatActivity() {
    fun loadUsers() {
        val users = repository.getUsers() // Network!
        recyclerView.adapter = UserAdapter(users) // UI!
    }
}

// Test?
@Test
fun testLoadUsers() {
    val activity = UserActivity() // ❌ DERLENMEZ!
    // Activity mock'lanamaz!
}
```

#### MVP - Test Edilebilir ✅

```kotlin
// Presenter testi
@Test
fun `when loadUsers called, should show users in view`() {
    // Given
    val mockView = mock<UserContract.View>()
    val mockRepository = mock<UserRepository>()
    whenever(mockRepository.getUsers()).thenReturn(listOf(user1, user2))

    val presenter = UserPresenter(mockView, mockRepository)

    // When
    presenter.loadUsers()

    // Then
    verify(mockView).showUsers(listOf(user1, user2))
}
```

#### MVVM - Çok Kolay Test ✅✅

```kotlin
@Test
fun `when init, should load users`() = runTest {
    // Given
    val mockRepository = mock<UserRepository>()
    whenever(mockRepository.getUsers()).thenReturn(listOf(user1, user2))

    // When
    val viewModel = UserViewModel(mockRepository)

    // Then
    assertEquals(listOf(user1, user2), viewModel.users.value)
}
```

#### Clean Architecture - Katman Katman Test ✅✅✅

```kotlin
// Use Case Test (Domain Layer)
@Test
fun `GetUsersUseCase should return users from repository`() = runTest {
    val mockRepository = mock<UserRepository>()
    whenever(mockRepository.getUsers()).thenReturn(Result.success(users))

    val useCase = GetUsersUseCase(mockRepository)
    val result = useCase()

    assertEquals(users, result.getOrNull())
}

// Repository Test (Data Layer)
@Test
fun `when API fails, should fallback to cache`() = runTest {
    // Test implementation...
}

// ViewModel Test (Presentation Layer)
@Test
fun `when loadUsers succeeds, should emit Success state`() = runTest {
    // Test implementation...
}
```

---

### 3. Memory Leak Riski

#### MVC - Yüksek Risk 🔴

```kotlin
class UserActivity : AppCompatActivity() {
    private val job = CoroutineScope(Dispatchers.IO).launch {
        // ❌ Activity leak! Job cancel edilmiyor
        delay(10000)
        val users = repository.getUsers()
        // Activity destroyed olmuş olabilir!
    }
}
```

#### MVP - Orta Risk 🟡

```kotlin
class UserPresenter(
    private val view: UserContract.View // ❌ View referansı leak olabilir!
) {
    fun loadUsers() {
        // Async call bitince view çağrılır
        // View destroy olmuşsa? → CRASH!
    }
}

// Çözüm: Manual cleanup
override fun onDestroy() {
    presenter.detachView() // Manuel temizlik gerekli
    super.onDestroy()
}
```

#### MVVM - Düşük Risk 🟢

```kotlin
class UserViewModel : ViewModel() {
    fun loadUsers() {
        viewModelScope.launch {
            // ✅ ViewModel destroy olunca otomatik iptal!
            val users = repository.getUsers()
            _users.value = users
        }
    }
}
```

#### Clean Architecture - Çok Düşük Risk 🟢

```kotlin
// Her katman lifecycle-aware
class UserViewModel(
    private val getUsersUseCase: GetUsersUseCase // ✅ Interface
) : ViewModel() {
    // ViewModel scope kullanıyor
    // Otomatik cleanup
}
```

---

## Veri Akışı Karşılaştırması

### MVC - Karmaşık, Çift Yönlü

```
User Input → Controller ──┐
                          ├──► Model
                          └──► View ──► Update ──► Controller
                                                      │
                                                      └──► Loop!

Sorun: Veri akışı tahmin edilemez
```

### MVP - Temiz ama Manuel

```
User Input → View (Activity)
              │
              ▼
            Presenter
              │
              ├──► Model (Data)
              │
              └──► View.showData() ✅

Veri akışı: View → Presenter → Model → Presenter → View
```

### MVVM - Reactive, Tek Yönlü

```
User Input → ViewModel
              │
              ├──► Repository
              │       │
              │       ▼
              └──► LiveData/StateFlow ──► View (Observe)
                       ▲
                       │
                   Reactive Update (Otomatik!)

Veri akışı: View → ViewModel → Repository → StateFlow → View
```

### Clean Architecture - Katmanlı, Dependency Rule

```
User Input → Activity
              │
              ▼
            ViewModel
              │
              ▼
            Use Case (Domain)
              │
              ▼
            Repository Interface (Domain)
              ▲
              │
            Repository Impl (Data)
              │
              ├──► API
              └──► Database

Dependency Rule: İçteki katmanlar dıştakileri bilmez!
```

---

## Mimari Bağımsız Yaygın Pratikler

### 1. Repository Pattern

```kotlin
// Repository pattern, data source soyutlaması sağlar
class UserRepository { ... }

// MVC gibi basit mimarilerde bile kullanılabilir:
class UserActivity : AppCompatActivity() {
    private val repository = UserRepository()
}
```

### 2. Dependency Injection

```kotlin
// DI, bağımlılıkların yönetimini kolaylaştırır
class UserViewModel @Inject constructor(
    private val repository: UserRepository
) : ViewModel()
```

### 3. Reactive Programming

```kotlin
// Reactive yaklaşım, veri akışını yönetir
private val _users = MutableStateFlow<List<User>>(emptyList())
val users: StateFlow<List<User>> = _users.asStateFlow()
```

### 4. Immutable State

```kotlin
// Immutable state, thread safety sağlar
data class User(val name: String, val age: Int)

// Mutable alternatif (değiştirilebilir)
data class User(var name: String, var age: Int)
```

---

## Kaynaklar

- [Android Architecture Guide - Google](https://developer.android.com/topic/architecture)
- [Guide to app architecture - Modern Best Practices](https://developer.android.com/topic/architecture/recommendations)
- [Clean Architecture - Robert C. Martin](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)
- [MVVM vs MVP - Detailed Comparison](https://proandroiddev.com/mvp-vs-mvvm-vs-mvi-6843b9313b00)

---

## Özet

**Mimari Pattern'ler Hakkında Gözlemler:**

Her mimari pattern'in farklı proje bağlamlarında avantajları ve dezavantajları bulunmaktadır. Mimari seçimi, proje gereksinimlerine, takım yapısına ve uzun vadeli hedeflere bağlı olarak değişkenlik gösterir.

**Endüstri Trendleri (2025):**

- MVVM ve Clean Architecture, Google'ın Android Architecture Guide'ında önerilen ve endüstride yaygın olarak benimsenen yaklaşımlardır
- MVC ve MVP, yeni projelerde nadiren tercih edilmekte, çoğunlukla legacy kod bakımında görülmektedir
- Jetpack Compose'un yaygınlaşmasıyla reactive state management yaklaşımları daha fazla benimsenmiştir

**Genel Değerlendirme:**

- Basit projeler genellikle daha az katmanlı mimarilerle yönetilebilir
- Kompleks projeler, katmanlı ve modüler mimarilerden fayda sağlayabilir
- Test edilebilirlik ve bakım kolaylığı, mimari seçiminde önemli faktörlerdir
