# Creational Design Patterns

Patterns that deal with **how objects are created** — they control, abstract or defer instantiation instead of calling `new` directly.

Code examples are Java.

## Patterns

| # | Pattern | Intent | Status |
|---|---|---|---|
| 1 | [Singleton](01-singleton.md) | Ensure a class has only one instance and give it a global access point | ✅ Written |
| 2 | Factory Method | Let a subclass decide which concrete class to instantiate | ⬜ Planned |
| 3 | Abstract Factory | Create families of related objects without naming their concrete classes | ⬜ Planned |
| 4 | Builder | Construct a complex object step by step | ⬜ Planned |
| 5 | Prototype | Create new objects by cloning an existing one | ⬜ Planned |

## The common thread

All five answer the same question — *who decides which object gets made, and when?*

- **Singleton** — exactly one, decided by the class itself.
- **Factory Method / Abstract Factory** — a subclass or factory decides.
- **Builder** — the caller decides, one piece at a time.
- **Prototype** — an existing object decides, by copying itself.
