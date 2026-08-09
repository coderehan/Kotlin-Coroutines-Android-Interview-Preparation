# 🔴 Hard — Kotlin Coroutines

> Internals, Flow, concurrency safety & senior-level scenarios.
---

### 1. Suspension Internals

The Kotlin compiler rewrites `suspend` functions using **Continuation-Passing Style (CPS)** — an extra hidden `Continuation` parameter is added so the function can pause and resume state later.

```kotlin
// what you write:
suspend fun getUser(): User

// roughly what the compiler generates:
fun getUser(continuation: Continuation<User>): Any?
```

---

### 2. Continuation

An object that remembers **where to resume** and holds the result/exception — think of it as a "callback with saved state."

```kotlin
interface Continuation<in T> {
    val context: CoroutineContext
    fun resumeWith(result: Result<T>)
}
```

---

### 3. What Happens When a Function Suspends?

The function returns a special marker (`COROUTINE_SUSPENDED`), the thread is freed, and later the `Continuation.resumeWith()` call resumes execution exactly where it paused.

🏠 Bookmarking a page mid-book — you close it (thread freed), and resume from that exact page later.

---

### 4. Coroutine State Machine

Each `suspend` function is compiled into a **state machine** — every suspension point becomes a numbered state, with a `when` switching between them.

```kotlin
// Conceptually:
when (label) {
    0 -> { /* before delay() */ label = 1; delay(this) }
    1 -> { /* after delay() resumes here */ }
}
```

---

### 5. `CoroutineContext`

A set of elements (Job, Dispatcher, name, exception handler) that describe **how and where** a coroutine runs — like a bundle of settings.

```kotlin
val context = Dispatchers.IO + Job() + CoroutineName("Sync")
```

---

### 6. Context Elements

Common elements: `Job` (lifecycle), `CoroutineDispatcher` (thread pool), `CoroutineName` (debug label), `CoroutineExceptionHandler` (error handling).

```kotlin
launch(Dispatchers.IO + CoroutineName("FetchUser")) { getUser() }
```

---

### 7. Dispatcher Internals

Dispatchers are backed by **thread pools** (`IO` is large/elastic for blocking calls, `Default` is sized to CPU cores for compute work); `launch`/`withContext` route work to them via the context.

---

### 8. Context Combination

Contexts combine with `+`, and a later element **overrides** an earlier one of the same key.

```kotlin
val ctx = Dispatchers.IO + CoroutineName("A") + CoroutineName("B")
// name is "B" — it overrides "A"
```

---

### 9. `CoroutineName`

A debugging label attached to a coroutine's context — shows up in thread dumps/logs when debugging is enabled.

```kotlin
launch(CoroutineName("SyncUserData")) { sync() }
```

---

### 10. Job Hierarchy

Jobs form a **parent-child tree**; a parent only completes after all children complete, and cancelling a parent cancels the whole subtree.

```kotlin
val parentJob = Job()
val child = Job(parentJob) // child.parent == parentJob
```

---

### 11. `SupervisorJob` — Deep Dive

Unlike a normal `Job`, a child's **failure doesn't propagate up** — but cancellation of the supervisor still cancels all children (failure isolation is one-directional).

```kotlin
val supervisor = SupervisorJob()
val scope = CoroutineScope(supervisor)
scope.launch { throw Exception() } // isolated failure
scope.launch { println("still runs") }
```

---

### 12. Advanced Exception Propagation

In structured concurrency, the **first** child exception cancels siblings; further exceptions from other cancelled children are added as **suppressed exceptions** on the first one.

---

### 13. Cancellation vs Exception

Cancellation is a *signal*, implemented internally by throwing `CancellationException` — but it's treated specially: it doesn't trigger `CoroutineExceptionHandler` and shouldn't be logged as a real error.

```kotlin
catch (e: CancellationException) {
    throw e // always rethrow, never swallow
} catch (e: Exception) {
    log(e)
}
```

---

### 14. `CancellationException`

The exception type used internally to unwind a cancelled coroutine's stack — coroutines machinery treats it as "normal" cancellation, not a failure.

---

### 15. Don't Swallow `CancellationException`

Catching `Exception` broadly and not rethrowing `CancellationException` breaks cooperative cancellation — the coroutine looks alive but never actually stops.

```kotlin
// ❌ breaks cancellation
try { work() } catch (e: Exception) { log(e) }

// ✅ correct
try { work() } catch (e: CancellationException) { throw e }
  catch (e: Exception) { log(e) }
```

---

### 16. Shared Mutable State

Multiple coroutines writing to the **same variable concurrently** (e.g. on `Dispatchers.Default`) can corrupt data — coroutines don't make shared state automatically safe.

```kotlin
var counter = 0
repeat(1000) { launch(Dispatchers.Default) { counter++ } } // unsafe!
```

---

### 17. Race Condition

Two coroutines read-modify-write the same value **at the same time**, and one update overwrites the other — the final result becomes unpredictable.

🏠 Two people editing the same shared spreadsheet cell at once — the last save wins, the other's edit is lost.

---

### 18. `Mutex`

A coroutine-friendly **lock** — only one coroutine can hold it at a time, and waiting suspends instead of blocking the thread.

```kotlin
val mutex = Mutex()
var counter = 0
launch { mutex.withLock { counter++ } }
```

---

### 19. Atomic Operations

Thread-safe operations on a single value using classes like `AtomicInteger`, useful for simple counters without a full `Mutex`.

```kotlin
val counter = AtomicInteger(0)
repeat(1000) { launch(Dispatchers.Default) { counter.incrementAndGet() } }
```

---

### 20. Thread-Safe Coroutine Design

Prefer confining mutable state to **one coroutine/dispatcher**, or protect it with `Mutex`/`Atomic*`, instead of sharing raw mutable variables across coroutines.

---

### 21. `StateFlow`

A **hot**, observable state holder that always has a current value and only emits distinct updates — ideal for UI state.

```kotlin
private val _uiState = MutableStateFlow(UiState.Loading)
val uiState: StateFlow<UiState> = _uiState
```

---

### 22. `SharedFlow`

A **hot** stream for events (no initial value required), configurable replay/buffer — good for one-off events like "show a toast."

```kotlin
private val _events = MutableSharedFlow<String>()
val events: SharedFlow<String> = _events
```

---

### 23. `StateFlow` vs `SharedFlow`

| | `StateFlow` | `SharedFlow` |
|---|---|---|
| Initial value | required | optional |
| Use for | UI state | one-time events |
| Duplicate emissions | dropped if same value | configurable |

---

### 24. Cold vs Hot Flow

A **cold** `Flow` starts producing only when collected (each collector gets its own run). A **hot** flow (`StateFlow`/`SharedFlow`) runs independently and broadcasts to all collectors.

```kotlin
val cold = flow { emit(fetchData()) } // runs per collector
val hot = MutableStateFlow(0)         // runs once, shared
```

---

### 25. `stateIn`

Converts a cold `Flow` into a hot `StateFlow`, sharing one upstream execution among collectors.

```kotlin
val uiState = repository.userFlow
    .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5000), initial)
```

---

### 26. `shareIn`

Converts a cold `Flow` into a hot `SharedFlow`, for sharing events across multiple collectors.

```kotlin
val sharedEvents = eventsFlow.shareIn(scope, SharingStarted.Eagerly, replay = 0)
```

---

### 27. `flowOn`

Changes the dispatcher used for **upstream** operators only — collection downstream stays on the caller's dispatcher.

```kotlin
flow { emit(readFile()) }
    .flowOn(Dispatchers.IO) // upstream runs on IO
    .collect { println(it) } // collect runs on caller's dispatcher
```

---

### 28. `buffer()`

Lets the producer keep emitting **without waiting** for a slow collector, storing items in a buffer.

```kotlin
flow { repeat(10) { emit(it) } }
    .buffer()
    .collect { delay(100); println(it) }
```

---

### 29. `conflate()`

If the collector is slow, **skip intermediate values** and only deliver the latest one — no buffering of old values.

```kotlin
flow { repeat(10) { emit(it); delay(50) } }
    .conflate()
    .collect { delay(200); println(it) } // misses some in-between values
```

---

### 30. `collectLatest()`

Cancels the **previous collector block** whenever a new value arrives, and restarts it with the new value.

```kotlin
flow.collectLatest { value ->
    delay(1000) // cancelled if a newer value arrives mid-way
    println(value)
}
```

---

### 31. `debounce()`

Only emits a value after a specified **quiet period** with no new emissions — great for search-as-you-type.

```kotlin
searchQueryFlow
    .debounce(300)
    .collect { query -> search(query) }
```

---

### 32. `combine()`

Combines the **latest** values from multiple flows whenever *any* of them emits.

```kotlin
combine(nameFlow, ageFlow) { name, age -> "$name is $age" }
    .collect { println(it) }
```

---

### 33. `zip()`

Pairs values from two flows **in order**, one-to-one — waits for both sides to emit before producing a pair.

```kotlin
flowA.zip(flowB) { a, b -> a + b }.collect { println(it) }
```

---

### 34. `flatMapLatest()`

Maps each emission to a new flow, **cancelling the previous inner flow** when a new value arrives — like `switchMap`.

```kotlin
searchQueryFlow
    .flatMapLatest { query -> api.searchFlow(query) }
    .collect { results -> show(results) }
```

---

### 35. Backpressure

When a producer emits **faster** than a collector can consume — handled with `buffer()`, `conflate()`, or `collectLatest()` depending on whether you want to keep, skip, or restart.

---

### 36. Channels

A **hot** communication pipe between coroutines — supports send/receive, and unlike Flow, each value goes to only *one* receiver.

```kotlin
val channel = Channel<Int>()
launch { channel.send(1) }
launch { println(channel.receive()) }
```

---

### 37. Channel vs Flow

| | Channel | Flow |
|---|---|---|
| Type | hot, imperative | cold (by default), declarative |
| Delivery | one item → one receiver | one item → all collectors (if shared) |

---

### 38. `callbackFlow`

Bridges a **callback-based API** (like a Firebase listener) into a cold `Flow`.

```kotlin
fun listenToUpdates() = callbackFlow {
    val listener = Listener { trySend(it) }
    api.register(listener)
    awaitClose { api.unregister(listener) }
}
```

---

### 39. `repeatOnLifecycle`

Automatically starts/stops collecting a Flow based on **Lifecycle state** — prevents collecting when the UI is in the background.

```kotlin
lifecycleScope.launch {
    repeatOnLifecycle(Lifecycle.State.STARTED) {
        viewModel.uiState.collect { render(it) }
    }
}
```

---

### 40. Flow Cancellation

Flows are cooperative like coroutines — cancelling the collecting coroutine stops the Flow at its next suspension point (e.g. inside `emit`).

```kotlin
val job = launch { longFlow.collect { println(it) } }
delay(500)
job.cancel() // stops flow collection
```

---

### 41. Performance

Common wins: avoid `Dispatchers.Default` for I/O (starves CPU work), avoid unnecessary `withContext` hops, prefer `flowOn`/`buffer` over manual thread juggling, and don't over-launch thousands of coroutines doing tiny work.

---

### 42. Senior-Level Mistakes

- Mixing `GlobalScope` into production Android code
- Using `StateFlow` for one-time events (causes replays on rotation)
- Not using `SupervisorJob` where independent failures are expected
- Blocking calls inside `Dispatchers.Default`

---

### 43. Real-World Architecture

```
UI → ViewModel (viewModelScope) → UseCase → Repository (Dispatchers.IO)
                                                 │
                                        Network / Database
                                                 │
                                        Repository → Flow<Data>
                                                 │
                                    ViewModel → StateFlow<UiState> → UI
```

```kotlin
class UserRepository(private val api: Api, private val dao: UserDao) {
    fun observeUser(id: String): Flow<User> = dao.observeUser(id)
        .flowOn(Dispatchers.IO)

    suspend fun refreshUser(id: String) = withContext(Dispatchers.IO) {
        dao.insert(api.fetchUser(id))
    }
}
```

---

### 44. Senior Interview Scenario

**"Two API calls must run in parallel, both must succeed, and a screen rotation shouldn't cancel or restart them."**

Answer: use `viewModelScope` (survives rotation, tied to ViewModel) + `coroutineScope` with `async`/`awaitAll` for all-or-nothing parallelism, exposing the result via `StateFlow` so the UI just re-reads the latest state after rotation.

```kotlin
fun loadDashboard() = viewModelScope.launch {
    val (user, posts) = coroutineScope {
        val u = async { repo.getUser() }
        val p = async { repo.getPosts() }
        u.await() to p.await()
    }
    _uiState.value = UiState.Success(user, posts)
}
```

---
