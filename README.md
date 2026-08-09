<div align="center">

# 🚀 Kotlin Coroutines — Android Interview Preparation

**85 interview questions.** Simple words. One real-life example. One Kotlin snippet. No fluff.

![Kotlin](https://img.shields.io/badge/Kotlin-Coroutines-7F52FF?logo=kotlin&logoColor=white)
![Android](https://img.shields.io/badge/Android-Interview-3DDC84?logo=android&logoColor=white)
![Level](https://img.shields.io/badge/Level-Easy%20→%20Hard-orange)
![PRs](https://img.shields.io/badge/PRs-welcome-brightgreen)

</div>

---

## 💡 Why this repo

Most coroutine guides either dump the entire official documentation on you, or skip the "why" and just show code. This repo does neither:

> **Question → Simple-words answer → 1 real-life analogy → 1 runnable Kotlin snippet.**

That's it. Nothing to memorize, nothing to skim past.

---

## 🗂️ What's inside

| Level | File | Questions | Focus |
|:---:|---|:---:|---|
| 🟢 Easy | [`01-Easy.md`](01-Easy.md) | 18 | Coroutine basics, `launch`/`async`, Dispatchers |
| 🟡 Medium | [`02-Medium.md`](02-Medium.md) | 27 | Structured concurrency, exceptions, cancellation, `viewModelScope` |
| 🔴 Hard | [`03-Hard.md`](03-Hard.md) | 40 | Internals, `Flow`, concurrency safety, senior-level scenarios |

---

## 📖 Sample format

Every question in this repo looks like this:

> **What is `withContext()`?**
>
> Switches the dispatcher for a block of code, then switches back automatically when done.
>
> 🏠 *Stepping into the kitchen to cook, then returning to the counter — same coroutine, different "room."*
>
> ```kotlin
> suspend fun getUser(): User = withContext(Dispatchers.IO) {
>     api.fetchUser()
> }
> ```

---

## 🧭 How to use this repo

1. **Read** — question, answer, analogy.
2. **Type** the snippet yourself — don't copy-paste.
3. **Close the file** and explain it out loud, like an interviewer is listening.
4. Move to the next question.

Go in order: `Easy → Medium → Hard`. Each level builds on the last.

---

## 🤝 Contributing

Found a question that's still confusing, or want to add one? PRs are welcome — keep the same format: **simple words + real-life example + Kotlin snippet.**

---

<div align="center">

⭐ **If this helped you prepare, star the repo — it helps others find it too.**

</div>
