# Composition Over Inheritance

## Giriş

**Composition Over Inheritance** (Kalıtım yerine Bileşim), "miras almak yerine bileşenlerden oluştur" prensibidir. Gang of Four (GoF) Design Patterns kitabında önerilen temel prensiplerden biridir.

**Basit Anlatım:**
- **Inheritance (Kalıtım)**: "IS-A" ilişkisi → "Araba bir araçtır"
- **Composition (Bileşim)**: "HAS-A" ilişkisi → "Araba bir motora sahiptir"

**Gerçek hayat analojisi:**
- **Inheritance**: Bir robotun DNA'sını değiştirip yeni robot yaratmak (kırılgan, riskli)
- **Composition**: LEGO blokları gibi parçaları birleştirip robot yapmak (esnek, güvenli)

> **Temel Fikir:** "Miras almak yerine, ihtiyacın olan davranışları bileşenler halinde kullan"

**Önemli Not:** Composition her zaman daha iyi değildir! Framework gereksinimleri ve gerçek IS-A ilişkileri için inheritance kullanılmalıdır.

---

## Inheritance'ın Sorunları

### 1. **Fragile Base Class Problem** (Kırılgan Ana Sınıf)

Parent class değiştiğinde child class'lar beklenmedik şekilde etkilenir.

```kotlin
// ❌ Sorunlu örnek
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

**Sorun:**
- Parent class değişti → Child class'lar etkilendi
- Bisiklet, sensör implementasyonundan haberdar değil
- Parent'ın private metodları child'ı etkiliyor

### 2. **Deep Inheritance Hierarchies** (Derin Miras Zincirleri)

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

**Sorunlar:**
- Değişiklik yapmak zor (hangi seviyeyi değiştireceğim?)
- Her seviye bir bağımlılık ekliyor
- Test etmek karmaşık (tüm parent'ları bilmen gerek)

### 3. **Tight Coupling** (Sıkı Bağımlılık)

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

**Sorunlar:**
- `BaseActivity` değişirse `ArticleActivity` etkilenir
- İhtiyacım olmayan metodları da aldım
- Test ederken tüm `BaseActivity`'yi mock'lamam gerekir

### 4. **Gorilla-Banana Problem** 🦍🍌

Joe Armstrong (Erlang'ın yaratıcısı) şöyle der:

> "Object-oriented languages'in sorunu: Bir muz istedin ama muzu tutan gorili ve tüm ormayı aldın!"

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

## Composition Çözümü

### ✅ Doğru Yaklaşım: Composition

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

**Avantajlar:**
- ✅ İhtiyacın olanı al, gereksizi alma
- ✅ Bağımlılık az (loose coupling)
- ✅ Test kolay (her delegate ayrı test edilir)
- ✅ Esneklik yüksek (runtime'da değiştirebilirsin)

---

## Kotlin Delegation

Kotlin, composition'ı kolaylaştıran **`by`** keyword'üne sahiptir.

### Delegation Nedir?

**Delegation = "Sen yap, ben sadece ileteceğim"**

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

### Kotlin Delegation Avantajları

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

**Avantajlar:**
- ✅ Boilerplate kod yok (by keyword ile otomatik)
- ✅ Runtime'da engine değiştirebilirsin
- ✅ Test kolay (mock engine inject et)

---

## Kotlin Delegation vs Multiple Inheritance

### Multiple Inheritance Nedir?

Bazı dillerde (C++ gibi) bir sınıf **birden fazla sınıftan** miras alabilir:

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

**Java/Kotlin:** Multiple class inheritance **desteklemez**! ❌

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

Kotlin delegation, multiple inheritance'ın **avantajlarını** alır ama **sorunlarını** almaz:

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

### Diamond Problem (Elmas Problemi) 💎

Multiple inheritance'ın en büyük sorunu:

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

**Görsel:**
```
       Vehicle
       /     \
     Car     Boat
       \     /
    AmphibiousCar

Elmas şekli → Diamond Problem
```

### Kotlin Diamond Problem'i Nasıl Çözer?

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

**Kotlin'in yaklaşımı:**
- ❌ Belirsizlik yok (Diamond Problem çözüldü)
- ✅ Conflict varsa **compile-time hata** (runtime'da patlama yok)
- ✅ Developer kontrolde (manuel override zorunlu)

### Karşılaştırma Tablosu

| Özellik | Multiple Inheritance (C++) | Kotlin Delegation |
|---------|---------------------------|-------------------|
| **Birden fazla class'tan miras** | ✅ Evet | ❌ Hayır (sadece 1 class) |
| **Birden fazla interface** | ✅ Evet | ✅ Evet |
| **Diamond Problem** | ❌ Var (büyük sorun!) | ✅ Yok (compile-time hata) |
| **Belirsizlik** | ❌ Runtime'da sorun | ✅ Compile-time'da yakalanır |
| **Boilerplate kod** | ✅ Az | ✅ Az (by keyword) |
| **Güvenlik** | ❌ Riskli | ✅ Güvenli |
| **Esneklik** | ❌ Az (compile-time fixed) | ✅ Çok (runtime değiştirilebilir) |

**Sonuç:**

> **"Kotlin delegation, multiple inheritance'ın avantajlarını sağlar ama Diamond Problem gibi sorunlarını çözer. Composition + Delegation = Safe Multiple Inheritance"**

---

## Android'de Gerçek Örnekler

### Örnek 1: ViewModel State Yönetimi

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

**Sorunlar:**
- BaseViewModel değişirse tüm ViewModeller etkilenir
- Test için BaseViewModel'i mock'lamak gerekir
- Bazı ViewModeller loading istemeyebilir ama mecbur alır

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

**Avantajlar:**
- ✅ Test kolay (LoadingStateDelegate'i mock'la)
- ✅ Esnek (ProfileViewModel loading almadı)
- ✅ Reusable (her ViewModel aynı delegate'i kullanabilir)

### Örnek 2: Analytics ve Logging

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

**Sorunlar:**
- Firebase'e sıkı bağımlı (Mixpanel'e geçemezsin)
- Her Activity analytics alır (bazıları istemeyebilir)
- Test için BaseActivity'yi mock'lamak gerekir

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

**Avantajlar:**
- ✅ Firebase → Mixpanel geçişi kolay (sadece implementation değiştir)
- ✅ Test kolay (AnalyticsHandler'ı mock'la)
- ✅ Esnek (ProfileActivity analytics almadı)

### Örnek 3: RecyclerView.Adapter (IS-A İlişkisi - Inheritance Doğru!)

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

**Neden inheritance doğru?**
- ✅ `ArticleAdapter` **IS-A** `RecyclerView.Adapter` (gerçekten bir adapter)
- ✅ RecyclerView framework, `Adapter` bekliyor
- ✅ Template method pattern (`onCreateViewHolder`, `onBindViewHolder` override zorunlu)

---

## Ne Zaman Inheritance, Ne Zaman Composition?

### ✅ Inheritance Kullan:

#### 1. **Framework Zorunluluğu** (Mecbursun!)

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

#### 2. **Gerçek IS-A İlişkisi**

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
> "Parent yerine child koyabilir miyim?"

- ✅ `Shape` yerine `Circle` koyabilir miyim? → Evet
- ✅ `RecyclerView.Adapter` yerine `ArticleAdapter` koyabilir miyim? → Evet

#### 3. **Template Method Pattern** (Nadir!)

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

### ✅ Composition Kullan:

#### 1. **Behavior Ekleme** (Davranış paylaşımı)

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

#### 3. **Multiple Behaviors** (Birden fazla davranış)

```kotlin
// ✅ Composition + Delegation
class ArticleViewModel(
    loadingState: LoadingState,
    errorState: ErrorState
) : ViewModel(),
    LoadingState by loadingState,
    ErrorState by errorState
```

#### 4. **Test Edilebilirlik**

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

#### 5. **Runtime Behavior Değiştirme**

```kotlin
// ✅ Composition - Runtime'da değişir
class Car(private var engine: Engine) {
    fun changeEngine(newEngine: Engine) {
        engine = newEngine  // Runtime'da motor değişti!
    }
}
```

---

## Pratik Kurallar

### Altın Kural

```
Inheritance kullanabilir miyim?
     │
     ├─► Framework zorunlu mu? (Activity, Fragment, ViewModel)
     │   └─► EVET → Inheritance kullan ✅
     │
     ├─► Gerçek IS-A ilişkisi mi? (Circle IS-A Shape)
     │   └─► EVET → Inheritance kullan ✅
     │
     └─► DİĞER TÜM DURUMLAR → Composition kullan ✅
```

**Şüphen varsa → Composition kullan!** 🚀

### IS-A vs HAS-A Test

| Durum | Cümle | Doğru mu? |
|-------|-------|-----------|
| `ArticleAdapter` | ArticleAdapter **IS-A** RecyclerView.Adapter | ✅ Evet → Inheritance |
| `CircularImageView` | CircularImageView **IS-A** ImageView | ✅ Evet → Inheritance |
| `ArticleViewModel` | ArticleViewModel **IS-A** Repository | ❌ Hayır → Composition |
| `ArticleViewModel` | ArticleViewModel **HAS-A** Repository | ✅ Evet → Composition |
| `ArticleActivity` | ArticleActivity **IS-A** AnalyticsHandler | ❌ Hayır → Composition |
| `ArticleActivity` | ArticleActivity **HAS-A** AnalyticsHandler | ✅ Evet → Composition |

---

## Özet Tablo

| Kriter | Inheritance | Composition |
|--------|-------------|-------------|
| **İlişki** | IS-A (Araba bir araçtır) | HAS-A (Araba bir motora sahiptir) |
| **Esneklik** | ❌ Az (compile-time fixed) | ✅ Çok (runtime değiştirilebilir) |
| **Bağımlılık** | ❌ Tight coupling | ✅ Loose coupling |
| **Test** | ❌ Zor (parent'ı mock'la) | ✅ Kolay (bileşenleri mock'la) |
| **Multiple behavior** | ❌ İmkansız (1 parent) | ✅ Mümkün (N bileşen) |
| **Değişiklik etkisi** | ❌ Yüksek (parent değişir → child patlar) | ✅ Düşük (izole değişiklik) |
| **Kullanım** | Framework zorunluluğu, gerçek IS-A | Behavior paylaşımı, cross-cutting concerns |
| **Kotlin desteği** | ✅ Var (inheritance) | ✅ Var + Kolay (delegation ile) |

---

## Composition Over Inheritance ve SOLID İlişkisi

| SOLID Prensibi | Composition'da Karşılığı |
|----------------|--------------------------|
| **Single Responsibility** | Her bileşen tek sorumluluk (LoadingDelegate, ErrorDelegate ayrı) |
| **Open/Closed** | Yeni behavior ekle (yeni delegate), mevcut kodu değiştirme |
| **Liskov Substitution** | Delegate'ler değiştirilebilir (MockLoadingState test için) |
| **Interface Segregation** | Küçük interface'ler (LoadingState, ErrorState ayrı) |
| **Dependency Inversion** | Interface'e bağımlı (LoadingState interface, delegate implementation) |

---

## Kaynaklar

- [Design Patterns: Elements of Reusable Object-Oriented Software - Gang of Four](https://www.amazon.com/Design-Patterns-Elements-Reusable-Object-Oriented/dp/0201633612)
- [Effective Java - Joshua Bloch (Item 18: Favor composition over inheritance)](https://www.amazon.com/Effective-Java-Joshua-Bloch/dp/0134685997)
- [Kotlin Delegation - Official Docs](https://kotlinlang.org/docs/delegation.html)
- [Android Architecture Guide - Google](https://developer.android.com/topic/architecture)
