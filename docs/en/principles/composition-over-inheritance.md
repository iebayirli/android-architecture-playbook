# Composition Over Inheritance

## Introduction

**Composition Over Inheritance** is the principle of "compose from components rather than inheriting". It is one of the fundamental principles recommended in the Gang of Four (GoF) Design Patterns book.

**Simple Explanation:**
- **Inheritance**: "IS-A" relationship → "A car is a vehicle"
- **Composition**: "HAS-A" relationship → "A car has an engine"

**Real-life analogy:**
- **Inheritance**: Changing a robot's DNA to create a new robot (fragile, risky)
- **Composition**: Making a robot by assembling parts like LEGO blocks (flexible, safe)

> **Core Idea:** "Instead of inheriting, use the behaviors you need as components"

**Important Note:** Composition is not always better! Inheritance should be used for framework requirements and genuine IS-A relationships.

---

## Problems with Inheritance

### 1. **Fragile Base Class Problem**

When the parent class changes, child classes are affected unexpectedly.

```kotlin
// ❌ Problematic example
open class Vehicle {
    open fun start() {
        initializeSensors()  // Tüm araçlar sensör kullanıyor varsayımı
        println("Starting vehicle")
    }

    private fun initializeSensors() {
        println("Initializing sensors...")
    }
}

class Bicycle : Vehicle() {
    // Sorun: Bisikletin sensörü yok ama initializeSensors() çalışıyor!
    override fun start() {
        super.start()  // Parent'ın initializeSensors() çağrılıyor
        println("Starting to pedal")
    }
}

// Kullanım
val bicycle = Bicycle()
bicycle.start()
// Output:
// Initializing sensors...  ❌ Bisikletin sensörü yok!
// Starting vehicle
// Starting to pedal
```

**Problem:**
- Parent class changed → Child classes affected
- Bicycle is unaware of the sensor implementation
- Parent's private methods affect the child

### 2. **Deep Inheritance Hierarchies**

```kotlin
open class Vehicle {
    open fun move() { println("Moving") }
}

open class LandVehicle : Vehicle() {
    open fun driveOnRoad() { println("Driving on road") }
}

open class MotorizedVehicle : LandVehicle() {
    open fun startEngine() { println("Engine started") }
}

open class Car : MotorizedVehicle() {
    open fun openTrunk() { println("Trunk opened") }
}

class ElectricCar : Car() {
    override fun startEngine() {
        // ❌ Sorun: ElectricCar motor sesiyle başlamaz!
        // Ama Car'dan 4 seviye miras aldığı için karmaşık
        println("Silent electric start")
    }
}

// ElectricCar 4 seviye miras almış!
// Vehicle → LandVehicle → MotorizedVehicle → Car → ElectricCar
```

**Problems:**
- Hard to make changes (which level should I modify?)
- Each level adds a dependency
- Complex to test (you need to know all parents)

### 3. **Tight Coupling**

```kotlin
open class BaseActivity : AppCompatActivity() {
    fun showLoading() { /* ... */ }
    fun hideLoading() { /* ... */ }
    fun showError(message: String) { /* ... */ }
    fun setupToolbar() { /* ... */ }
    fun handleDeepLink(url: String) { /* ... */ }
    fun setupAds() { /* ... */ }
    fun trackAnalytics(event: String) { /* ... */ }
}

class ArticleActivity : BaseActivity() {
    // ❌ Sorun: 7 metodu miras aldım ama sadece 2 tanesini kullanıyorum!
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        showLoading()  // Kullanıyorum
        hideLoading()  // Kullanıyorum
        // Diğer 5 metod gereksiz ama yine de var!
    }
}
```

**Problems:**
- If `BaseActivity` changes, `ArticleActivity` is affected
- I inherited methods I don't need
- When testing, I need to mock the entire `BaseActivity`

### 4. **Gorilla-Banana Problem** 🦍🍌

Joe Armstrong (creator of Erlang) says:

> "The problem with object-oriented languages: You wanted a banana but got a gorilla holding the banana and the entire jungle!"

```kotlin
open class BaseViewModel : ViewModel() {
    protected val _loading = MutableStateFlow(false)      // İstiyorum ✅
    val loading = _loading.asStateFlow()                  // İstiyorum ✅

    protected val _error = MutableStateFlow<String?>(null) // İstiyorum ✅
    val error = _error.asStateFlow()                       // İstiyorum ✅

    protected fun handleError(e: Exception) { /* ... */ }  // İstiyorum ✅

    // Aşağıdakiler gereksiz ama miras aldım ❌
    protected fun trackAnalytics(event: String) { /* ... */ }
    protected fun setupRemoteConfig() { /* ... */ }
    protected fun checkSubscription() { /* ... */ }
}

class ArticleViewModel : BaseViewModel() {
    // Sadece loading ve error istedim
    // Ama analytics, remote config, subscription da geldi! ❌
}
```

---

## Composition Solution

### ✅ Correct Approach: Composition

```kotlin
// 1. Her davranışı ayrı interface yap
interface LoadingHandler {
    fun showLoading()
    fun hideLoading()
}

interface ErrorHandler {
    fun showError(message: String)
}

interface ToolbarHandler {
    fun setupToolbar()
}

// 2. Implementation'lar
class LoadingDelegate : LoadingHandler {
    private var isLoading = false

    override fun showLoading() {
        isLoading = true
        println("Loading...")
    }

    override fun hideLoading() {
        isLoading = false
        println("Loaded!")
    }
}

class ErrorDelegate : ErrorHandler {
    override fun showError(message: String) {
        println("Error: $message")
    }
}

// 3. Activity sadece ihtiyacını alır
class ArticleActivity : AppCompatActivity(),
    LoadingHandler by LoadingDelegate(),
    ErrorHandler by ErrorDelegate() {

    // ✅ Sadece loading ve error aldım
    // ✅ Toolbar almadım (gereksizdi)

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        showLoading()  // LoadingDelegate'ten geliyor
        // API call...
        hideLoading()  // LoadingDelegate'ten geliyor
    }
}

// 4. Başka activity farklı kombinasyon alır
class ProfileActivity : AppCompatActivity(),
    ErrorHandler by ErrorDelegate(),
    ToolbarHandler by ToolbarDelegate() {

    // ✅ Sadece error ve toolbar aldım
    // ✅ Loading almadım (gereksizdi)
}
```

**Advantages:**
- ✅ Take what you need, don't take what's unnecessary
- ✅ Low coupling (loose coupling)
- ✅ Easy to test (each delegate is tested separately)
- ✅ High flexibility (can be changed at runtime)

---

## Kotlin Delegation

Kotlin has the **`by`** keyword that makes composition easier.

### What is Delegation?

**Delegation = "You do it, I'll just forward it"**

```kotlin
interface Printer {
    fun print(message: String)
}

class ConsolePrinter : Printer {
    override fun print(message: String) {
        println("Console: $message")
    }
}

// ❌ Manuel delegation (zahmetli)
class Logger(private val printer: Printer) : Printer {
    override fun print(message: String) {
        printer.print(message)  // Manuel olarak delegate ediyoruz
    }
}

// ✅ Kotlin delegation (kolay!)
class Logger(printer: Printer) : Printer by printer {
    // Hiçbir kod yazmadık!
    // Kotlin otomatik olarak printer.print() çağırıyor
}

// Kullanım
val logger = Logger(ConsolePrinter())
logger.print("Hello")  // "Console: Hello"
```

### Kotlin Delegation Advantages

```kotlin
interface Engine {
    fun start()
    fun stop()
    fun getSpeed(): Int
}

class ElectricEngine : Engine {
    override fun start() { println("Electric start") }
    override fun stop() { println("Electric stop") }
    override fun getSpeed() = 100
}

class GasEngine : Engine {
    override fun start() { println("Gas start") }
    override fun stop() { println("Gas stop") }
    override fun getSpeed() = 120
}

// ✅ Delegation ile
class Car(engine: Engine) : Engine by engine {
    // Engine metodları otomatik geldi: start(), stop(), getSpeed()

    fun drive() {
        start()  // engine.start() çağrılır
        println("Driving at ${getSpeed()} km/h")
    }
}

// Kullanım
val electricCar = Car(ElectricEngine())
electricCar.drive()
// Output:
// Electric start
// Driving at 100 km/h

val gasCar = Car(GasEngine())
gasCar.drive()
// Output:
// Gas start
// Driving at 120 km/h
```

**Advantages:**
- ✅ No boilerplate code (automatic with by keyword)
- ✅ You can change engine at runtime
- ✅ Easy to test (inject mock engine)

---

## Kotlin Delegation vs Multiple Inheritance

### What is Multiple Inheritance?

In some languages (like C++), a class can inherit from **multiple classes**:

```cpp
// C++ - Multiple Inheritance
class Engine {
    void start() { cout << "Engine started"; }
};

class GPS {
    void navigate() { cout << "Navigating"; }
};

// ✅ C++ multiple inheritance destekler
class Car : public Engine, public GPS {
    // Car hem Engine hem GPS'ten miras aldı
};

Car car;
car.start();      // Engine'den
car.navigate();   // GPS'den
```

**Java/Kotlin:** Multiple class inheritance **not supported**! ❌

```kotlin
// ❌ Kotlin - Multiple class inheritance YASAK!
open class Engine {
    fun start() { println("Engine started") }
}

open class GPS {
    fun navigate() { println("Navigating") }
}

class Car : Engine(), GPS() {
    // ❌ DERLENMEZ! Sadece 1 class'tan miras alabilirsin
}
```

### Kotlin Delegation = Safe Multiple Inheritance ✅

Kotlin delegation gets the **advantages** of multiple inheritance but avoids the **problems**:

```kotlin
// ✅ Kotlin - Multiple interface implementation + Delegation
interface Engine {
    fun start()
}

class ElectricEngine : Engine {
    override fun start() { println("Electric start") }
}

interface GPS {
    fun navigate()
}

class GoogleMaps : GPS {
    override fun navigate() { println("Google Maps navigating") }
}

// ✅ Kotlin delegation ile "sanki multiple inheritance gibi"
class Car(
    engine: Engine,
    gps: GPS
) : Engine by engine, GPS by gps {
    // start() ve navigate() otomatik geldi!
    // Sanki hem Engine hem GPS'ten miras almış gibi
}

val car = Car(ElectricEngine(), GoogleMaps())
car.start()      // "Electric start"
car.navigate()   // "Google Maps navigating"
```

### Diamond Problem 💎

The biggest problem of multiple inheritance:

```cpp
// C++ - Diamond Problem
class Vehicle {
    void start() { cout << "Vehicle started"; }
};

class Car : public Vehicle {
    void start() { cout << "Car started"; }
};

class Boat : public Vehicle {
    void start() { cout << "Boat started"; }
};

// ❌ Sorun: AmphibiousCar hem Car hem Boat'tan miras alıyor
class AmphibiousCar : public Car, public Boat {
    // start() metodunu kim çağıracak? Car.start() mı Boat.start() mı?
};

AmphibiousCar amphibious;
amphibious.start();  // ❌ BELIRSIZ! Hangi start()?
```

**Visual:**
```
       Vehicle
       /     \
     Car     Boat
       \     /
    AmphibiousCar

Elmas şekli → Diamond Problem
```

### How Does Kotlin Solve the Diamond Problem?

```kotlin
// ✅ Kotlin - Diamond Problem yok!
interface Startable {
    fun start()
}

class CarEngine : Startable {
    override fun start() { println("Car engine start") }
}

class BoatEngine : Startable {
    override fun start() { println("Boat engine start") }
}

// ❌ Eğer ikisini de aynı interface için delegate edersen DERLENMEZ!
class AmphibiousCar(
    carEngine: Startable,
    boatEngine: Startable
) : Startable by carEngine, Startable by boatEngine {
    // ❌ HATA: "Multiple implementations of Startable"
}

// ✅ Çözüm 1: Manuel override et, sen karar ver!
class AmphibiousCar(
    private val carEngine: Startable,
    private val boatEngine: Startable
) : Startable {
    override fun start() {
        // SEN karar veriyorsun, belirsizlik yok!
        if (onLand) {
            carEngine.start()
        } else {
            boatEngine.start()
        }
    }
}

// ✅ Çözüm 2: Farklı interface'ler kullan
interface LandVehicle {
    fun startOnLand()
}

interface WaterVehicle {
    fun startOnWater()
}

class AmphibiousCar(
    landEngine: LandVehicle,
    waterEngine: WaterVehicle
) : LandVehicle by landEngine, WaterVehicle by waterEngine {
    // ✅ Conflict yok, her interface ayrı!
}
```

**Kotlin's approach:**
- ❌ No ambiguity (Diamond Problem solved)
- ✅ If there's a conflict, **compile-time error** (no runtime crashes)
- ✅ Developer in control (manual override required)

### Comparison Table

| Feature | Multiple Inheritance (C++) | Kotlin Delegation |
|---------|---------------------------|-------------------|
| **Multiple class inheritance** | ✅ Yes | ❌ No (only 1 class) |
| **Multiple interfaces** | ✅ Yes | ✅ Yes |
| **Diamond Problem** | ❌ Exists (major issue!) | ✅ None (compile-time error) |
| **Ambiguity** | ❌ Runtime problem | ✅ Caught at compile-time |
| **Boilerplate code** | ✅ Low | ✅ Low (by keyword) |
| **Safety** | ❌ Risky | ✅ Safe |
| **Flexibility** | ❌ Low (compile-time fixed) | ✅ High (can be changed at runtime) |

**Conclusion:**

> **"Kotlin delegation provides the advantages of multiple inheritance but solves problems like the Diamond Problem. Composition + Delegation = Safe Multiple Inheritance"**

---

## Real Examples in Android

### Example 1: ViewModel State Management

**❌ Inheritance:**

```kotlin
open class BaseViewModel : ViewModel() {
    protected val _loading = MutableStateFlow(false)
    val loading = _loading.asStateFlow()

    protected val _error = MutableStateFlow<String?>(null)
    val error = _error.asStateFlow()

    protected fun handleError(e: Exception) {
        _error.value = e.message
    }
}

class ArticleViewModel : BaseViewModel() {
    // loading, error miras aldı
}

class ProfileViewModel : BaseViewModel() {
    // loading, error miras aldı
}
```

**Problems:**
- If BaseViewModel changes, all ViewModels are affected
- BaseViewModel needs to be mocked for testing
- Some ViewModels may not want loading but are forced to take it

**✅ Composition:**

```kotlin
// 1. State interface'leri
interface LoadingState {
    val loading: StateFlow<Boolean>
    fun setLoading(isLoading: Boolean)
}

interface ErrorState {
    val error: StateFlow<String?>
    fun setError(message: String?)
}

// 2. Delegate implementations
class LoadingStateDelegate : LoadingState {
    private val _loading = MutableStateFlow(false)
    override val loading: StateFlow<Boolean> = _loading.asStateFlow()

    override fun setLoading(isLoading: Boolean) {
        _loading.value = isLoading
    }
}

class ErrorStateDelegate : ErrorState {
    private val _error = MutableStateFlow<String?>(null)
    override val error: StateFlow<String?> = _error.asStateFlow()

    override fun setError(message: String?) {
        _error.value = message
    }
}

// 3. ViewModel composition ile
class ArticleViewModel(
    loadingState: LoadingState = LoadingStateDelegate(),
    errorState: ErrorState = ErrorStateDelegate()
) : ViewModel(),
    LoadingState by loadingState,
    ErrorState by errorState {

    // loading, setLoading(), error, setError() otomatik var!

    fun loadArticle() {
        setLoading(true)

        viewModelScope.launch {
            try {
                // API call...
                setLoading(false)
            } catch (e: Exception) {
                setError(e.message)
                setLoading(false)
            }
        }
    }
}

// 4. Başka ViewModel sadece error alır
class ProfileViewModel(
    errorState: ErrorState = ErrorStateDelegate()
) : ViewModel(),
    ErrorState by errorState {

    // Sadece error var, loading yok! ✅
}
```

**Advantages:**
- ✅ Easy to test (mock LoadingStateDelegate)
- ✅ Flexible (ProfileViewModel didn't take loading)
- ✅ Reusable (every ViewModel can use the same delegate)

### Example 2: Analytics and Logging

**❌ Inheritance:**

```kotlin
open class BaseActivity : AppCompatActivity() {
    protected fun trackScreenView(screenName: String) {
        FirebaseAnalytics.getInstance(this).logEvent("screen_view", Bundle().apply {
            putString("screen_name", screenName)
        })
    }

    protected fun logDebug(message: String) {
        Log.d(javaClass.simpleName, message)
    }
}

class ArticleActivity : BaseActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        trackScreenView("Article")
        logDebug("Article loaded")
    }
}
```

**Problems:**
- Tightly coupled to Firebase (can't switch to Mixpanel)
- Every Activity gets analytics (some might not want it)
- BaseActivity needs to be mocked for testing

**✅ Composition:**

```kotlin
// 1. Analytics interface
interface AnalyticsHandler {
    fun trackScreenView(screenName: String)
    fun trackEvent(eventName: String, params: Map<String, Any>)
}

class FirebaseAnalyticsHandler(
    private val analytics: FirebaseAnalytics
) : AnalyticsHandler {
    override fun trackScreenView(screenName: String) {
        analytics.logEvent("screen_view", Bundle().apply {
            putString("screen_name", screenName)
        })
    }

    override fun trackEvent(eventName: String, params: Map<String, Any>) {
        analytics.logEvent(eventName, Bundle().apply {
            params.forEach { (key, value) ->
                putString(key, value.toString())
            }
        })
    }
}

// 2. Logger interface
interface Logger {
    fun debug(message: String)
    fun error(message: String, throwable: Throwable?)
}

class AndroidLogger(private val tag: String) : Logger {
    override fun debug(message: String) {
        Log.d(tag, message)
    }

    override fun error(message: String, throwable: Throwable?) {
        Log.e(tag, message, throwable)
    }
}

// 3. Activity composition ile
class ArticleActivity : AppCompatActivity() {
    private val analyticsHandler: AnalyticsHandler by lazy {
        FirebaseAnalyticsHandler(FirebaseAnalytics.getInstance(this))
    }

    private val logger: Logger by lazy {
        AndroidLogger("ArticleActivity")
    }

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        analyticsHandler.trackScreenView("Article")
        logger.debug("Article loaded")
    }
}

// 4. Başka Activity analytics almaz
class ProfileActivity : AppCompatActivity() {
    // Analytics yok, sadece logger var ✅
    private val logger: Logger by lazy {
        AndroidLogger("ProfileActivity")
    }
}
```

**Advantages:**
- ✅ Firebase → Mixpanel migration is easy (just change implementation)
- ✅ Easy to test (mock AnalyticsHandler)
- ✅ Flexible (ProfileActivity didn't take analytics)

### Example 3: RecyclerView.Adapter (IS-A Relationship - Inheritance Correct!)

```kotlin
// ✅ Inheritance kullan - RecyclerView framework gerektirir
class ArticleAdapter(
    private val onItemClick: (Article) -> Unit
) : RecyclerView.Adapter<ArticleAdapter.ViewHolder>() {

    private val articles = mutableListOf<Article>()

    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): ViewHolder {
        val view = LayoutInflater.from(parent.context)
            .inflate(R.layout.item_article, parent, false)
        return ViewHolder(view)
    }

    override fun onBindViewHolder(holder: ViewHolder, position: Int) {
        holder.bind(articles[position])
    }

    override fun getItemCount() = articles.size

    inner class ViewHolder(itemView: View) : RecyclerView.ViewHolder(itemView) {
        fun bind(article: Article) {
            itemView.findViewById<TextView>(R.id.title).text = article.title
            itemView.setOnClickListener { onItemClick(article) }
        }
    }
}
```

**Why is inheritance correct?**
- ✅ `ArticleAdapter` **IS-A** `RecyclerView.Adapter` (truly an adapter)
- ✅ RecyclerView framework expects `Adapter`
- ✅ Template method pattern (`onCreateViewHolder`, `onBindViewHolder` must be overridden)

---

## When to Use Inheritance, When to Use Composition?

### ✅ Use Inheritance:

#### 1. **Framework Requirement** (You have to!)

```kotlin
// ✅ Android framework gerektirir
class MyActivity : AppCompatActivity()
class MyFragment : Fragment()
class MyViewModel : ViewModel()
class MyView(context: Context) : View(context)
class MyAdapter : RecyclerView.Adapter<ViewHolder>()
class MyWorker(context: Context, params: WorkerParameters) : Worker(context, params)
class MyBroadcastReceiver : BroadcastReceiver()
class MyService : Service()
```

#### 2. **Genuine IS-A Relationship**

```kotlin
// ✅ IS-A ilişkisi var
abstract class Shape {
    abstract fun area(): Double
}

class Circle(val radius: Double) : Shape() {
    override fun area() = Math.PI * radius * radius
}

class Rectangle(val width: Double, val height: Double) : Shape() {
    override fun area() = width * height
}

// Circle IS-A Shape ✅
// Rectangle IS-A Shape ✅
```

**IS-A Test:**
> "Can I put a child in place of a parent?"

- ✅ Can I put `Circle` in place of `Shape`? → Yes
- ✅ Can I put `ArticleAdapter` in place of `RecyclerView.Adapter`? → Yes

#### 3. **Template Method Pattern** (Rare!)

```kotlin
// ✅ Template method pattern
abstract class DataParser {
    fun parse(data: String): Result {
        validate(data)
        val parsed = doParse(data)
        return transform(parsed)
    }

    protected abstract fun doParse(data: String): ParsedData

    private fun validate(data: String) { /* ... */ }
    private fun transform(data: ParsedData): Result { /* ... */ }
}
```

### ✅ Use Composition:

#### 1. **Adding Behavior** (Behavior sharing)

```kotlin
// ✅ Composition
interface LoadingHandler {
    fun showLoading()
    fun hideLoading()
}

class ArticleViewModel(
    private val loadingHandler: LoadingHandler
) : ViewModel()
```

#### 2. **Cross-cutting Concerns** (Analytics, Logging, Auth)

```kotlin
// ✅ Composition
class ArticleViewModel(
    private val analytics: AnalyticsHandler,
    private val logger: Logger,
    private val authRepository: AuthRepository
) : ViewModel()
```

#### 3. **Multiple Behaviors**

```kotlin
// ✅ Composition + Delegation
class ArticleViewModel(
    loadingState: LoadingState,
    errorState: ErrorState
) : ViewModel(),
    LoadingState by loadingState,
    ErrorState by errorState
```

#### 4. **Testability**

```kotlin
// ✅ Composition - Mock kolay
@Test
fun testLoading() {
    val mockLoading = MockLoadingState()
    val viewModel = ArticleViewModel(loadingState = mockLoading)

    viewModel.load()

    assertTrue(mockLoading.isLoading)
}
```

#### 5. **Changing Behavior at Runtime**

```kotlin
// ✅ Composition - Runtime'da değişir
class Car(private var engine: Engine) {
    fun changeEngine(newEngine: Engine) {
        engine = newEngine  // Runtime'da motor değişti!
    }
}
```

---

## Practical Rules

### Golden Rule

```
Can I use inheritance?
     │
     ├─► Framework requirement? (Activity, Fragment, ViewModel)
     │   └─► YES → Use Inheritance ✅
     │
     ├─► Genuine IS-A relationship? (Circle IS-A Shape)
     │   └─► YES → Use Inheritance ✅
     │
     └─► ALL OTHER CASES → Use Composition ✅
```

**If in doubt → Use Composition!** 🚀

### IS-A vs HAS-A Test

| Case | Sentence | Correct? |
|-------|-------|-----------|
| `ArticleAdapter` | ArticleAdapter **IS-A** RecyclerView.Adapter | ✅ Yes → Inheritance |
| `CircularImageView` | CircularImageView **IS-A** ImageView | ✅ Yes → Inheritance |
| `ArticleViewModel` | ArticleViewModel **IS-A** Repository | ❌ No → Composition |
| `ArticleViewModel` | ArticleViewModel **HAS-A** Repository | ✅ Yes → Composition |
| `ArticleActivity` | ArticleActivity **IS-A** AnalyticsHandler | ❌ No → Composition |
| `ArticleActivity` | ArticleActivity **HAS-A** AnalyticsHandler | ✅ Yes → Composition |

---

## Summary Table

| Criteria | Inheritance | Composition |
|--------|-------------|-------------|
| **Relationship** | IS-A (A car is a vehicle) | HAS-A (A car has an engine) |
| **Flexibility** | ❌ Low (compile-time fixed) | ✅ High (can be changed at runtime) |
| **Coupling** | ❌ Tight coupling | ✅ Loose coupling |
| **Testing** | ❌ Hard (mock parent) | ✅ Easy (mock components) |
| **Multiple behavior** | ❌ Impossible (1 parent) | ✅ Possible (N components) |
| **Change impact** | ❌ High (parent changes → child breaks) | ✅ Low (isolated change) |
| **Usage** | Framework requirement, genuine IS-A | Behavior sharing, cross-cutting concerns |
| **Kotlin support** | ✅ Available (inheritance) | ✅ Available + Easy (with delegation) |

---

## Composition Over Inheritance and SOLID Relationship

| SOLID Principle | Composition Equivalent |
|----------------|--------------------------|
| **Single Responsibility** | Each component has a single responsibility (LoadingDelegate, ErrorDelegate separate) |
| **Open/Closed** | Add new behavior (new delegate), don't modify existing code |
| **Liskov Substitution** | Delegates are interchangeable (MockLoadingState for testing) |
| **Interface Segregation** | Small interfaces (LoadingState, ErrorState separate) |
| **Dependency Inversion** | Depend on interfaces (LoadingState interface, delegate implementation) |

---

## Resources

- [Design Patterns: Elements of Reusable Object-Oriented Software - Gang of Four](https://www.amazon.com/Design-Patterns-Elements-Reusable-Object-Oriented/dp/0201633612)
- [Effective Java - Joshua Bloch (Item 18: Favor composition over inheritance)](https://www.amazon.com/Effective-Java-Joshua-Bloch/dp/0134685997)
- [Kotlin Delegation - Official Docs](https://kotlinlang.org/docs/delegation.html)
- [Android Architecture Guide - Google](https://developer.android.com/topic/architecture)
