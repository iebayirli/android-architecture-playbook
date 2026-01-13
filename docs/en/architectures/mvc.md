### 1. MVC (Model-View-Controller)

#### Definition

Model-View-Controller architecture is one of the oldest architectural patterns in software development. It divides the application into three main components: Model (data layer), View (presentation), and Controller (control layer). In Android, Activity/Fragment serves as both Controller and partially as View. User interactions, data operations, and screen updates are managed through the Controller (Activity/Fragment).

#### Structure in Android

- **Model:** Data layer (Repository, DB, API)
- **View:** XML layouts
- **Controller:** Activity/Fragment

#### Code Example

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
    private lateinit var nameTextView: TextView // VIEW reference

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

#### Advantages

- Easy to learn and simple.
- Enables fast development for small projects.

#### Disadvantages

- Activity/Fragment becomes bloated (God Object problem).
- Not testable (business logic is tightly coupled to UI).
- Business logic cannot be isolated as it depends on UI.
- Data is lost on configuration changes (e.g., screen rotation).
- Tight coupling (tight dependencies between layers).

#### Use Cases

- Can be used in very simple, single-screen applications.
- Can be used in prototype or demo work.
- **Not recommended for modern Android development today.**
