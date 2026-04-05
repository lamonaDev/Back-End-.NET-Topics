# Interlocked in C#

## 🧠 What Is It?

`Interlocked` is a static class that provides **atomic operations** on shared variables. "Atomic" means the operation completes as a single, uninterruptible unit — no thread can observe an intermediate state.

It's the **fastest and most lightweight** way to safely modify a shared numeric value across threads.

---

## 🌍 Real-World Analogy

Imagine a bank ATM. Two people withdraw from the same account simultaneously. If the ATM reads the balance, then another ATM reads the balance before the first one writes back — you get a corrupted total. 

`Interlocked` is like having a **single atomic teller window**: the read-modify-write is one indivisible action. Nobody else can sneak in between.

---

## ⚙️ Why Normal `counter++` Is Broken in Multithreading

`counter++` looks like one operation, but it's actually **three CPU instructions**:

```
1. READ  counter from memory → register
2. ADD   1 to register
3. WRITE register → memory
```

Two threads doing this simultaneously:
```
Thread A: READ  (counter = 5)
Thread B: READ  (counter = 5)   ← reads BEFORE A writes back
Thread A: WRITE (counter = 6)
Thread B: WRITE (counter = 6)   ← overwrites A's result!
```

**Result: counter = 6 instead of 7.** This is a **lost update** race condition.

---

## ✅ The Fix: `Interlocked.Increment`

```csharp
using System;
using System.Threading;
using System.Threading.Tasks;

class Program
{
    static int _counter = 0;

    static async Task Main()
    {
        // Launch 10 tasks, each incrementing 1000 times
        var tasks = Enumerable.Range(0, 10)
            .Select(_ => Task.Run(() =>
            {
                for (int i = 0; i < 1000; i++)
                    Interlocked.Increment(ref _counter); // Atomic!
            }));

        await Task.WhenAll(tasks);

        Console.WriteLine($"Final counter: {_counter}"); // Always 10000
    }
}
```

Without `Interlocked`, the result would be unpredictably **less than 10000**. With it — always exactly **10000**.

---

## 📚 All Interlocked Operations

```csharp
int val = 0;

// Atomic increment / decrement
Interlocked.Increment(ref val);      // val++ atomically
Interlocked.Decrement(ref val);      // val-- atomically

// Atomic add (returns NEW value)
int newVal = Interlocked.Add(ref val, 5);  // val += 5

// Atomic exchange (set and return old value)
int old = Interlocked.Exchange(ref val, 100); // val = 100; returns old val

// Compare-and-swap (CAS): only swaps if current value matches expected
int result = Interlocked.CompareExchange(ref val, 200, 100);
// If val == 100 → set val = 200, return 100
// If val != 100 → do nothing, return current val

// Read 64-bit value atomically (important on 32-bit OS)
long bigVal = 0;
long read = Interlocked.Read(ref bigVal);
```

---

## ⚙️ Memory Model — How It Works Under the Hood

```
CPU Core 1                  CPU Core 2
──────────────              ──────────────
Interlocked.Increment       Interlocked.Increment
     │                           │
     ▼                           ▼
[LOCK XADD instruction]     [LOCK XADD instruction]
     │                           │
     └──── Memory Bus Lock ──────┘
              (serialized at hardware level)
```

> `Interlocked` uses **CPU-level LOCK prefix** instructions (e.g., `LOCK XADD`, `LOCK CMPXCHG`). These are handled entirely in hardware — no OS kernel call, no context switch. Far cheaper than `lock {}` or `Monitor`.

---

## 🆚 Interlocked vs `lock`

| Feature | `Interlocked` | `lock` / Monitor |
|---|---|---|
| Performance | 🔥 Extremely fast | Slower (OS assist) |
| Works on | Single numeric/reference value | Any code block |
| Blocking | ❌ Never blocks | ✅ Can block threads |
| Scope | Single variable | Arbitrary code section |
| Best use | Counters, flags, CAS loops | Complex multi-step operations |

---

## 🧠 Compare-and-Swap (CAS) Pattern

CAS is the foundation of **lock-free data structures**:

```csharp
// Lock-free "lazy initialization" using CAS
private static MyService? _instance;

public static MyService GetInstance()
{
    if (_instance == null)
    {
        var newInstance = new MyService();
        // Only assign if _instance is still null
        Interlocked.CompareExchange(ref _instance, newInstance, null);
    }
    return _instance!;
}
```

---

## 📌 Summary

> `Interlocked` gives you **lock-free, thread-safe atomic operations** on single values at hardware speed. Use it for counters, flags, and simple shared state. For complex multi-step operations involving multiple variables, use `lock` instead.
