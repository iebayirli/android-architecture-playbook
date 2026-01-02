### 2. MVP (Model-View-Presenter)

#### Tanım

Model-View-Presenter mimarisi, MVC'nin geliştirilmiş halidir. Business logic'i UI'dan tamamen ayırmak için Presenter katmanı ekler. View, sadece kullanıcı etkileşimlerini yakalar ve ekranı günceller; herhangi bir iş mantığı içermez. Presenter, View ile Model arasında köprü görevi görür ve tüm iş mantığını yönetir. View ile Presenter arasındaki iletişim interface'ler (Contract) aracılığıyla sağlanır, bu sayede test edilebilirlik artar.

#### Android'de Yapısı

- **Model:** Data katmanı (Repository, DB, API)
- **View:** Activity/Fragment (sadece UI işlemleri)
- **Presenter:** Business logic (View'den bağımsız)

#### Kod Örneği

```kotlin
// CONTRACT (Interface)
interface UserContract {
    interface View {
        fun showUser(name: String)
        fun showError(message: String)
        fun showLoading()
    }

    interface Presenter {
        fun loadUser(id: String)
        fun onDestroy()
    }
}

// PRESENTER
class UserPresenter(
    private val view: UserContract.View,
    private val repository: UserRepository
) : UserContract.Presenter {

    override fun loadUser(id: String) {
        view.showLoading()

        // Async call (coroutine, callback, etc.)
        repository.getUser(id) { result ->
            when (result) {
                is Success -> view.showUser(result.data.name)
                is Error -> view.showError(result.message)
            }
        }
    }

    override fun onDestroy() {
        // Cleanup
    }
}

// VIEW (Activity/Fragment)
class UserActivity : AppCompatActivity(), UserContract.View {
    private lateinit var presenter: UserContract.Presenter

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_user)

        presenter = UserPresenter(this, UserRepository())
        presenter.loadUser("123")
    }

    override fun showUser(name: String) {
        nameTextView.text = name
    }

    override fun showLoading() {
        progressBar.visibility = View.VISIBLE
    }

    override fun showError(message: String) {
        Toast.makeText(this, message, Toast.LENGTH_SHORT).show()
    }

    override fun onDestroy() {
        presenter.onDestroy()
        super.onDestroy()
    }
}
```

#### Avantajları

- Logic UI'dan bağımsız olduğu için test edilebilir.
- Presenter ayrı test edilebilir (UI mock'lanabilir).
- Separation of concerns (Her katmanın tek sorumluluğu var).
- MVC'ye göre daha iyi kod organizasyonu.

#### Dezavantajları

- Çok fazla interface gerektirir (Boilerplate kod).
- View reference leak riski (Presenter, View referansını tutuyor).
- Lifecycle yönetimi manuel olarak yapılmalı.
- Her ekran için Contract + Presenter + View gerekir.

#### Kullanım Senaryoları

- Test edilebilirlik önemliyse.
- Orta ölçekli projelerde kullanılabilir.
- **Artık MVVM tercih ediliyor.**
