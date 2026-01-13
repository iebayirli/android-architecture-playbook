### 3. MVVM (Model-View-ViewModel)

#### Definition

Model-View-ViewModel is the most widely used architecture in modern Android development, recommended by Google. Unlike MVP, the ViewModel class does not hold a reference to the View; instead, it communicates with the View using **reactive programming** (LiveData, StateFlow). ViewModel is one of the Android Jetpack components and has a lifecycle-aware structure, so it is not affected by configuration changes (screen rotation, etc.). The View and Model layers are similar to the structure in MVP, but the coupling (dependency) between ViewModel and View is completely eliminated.

#### Structure in Android

- **Model:** Data layer (Repository, DB, API)
- **View:** Activity/Fragment + XML Layouts (or Jetpack Compose)
- **ViewModel:** Business Logic (Lifecycle-aware)

#### Code Example

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

#### Advantages

- UI and Business Logic are completely independent from each other.
- UI and Business Logic can be tested separately.
- ViewModel is lifecycle-aware, so it is not affected by configuration changes.
- Since ViewModel doesn't know about the View, it can be reused in different Views.
- Data flow is more manageable with reactive programming.
- Supported by Google (Jetpack component).

#### Disadvantages

- Can be overkill for small projects (unnecessary complexity).
- There is a learning curve for reactive programming (LiveData/Flow).
- If too much logic accumulates in the ViewModel, it creates a "Fat ViewModel" problem.
- If data binding is used, writing logic in XML makes debugging difficult.

#### Use Cases

- **Standard architecture in modern Android development** (recommended by Google).
- Can be used in projects of all sizes (small, medium, large).
- Especially in applications where configuration changes are frequent (tablet support).
- When testability is important.
- Recommended for use with Jetpack Compose.
- **If you want to use reactive programming** (LiveData, StateFlow, Flow).
