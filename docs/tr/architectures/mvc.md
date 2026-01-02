### 1. MVC (Model-View-Controller)

#### Tanım

Model-View-Controller mimarisi, yazılım geliştirmede en eski mimari desenlerden biridir. Uygulamayı üç ana bileşene ayırır: Model (veri katmanı), View (görünüm) ve Controller (kontrol katmanı). Android'de Activity/Fragment hem Controller hem de kısmen View görevi görür. Kullanıcı etkileşimleri, veri işlemleri ve ekran güncellemeleri Controller (Activity/Fragment) üzerinden yönetilir.

#### Android'de Yapısı

- **Model:** Data katmanı (Repository, DB, API)
- **View:** XML layouts
- **Controller:** Activity/Fragment

#### Kod Örneği

```kotlin
// MODEL
class UserRepository {
    fun getUser(id: String): User {
        // API call or DB query
    }
}

// CONTROLLER (Activity/Fragment)
class UserActivity : AppCompatActivity() {
    private val repository = UserRepository()
    private lateinit var nameTextView: TextView // VIEW referansı

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_user)

        nameTextView = findViewById(R.id.nameTextView)

        // Button click (User input)
        findViewById<Button>(R.id.loadButton).setOnClickListener {
            // Controller logic
            val user = repository.getUser("123")

            // Update view directly
            nameTextView.text = user.name
        }
    }
}

// VIEW
// activity_user.xml
<TextView android:id="@+id/nameTextView" />
<Button android:id="@+id/loadButton" />
```

#### Avantajları

- Öğrenmesi kolay ve basit.
- Küçük projeler için hızlı geliştirme imkanı sağlar.

#### Dezavantajları

- Activity/Fragment çok şişer (God Object problemi).
- Test edilemez (iş mantığı UI'a sıkı bağlı).
- Business logic UI'a bağımlı olduğu için izole edilemez.
- Configuration change'de (örn: ekran döndürme) veri kaybolur.
- Tight coupling (katmanlar arası sıkı bağımlılık).

#### Kullanım Senaryoları

- Çok basit, tek ekranlı uygulamalarda kullanılabilir.
- Prototip veya demo çalışmalarında kullanılabilir.
- **Günümüz modern Android geliştirmesinde önerilmez.**
