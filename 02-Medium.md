# 🟡 Medium — Kotlin Coroutines

> Structured concurrency, exceptions, cancellation, and real Android usage — explained with enough depth to actually stick.

---

### 1. Structured Concurrency

Structured concurrency is the rule that a coroutine's **lifetime is always tied to its parent scope** — a coroutine can never outlive the scope that launched it, and a parent won't be considered "done" until all of its children are done too. This is what prevents orphaned background work: with plain threads or callbacks, it's very easy to accidentally leave work running after the screen that started it is gone. Structured concurrency makes that class of bug structurally impossible, because the relationship between parent and child is enforced by the coroutine hierarchy itself.

🏠 A manager (parent) doesn't clock out and leave the building until every team member (child) working under them has finished their task for the day.

```kotlin
scope.launch {
    launch { task1() }
    launch { task2() }
} // the outer coroutine only completes after both inner ones finish
```

---

### 2. Parent & Child Coroutines

Any coroutine started **inside** another coroutine's scope automatically becomes its child. This parent-child relationship matters for two things: cancellation flows downward (cancelling the parent cancels all children), and completion flows upward (the parent only finishes once all its children have finished).

```kotlin
val parent = launch {
    launch { println("child 1") }
    launch { println("child 2") }
}
parent.cancel() // cancels both children too, automatically
```

---

### 3. Coroutine Hierarchy

Every `Job` (and its children, and their children) forms a **tree structure**, similar to a family tree or an org chart. Cancellation signals travel down this tree (parent → children → grandchildren), and failures typically travel up it (a failing grandchild can cancel its parent, which then cancels its siblings). Understanding this tree is key to reasoning about how cancellation and exceptions behave in complex coroutine code.

```kotlin
launch { // parent Job
    launch { // child Job
        launch { /* grandchild Job */ }
    }
}
```

---

### 4. `coroutineScope` function

`coroutineScope { }` is a suspend function that creates a **new child scope** and suspends the caller until every coroutine launched inside it completes. Its defining behavior is: if *any* child inside it fails, the whole `coroutineScope` block fails and cancels its other children too — it's an all-or-nothing unit. This is the go-to tool when you need multiple pieces of work to complete together, like fetching several pieces of data needed to render one screen.

🏠 A group photo: everyone has to be ready before the shutter clicks. If one person messes up their pose, the whole photo (the whole operation) is considered a failure — you don't get a partial photo.

```kotlin
suspend fun loadDashboard() = coroutineScope {
    val user = async { getUser() }
    val posts = async { getPosts() }
    Dashboard(user.await(), posts.await())
}
```

---

### 5. `supervisorScope`

`supervisorScope { }` behaves like `coroutineScope`, except that a **child's failure doesn't cancel its siblings** — each child coroutine fails or succeeds independently. This is useful when you're launching several unrelated tasks together and one failing shouldn't derail the others — for example, loading multiple independent widgets on a dashboard, where one widget's API failing shouldn't blank out the entire screen.

🏠 Independent food stalls at a festival — if one stall runs out of ingredients and has to close, the other stalls keep serving customers as normal.

```kotlin
supervisorScope {
    launch { riskyTask1() } // if this fails, it doesn't affect the line below
    launch { riskyTask2() }
}
```

---

### 6. `SupervisorJob`

`SupervisorJob` is a special `Job` implementation where **a child's failure is isolated** and does not cancel the job's other children or the job itself. It's commonly used when building a long-lived `CoroutineScope` (like a custom application-level scope) where you want independent pieces of background work that shouldn't take each other down if one fails.

```kotlin
val scope = CoroutineScope(SupervisorJob() + Dispatchers.Main)
scope.launch { throw Exception("A") } // this failure doesn't cancel the scope
scope.launch { println("still runs") }
```

---

### 7. `coroutineScope` vs `supervisorScope`

The key distinction is how they respond to **one child failing**. Use `coroutineScope` for all-or-nothing operations where every piece of work depends on the others succeeding together. Use `supervisorScope` for independent operations where one failing shouldn't take down the rest.

| | `coroutineScope` | `supervisorScope` |
|---|---|---|
| One child fails | all siblings are cancelled | others continue running |
| Use case | all-or-nothing tasks (e.g. combined data needed for one result) | independent tasks (e.g. multiple unrelated widgets) |

---

### 8. Exception Propagation

By default, an unhandled exception thrown inside a child coroutine **cancels its parent**, which in turn **cancels all of the parent's other children**, and the exception continues propagating up the hierarchy until something handles it (or it reaches the top and crashes the app). This aggressive propagation is intentional — it reflects structured concurrency's philosophy that a failure in one part of a "unit of work" usually means the whole unit is no longer meaningful.

```kotlin
launch {
    launch { throw RuntimeException("boom") } // this cancels the parent too
    launch { /* and this sibling gets cancelled as a result */ }
}
```

---

### 9. `launch` Exception Behaviour

An exception thrown inside a `launch` block is thrown **immediately, as soon as it happens** — it's not held back until you call `.join()` on the Job. It propagates up the coroutine hierarchy right away, following the exception propagation rules above.

```kotlin
val job = launch {
    throw IllegalStateException("fails right away, not when join() is called")
}
```

---

### 10. `async` Exception Behaviour

Unlike `launch`, an exception thrown inside `async` is **not thrown immediately** — it's captured and stored inside the resulting `Deferred`, and only re-thrown when you actually call `.await()` on it. If you never call `.await()`, you'll never see that exception surface directly (though it can still affect structured concurrency around it).

```kotlin
val deferred = async { throw IllegalStateException("boom") }
// nothing happens yet — the exception is just sitting inside deferred...
deferred.await() // the exception is thrown here, at the point of awaiting
```

---

### 11. `try/catch` with Coroutines

`try/catch` works around suspend calls exactly like normal Kotlin code — but there's a subtlety with `async`: since its exception is only thrown at `.await()`, your `try` block needs to wrap the `.await()` call itself, not just the `async { }` call, or you'll miss the exception.

```kotlin
try {
    val result = async { riskyCall() }.await() // await() must be inside the try
} catch (e: Exception) {
    println("Caught: ${e.message}")
}
```

---

### 12. `CoroutineExceptionHandler`

`CoroutineExceptionHandler` is a **global catch-all** you attach to a coroutine's context to handle exceptions that would otherwise crash the app — but it only works for `launch`-style coroutines (root coroutines specifically), not for exceptions thrown inside `async`, since those are meant to be handled at `.await()` instead.

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

Think of `try/catch` as handling failures **locally**, around a specific call, while `CoroutineExceptionHandler` is a **safety net** for the whole scope, catching anything that wasn't already handled locally. They're complementary, not competing tools — use `try/catch` where you can meaningfully react to a specific failure, and a handler as a last line of defense.

| | `try/catch` | `CoroutineExceptionHandler` |
|---|---|---|
| Scope | one specific block of code | the whole coroutine/scope |
| Works with `async` | yes, if wrapped around `.await()` | ❌ no, `async` exceptions surface at `.await()` |

---

### 14. Cancellation Propagation

Cancelling a **parent** coroutine cancels every one of its children, recursively down the whole tree. But cancellation does **not** travel upward or sideways: cancelling a *child* does not cancel its parent, and does not affect its siblings. This asymmetry (down = yes, up/sideways = no) is different from exception propagation, which does travel upward — it's a common point of confusion worth remembering clearly.

```kotlin
val parent = launch {
    val child = launch { delay(2000) }
} // calling parent.cancel() cancels the child too — but child.cancel() alone wouldn't touch the parent
```

---

### 15. `isActive`

`isActive` is a property you can check manually inside a coroutine to see if it's **still supposed to be running**. It's mainly useful inside tight, CPU-heavy loops that have no natural suspension points (like `delay()`) where cancellation would otherwise never get a chance to be noticed.

```kotlin
launch {
    while (isActive) {
        // do a chunk of work, checking cancellation status on every loop iteration
    }
}
```

---

### 16. `ensureActive()`

`ensureActive()` is essentially a shorthand for "if the coroutine was cancelled, throw `CancellationException` right now" — it's a cleaner, more idiomatic way to write `if (!isActive) throw CancellationException()` inside CPU-bound loops.

```kotlin
launch {
    for (i in 1..1_000_000) {
        ensureActive() // throws immediately if cancelled, stopping the loop cleanly
        compute(i)
    }
}
```

---

### 17. `join()`

`join()` is a suspending function that **pauses the caller until the target `Job` finishes** — whether it completes normally, throws, or is cancelled. Unlike `.await()`, `join()` doesn't give you back a result value; it's purely for waiting.

```kotlin
val job = launch { doWork() }
job.join() // suspends here until doWork() completes
println("Done!")
```

---

### 18. `cancelAndJoin()`

`cancelAndJoin()` combines `cancel()` and `join()` into one call: it requests cancellation **and** suspends until the coroutine has actually finished cancelling. This matters because cancellation is cooperative and not instant — calling plain `cancel()` doesn't guarantee the coroutine has fully stopped by the time your next line of code runs, but `cancelAndJoin()` does.

```kotlin
val job = launch { repeat(10) { delay(300) } }
job.cancelAndJoin() // guaranteed to be fully stopped before the next line runs
```

---

### 19. `NonCancellable`

`NonCancellable` is a special coroutine context used to run **cleanup code that absolutely must complete**, even if the surrounding coroutine has already been cancelled. Normally, once a coroutine is cancelled, any suspend call inside it (like a `delay()` in a `finally` block) would immediately throw — wrapping it in `withContext(NonCancellable)` protects it from that.

🏠 A restaurant closing early for the night still lets the chef finish plating the dish currently on the stove before locking the doors — that last step is protected, even though everything else has stopped.

```kotlin
launch {
    try {
        doWork()
    } finally {
        withContext(NonCancellable) {
            saveState() // this still completes even though the coroutine was cancelled
        }
    }
}
```

---

### 20. Sequential vs Parallel

If you call `async { }.await()` one after another, each one waits for the previous to finish before starting — effectively running **sequentially**, even though you used `async`. To actually run work in **parallel**, you need to start all the `async` calls first (without awaiting immediately), and only call `.await()` on them afterward.

```kotlin
// Sequential (slower): roughly 2000ms total
val a = async { task1() }.await()
val b = async { task2() }.await()

// Parallel (faster): roughly 1000ms total — both run at the same time
val a2 = async { task1() }
val b2 = async { task2() }
val results = awaitAll(a2, b2)
```

---

### 21. Parallel API Calls

To run multiple independent API calls at the same time, launch them all as `async` first, then gather the results together with `awaitAll()`. The total time taken is roughly the duration of the **slowest** call, rather than the sum of all of them — this is the main performance win of running things in parallel.

```kotlin
suspend fun loadHome() = coroutineScope {
    val user = async { api.getUser() }
    val feed = async { api.getFeed() }
    HomeData(user.await(), feed.await()) // both calls already ran concurrently
}
```

---

### 22. `async` and Structured Concurrency

Even if you never call `.await()` on an `async` coroutine, it still **runs to completion** as a proper child of its scope — it's not silently dropped. However, its exception will only actually surface (be thrown) if and when you call `.await()` on it; an unawaited failing `async` inside a `coroutineScope` can still affect the scope's structured concurrency behavior even without being awaited.

```kotlin
coroutineScope {
    async { riskyCall() } // this runs regardless — its exception only "surfaces" if awaited
}
```

---

### 23. `withContext` vs `async`

`withContext` runs a single suspend block on a different dispatcher, **sequentially** — it doesn't start new concurrent work, it just moves where the current work runs and waits for it to finish before continuing. `async` actually starts a **new, concurrently running** coroutine, useful specifically when you want multiple things happening at once.

| | `withContext` | `async` |
|---|---|---|
| Runs | sequentially, just switches dispatcher | starts genuinely *concurrent* work |
| Use when | one suspend call that needs a dispatcher switch | multiple things need to happen in parallel |

```kotlin
val user = withContext(Dispatchers.IO) { api.getUser() } // one sequential call
val (a, b) = coroutineScope { async { f1() } to async { f2() } } // genuinely parallel
```

---

### 24. Choosing a Dispatcher

Pick the dispatcher based on the **nature of the work**, not habit: `Main` for anything touching the UI, `IO` for network/disk/database calls (work that mostly waits), and `Default` for CPU-heavy work like sorting, JSON parsing, or image processing (work that mostly computes). Using the wrong dispatcher — like doing heavy computation on `IO`, or blocking calls on `Default` — hurts performance even though the code will "work."

```kotlin
withContext(Dispatchers.IO) { db.insert(user) }        // I/O-bound
withContext(Dispatchers.Default) { image.applyFilter() } // CPU-bound
```

---

### 25. Common Mistakes

A few mistakes come up over and over in real codebases and interviews:
- Using `GlobalScope` — its coroutines aren't tied to any lifecycle, so they can leak and outlive the screen that started them.
- Catching `Exception` broadly and accidentally swallowing `CancellationException`, which quietly breaks cancellation.
- Calling `.await()` immediately after each `async`, which accidentally turns parallel work back into sequential work.
- Using blocking calls like `Thread.sleep()` inside a coroutine instead of the suspending `delay()`, which defeats the purpose of using coroutines at all.

```kotlin
// ❌ swallows cancellation silently, which is a real bug
catch (e: Exception) { log(e) }

// ✅ always rethrow cancellation so it can propagate properly
catch (e: Exception) {
    if (e is CancellationException) throw e
    log(e)
}
```

---

### 26. Android `viewModelScope`

`viewModelScope` is a `CoroutineScope` built into Android's `ViewModel` class, automatically **cancelled when the ViewModel is cleared** (`onCleared()`), which typically happens when the screen is finally destroyed (not just rotated). This is exactly the kind of automatic lifecycle-tied cancellation that structured concurrency is designed to give you for free — no manual bookkeeping needed.

```kotlin
class UserViewModel : ViewModel() {
    fun loadUser() = viewModelScope.launch {
        val user = repository.getUser() // automatically cancelled if the screen is destroyed
    }
}
```

---

### 27. Real-World Repository Example

In a typical layered Android architecture, the `ViewModel` calls a `Repository`, and the `Repository` is responsible for actually performing the network/database call on the correct dispatcher (usually `Dispatchers.IO`) using `withContext`. This keeps dispatcher decisions localized to the data layer, so the `ViewModel` and UI layers don't need to know or care which thread the work happens on.

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
