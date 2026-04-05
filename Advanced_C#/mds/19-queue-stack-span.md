# Queue, Stack, Span\<T\> & Collections Guide

## 🧠 Overview

This guide covers three essential C# types: `Queue<T>` (FIFO), `Stack<T>` (LIFO), and `Span<T>` (zero-copy memory windows) — plus a full comparison of all collection types to help you choose the right one.

---

## 🟢 Part 1: Queue\<T\> — First In, First Out

### What Is It?

`Queue<T>` stores elements in **FIFO order** — first element added is the first one removed. Like a checkout line at a store: the first person to join is the first to be served.

### 🌍 Real-World Analogy

A **restaurant queue** — people enter at the back and leave from the front. The first person to arrive gets seated first. No cutting in line.

---

### ⚙️ How Queue Flows

```
Enqueue("First")  →  [First]
Enqueue("Second") →  [First][Second]
Enqueue("Third")  →  [First][Second][Third]
Dequeue()         →  returns "First"  →  [Second][Third]
Peek()            →  returns "Second" (not removed)  →  [Second][Third]
```

---

### 💻 Code Example

```csharp
Queue<string> printQueue = new();
printQueue.Enqueue("Document1.pdf"); // O(1) — add to back
printQueue.Enqueue("Document2.pdf");
printQueue.Enqueue("Document3.pdf");

Console.WriteLine($"Count: {printQueue.Count}");       // 3
Console.WriteLine($"Peek:  {printQueue.Peek()}");      // "Document1.pdf" (not removed)

// Process all items in FIFO order
while (printQueue.Count > 0)
{
    string doc = printQueue.Dequeue(); // O(1) — removes from front
    Console.WriteLine($"Printing: {doc}");
}
// → Document1.pdf → Document2.pdf → Document3.pdf

// ── Safe operations (no exception on empty) ────────────────────
Queue<int> emptyQ = new();
if (emptyQ.TryDequeue(out int result))
    Console.WriteLine($"Dequeued: {result}");
else
    Console.WriteLine("Queue is empty"); // ← no InvalidOperationException

emptyQ.TryPeek(out int peeked); // safe peek too
```

| Operation | Complexity | Notes |
|---|---|---|
| `Enqueue(x)` | O(1) amortized | Adds to back |
| `Dequeue()` | O(1) | Removes from front — advances head index |
| `Peek()` | O(1) | Reads front without removing |
| `TryDequeue` | O(1) | Safe — returns false if empty |
| `Count` | O(1) | |

> 💡 **Why not just use List?** `List.RemoveAt(0)` is O(n) — it shifts all remaining elements. `Queue.Dequeue()` is O(1) — it just advances an internal head index. No data moves at all.

---

### ⚙️ Internal Implementation — Circular Array

```
Queue<T> uses a CIRCULAR ARRAY internally (not a linked list!)

Key fields:
  T[]  _array  — backing circular array
  int  _head   — index of the front element
  int  _tail   — index of next free slot at the back
  int  _size   — Count

Why circular?
  After Dequeue(), instead of shifting all elements,
  _head simply advances: _head = (_head + 1) % _array.Length
  → Dequeue() is O(1) — no data moves at all!

  Enqueue A,B,C,D → [A,B,C,D,_], _head=0, _tail=4
  Dequeue()       → [_,B,C,D,_], _head=1  ← just moves pointer!
  Enqueue E,F     → [F,B,C,D,E], _tail wraps around ← circular!
```

---

### 🔴 Real-World Queue Patterns

```csharp
// Pattern 1: Job Queue
Queue<WorkItem> jobQueue = new();
jobQueue.Enqueue(new WorkItem("Send email"));
jobQueue.Enqueue(new WorkItem("Resize image"));
while (jobQueue.TryDequeue(out var job))
    job.Execute(); // processes in submission order

// Pattern 2: BFS (Breadth-First Search)
Queue<TreeNode> bfsQueue = new();
bfsQueue.Enqueue(root);
while (bfsQueue.Count > 0)
{
    var node = bfsQueue.Dequeue();
    Console.WriteLine(node.Value);
    foreach (var child in node.Children)
        bfsQueue.Enqueue(child); // level-by-level traversal
}
```

---

## 🔵 Part 2: Stack\<T\> — Last In, First Out

### What Is It?

`Stack<T>` stores elements in **LIFO order** — the last element added is the first one removed. Like a stack of plates: you always take the top plate first.

### 🌍 Real-World Analogy

A **stack of plates** — you place a plate on top and take from the top. The last plate placed is the first one removed.

---

### ⚙️ How Stack Flows

```
Push(1)      →  [1]
Push(2)      →  [1][2]
Push(3)      →  [1][2][3]
Pop()        →  returns 3  →  [1][2]
Peek()       →  returns 2 (not removed)  →  [1][2]
```

---

### 💻 Code Example

```csharp
Stack<string> browserHistory = new();
browserHistory.Push("google.com");       // O(1) — push to top
browserHistory.Push("github.com");
browserHistory.Push("stackoverflow.com");

Console.WriteLine($"Count: {browserHistory.Count}"); // 3
Console.WriteLine($"Peek:  {browserHistory.Peek()}"); // "stackoverflow.com"

// Back button — pops in reverse (LIFO) order
while (browserHistory.Count > 0)
{
    string page = browserHistory.Pop(); // O(1) — removes from top
    Console.WriteLine($"Back to: {page}");
}
// → stackoverflow.com → github.com → google.com

// ── Safe operations ────────────────────────────────────────────
Stack<int> emptyS = new();
if (emptyS.TryPop(out int val))
    Console.WriteLine(val);
else
    Console.WriteLine("Stack is empty"); // no exception
```

| Operation | Complexity |
|---|---|
| `Push(x)` | O(1) amortized |
| `Pop()` | O(1) |
| `Peek()` | O(1) |
| `Count` | O(1) |

---

### ⚙️ Internal Implementation — Simple Array

```
Stack<T> uses a simple ARRAY — not a linked list!

  T[]  _array  — backing array
  int  _size   — index of next free slot (= Count)

Push: write at _array[_size], then _size++
Pop:  _size--, return _array[_size]

Both are just index arithmetic + one array read/write → O(1)
The "top" is the end of the array — no shifting ever.

Why not a linked list?
  Arrays are cache-friendly (sequential memory).
  Linked lists have pointer overhead + scattered memory = cache misses.
```

---

### 🔴 Real-World Stack Patterns

```csharp
// Pattern 1: Undo / Redo
Stack<ICommand> undoStack = new();
Stack<ICommand> redoStack = new();

void Execute(ICommand cmd) {
    cmd.Execute();
    undoStack.Push(cmd);
    redoStack.Clear(); // new action clears redo history
}
void Undo() {
    if (undoStack.TryPop(out var cmd)) {
        cmd.Undo();
        redoStack.Push(cmd);
    }
}

// Pattern 2: Balanced Brackets
bool IsBalanced(string expr) {
    Stack<char> stack = new();
    foreach (char c in expr) {
        if (c == '(') stack.Push(c);
        else if (c == ')' && !stack.TryPop(out _)) return false;
    }
    return stack.Count == 0;
}

// Pattern 3: DFS (Depth-First Search)
Stack<TreeNode> dfsStack = new();
dfsStack.Push(root);
while (dfsStack.Count > 0)
{
    var node = dfsStack.Pop();
    Console.WriteLine(node.Value);
    foreach (var child in node.Children)
        dfsStack.Push(child);
}
```

---

## 🆚 Queue vs Stack — Side by Side

| Aspect | `Queue<T>` | `Stack<T>` |
|---|---|---|
| Order | FIFO — first in, first out | LIFO — last in, first out |
| Add | `Enqueue(x)` — to back | `Push(x)` — to top |
| Remove | `Dequeue()` — from front | `Pop()` — from top |
| Peek (no remove) | `Peek()` — front item | `Peek()` — top item |
| Safe remove | `TryDequeue(out T)` | `TryPop(out T)` |
| Internal structure | Circular array | Simple array |
| Analogy | 🍽️ Restaurant queue | 📚 Stack of plates |
| All operations | O(1) amortized | O(1) amortized |

> 🔴 **`Dequeue()` on empty Queue** and **`Pop()` on empty Stack** both throw `InvalidOperationException`. Always use `TryDequeue()`/`TryPop()` or check `.Count > 0` first.

---

## 🟡 Part 3: Span\<T\> & ReadOnlySpan\<T\>

### What Is It?

`Span<T>` is a **stack-only struct** that represents a safe, high-performance window over a contiguous block of existing memory — arrays, strings, stack-allocated buffers — **without allocating or copying anything**.

### 🌍 Real-World Analogy

Instead of photocopying a chapter from a book (allocation), you just use a **bookmark and a ruler** to indicate "pages 5–12." Same information, zero copying.

---

### ⚙️ The Allocation Problem

```csharp
// ❌ Without Span — copies data, creates garbage
int[] arr = [1, 2, 3, 4, 5];
int[] slice = arr.Skip(1).Take(3).ToArray(); // 💸 New heap allocation!

string text = "Hello, World!";
string sub = text.Substring(0, 5); // 💸 New string on heap!

// ✅ With Span — zero copy, zero allocation
int[] arr = [1, 2, 3, 4, 5];
Span<int> slice = arr.AsSpan(1, 3);  // ✅ Pointer + length only (16 bytes on stack)

string text = "Hello, World!";
ReadOnlySpan<char> sub = text.AsSpan(0, 5); // ✅ Zero allocation!
```

### Visual: Span is a Window, Not a Copy

```
Original array:  [ 1 ][ 2 ][ 3 ][ 4 ][ 5 ]
                   0    1    2    3    4

arr.AsSpan(1, 3) points to:
                       [ 2 ][ 3 ][ 4 ]
                   ← same memory — no copy ←

slice[0] = 99 → MODIFIES the original: array[1] = 99
```

---

### 💻 Code Examples

```csharp
// ── Array slicing (read/write) ─────────────────────────────────
int[] array = [1, 2, 3, 4, 5];
Span<int> slice = array.AsSpan(1, 3); // window over indices 1,2,3 → {2,3,4}
Console.WriteLine(slice[0]); // → 2
Console.WriteLine(slice.Length); // → 3
slice[0] = 99; // modifies the ORIGINAL: array is now [1, 99, 3, 4, 5]

// ── String slicing — zero allocation ──────────────────────────
string text = "Hello, World!";
ReadOnlySpan<char> hello = text.AsSpan(0, 5); // "Hello" — no new string!
ReadOnlySpan<char> world = text.AsSpan(7, 5); // "World" — no new string!
Console.WriteLine(hello.ToString()); // → Hello
// hello[0] = 'X'; // ❌ compile error — ReadOnlySpan is read-only

// ── Zero-alloc CSV parsing (real-world pattern) ────────────────
ReadOnlySpan<char> line = "Ahmed,25,Cairo".AsSpan();
int comma1 = line.IndexOf(',');
ReadOnlySpan<char> name = line[..comma1];      // "Ahmed" — no allocation
ReadOnlySpan<char> rest = line[(comma1 + 1)..]; // "25,Cairo"
```

---

### ⚡ Performance: Span vs LINQ

```csharp
// ❌ LINQ — allocates new array
int[] big = Enumerable.Range(1, 1000).ToArray();
int[] sliced = big.Skip(100).Take(50).ToArray(); // New int[50] on heap!

// ✅ Span — zero bytes allocated
int[] big = Enumerable.Range(1, 1000).ToArray();
Span<int> sliced = big.AsSpan(100, 50); // 16 bytes on stack only. That's it.
```

---

### ⚙️ How Span Works Internally

```csharp
// Span<T> is a ref struct — lives ONLY on the stack, never on heap
public readonly ref struct Span<T>
{
    internal readonly ref T _reference; // pointer to first element
    private readonly int _length;       // number of elements
    // this[i] computes: address = _reference + (i × sizeof(T))
}

// ReadOnlySpan<T> is identical but indexer returns readonly ref T
// Key: Span is just 16 bytes (pointer + int) on the stack,
//      regardless of how large the underlying data is. Zero GC impact.
```

---

### Span\<T\> vs ReadOnlySpan\<T\>

| Feature | `Span<T>` | `ReadOnlySpan<T>` |
|---|---|---|
| Read | ✅ | ✅ |
| Write back to original | ✅ | ❌ |
| Can wrap strings | ❌ | ✅ (strings are immutable) |
| Use case | Mutable buffers | Strings, read-only data |

### ⚠️ Span Limitations

```
❌ Cannot be stored as a class field
❌ Cannot be used in async methods (stack frame is destroyed on await)
❌ Cannot be boxed
❌ Cannot be used as a generic type argument to other types
✅ For long-lived memory windows: use Memory<T> instead
```

---

## 📊 Part 4: All Collections — Complete Comparison

| Collection | Structure | Add | Find | Remove | Order | Unique? | Use When |
|---|---|---|---|---|---|---|---|
| `List<T>` | Dynamic array | O(1)* | O(n) | O(n) | Insertion | — | Default ordered collection, index access |
| `Dictionary<K,V>` | Hash table | O(1)* | O(1)* | O(1)* | None | Keys ✅ | Key-value lookup |
| `HashSet<T>` | Hash table | O(1)* | O(1)* | O(1)* | None | ✅ | Uniqueness, O(1) membership |
| `SortedList<K,V>` | Two sorted arrays | O(n) | O(log n) | O(n) | Key-sorted | Keys ✅ | Sorted KV + positional access |
| `SortedDictionary<K,V>` | Red-Black tree | O(log n) | O(log n) | O(log n) | Key-sorted | Keys ✅ | Sorted KV + frequent inserts |
| `SortedSet<T>` | Red-Black tree | O(log n) | O(log n) | O(log n) | Sorted | ✅ | Unique + always-sorted elements |
| `Queue<T>` | Circular array | O(1)* | — | O(1) | FIFO | — | Task queues, BFS |
| `Stack<T>` | Array | O(1)* | — | O(1) | LIFO | — | Undo/redo, DFS, parsing |
| `LinkedList<T>` | Doubly linked | O(1) at node | O(n) | O(1) at node | Insertion | — | Frequent mid-list insert/remove |
| `Span<T>` | Pointer + length | — | O(1) | — | Existing | — | Zero-alloc slicing, hot paths |

*amortized — occasional O(n) resize

### ✅ Quick Decision Rule

```
Need ordered, indexed data?          → List<T>
Need key-value, fast lookup?         → Dictionary<K,V>
Need uniqueness, O(1) membership?    → HashSet<T>
Need sorted + positional access?     → SortedList<K,V>
Need sorted + frequent inserts?      → SortedDictionary<K,V>
Need FIFO order?                     → Queue<T>
Need LIFO order?                     → Stack<T>
Need zero-alloc slicing?             → Span<T> / ReadOnlySpan<T>
Need frequent insert at known node?  → LinkedList<T>
```

---

## 📌 Summary

> `Queue<T>` and `Stack<T>` are both backed by **arrays** (not linked lists) for cache efficiency, with all core operations at O(1) amortized. `Span<T>` is a **16-byte stack value** that provides a zero-allocation window into existing memory — critical for high-performance parsing and slicing. Always choose the collection that matches your **access pattern**: FIFO → Queue, LIFO → Stack, indexed → List, key-based → Dictionary, unique → HashSet.
