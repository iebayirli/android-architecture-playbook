### 2. MVP (Model-View-Presenter)

#### Definition

Model-View-Presenter architecture is an improved version of MVC. It adds a Presenter layer to completely separate business logic from the UI. The View only captures user interactions and updates the screen; it doesn't contain any business logic. The Presenter acts as a bridge between View and Model and manages all business logic. Communication between View and Presenter is provided through interfaces (Contract), which increases testability.

#### Structure in Android

- **Model:** Data layer (Repository, DB, API)
- **View:** Activity/Fragment (UI operations only)
- **Presenter:** Business logic (independent from View)

#### Code Example

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

#### Advantages

- Logic is testable because it's independent from UI.
- Presenter can be tested separately (UI can be mocked).
- Separation of concerns (Each layer has a single responsibility).
- Better code organization compared to MVC.

#### Disadvantages

- Requires too many interfaces (Boilerplate code).
- View reference leak risk (Presenter holds a reference to View).
- Lifecycle management must be done manually.
- Requires Contract + Presenter + View for each screen.

#### Use Cases

- When testability is important.
- Can be used in medium-scale projects.
- **MVVM is now preferred.**
