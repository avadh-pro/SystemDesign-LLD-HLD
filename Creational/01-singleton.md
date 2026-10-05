# Singleton Pattern

> **Verdict:** One instance for the whole program, with a private constructor enforcing it. Easy to write, easy to misuse — the trade-off is always lazy-vs-eager and thread safety.

**Category:** Creational · **Language:** Java

---

## What it is

The Singleton Pattern ensures that a class has **only one instance** and provides a **global point of access** to that instance.

In simpler terms: imagine you're building an application where you only want one shared object throughout the lifecycle of the program. This is where Singleton comes into play — it restricts object creation and guarantees that all parts of your application use the same object.

---

## The problem it solves

In a typical application, creating multiple objects of a class might not be problematic. However, in certain scenarios — like **logging**, **configuration handling**, or **managing a database connection** — you want just one instance to avoid:

- redundancy
- excessive memory use
- inconsistent behaviour

### Real-world analogy: the operating system's print spooler

Imagine you're in an office with multiple employees, and everyone sends documents to a single shared printer. Now, if each computer tried to talk directly to the printer on its own terms, the printer would get overwhelmed — prints might get jumbled, overlap, or crash the device.

Instead, there's a **Print Spooler** — a background service that manages all print jobs. No matter who initiates the print, they all go through one centralised spooler instance that queues and handles the tasks in order.

---

## Why is it a creational pattern?

The Singleton Pattern falls under the **creational** design patterns. This is because it deals with *how objects are created*. Unlike simple instantiation (`new`), Singleton **controls the object creation process** by returning an existing instance rather than creating a new one.

---

## Identifying the need for a Singleton

Imagine you're developing a logging service. You need a class that writes logs to a file. If every part of your application creates a new logger instance, the result might be:

- Overwritten logs
- Multiple file handles
- Synchronisation issues

Instead, if there's only one logger instance (a Singleton), all parts of the program write to the same log file in a controlled manner.

---

## Working of the Singleton Pattern

The Singleton Pattern typically involves the following steps:

| Step | Purpose |
|---|---|
| **Private constructor** | Prevents instantiation from outside the class |
| **Static variable** | Holds the single instance of the class |
| **Public static method** | Provides a global access point to get the instance |

This ensures that no matter how many times you call the method to get an instance, it will always return the same object.

---

## Approaches to implement the Singleton Pattern

In the real world, while designing the product, there are two primary ways to implement the Singleton pattern:

1. **Eager Loading**
2. **Lazy Loading**

Each with its own trade-offs in terms of performance, memory usage, and thread safety.

---

### 1. Eager Loading (early initialisation)

In Eager Loading, the Singleton instance is created **as soon as the class is loaded**, regardless of whether it's ever used.

> **Real-world analogy — fire extinguisher in a building:** a fire extinguisher is always present, even if a fire never occurs. Similarly, eager loading creates the Singleton instance upfront, just in case it's needed.

```java
// Class implementing Eager Loading
class EagerSingleton {

    private static final EagerSingleton instance = new EagerSingleton();

    // private constructor
    private EagerSingleton() {
        // Declaring it private prevents creation of its object using the new keyword
    }

    // Method to get the instance of class
    public static EagerSingleton getInstance() {
        return instance;    // Always returns the same instance
    }
}
```

**Understanding**
- The object is created immediately when the class is loaded.
- It's always available and inherently thread-safe.

| Pros | Cons |
|---|---|
| Very simple to implement | Wastes memory if the instance is never used |
| Thread-safe without any extra handling | Not suitable for heavy objects |

---

### 2. Lazy Loading (on-demand initialisation)

In Lazy Loading, the Singleton instance is created **only when it's needed** — the first time the `getInstance()` method is called.

> **Real-world analogy — coffee machine:** imagine a coffee machine that only brews coffee when you press the button. It doesn't waste energy or resources until you actually want a cup. Similarly, lazy loading creates the Singleton instance only when it's requested.

```java
// Class implementing Lazy Loading
class LazySingleton {

    // Object declaration
    private static LazySingleton instance;

    // private constructor
    private LazySingleton() {
        // Declaring it private prevents creation of its object using the new keyword
    }

    // Method to get the instance of class
    public static LazySingleton getInstance() {
        // If the object is not created
        if (instance == null) {
            // A new object is created
            instance = new LazySingleton();
        }
        // Otherwise the already created object is returned
        return instance;
    }
}
```

**Understanding**
- The instance starts as `null`.
- It is only created when `getInstance()` is first called.
- Future calls return the already created instance.

| Pros | Cons |
|---|---|
| Saves memory if the instance is never used | **Not thread-safe by default** |
| Object creation is deferred until required | Requires synchronisation in multi-threaded environments |

---

## Thread safety: a critical concern

In a single-threaded environment, implementing a Singleton is straightforward. However, things get complicated in **multi-threaded applications**, which are very common in modern software (especially web servers, mobile apps, etc.).

### The problem

Let's say two threads simultaneously call `getInstance()` for the first time in a lazy-loaded Singleton. If the instance hasn't been created yet, **both threads might pass the null check** and end up creating two different instances — completely breaking the Singleton guarantee.

```
Thread A                          Thread B
   │                                 │
   ├─ if (instance == null) ──▶ true │
   │                                 ├─ if (instance == null) ──▶ true
   ├─ instance = new Singleton()     │
   │                                 ├─ instance = new Singleton()
   ▼                                 ▼
        TWO different objects exist
```

This is not theoretical. Running the plain lazy version above with 50 threads hitting `getInstance()` simultaneously:

```
distinct instances created: 47   <-- SINGLETON BROKEN
```

Forty-seven objects where there should have been one.

This kind of bug is:
- **Hard to detect**, as it may not occur every time.
- **Severe**, because it defeats the whole purpose of the pattern.
- **Costly**, especially if the Singleton manages critical resources like logging, configuration, or DB connections.

---

## Different ways to achieve thread safety

### 1. Synchronized method

This is the simplest way to ensure thread safety. By synchronising the method that creates the instance, we can prevent multiple threads from creating separate instances at the same time. However, this approach can lead to performance issues due to the overhead of synchronisation.

```java
public class Singleton {

    // Object declaration
    private static Singleton instance;

    // Private constructor
    private Singleton() {}

    // Synchronized keyword used
    public static synchronized Singleton getInstance() {
        if (instance == null) {
            instance = new Singleton();
        }
        return instance;
    }
}
```

**What the `synchronized` keyword does:** it ensures that only one thread at a time can execute the `getInstance()` method. This prevents multiple threads from entering the method simultaneously and creating multiple instances.

| Pros | Cons |
|---|---|
| Simple and easy to implement | **Performance overhead** — every call is synchronised, even after the instance is created |
| Thread-safe without needing complex logic | May slow down the application in high-concurrency scenarios |

---

### 2. Double-checked locking

This is a more efficient way to achieve thread safety. The idea is to check if the instance is `null` **before** acquiring the lock. If it is, then we synchronise the block and check again. This reduces the overhead of synchronisation after the instance has been created.

```java
public class Singleton {

    // Volatile object declaration
    private static volatile Singleton instance;

    // Private constructor
    private Singleton() {}

    // Thread-safe method using double-checked locking
    public static Singleton getInstance() {
        if (instance == null) {                      // 1st check — no lock
            synchronized (Singleton.class) {
                if (instance == null) {              // 2nd check — with lock
                    instance = new Singleton();
                }
            }
        }
        return instance;
    }
}
```

**Understanding**
- The **outer `if`** check avoids synchronisation once the instance is created.
- The **inner `if`** inside `synchronized` ensures that only one thread creates the instance.
- The **`volatile`** keyword ensures changes made by one thread are visible to others. Without `volatile`, one thread might create the Singleton instance, but other threads may not see the updated value due to caching. `volatile` ensures that the instance is always read from main memory, so all threads see the most up-to-date version.

| Pros | Cons |
|---|---|
| Efficient — synchronisation only happens once, at creation | Slightly more complex than the synchronized method |
| Safe and fast in concurrent environments | Requires Java 1.5 or above due to reliance on `volatile` |

---

### 3. Bill Pugh Singleton — best practice for lazy loading

This is a highly efficient way to implement the Singleton pattern. It uses a **static inner helper class** to hold the Singleton instance. The instance is created only when the inner class is loaded, which happens only when `getInstance()` is called for the first time.

```java
public class Singleton {

    // Private constructor
    private Singleton() {}

    // Static inner class to hold the Singleton instance
    private static class Holder {
        private static final Singleton INSTANCE = new Singleton();
    }

    // Public method to return the Singleton instance
    public static Singleton getInstance() {
        return Holder.INSTANCE;
    }
}
```

**Explanation**
- The Singleton instance is **not created until `getInstance()` is called**.
- The static inner class (`Holder`) is not loaded until referenced, thanks to Java's class loading mechanism.
- It ensures thread safety, lazy loading, and high performance **without synchronisation overhead**.

| Pros | Cons |
|---|---|
| Best of both worlds — lazy **and** thread-safe | Slightly less intuitive for beginners due to the nested static class |
| No need for `synchronized` or `volatile` | |
| Clean and efficient | |

---

### 4. Eager loading

As discussed earlier, eager loading does not face thread safety issues. This approach **avoids thread issues altogether** by creating the instance upfront — at the cost of potential memory waste. Thus, it is not a preferred method in most cases, but is still a valid option.

---

## Comparing the approaches

| Approach | Lazy? | Thread-safe? | Lock on every call? | Verdict |
|---|---|---|---|---|
| Eager loading | ❌ | ✅ | ❌ | Fine for cheap objects |
| Lazy loading (plain) | ✅ | ❌ | ❌ | Single-threaded only |
| Synchronized method | ✅ | ✅ | ✅ | Simple but slow |
| Double-checked locking | ✅ | ✅ | ❌ | Fast, needs `volatile` |
| **Bill Pugh (holder)** | ✅ | ✅ | ❌ | **Best general choice** |

---

## Pros of the Singleton Pattern

- **Cleaner implementation** — Singleton offers a straightforward and tidy way to manage a single instance of a class, especially when designed with thread safety and simplicity in mind.
- **Guarantees one instance** — this pattern enforces that only one instance of the class can exist, making it ideal for shared resources.
- **Provides a way to maintain a global resource** — it allows centralised access to a global resource or service, which can be useful in managing application-wide configurations or state.
- **Supports lazy loading** — many Singleton implementations allow the instance to be created only when it is first accessed, optimising memory usage and startup performance.

## Cons of the Singleton Pattern

- **Used with parameters and confused with Factory** — when a Singleton class requires parameters for instantiation, it may blur lines with the Factory pattern, leading to design confusion.
- **Hard to write unit tests** — since the Singleton holds global state, it becomes difficult to isolate and mock for unit testing, thus potentially hindering testability.
- **Classes using it are highly coupled to it** — components that depend on the Singleton become tightly coupled to its implementation, which reduces flexibility and makes code harder to maintain or refactor.
- **Special cases to avoid race conditions** — in multi-threaded environments, care must be taken to avoid race conditions during the instance creation phase, complicating implementation.
- **Violates the Single Responsibility Principle (SRP)** — a Singleton often handles both instance control and its core functionality, thereby violating the SRP, a key principle of clean software design.

---

## Conclusion

The Singleton pattern can be a powerful tool when used appropriately, particularly for managing global states and shared resources. However, developers should be mindful of its drawbacks, especially regarding testing and maintainability. Consider alternatives or enhanced implementations (like **dependency injection**) where appropriate to maintain clean and scalable codebases.

---

## Skim cheat sheet

- **Definition:** one instance, global access point.
- **Three ingredients:** private constructor · static variable · public static accessor.
- **Why creational:** it controls *how* the object is created, returning an existing one instead of a new one.
- **Use for:** logging, configuration, DB connection pools, print spooler.
- **Eager** = built at class load. Simple, thread-safe, wasteful.
- **Lazy** = built on first `getInstance()`. Saves memory, **not thread-safe by default**.
- **The race:** two threads both pass `if (instance == null)` → two objects.
- **Thread-safe options:** `synchronized` (slow) · double-checked locking + `volatile` (fast) · **Bill Pugh holder (best)**.
- **`volatile` is mandatory** in double-checked locking — without it, other threads may read a stale cached value.
- **Biggest downside:** global state → hard to unit-test, tight coupling, violates SRP.
- **Modern alternative:** dependency injection.
