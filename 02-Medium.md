🟡 Kotlin Coroutines — Medium Level

«Goal: Move from knowing Coroutine APIs to understanding how Coroutines behave in a real Android application.»

At Easy level, we learned:

Coroutine
   ↓
Scope
   ↓
launch / async
   ↓
Dispatcher
   ↓
suspend
   ↓
Job
   ↓
Cancellation

Now we go one level deeper:

                STRUCTURED CONCURRENCY
                         │
              ┌──────────┴──────────┐
              ↓                     ↓
        Parent / Child          Cancellation
              │                     │
              ↓                     ↓
        CoroutineScope        Propagation
              │
              ↓
        Exception Handling
              │
       ┌──────┴──────┐
       ↓             ↓
    Regular       Supervisor
     Scope          Scope

---

📚 Topics Covered

1. Structured Concurrency
2. Parent and Child Coroutines
3. Coroutine Hierarchy
4. "coroutineScope"
5. "supervisorScope"
6. "SupervisorJob"
7. "coroutineScope" vs "supervisorScope"
8. Exception Propagation
9. "launch" Exception Behaviour
10. "async" Exception Behaviour
11. "try/catch" with Coroutines
12. "CoroutineExceptionHandler"
13. "CoroutineExceptionHandler" vs "try/catch"
14. Cancellation Propagation
15. "isActive"
16. "ensureActive()"
17. "join()"
18. "cancelAndJoin()"
19. "NonCancellable"
20. Sequential vs Parallel Coroutines
21. Parallel API Calls
22. "async" and Structured Concurrency
23. "withContext" vs "async"
24. Dispatcher Selection
25. Common Coroutine Mistakes
26. Android "viewModelScope"
27. Real-World Repository Example

---

1. 🌳 What is Structured Concurrency?

🧠 Simple Explanation

Structured concurrency means:

«Coroutines should have a clear parent, lifetime and ownership.»

Instead of creating random background work, we create Coroutines inside a structured scope.

        Parent Scope
             │
       ┌─────┼─────┐
       ↓     ↓     ↓
    Child  Child  Child

The parent manages the children.

---

🏠 Real-Life Example

Imagine a manager with three employees.

              Manager
                 │
       ┌─────────┼─────────┐
       ↓         ↓         ↓
   Employee A Employee B Employee C

The manager knows:

- Who is working
- What they are doing
- When their work finishes
- When their work should stop

That's structured concurrency.

---

💻 Example

coroutineScope {

    launch {
        taskA()
    }

    launch {
        taskB()
    }
}

The parent scope waits for its children to complete.

---

🔑 Remember

«Structured concurrency = Coroutines have a clear parent and lifecycle.»

---

2. 👨‍👧 Parent and Child Coroutines

When you create a Coroutine inside another Coroutine, the inner Coroutine normally becomes a child of the outer Coroutine.

scope.launch {

    launch {
        taskA()
    }

    launch {
        taskB()
    }
}

Structure:

Parent
  │
  ├── Child A
  │
  └── Child B

---

🏠 Real-Life Example

A project manager assigns two tasks:

Project
  │
  ├── Task A
  └── Task B

The tasks belong to the project.

If the project is cancelled, its tasks should also stop.

---

3. 🌳 Coroutine Hierarchy

Think of a Coroutine hierarchy like a family tree.

               Parent
                 │
        ┌────────┴────────┐
        ↓                 ↓
      Child A           Child B
        │
     ┌──┴──┐
     ↓     ↓
 Grandchild

This hierarchy determines:

- Lifetime
- Cancellation
- Exception propagation

---

4. 🧩 What is "coroutineScope"?

🧠 Simple Explanation

"coroutineScope" creates a structured scope where child Coroutines must complete before the scope completes.

Example:

suspend fun loadData() {

    coroutineScope {

        launch {
            loadUser()
        }

        launch {
            loadOrders()
        }
    }

    println("Both completed")
}

The ""Both completed"" line executes only after the children complete successfully.

---

🏠 Real-Life Example

Imagine a manager says:

«"I need both employees to finish their tasks before I can submit the project."»

Manager
   │
   ├── Employee A
   │      ↓
   │   Complete
   │
   └── Employee B
          ↓
       Complete
            ↓
       Submit Project

That's similar to "coroutineScope".

---

🔑 Remember

«"coroutineScope" waits for all its children and follows normal failure propagation.»

---

5. 🛡️ What is "supervisorScope"?

"supervisorScope" is similar to "coroutineScope", but it changes how child failures affect sibling Coroutines.

Example:

supervisorScope {

    launch {
        throw Exception("Task A failed")
    }

    launch {
        println("Task B continues")
    }
}

Task A fails.

Task B can continue.

---

🏠 Real-Life Example

Imagine a manager with two independent employees.

             Manager
                │
        ┌───────┴───────┐
        ↓               ↓
   Employee A       Employee B
        ↓               ↓
      FAILED          CONTINUES

One employee's failure does not automatically stop the other independent employee.

---

🔑 Remember

«"supervisorScope" = children fail independently.»

---

6. 🛡️ What is "SupervisorJob"?

"SupervisorJob" provides supervisor-style failure behaviour for a Coroutine hierarchy.

Example:

val scope = CoroutineScope(
    SupervisorJob() + Dispatchers.Main
)

Now one child failing does not automatically cancel its sibling.

---

🏠 Real-Life Example

Think of a manager who says:

«"If one employee makes a mistake, don't fire everyone else."»

That's the idea behind supervision.

---

7. ⚔️ "coroutineScope" vs "supervisorScope"

Very common interview question.

"coroutineScope"| "supervisorScope"
Child failure affects the scope| Child failures are isolated
Failure propagates normally| Sibling can continue
Good for dependent work| Good for independent work
All-or-nothing style| Independent tasks

---

🧠 Easy Memory Trick

coroutineScope
      ↓
"One fails → group fails"

supervisorScope
      ↓
"One fails → others can continue"

---

8. 💥 Exception Propagation

Exceptions are another important part of Coroutine interviews.

Suppose:

coroutineScope {

    launch {
        throw Exception("Something went wrong")
    }

    launch {
        taskB()
    }
}

The exception can propagate to the parent.

That can cause sibling Coroutines to be cancelled.

---

🏠 Real-Life Example

Imagine a team working on one critical project.

If a critical task fails:

Task A
  ↓
FAILED
  ↓
Project fails
  ↓
Other project tasks stop

This is roughly the idea of regular structured failure propagation.

---

9. 💥 "launch" Exception Behaviour

If a Coroutine launched with "launch" throws an uncaught exception, that exception is propagated through its Coroutine hierarchy.

Example:

scope.launch {

    throw RuntimeException("Failed")
}

The exception doesn't become a normal return value.

You generally handle it using:

try/catch
CoroutineExceptionHandler
supervision

depending on where the exception originates and what behaviour you need.

---

10. 💥 "async" Exception Behaviour

"async" is different because the exception is associated with the "Deferred" result and is normally observed when you call:

await()

Example:

val result = async {

    throw Exception("Failed")
}

try {

    result.await()

} catch (e: Exception) {

    println("Handled")
}

---

🧠 Important Interview Point

Don't simply say:

«""async" always hides exceptions until "await()"."»

That's too simplistic.

In structured concurrency, an exception from an "async" child can still affect its parent/scope even if you haven't reached "await()" yet.

The key point is:

«"await()" is where you retrieve the result and can observe the Deferred's exception directly.»

---

11. 🧯 "try/catch" with Coroutines

You can use normal Kotlin "try/catch".

Example:

scope.launch {

    try {

        val user = repository.getUser()

        println(user)

    } catch (e: Exception) {

        println("Something went wrong")
    }
}

---

🏠 Real-Life Example

Something goes wrong while performing a task.

Task
 ↓
Error
 ↓
Catch
 ↓
Handle

---

🔑 Remember

«Use "try/catch" when you want to handle an exception directly around the operation.»

---

12. 🚨 "CoroutineExceptionHandler"

"CoroutineExceptionHandler" can handle uncaught exceptions from a Coroutine.

Example:

val handler = CoroutineExceptionHandler { _, exception ->

    println("Caught: ${exception.message}")
}

scope.launch(handler) {

    throw RuntimeException("Something went wrong")
}

---

⚠️ Important

A common interview mistake is saying:

«"CoroutineExceptionHandler catches every Coroutine exception."»

That's not correct.

It is primarily for uncaught exceptions.

It is not a replacement for "try/catch".

---

13. ⚔️ "CoroutineExceptionHandler" vs "try/catch"

"try/catch"| "CoroutineExceptionHandler"
Explicitly catches exceptions| Handles uncaught exceptions
Local handling| Coroutine-level handler
Can recover/control flow| Mainly final handling/logging
Works naturally around suspending calls| Not a replacement for "try/catch"

---

🧠 Memory Trick

try/catch
   ↓
"I want to handle this error here."

ExceptionHandler
   ↓
"This exception escaped without being handled."

---

14. 🛑 Cancellation Propagation

Cancellation usually flows from parent to child.

Parent cancelled
      ↓
Child A cancelled
      ↓
Child B cancelled
      ↓
Child C cancelled

Example:

val job = scope.launch {

    launch {
        delay(5000)
    }

    launch {
        delay(5000)
    }
}

job.cancel()

The children are cancelled as part of the structured hierarchy.

---

🏠 Real-Life Example

A manager cancels a project.

Project CANCELLED
       ↓
Task A CANCELLED
Task B CANCELLED
Task C CANCELLED

---

15. 🔍 What is "isActive"?

"isActive" tells you whether the current Coroutine is still active.

Example:

scope.launch {

    while (isActive) {

        doWork()
    }
}

When the Coroutine is cancelled:

isActive
   ↓
false
   ↓
Loop stops

---

🔑 Remember

«"isActive" = "Should I continue working?"»

---

16. 🛑 What is "ensureActive()"?

"ensureActive()" checks whether the Coroutine is still active.

If it has been cancelled, it throws a cancellation exception.

Example:

scope.launch {

    repeat(1000) {

        ensureActive()

        doWork()
    }
}

This is useful for CPU-heavy loops where you need explicit cancellation checks.

---

Difference

isActive
   ↓
Returns Boolean

ensureActive()
   ↓
Throws if cancelled

---

17. ⏳ What is "join()"?

"join()" waits for a Job to complete.

Example:

val job = launch {

    delay(1000)

    println("Finished")
}

job.join()

println("Now continue")

Output:

Finished
Now continue

---

🧠 Remember

launch()
   ↓
Job
   ↓
join()
   ↓
Wait for completion

---

18. 🛑 What is "cancelAndJoin()"?

Sometimes you want to:

1. Cancel a Job
2. Wait until it actually finishes cancellation

Use:

job.cancelAndJoin()

Equivalent idea:

job.cancel()
job.join()

---

🧠 Memory Trick

cancel()
   ↓
Stop it

join()
   ↓
Wait for it

cancelAndJoin()
   ↓
Do both

---

19. 🛡️ What is "NonCancellable"?

Sometimes cleanup must happen even when a Coroutine is being cancelled.

Example:

try {

    doWork()

} finally {

    withContext(NonCancellable) {
        saveCleanupData()
    }
}

---

🏠 Real-Life Example

Imagine cancelling a flight booking process.

Even though the booking is cancelled, you still need to:

Cancel booking
     ↓
Cleanup
     ↓
Release resources
     ↓
Finish

---

⚠️ Important

Don't use "NonCancellable" everywhere.

It deliberately prevents cancellation from affecting that block.

Use it only for short, necessary cleanup work.

---

20. 🔄 Sequential vs Parallel Coroutines

Suppose we need:

Get User
Get Orders

If they don't depend on each other, we can run them in parallel.

---

❌ Sequential

val user = getUser()

val orders = getOrders()

Timeline:

Get User
   ↓
   Done
   ↓
Get Orders
   ↓
   Done

---

✅ Parallel

coroutineScope {

    val user = async {
        getUser()
    }

    val orders = async {
        getOrders()
    }

    val userResult = user.await()
    val ordersResult = orders.await()
}

Timeline:

       ┌── Get User ──┐
Start ─┤              ├─ Complete
       └── Get Orders ┘

This can reduce total waiting time when the operations are independent.

---

21. ⚡ Parallel API Calls

This is a very common Android interview scenario.

Suppose a Home screen needs:

User Profile
Recommendations
Notifications

They don't depend on each other.

Instead of:

val profile = getProfile()
val recommendations = getRecommendations()
val notifications = getNotifications()

We can use:

coroutineScope {

    val profile = async {
        getProfile()
    }

    val recommendations = async {
        getRecommendations()
    }

    val notifications = async {
        getNotifications()
    }

    val profileResult = profile.await()
    val recommendationsResult = recommendations.await()
    val notificationsResult = notifications.await()
}

---

🏠 Real-Life Example

Three employees are asked to do three independent tasks:

             Manager
                │
      ┌─────────┼─────────┐
      ↓         ↓         ↓
   Profile   Recommend  Notify
      │         │         │
      └─────────┼─────────┘
                ↓
           Final Screen

---

22. 🧠 "async" and Structured Concurrency

A common mistake is:

GlobalScope.async {
    getUser()
}

This creates work that isn't tied to the caller's structured lifecycle.

Prefer:

coroutineScope {

    val user = async {
        getUser()
    }

    user.await()
}

---

🔑 Remember

«Use "async" inside an appropriate structured scope when the result belongs to the current operation.»

---

23. ⚔️ "withContext" vs "async"

Very common interview question.

"withContext"

Use when you want to:

«Switch context and get one result.»

val user = withContext(Dispatchers.IO) {
    repository.getUser()
}

"async"

Use when you want:

«Concurrent work that produces a result.»

val user = async {
    getUser()
}

val orders = async {
    getOrders()
}

---

🧠 Memory Trick

withContext
     ↓
"Switch context"

async
     ↓
"Run concurrent result-producing work"

---

24. 🚦 Choosing the Right Dispatcher

Use this mental table:

Work| Dispatcher
UI updates| "Main"
Network| "IO"
Database| "IO"
File operations| "IO"
Heavy calculations| "Default"
Large data processing| "Default"

---

⚠️ Important Interview Nuance

Don't blindly say:

«"Every repository call should use "Dispatchers.IO"."»

The layer that knows what kind of work is being done should generally choose the appropriate execution context, and good repository/API abstractions may already manage their own threading.

The important principle is:

«Don't perform blocking work on the Main thread.»

---

25. 🚫 Common Coroutine Mistakes

Mistake 1 — "GlobalScope"

GlobalScope.launch {
    doWork()
}

Problem:

No meaningful lifecycle owner
       ↓
Can outlive the screen/component

Prefer an appropriate scope:

viewModelScope.launch {
    doWork()
}

---

Mistake 2 — Blocking the Main Thread

Bad:

Thread.sleep(5000)

Better:

delay(5000)

---

Mistake 3 — Heavy CPU Work on Main

Bad:

withContext(Dispatchers.Main) {
    performHugeCalculation()
}

Better:

withContext(Dispatchers.Default) {
    performHugeCalculation()
}

---

Mistake 4 — Using "async" unnecessarily

Don't use:

val result = async {
    getUser()
}.await()

when simple:

val result = getUser()

is sufficient.

"async" is useful when you actually need concurrent result-producing work.

---

26. 📱 Android "viewModelScope"

Android provides:

viewModelScope

It is tied to the lifecycle of the ViewModel.

Example:

class UserViewModel(
    private val repository: UserRepository
) : ViewModel() {

    fun loadUser() {

        viewModelScope.launch {

            val user = repository.getUser()

            // Update UI state
        }
    }
}

When the ViewModel is cleared:

ViewModel cleared
       ↓
viewModelScope cancelled
       ↓
Child Coroutines cancelled

---

🏠 Real-Life Example

Think of a manager assigned to a project.

Manager exists
    ↓
Tasks running

Manager leaves project
    ↓
Owned tasks are cancelled

---

27. 🏗️ Real-World Repository Example

Imagine an Android application:

                 UI
                  ↓
              ViewModel
                  ↓
            viewModelScope
                  ↓
              Repository
                  ↓
              Network

Example:

class UserViewModel(
    private val repository: UserRepository
) : ViewModel() {

    fun loadUser() {

        viewModelScope.launch {

            try {

                val user = repository.getUser()

                println(user)

            } catch (e: Exception) {

                println("Failed to load user")
            }
        }
    }
}

Repository:

class UserRepository(
    private val api: UserApi
) {

    suspend fun getUser(): User {

        return api.getUser()
    }
}

The API call can be a suspending operation.

---

🔥 Real Android Flow

                  UI
                   │
                   │ User clicks
                   ↓
               ViewModel
                   │
                   │ viewModelScope.launch
                   ↓
               Repository
                   │
                   │ suspend
                   ↓
               Network
                   │
                   ↓
                Result
                   │
                   ↓
               ViewModel
                   │
                   ↓
               StateFlow
                   │
                   ↓
                  UI

---

🎯 Medium-Level Interview Questions

Q1. What is structured concurrency?

«Structured concurrency ensures Coroutines have a clear parent, lifecycle and ownership, making cancellation and failure management predictable.»

---

Q2. "coroutineScope" vs "supervisorScope"?

«"coroutineScope" follows normal structured failure propagation, while "supervisorScope" isolates child failures so one child failing doesn't automatically cancel its siblings.»

---

Q3. What is "SupervisorJob"?

«"SupervisorJob" creates a Job whose children can fail independently without automatically cancelling their siblings.»

---

Q4. What happens when a parent Coroutine is cancelled?

«Cancellation normally propagates to its child Coroutines.»

---

Q5. Is cancellation immediate?

«Cancellation is cooperative. Suspending functions usually check cancellation, while CPU-heavy code may need explicit checks such as "isActive" or "ensureActive()".»

---

Q6. "isActive" vs "ensureActive()"?

«"isActive" returns whether the Coroutine is active. "ensureActive()" throws a cancellation exception if the Coroutine is no longer active.»

---

Q7. "join()" vs "await()"?

«"join()" waits for a "Job" to complete. "await()" waits for a "Deferred" and returns its result.»

Job
 ↓
join()

Deferred<T>
 ↓
await()
 ↓
T

---

Q8. What is "NonCancellable"?

«It provides a context where cancellation does not cancel the work, typically used for short cleanup operations in "finally".»

---

Q9. "withContext" vs "async"?

«"withContext" is generally used to switch context and get a result, while "async" is used for concurrent result-producing work.»

---

Q10. How would you make three API calls in parallel?

«Use "async" inside a structured scope and call "await()" on the results.»

coroutineScope {

    val a = async { apiA() }
    val b = async { apiB() }
    val c = async { apiC() }

    val resultA = a.await()
    val resultB = b.await()
    val resultC = c.await()
}

---

🧠 Medium-Level Memory Map

                 COROUTINE
                     │
                 HAS OWNER
                     ↓
              CoroutineScope
                     │
                 Parent Job
                     │
          ┌──────────┴──────────┐
          ↓                     ↓
       Child A               Child B
          │                     │
          └──────────┬──────────┘
                     ↓
               STRUCTURED
               CO
