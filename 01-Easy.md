# 🟢 Easy — Kotlin Coroutines

> Foundation questions. Every answer = **simple words + real-life example + Kotlin code.**

---

### 1. What is a Coroutine?

A lightweight "task" that can **pause and resume** without blocking a thread.

🏠 Like a waiter who places your order and serves other tables instead of standing at the kitchen door waiting.

```kotlin
CoroutineScope(Dispatchers.IO).launch {
    val user = fetchUser() // can suspend here, thread is free meanwhile
}
```

---

### 2. Why do we need Coroutines?

To do slow work (network, DB) **without freezing the UI**, and without the complexity of raw threads/callbacks.

🏠 Instead of standing in a queue for a coffee, you get a token and sit down — you're notified when it's ready.

```kotlin
viewModelScope.launch {
    val data = repository.getData() // UI thread stays responsive
    showResult(data)
}
```

---

### 3. Coroutine vs Thread

A **Thread** is a real OS worker (heavy). A **Coroutine** is a task that runs *on* a thread and can be paused — thousands can share a few threads.

🏠 Threads are cashiers; coroutines are customers. One cashier can serve many customers by handling one, pausing, serving another.

```kotlin
// 1 thread, 10,000 coroutines — totally fine
repeat(10_000) {
    launch(Dispatchers.Default) { delay(1000) }
}
```

---

### 4. What is `suspend`?

A marker that says: "this function may pause here and resume later, without blocking the thread."

🏠 Like a pause button on a video call — the call isn't dropped, it just resumes later.

```kotlin
suspend fun fetchUser(): User {
    delay(500) // pauses the coroutine, not the thread
    return repository.getUser()
}
```

> ⚠️ `suspend` does **not** mean "runs on a background thread automatically."

---

### 5. What is `CoroutineScope`?

Defines the **lifetime** — when its scope dies, all coroutines launched in it are cancelled.

🏠 A restaurant shift: when the shift ends, every pending order tied to it stops.

```kotlin
class MyViewModel : ViewModel() {
    fun load() {
        viewModelScope.launch { fetchUser() } // dies with the ViewModel
    }
}
```

---

### 6. What is `launch`?

Starts a coroutine that runs **fire-and-forget** — returns a `Job`, no result value.

🏠 Telling the kitchen "start cooking" — you don't wait at the counter.

```kotlin
scope.launch {
    println("Cooking started")
    delay(1000)
    println("Dish ready")
}
```

---

### 7. What is `async`?

Starts a coroutine that **returns a result** wrapped in a `Deferred`, fetched later via `.await()`.

🏠 Ordering food and getting a token number — you collect the result when it's ready.

```kotlin
val deferred = scope.async { fetchUser() }
// do other work here
val user = deferred.await()
```

---

### 8. What is `await()`?

Suspends until the `Deferred` (from `async`) has a result, then returns it.

🏠 Checking your token board until your order number lights up.

```kotlin
val user = async { fetchUser() }.await()
println(user.name)
```

---

### 9. `launch` vs `async`

| | `launch` | `async` |
|---|---|---|
| Returns | `Job` | `Deferred<T>` |
| Use when | no result needed | you need a result |

```kotlin
launch { saveLogs() }              // fire and forget
val result = async { getScore() }  // need the value
```

---

### 10. What is `runBlocking`?

Starts a coroutine and **blocks the current thread** until it finishes. Mainly for `main()` and tests — avoid in Android UI code.

🏠 A manager who stops all other work and personally waits for one task to finish.

```kotlin
fun main() = runBlocking {
    delay(1000)
    println("Done") // main() blocks here until this completes
}
```

---

### 11. What is `delay()`?

A **suspending** version of `Thread.sleep()` — pauses the coroutine, frees the thread for other work.

```kotlin
launch {
    println("A")
    delay(1000) // thread is free to do other work meanwhile
    println("B")
}
```

---

### 12. What are Dispatchers?

They decide **which thread pool** a coroutine runs on: `Main`, `IO`, or `Default`.

🏠 Departments in an office — reception (Main), the mailroom (IO), and the number-crunching team (Default).

```kotlin
launch(Dispatchers.IO) { readFile() }
```

---

### 13. `Dispatchers.Main`

Runs on the **UI thread** — use it for updating views/state.

```kotlin
withContext(Dispatchers.Main) {
    textView.text = "Updated!"
}
```

---

### 14. `Dispatchers.IO`

Optimized for **network/disk/database** work — lots of threads, mostly waiting.

```kotlin
withContext(Dispatchers.IO) {
    val response = api.getUser()
}
```

---

### 15. `Dispatchers.Default`

Optimized for **CPU-heavy** work like sorting, parsing, image processing — thread count matches CPU cores.

```kotlin
withContext(Dispatchers.Default) {
    val sorted = hugeList.sortedBy { it.value }
}
```

---

### 16. What is `withContext()`?

Switches the dispatcher for a block of code, then **switches back automatically** when done.

🏠 Stepping into the kitchen to cook, then returning to the counter — same coroutine, different "room."

```kotlin
suspend fun getUser(): User = withContext(Dispatchers.IO) {
    api.fetchUser() // runs on IO, caller gets result back on original dispatcher
}
```

---

### 17. What is a `Job`?

A **handle** to a coroutine — lets you track, cancel, or wait for it.

```kotlin
val job = launch { doWork() }
job.cancel()   // stop it
job.join()     // wait for it to finish
```

---

### 18. Basic Cancellation

Calling `job.cancel()` cooperatively stops a coroutine — it checks for cancellation at suspend points like `delay()`.

🏠 Telling a worker "stop" — they finish their current step, notice the message, and stop.

```kotlin
val job = launch {
    repeat(1000) { i ->
        println("Working $i")
        delay(500) // cancellation is checked here
    }
}
delay(1200)
job.cancel() // stops around iteration 2-3
```

---
