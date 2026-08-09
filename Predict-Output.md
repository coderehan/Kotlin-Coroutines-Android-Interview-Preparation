<div align="center">

# 🧪 Kotlin Coroutines — Predict the Output

**21 scenario-based questions.** Read the code, guess the output, then reveal the answer.

</div>

---

## 💡 Why this file is different

Interviewers rarely ask *"what is `launch`?"* anymore — they paste a code snippet and ask **"what gets printed, and in what order?"**

This file trains exactly that skill. Every question:

1. Shows a small, **fully commented** piece of code
2. Lets you **think first** — the output is hidden behind a spoiler
3. Reveals the **output + a step-by-step "why"** once you click

> 🧠 Cover the answer with your hand (mentally), predict the output on paper first, *then* click to reveal. That's how this repo actually helps you remember.

---

## 🟢 Level 1 — Basic

### Q1

```kotlin
fun main() = runBlocking {
    println("1") // runs first — plain sequential code
    println("2") // runs second — sequential
    println("3") // runs third — sequential
}
```

<details>
<summary>📤 Reveal Output & Explanation</summary>

**Output:**
```
1
2
3
```

**Why:** No coroutine is actually launched here — `runBlocking` just gives you a coroutine scope to call suspend functions in. With nothing suspending or running concurrently, this is just plain, top-to-bottom sequential code.

</details>

---

### Q2

```kotlin
fun main() = runBlocking {
    println("Start") // 1. prints immediately

    launch {
        // this block is only *scheduled* here —
        // it won't actually run until the current
        // coroutine (main) suspends or finishes
        println("Launch") // 3. prints after "End"
    }

    // main coroutine keeps running immediately after
    // launch{} — it does NOT wait for launch to run
    println("End") // 2. prints before "Launch"
}
```

<details>
<summary>📤 Reveal Output & Explanation</summary>

**Output:**
```
Start
End
Launch
```

**Why:** `launch` is fire-and-forget — calling it only *schedules* the new coroutine, it doesn't hand control over immediately. The `main` coroutine keeps running past the `launch{}` block and prints `"End"` first. Only once `main` has nothing left to do does the scheduled `launch` coroutine actually get a turn to run.

</details>

---

### Q3

```kotlin
fun main() = runBlocking {
    println("Start") // 1. prints immediately

    launch {
        delay(1000) // suspends THIS coroutine for 1 second —
                    // does not block the thread
        println("Launch") // 3. prints after 1 second
    }

    // main keeps going immediately, it doesn't wait for launch
    println("End") // 2. prints before "Launch"
}
```

<details>
<summary>📤 Reveal Output & Explanation</summary>

**Output:**
```
Start
End
Launch
```

**Why:** The `launch` coroutine starts, immediately hits `delay(1000)`, and suspends itself for a second. While it's suspended, `main` is free to keep running and prints `"End"`. A full second later, the `launch` coroutine resumes and prints `"Launch"`.

</details>

---

### Q4

```kotlin
fun main() = runBlocking {
    // repeat() creates 3 launch coroutines back-to-back,
    // rapidly — all 3 are *scheduled* immediately,
    // but none of them actually run yet
    repeat(3) { index ->
        launch {
            println("Launch $index")
        }
    }
    // main keeps running past the loop and prints
    // "End" before any of the 3 launches get a turn
    println("End")
}
```

<details>
<summary>📤 Reveal Output & Explanation</summary>

**Output:**
```
End
Launch 0
Launch 1
Launch 2
```

**Why:** All three `launch` calls only *schedule* their coroutines — none of them run yet. `main` continues past the loop and prints `"End"` first. Once `main` is done, the three scheduled coroutines run **in the order they were launched**: 0, 1, 2.

</details>

---

## 🟡 Level 2 — Intermediate

### Q5

```kotlin
fun main() = runBlocking {
    println("Start") // 1. prints immediately

    launch {
        // 3. runs once main suspends below
        println("Launch 1")      // 3. prints
        delay(200)                // suspends for 200ms
        println("Launch 1 done") // 6. prints after 200ms
    }

    launch {
        // 4. runs right after Launch 1 starts
        println("Launch 2")      // 4. prints
        delay(100)                // suspends for 100ms
        println("Launch 2 done") // 5. prints after 100ms
    }

    // main continues immediately, doesn't wait for either launch
    println("End") // 2. prints before both launches
}
```

<details>
<summary>📤 Reveal Output & Explanation</summary>

**Output:**
```
Start
End
Launch 1
Launch 2
Launch 2 done
Launch 1 done
```

**Why:** `"Start"` prints immediately, both `launch` blocks are scheduled but don't run yet, so `"End"` prints next. Then Launch 1 runs up to its `delay(200)`, followed by Launch 2 running up to its `delay(100)`. Since Launch 2's delay is shorter, it resumes and finishes first (`"Launch 2 done"`), then Launch 1 finishes 100ms later (`"Launch 1 done"`).

</details>

---

### Q6

```kotlin
fun main() = runBlocking {
    println("Start") // 1. prints immediately

    val job = launch {
        delay(500)              // would wait 500ms...
        println("Launch done") // ...but this is never reached!
    }

    // cancel() is called immediately, before the coroutine
    // even gets a chance to start running its body
    job.cancel()
    println("Job cancelled") // 2. prints immediately
    println("End")           // 3. prints immediately
}
```

<details>
<summary>📤 Reveal Output & Explanation</summary>

**Output:**
```
Start
Job cancelled
End
```

**Why:** The `launch` coroutine is only scheduled — it hasn't started running its body yet. `job.cancel()` is called right after, before the coroutine ever gets a turn, so it's cancelled before `delay(500)` even begins. `"Launch done"` never prints.

</details>

---

### Q7

```kotlin
fun main() = runBlocking {
    println("Start") // 1. prints immediately

    launch {
        // this whole launch is scheduled — runs
        // once main continues past it

        withContext(Dispatchers.IO) {
            // switches to an IO thread and
            // SUSPENDS this coroutine until done
            println("Inside withContext") // 3. prints on IO thread
        }
        // back on the original dispatcher after withContext returns
        println("After withContext") // 4. prints right after
    }

    // main continues immediately, doesn't wait for launch
    println("End") // 2. prints before launch runs at all
}
```

<details>
<summary>📤 Reveal Output & Explanation</summary>

**Output:**
```
Start
End
Inside withContext
After withContext
```

**Why:** `launch` is scheduled but doesn't block `main`, so `"End"` prints first. Once `main` is done, the `launch` coroutine runs: it switches to `Dispatchers.IO` via `withContext`, prints inside it, then automatically switches back and prints `"After withContext"`.

</details>

---

### Q8

```kotlin
fun main() = runBlocking {
    println("Start") // 1. prints immediately

    val result = async {
        // async starts running CONCURRENTLY, right away
        delay(100) // suspends for 100ms
        42         // this becomes the eventual result
    }

    // async doesn't block main — this prints immediately
    println("Before await") // 2. prints immediately

    // await() SUSPENDS main until async's result is ready
    println("Result: ${result.await()}") // 3. prints after 100ms

    println("End") // 4. prints last
}
```

<details>
<summary>📤 Reveal Output & Explanation</summary>

**Output:**
```
Start
Before await
Result: 42
End
```

**Why:** `async` starts running concurrently as soon as it's called, but doesn't block `main` — so `"Before await"` prints immediately. `await()` then suspends `main` until the `async` block finishes its `delay(100)` and returns `42`, at which point `"Result: 42"` prints.

</details>

---

### Q9

```kotlin
fun main() = runBlocking {
    println("Start") // 1. prints immediately

    launch {
        try {
            delay(100)              // suspends for 100ms
            println("Launch done") // never reached — cancelled first
        } finally {
            // finally ALWAYS runs, even on cancellation —
            // this is exactly why it's used for cleanup
            println("Finally in launch") // 4. prints on cancel
        }
    }

    delay(50) // 2. main suspends for 50ms;
              // the launch above has started but is still
              // waiting inside its own delay(100)

    println("Cancelling") // 3. prints after 50ms

    // cancels every child coroutine of the current context
    coroutineContext.cancelChildren()

    println("End") // 5. prints last
}
```

<details>
<summary>📤 Reveal Output & Explanation</summary>

**Output:**
```
Start
Cancelling
Finally in launch
End
```

**Why:** The `launch` coroutine starts and suspends at `delay(100)`. Meanwhile `main` suspends for 50ms, then resumes and prints `"Cancelling"`. `cancelChildren()` cancels the still-waiting `launch`, which triggers its `finally` block (which always runs, cancelled or not) — printing `"Finally in launch"`. Finally, `"End"` prints.

**Key rule:** a `finally` block always runs, even when its coroutine is cancelled — that's exactly why it's the right place for cleanup like closing files or releasing resources.

</details>

---

### Q10

```kotlin
fun main() = runBlocking {
    println("Start") // 1. prints immediately

    launch {
        println("Launch 1 start") // 3. prints
        delay(200)                 // suspends 200ms
        println("Launch 1 end")   // 6. prints after 200ms
    }

    launch {
        println("Launch 2 start") // 4. prints
        delay(100)                 // suspends 100ms
        println("Launch 2 end")   // 5. prints after 100ms
    }

    // main suspends for 300ms — long enough for
    // BOTH launches above to fully complete
    delay(300)

    println("End") // 7. prints last, after 300ms
}
```

<details>
<summary>📤 Reveal Output & Explanation</summary>

**Output:**
```
Start
Launch 1 start
Launch 2 start
Launch 2 end
Launch 1 end
End
```

**Why:** Both `launch` blocks start and print their "start" messages before hitting their own `delay()`. Launch 2's shorter 100ms delay finishes first (`"Launch 2 end"`), then Launch 1's 200ms delay finishes (`"Launch 1 end"`). Once `main`'s own 300ms delay expires, `"End"` prints last.

</details>

---

## 🔴 Level 3 — Advanced

### Q11

```kotlin
fun main() = runBlocking {
    println("Start") // 1. prints immediately

    // Both async blocks start IMMEDIATELY and
    // run CONCURRENTLY — at the same time
    val deferred1 = async {
        delay(300)               // waits 300ms
        println("Async 1 done") // 4. prints after 300ms
        10                       // returns 10
    }

    val deferred2 = async {
        delay(100)               // waits only 100ms
        println("Async 2 done") // 3. prints after 100ms
        20                       // returns 20
    }

    // Awaiting deferred1 first suspends main until it
    // finishes (300ms) — but deferred2 is ALSO running
    // during that same wait, since both started already
    val sum = deferred1.await() + deferred2.await()
    // total wait ends up being ~300ms, NOT 400ms,
    // because the two run in parallel

    println("Sum: $sum") // 5. prints 30
    println("End")       // 6. prints last
}
```

<details>
<summary>📤 Reveal Output & Explanation</summary>

**Output:**
```
Start
Async 2 done
Async 1 done
Sum: 30
End
```

**Why:** Both `async` blocks start immediately and run concurrently, so `deferred2` (100ms) finishes before `deferred1` (300ms) — hence `"Async 2 done"` prints first. Total elapsed time is only ~300ms (the longer of the two), not 400ms — this is the entire point of `async`: parallel execution, not sequential.

</details>

---

### Q12

```kotlin
fun main() = runBlocking {
    println("Start") // 1. prints immediately

    launch {
        // outer launch is scheduled, runs after main continues

        launch {
            // inner launch is scheduled INSIDE outer —
            // it doesn't block outer from continuing
            delay(100)
            println("Inner launch") // 4. prints after 100ms
        }

        // outer continues immediately, doesn't wait for inner
        println("Outer launch") // 3. prints before inner
    }

    // main continues immediately past the outer launch
    println("End") // 2. prints first
}
```

<details>
<summary>📤 Reveal Output & Explanation</summary>

**Output:**
```
Start
End
Outer launch
Inner launch
```

**Why:** `main` runs past the outer `launch` and prints `"End"` first. The outer coroutine then runs: it schedules the inner `launch` (without waiting for it) and immediately prints `"Outer launch"`. Only after the outer block finishes does the inner coroutine get its turn, eventually printing `"Inner launch"` after its 100ms delay.

</details>

---

### Q13 — Tricky!

```kotlin
fun main() = runBlocking {
    println("Start") // 1. prints immediately

    try {
        launch {
            // an exception thrown inside launch propagates
            // to its PARENT SCOPE (runBlocking here) —
            // it does NOT go to the surrounding try/catch below
            throw Exception("Launch failed")
        }

        delay(100) // main suspends here; the launch above
                   // runs, throws, and the exception
                   // propagates up to runBlocking

        println("After launch") // never reached

    } catch (e: Exception) {
        // this does NOT catch the launch{} exception —
        // launch exceptions bypass a surrounding try/catch
        println("Caught: ${e.message}")
    }

    println("End") // may not even be reached
}
```

<details>
<summary>📤 Reveal Output & Explanation</summary>

**Output:**
```
Start
```
...then the app crashes with:
```
Exception in thread "main"
java.lang.Exception: Launch failed
```

**Why:** This is a classic trap. Exceptions thrown inside `launch` propagate up to the coroutine's **parent scope** — not to a `try/catch` wrapped around the `launch{}` call in the surrounding code. So the `catch` block here never fires, and the exception crashes the whole `runBlocking` scope instead.

**Correct way to catch it:**
```kotlin
launch {
    try {
        throw Exception("Launch failed")
    } catch (e: Exception) {
        println("Caught inside: ${e.message}") // ✅ this works
    }
}
```

</details>

---

### Q14

```kotlin
fun main() = runBlocking {
    println("Start") // 1. prints immediately

    // SupervisorJob makes children independent —
    // if one child fails, the others are NOT cancelled
    val scope = CoroutineScope(SupervisorJob())

    scope.launch {
        // Child 1 throws — with a plain Job(), this would
        // cancel Child 2 as well. With SupervisorJob,
        // only THIS child fails.
        throw Exception("Child 1 failed")
    }

    scope.launch {
        delay(100)
        // Child 2 is protected from Child 1's failure
        // thanks to SupervisorJob
        println("Child 2 done") // 3. prints after 100ms
    }

    // main waits long enough for both children to settle
    delay(200)

    println("End") // 4. prints last
}
```

<details>
<summary>📤 Reveal Output & Explanation</summary>

**Output:**
```
Start
Child 2 done
End
```

**Why:** With a `SupervisorJob`, a failing child does not cancel its siblings. Child 1 throws and fails on its own; Child 2 is unaffected and completes normally, printing `"Child 2 done"` after its 100ms delay.

> If a plain `Job()` had been used instead, Child 1's failure would cancel Child 2 too — and `"Child 2 done"` would never print.

</details>

---

### Q15

```kotlin
fun main() = runBlocking {
    println("Start") // 1. prints immediately

    withContext(Dispatchers.IO) {
        // switches to an IO thread — SUSPENDS runBlocking
        // until this whole block (including its children) is done
        println("IO 1") // 2. prints on IO thread

        launch {
            // this launch inherits the IO dispatcher,
            // but doesn't block withContext from continuing
            println("Launch inside IO") // 4. prints here
        }

        // withContext's own body continues immediately,
        // it doesn't wait for the launch above
        println("IO 2") // 3. prints immediately
    }
    // withContext only returns once ALL its children —
    // including the inner launch — have completed

    println("End") // 5. prints after everything above
}
```

<details>
<summary>📤 Reveal Output & Explanation</summary>

**Output:**
```
Start
IO 1
IO 2
Launch inside IO
End
```

**Why:** `withContext` suspends `main` and switches to `Dispatchers.IO`. Inside it, `"IO 1"` prints, then the inner `launch` is scheduled (without blocking), so `"IO 2"` prints next. Because `withContext` waits for **all** of its children to finish before returning, the inner `launch` runs and prints `"Launch inside IO"` before `withContext` completes and `main` resumes to print `"End"`.

</details>

---

### Q16

```kotlin
fun main() = runBlocking {
    println("Start") // 1. prints immediately

    val job1 = launch {
        delay(200)
        println("Job 1") // 4. prints after 200ms
    }

    val job2 = launch {
        delay(100)
        println("Job 2") // 3. prints after 100ms
    }

    // join() suspends main until job1 finishes —
    // but job2 keeps running freely during this wait too!
    job1.join() // waits for job1 (200ms)

    // by the time job1 finishes, job2 (only 100ms) has
    // already finished as well
    println("After join") // 5. prints after job1

    println("End") // 6. prints last
}
```

<details>
<summary>📤 Reveal Output & Explanation</summary>

**Output:**
```
Start
Job 2
Job 1
After join
End
```

**Why:** Both jobs start and run concurrently regardless of `join()`. `job2` (100ms) finishes first, `job1` (200ms) finishes second. `job1.join()` only waits for `job1` specifically — it doesn't pause `job2`, which is why `job2` is free to finish on its own in the meantime.

</details>

---

### Q17 — Tricky!

```kotlin
fun main() = runBlocking {
    println("Start") // 1. prints immediately

    async {
        delay(100)
        println("Async done") // 3. still prints, even unused!
        42                    // this value is simply never read
    }
    // Note: .await() is never called here —
    // but async is NOT lazy by default, so it still RUNS
    // in the background regardless of whether we use its result

    // main waits long enough for the async above to finish
    delay(200)

    println("End") // 4. prints last
}
```

<details>
<summary>📤 Reveal Output & Explanation</summary>

**Output:**
```
Start
Async done
End
```

**Why:** `async` is **not lazy by default** — it starts running the moment it's called, whether or not you ever call `.await()` on it. We just never read its returned value (`42`), but the body of the block still fully executes. Since `main`'s `delay(200)` gives it enough time, `"Async done"` prints before `"End"`.

> If `delay(200)` were removed, `main` (and `runBlocking`) could finish before the async block gets a chance to complete — and `"Async done"` might never print.

</details>

---

### Q18 — Tricky!

```kotlin
fun main() = runBlocking {
    println("Start") // 1. prints immediately

    // CoroutineStart.LAZY means this coroutine does NOT
    // start until something explicitly starts it —
    // either .start() or .await() being called
    val deferred = async(start = CoroutineStart.LAZY) {
        println("Async running") // 3. only prints once started
        42
    }

    // at this exact point, the async body has NOT run yet

    println("Before await") // 2. prints immediately

    // await() both STARTS the lazy async AND waits for its result
    println("Result: ${deferred.await()}") // 4. starts it, then waits

    println("End") // 5. prints last
}
```

<details>
<summary>📤 Reveal Output & Explanation</summary>

**Output:**
```
Start
Before await
Async running
Result: 42
End
```

**Why:** Because of `CoroutineStart.LAZY`, the `async` block doesn't run at all until something explicitly triggers it. `"Before await"` prints while the coroutine is still idle. Calling `.await()` is what actually **starts** the lazy coroutine — only then does `"Async running"` print, followed by the result.

> A regular `async` (without `LAZY`) starts immediately when called. `LAZY` only starts when `.start()` or `.await()` is explicitly invoked.

</details>

---

### Q19 — Very Tricky!

```kotlin
fun main() = runBlocking {
    println("Start") // 1. prints immediately

    launch(Dispatchers.Unconfined) {
        // Dispatchers.Unconfined is special: it starts running
        // on the CURRENT thread immediately, with no scheduling
        // delay at all — unlike a normal launch
        println("Unconfined 1") // 2. prints IMMEDIATELY

        delay(100) // after this suspension point, execution
                   // resumes on whatever thread the delay
                   // mechanism happens to use — not
                   // necessarily the original thread

        println("Unconfined 2") // 4. prints after the delay
    }

    // by the time we get here, "Unconfined 1" has
    // ALREADY printed — Unconfined ran immediately, inline
    println("End") // 3. prints after Unconfined 1, before Unconfined 2
}
```

<details>
<summary>📤 Reveal Output & Explanation</summary>

**Output:**
```
Start
Unconfined 1
End
Unconfined 2
```

**Why:** `Dispatchers.Unconfined` is unlike any other dispatcher — it runs on the **caller's current thread immediately**, without waiting to be scheduled, right up until its first suspension point. So `"Unconfined 1"` prints instantly, inline, before `main` even gets to its next line. `delay(100)` is that first suspension point — the coroutine pauses there, `main` continues and prints `"End"`, and only after the delay does `"Unconfined 2"` print.

> ⚠️ Avoid `Dispatchers.Unconfined` in production code — its behavior is unpredictable and hard to reason about compared to `Default`, `IO`, or `Main`.

</details>

---

### Q20 — SDE-2 Level!

```kotlin
fun main() = runBlocking {
    println("Start") // 1. prints immediately

    // coroutineScope suspends the CALLER until every
    // coroutine launched inside it has completed —
    // similar to runBlocking, but itself a suspend function
    coroutineScope {

        launch {
            delay(200)
            println("Launch 1") // 4. prints after 200ms
        }

        launch {
            delay(100)
            println("Launch 2") // 3. prints after 100ms
        }

        // this prints immediately — it doesn't wait
        // for either launch above to finish
        println("Inside coroutineScope") // 2. prints immediately
    }
    // execution is BLOCKED right here until both
    // launches above have fully completed

    println("After coroutineScope") // 5. prints after both are done
    println("End")                  // 6. prints last
}
```

<details>
<summary>📤 Reveal Output & Explanation</summary>

**Output:**
```
Start
Inside coroutineScope
Launch 2
Launch 1
After coroutineScope
End
```

**Why:** `coroutineScope` suspends `runBlocking` until everything inside it finishes. `"Inside coroutineScope"` prints immediately since it doesn't wait for the two `launch` blocks. Launch 2 (100ms) finishes before Launch 1 (200ms). Only once **both** are done does `coroutineScope` itself complete, letting `"After coroutineScope"` and `"End"` print.

**Key difference to remember:**
| | Behavior |
|---|---|
| `coroutineScope` | suspends the caller until all children finish, and propagates exceptions |
| `launch` | fire-and-forget, doesn't block the caller |

</details>

---

## 🎁 Bonus — Concept

### Q21

```kotlin
// coroutineScope — if ONE child fails, ALL siblings are cancelled
coroutineScope {
    launch { throw Exception("Failed") }
    launch {
        delay(100)
        println("This is CANCELLED") // never runs
    }
}

// supervisorScope — if ONE child fails, OTHER siblings continue
supervisorScope {
    launch { throw Exception("Failed") }
    launch {
        delay(100)
        println("This STILL runs") // ✅ runs normally
    }
}
```

<details>
<summary>📤 Reveal Explanation</summary>

**Why:** `coroutineScope` treats all of its children as one all-or-nothing unit — one failure cancels the rest. `supervisorScope` isolates failures per child, so a sibling's crash doesn't take down the others. Reach for `supervisorScope` whenever the tasks you're launching are genuinely independent of each other.

</details>

---
