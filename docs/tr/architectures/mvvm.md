### 3. MVVM (Model-View-ViewModel)

#### Tanım

Model-View-ViewModel, Google tarafından önerilen ve modern Android geliştirmede en yaygın kullanılan mimaridir. MVP'den farklı olarak, ViewModel sınıfı View'ın referansını tutmaz; bunun yerine **reactive programming** (LiveData, StateFlow) kullanarak View ile iletişim kurar. ViewModel, Android Jetpack bileşenlerinden biridir ve lifecycle-aware bir yapıdadır, böylece configuration change'lerden (ekran döndürme vb.) etkilenmez. View ve Model katmanları MVP'deki yapıyla benzerdir ancak ViewModel ile View arasındaki coupling (bağımlılık) tamamen ortadan kalkmıştır.

#### Android'de Yapısı

- **Model:** Data katmanı (Repository, DB, API)
- **View:** Activity/Fragment + XML Layouts (veya Jetpack Compose)
- **ViewModel:** Business Logic (Lifecycle-aware)

#### Kod Örneği

```kotlin
// MODEL - Repository
class UserRepository {
    suspend fun getUser(id: String): Result<User> {
        return try {
            val response = apiService.getUser(id)
            Result.success(response)
        } catch (e: Exception) {
            Result.failure(e)
        }
    }
}

// VIEWMODEL - Business Logic
class UserViewModel(
    private val repository: UserRepository
) : ViewModel() {

    // UI State (LiveData veya StateFlow)
    private val _uiState = MutableLiveData<UiState>()
    val uiState: LiveData<UiState> = _uiState

    sealed class UiState {
        object Loading : UiState()
        data class Success(val userName: String) : UiState()
        data class Error(val message: String) : UiState()
    }

    fun loadUser(id: String) {
        _uiState.value = UiState.Loading

        viewModelScope.launch {
            repository.getUser(id).fold(
                onSuccess = { user ->
                    _uiState.value = UiState.Success(user.name)
                },
                onFailure = { error ->
                    _uiState.value = UiState.Error(error.message ?: "Unknown error")
                }
            )
        }
    }
}

// VIEW - Activity/Fragment
class UserActivity : AppCompatActivity() {

    // ViewModel oluşturulur (Factory pattern)
    private val viewModel: UserViewModel by viewModels {
        UserViewModelFactory(UserRepository())
    }

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_user)

        // Observer pattern - UI State'i gözle
        viewModel.uiState.observe(this) { state ->
            when (state) {
                is UserViewModel.UiState.Loading -> {
                    progressBar.visibility = View.VISIBLE
                    errorText.visibility = View.GONE
                }
                is UserViewModel.UiState.Success -> {
                    progressBar.visibility = View.GONE
                    nameTextView.text = state.userName
                }
                is UserViewModel.UiState.Error -> {
                    progressBar.visibility = View.GONE
                    errorText.visibility = View.VISIBLE
                    errorText.text = state.message
                }
            }
        }

        // Buton tıklandığında ViewModel'e haber ver
        loadButton.setOnClickListener {
            viewModel.loadUser("123")
        }
    }
}

// ViewModel Factory (Dependency Injection için)
class UserViewModelFactory(
    private val repository: UserRepository
) : ViewModelProvider.Factory {
    override fun <T : ViewModel> create(modelClass: Class<T>): T {
        if (modelClass.isAssignableFrom(UserViewModel::class.java)) {
            @Suppress("UNCHECKED_CAST")
            return UserViewModel(repository) as T
        }
        throw IllegalArgumentException("Unknown ViewModel class")
    }
}
```

#### Avantajları

- UI ve Business Logic birbirinden tamamen bağımsız.
- UI ve Business Logic ayrı ayrı test edilebilir.
- ViewModel lifecycle-aware olduğu için configuration change'den etkilenmez.
- ViewModel, View'ı bilmediği için farklı View'lerde tekrar kullanılabilir.
- Reactive programming ile veri akışı daha yönetilebilir.
- Google tarafından destekleniyor (Jetpack bileşeni).

#### Dezavantajları

- Küçük projeler için overkill olabilir (gereksiz karmaşıklık).
- Reactive programming (LiveData/Flow) öğrenme eğrisi var.
- ViewModel'de çok fazla logic birikirse "Fat ViewModel" problemi oluşur.
- Data binding kullanılırsa XML'de logic yazmak debugging'i zorlaştırır.

#### Kullanım Senaryoları

- **Modern Android geliştirmede standart mimari** (Google tarafından önerilir).
- Her ölçekteki projede kullanılabilir (küçük, orta, büyük).
- Özellikle configuration change'lerin sık olduğu uygulamalarda (tablet destekli).
- Test edilebilirlik önemliyse.
- Jetpack Compose ile birlikte kullanımı önerilir.
- **Reactive programming kullanmak istiyorsan** (LiveData, StateFlow, Flow).
