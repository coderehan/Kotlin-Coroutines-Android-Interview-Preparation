# 🟡 Medium — Kotlin Coroutines

> Structured concurrency, exceptions, cancellation, and real Android usage.
---

### 1. Structured Concurrency

The rule that a coroutine's **lifetime is tied to its scope** — no orphaned background work, parents wait for children.

🏠 A manager (parent) doesn't clock out until every team member (child) finishes their task.

```kotlin
scope.launch {
    launch { task1() }
    launch { task2() }
} // parent completes only after both children finish
```

---

### 2. Parent & Child Coroutines

A coroutine started inside another becomes its **child** — cancelling the parent cancels all children.

```kotlin
val parent = launch {
    launch { println("child 1") }
    launch { println("child 2") }
}
parent.cancel() // cancels both children too
```

---

### 3. Coroutine Hierarchy

Jobs form a **tree**: `Job` → children `Job`s → grandchildren. Cancellation and failure flow up/down this tree.

```kotlin
launch { // parent Job
    launch { // child Job
        launch { /* grandchild Job */ }
    }
}
```

---

### 4. `coroutineScope` function

A suspend function that creates a **new scope** and waits for all children — if any child fails, the whole scope fails.

🏠 A group photo: everyone must be ready before the picture is taken; one person messing up ruins it for all.

```kotlin
suspend fun loadDashboard() = coroutineScope {
    val user = async { getUser() }
    val posts = async { getPosts() }
    Dashboard(user.await(), posts.await())
}
```

---

### 5. `supervisorScope`

Like `coroutineScope`, but a **child's failure doesn't cancel siblings** — each fails independently.

🏠 Independent food stalls at a festival: one stall closing doesn't shut down the others.

```kotlin
supervisorScope {
    launch { riskyTask1() } // fails, doesn't affect below
    launch { riskyTask2() }
}
```

---

### 6. `SupervisorJob`

A `Job` type where a **child's failure is isolated** — used to build scopes where one task's crash shouldn't kill others.

```kotlin
val scope = CoroutineScope(SupervisorJob() + Dispatchers.Main)
scope.launch { throw Exception("A") } // doesn't cancel scope
scope.launch { println("still runs") }
```

---

### 7. `coroutineScope` vs `supervisorScope`

| | `coroutineScope` | `supervisorScope` |
|---|---|---|
| One child fails | all cancelled | others continue |
| Use case | all-or-nothing tasks | independent tasks |

---

### 8. Exception Propagation

By default, an exception in a child **cancels the parent and siblings**, then propagates up.

```kotlin
launch {
    launch { throw RuntimeException("boom") } // cancels parent too
    launch { /* also cancelled */ }
}
```

---

### 9. `launch` Exception Behaviour

Exceptions in `launch` are thrown **immediately** when they happen (not on `.join()`), and propagate up the hierarchy.

```kotlin
val job = launch {
    throw IllegalStateException("fails right away")
}
```

---

### 10. `async` Exception Behaviour

Exceptions in `async` are **held inside the `Deferred`** and only thrown when you call `.await()`.

```kotlin
val deferred = async { throw IllegalStateException("boom") }
// nothing happens yet...
deferred.await() // exception thrown here
```

---

### 11. `try/catch` with Coroutines

Works normally around suspend calls — but wrapping `async` won't catch its exception until `.await()` is inside the `try`.

```kotlin
try {
    val result = async { riskyCall() }.await()
} catch (e: Exception) {
    println("Caught: ${e.message}")
}
```

---

### 12. `CoroutineExceptionHandler`

A **global catch-all** for uncaught exceptions in `launch` coroutines — attach it to the scope/context. Doesn't work with `async`.

```kotlin
val handler = CoroutineExceptionHandler { _, e ->
    println("Caught: ${e.message}")
}
CoroutineScope(SupervisorJob() + handler).launch {
    throw RuntimeException("boom")
}
```

---

### 13. `CoroutineExceptionHandler` vs `try/catch`

| | `try/catch` | `CoroutineExceptionHandler` |
|---|---|---|
| Scope | one specific block | whole coroutine/scope |
| Works with `async` | yes (at `.await()`) | ❌ no |

---

### 14. Cancellation Propagation

Cancelling a **parent** cancels all its children. Cancelling a **child** does *not* cancel the parent or siblings.

```kotlin
val parent = launch {
    val child = launch { delay(2000) }
} // parent.cancel() -> child cancelled too
```

---

### 15. `isActive`

A property to **manually check** if a coroutine is still active — useful inside CPU-heavy loops with no suspend points.

```kotlin
launch {
    while (isActive) {
        // do chunk of work, checks cancellation each loop
    }
}
```

---

### 16. `ensureActive()`

Throws `CancellationException` immediately if the coroutine was cancelled — a shorthand for `if (!isActive) throw ...`.

```kotlin
launch {
    for (i in 1..1_000_000) {
        ensureActive() // throws if cancelled, stops the loop
        compute(i)
    }
}
```

---

### 17. `join()`

Suspends the caller until the coroutine (`Job`) **finishes** — doesn't return a value.

```kotlin
val job = launch { doWork() }
job.join() // waits here until doWork() completes
println("Done!")
```

---

### 18. `cancelAndJoin()`

Cancels a `Job` **and** waits for it to actually finish cancelling — combines `cancel()` + `join()`.

```kotlin
val job = launch { repeat(10) { delay(300) } }
job.cancelAndJoin() // guaranteed stopped before next line runs
```

---

### 19. `NonCancellable`

A special context used to run **cleanup code that must complete even if the coroutine is cancelled**.

🏠 A restaurant closing early still lets the chef finish plating the current dish before locking up.

```kotlin
launch {
    try {
        doWork()
    } finally {
        withContext(NonCancellable) {
            saveState() // runs even if cancelled
        }
    }
}
```

---

### 20. Sequential vs Parallel

`launch`/`async` calls run **sequentially** if you `await()` immediately; run them together first, then `await()`, for parallel execution.

```kotlin
// Sequential (slower): ~2000ms
val a = async { task1() }.await()
val b = async { task2() }.await()

// Parallel (faster): ~1000ms
val a2 = async { task1() }
val b2 = async { task2() }
val results = awaitAll(a2, b2)
```

---

### 21. Parallel API Calls

Launch multiple `async` calls, then gather results with `awaitAll()` — total time = the *slowest* call, not the sum.

```kotlin
suspend fun loadHome() = coroutineScope {
    val user = async { api.getUser() }
    val feed = async { api.getFeed() }
    HomeData(user.await(), feed.await())
}
```

---

### 22. `async` and Structured Concurrency

An `async` started but never `await`-ed **still runs to completion** as a child of its scope — its exception is only surfaced when awaited.

```kotlin
coroutineScope {
    async { riskyCall() } // runs, but exception only surfaces if awaited
}
```

---

### 23. `withContext` vs `async`

| | `withContext` | `async` |
|---|---|---|
| Runs | sequentially, switches dispatcher | starts *concurrently* |
| Use when | one suspend call, need dispatcher switch | multiple things in parallel |

```kotlin
val user = withContext(Dispatchers.IO) { api.getUser() } // one call
val (a, b) = coroutineScope { async { f1() } to async { f2() } } // parallel
```

---

### 24. Choosing a Dispatcher

`Main` for UI, `IO` for network/disk/DB, `Default` for CPU-heavy work (sorting, JSON parsing, image processing).

```kotlin
withContext(Dispatchers.IO) { db.insert(user) }
withContext(Dispatchers.Default) { image.applyFilter() }
```

---

### 25. Common Mistakes

- Using `GlobalScope` (leaks, outlives the screen)
- Swallowing `CancellationException` in `catch (e: Exception)`
- Calling `.await()` right after each `async` (kills parallelism)
- Blocking calls (`Thread.sleep`) instead of `delay()`

```kotlin
// ❌ swallows cancellation silently
catch (e: Exception) { log(e) }

// ✅ rethrow cancellation
catch (e: Exception) {
    if (e is CancellationException) throw e
    log(e)
}
```

---

### 26. Android `viewModelScope`

A built-in `CoroutineScope` tied to the `ViewModel`'s lifecycle — auto-cancelled in `onCleared()`.

```kotlin
class UserViewModel : ViewModel() {
    fun loadUser() = viewModelScope.launch {
        val user = repository.getUser() // cancelled automatically on screen exit
    }
}
```

---

### 27. Real-World Repository Example

A typical layered flow: `ViewModel` → `Repository` → `Dispatchers.IO` for the actual network/DB call.

```kotlin
class UserRepository(private val api: ApiService) {
    suspend fun getUser(id: String): User = withContext(Dispatchers.IO) {
        api.fetchUser(id)
    }
}

class UserViewModel(private val repo: UserRepository) : ViewModel() {
    fun load(id: String) = viewModelScope.launch {
        val user = repo.getUser(id)
        _uiState.value = UiState.Success(user)
    }
}
```

---

        
