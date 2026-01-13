# Architecture Comparison

## Introduction

This document compares commonly used architectural patterns in Android (MVC, MVP, MVVM, Clean Architecture) and objectively examines the strengths and weaknesses of each.

---

## Table of Contents

1. [Detailed Comparison](#detailed-comparison)
2. [Data Flow Comparison](#data-flow-comparison)
3. [Architecture-Independent Common Practices](#architecture-independent-common-practices)

---

## Detailed Comparison

### 1. Architectural Structure

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

Characteristics:
- Controller (Activity/Fragment) knows everything
- Direct access to Model and View
- Tight coupling
- Weak separation of concerns
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

Characteristics:
- Presenter holds View reference (through interface)
- Two-way communication (View ↔ Presenter)
- View depends on Presenter
- Logic separated but View leak risk exists
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

Characteristics:
- ViewModel does not hold View reference
- One-way communication (ViewModel → View)
- View observes ViewModel
- Reactive and Lifecycle-aware
- Low memory leak risk
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

Characteristics:
- Clear boundaries between layers (Dependency Rule)
- Inner layers do not know outer layers
- Domain layer is framework-independent (Pure Kotlin)
- Each layer can be isolated and tested
- Dependency Inversion principle is applied
```

---

### 2. Testability

#### MVC - Not Testable ❌

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
    val activity = UserActivity() // ❌ DOES NOT COMPILE!
    // Activity cannot be mocked!
}
```

#### MVP - Testable ✅

```kotlin
// Presenter test
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

#### MVVM - Very Easy to Test ✅✅

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

#### Clean Architecture - Layer by Layer Testing ✅✅✅

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

### 3. Memory Leak Risk

#### MVC - High Risk 🔴

```kotlin
class UserActivity : AppCompatActivity() {
    private val job = CoroutineScope(Dispatchers.IO).launch {
        // ❌ Activity leak! Job is not cancelled
        delay(10000)
        val users = repository.getUsers()
        // Activity may have been destroyed!
    }
}
```

#### MVP - Medium Risk 🟡

```kotlin
class UserPresenter(
    private val view: UserContract.View // ❌ View reference can leak!
) {
    fun loadUsers() {
        // When async call finishes, view is called
        // What if View is destroyed? → CRASH!
    }
}

// Solution: Manual cleanup
override fun onDestroy() {
    presenter.detachView() // Manual cleanup required
    super.onDestroy()
}
```

#### MVVM - Low Risk 🟢

```kotlin
class UserViewModel : ViewModel() {
    fun loadUsers() {
        viewModelScope.launch {
            // ✅ Automatically cancelled when ViewModel is destroyed!
            val users = repository.getUsers()
            _users.value = users
        }
    }
}
```

#### Clean Architecture - Very Low Risk 🟢

```kotlin
// Each layer is lifecycle-aware
class UserViewModel(
    private val getUsersUseCase: GetUsersUseCase // ✅ Interface
) : ViewModel() {
    // Uses ViewModel scope
    // Automatic cleanup
}
```

---

## Data Flow Comparison

### MVC - Complex, Bidirectional

```
User Input → Controller ──┐
                          ├──► Model
                          └──► View ──► Update ──► Controller
                                                      │
                                                      └──► Loop!

Problem: Data flow is unpredictable
```

### MVP - Clean but Manual

```
User Input → View (Activity)
              │
              ▼
            Presenter
              │
              ├──► Model (Data)
              │
              └──► View.showData() ✅

Data flow: View → Presenter → Model → Presenter → View
```

### MVVM - Reactive, Unidirectional

```
User Input → ViewModel
              │
              ├──► Repository
              │       │
              │       ▼
              └──► LiveData/StateFlow ──► View (Observe)
                       ▲
                       │
                   Reactive Update (Automatic!)

Data flow: View → ViewModel → Repository → StateFlow → View
```

### Clean Architecture - Layered, Dependency Rule

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

Dependency Rule: Inner layers do not know outer layers!
```

---

## Architecture-Independent Common Practices

### 1. Repository Pattern

```kotlin
// Repository pattern provides data source abstraction
class UserRepository { ... }

// Can be used even in simple architectures like MVC:
class UserActivity : AppCompatActivity() {
    private val repository = UserRepository()
}
```

### 2. Dependency Injection

```kotlin
// DI makes dependency management easier
class UserViewModel @Inject constructor(
    private val repository: UserRepository
) : ViewModel()
```

### 3. Reactive Programming

```kotlin
// Reactive approach manages data flow
private val _users = MutableStateFlow<List<User>>(emptyList())
val users: StateFlow<List<User>> = _users.asStateFlow()
```

### 4. Immutable State

```kotlin
// Immutable state provides thread safety
data class User(val name: String, val age: Int)

// Mutable alternative
data class User(var name: String, var age: Int)
```

---

## Resources

- [Android Architecture Guide - Google](https://developer.android.com/topic/architecture)
- [Guide to app architecture - Modern Best Practices](https://developer.android.com/topic/architecture/recommendations)
- [Clean Architecture - Robert C. Martin](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)
- [MVVM vs MVP - Detailed Comparison](https://proandroiddev.com/mvp-vs-mvvm-vs-mvi-6843b9313b00)

---

## Summary

**Observations on Architectural Patterns:**

Each architectural pattern has advantages and disadvantages in different project contexts. The choice of architecture varies depending on project requirements, team structure, and long-term goals.

**Industry Trends (2025):**

- MVVM and Clean Architecture are approaches recommended in Google's Android Architecture Guide and widely adopted in the industry
- MVC and MVP are rarely preferred in new projects, mostly seen in legacy code maintenance
- With the widespread adoption of Jetpack Compose, reactive state management approaches have been increasingly embraced

**General Assessment:**

- Simple projects can generally be managed with less layered architectures
- Complex projects can benefit from layered and modular architectures
- Testability and ease of maintenance are important factors in architecture selection
