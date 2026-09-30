# 2. OOP Foundations

> **Verdict:** LLD is applied OOP. The four pillars aren't trivia — each one is a specific defence against a specific way code rots.

---

## Why this chapter exists

Every LLD interview scores you on whether your classes have clear
responsibilities and clean seams. The four pillars are the vocabulary for that.
You will not be asked "define polymorphism" — you will be asked to *use* it and
then explain what you did.

Each pillar below is shown twice: the version that hurts, then the version that
works.

---

## 1. Encapsulation

**What it is:** the object owns its data. Outsiders go through methods, never
reach in and grab fields.

**What it defends against:** an object that can be put into an impossible state
by anyone, from anywhere.

### Bad

```python
class BankAccount:
    def __init__(self, balance: float) -> None:
        self.balance = balance          # public


account = BankAccount(100.0)
account.balance = -5000                 # nothing stopped this
```

The account is now invalid and the class it belongs to never found out. When a
bug report says "some accounts have negative balances", you have to search every
file in the codebase to find who did it.

### Good

```python
class InsufficientFunds(Exception):
    pass


class BankAccount:
    def __init__(self, balance: float = 0.0) -> None:
        if balance < 0:
            raise ValueError("opening balance cannot be negative")
        self._balance = balance         # underscore = internal

    @property
    def balance(self) -> float:
        return self._balance            # read allowed

    def deposit(self, amount: float) -> None:
        if amount <= 0:
            raise ValueError("deposit must be positive")
        self._balance += amount

    def withdraw(self, amount: float) -> None:
        if amount > self._balance:
            raise InsufficientFunds(f"balance is {self._balance}")
        self._balance -= amount
```

Now there is exactly one place where the balance changes, and it is impossible
to go negative. The bug report can't happen.

**In plain words:** encapsulation means *"there is only one door, and I guard
it."*

> **Python note:** Python has no `private`. `_balance` is a convention, not
> enforcement. Interviewers know this — use the underscore and say "this is
> private by convention; in Java it'd be `private final`." That shows you
> understand the intent, not just the syntax.

---

## 2. Abstraction

**What it is:** expose *what* the thing does, hide *how*. The caller learns a
small interface instead of a large implementation.

**What it defends against:** callers that break every time you change your
internals.

### Bad

```python
class ReportService:
    def generate(self, rows: list) -> str:
        # caller must know we use CSV, and must know about quoting rules
        header = "id,name,amount"
        lines = [header]
        for r in rows:
            lines.append(f'{r["id"]},"{r["name"]}",{r["amount"]}')
        return "\n".join(lines)
```

The day the business wants PDF, every caller changes.

### Good

```python
from abc import ABC, abstractmethod


class ReportFormat(ABC):
    @abstractmethod
    def render(self, rows: list[dict]) -> bytes:
        ...


class CsvFormat(ReportFormat):
    def render(self, rows: list[dict]) -> bytes:
        header = "id,name,amount"
        lines = [header] + [
            f'{r["id"]},"{r["name"]}",{r["amount"]}' for r in rows
        ]
        return "\n".join(lines).encode()


class PdfFormat(ReportFormat):
    def render(self, rows: list[dict]) -> bytes:
        return b"%PDF-1.4 ..."          # real impl elsewhere


class ReportService:
    def __init__(self, fmt: ReportFormat) -> None:
        self._fmt = fmt

    def generate(self, rows: list[dict]) -> bytes:
        return self._fmt.render(rows)
```

`ReportService` now knows one thing: *something can render rows*. Adding PDF
touched zero existing callers.

**In plain words:** abstraction is *"you get a steering wheel, not the engine."*

**Encapsulation vs abstraction** — they get confused constantly:
- Encapsulation hides **data** (the balance field).
- Abstraction hides **implementation** (how the report is built).

One protects state, the other protects callers.

---

## 3. Inheritance

**What it is:** a subclass *is a* kind of its parent and reuses its behaviour.

**What it defends against:** duplicating shared behaviour across related types.

**What it causes when misused:** the single most common LLD mistake.

### Bad — inheritance used for code reuse

```python
class Rectangle:
    def __init__(self, w: float, h: float) -> None:
        self.w, self.h = w, h

    def set_width(self, w: float) -> None:
        self.w = w

    def area(self) -> float:
        return self.w * self.h


class Square(Rectangle):            # "a square IS a rectangle, right?"
    def set_width(self, w: float) -> None:
        self.w = self.h = w         # must break the parent's contract
```

Maths says a square is a rectangle. Code says no. Any function written against
`Rectangle` that sets width and height independently now silently misbehaves
when handed a `Square`. This is the classic **Liskov violation** (chapter 3).

### Good — model the shared thing, not the convenient one

```python
from abc import ABC, abstractmethod


class Shape(ABC):
    @abstractmethod
    def area(self) -> float:
        ...


class Rectangle(Shape):
    def __init__(self, w: float, h: float) -> None:
        self._w, self._h = w, h

    def area(self) -> float:
        return self._w * self._h


class Square(Shape):
    def __init__(self, side: float) -> None:
        self._side = side

    def area(self) -> float:
        return self._side ** 2
```

**The rule to say out loud in an interview:**

> *Inherit for "is-a", compose for "has-a". If you're inheriting only to reuse a
> method, use composition instead.*

### Composition — the safer default

```python
class Engine:
    def start(self) -> None:
        print("vroom")


class Car:
    def __init__(self, engine: Engine) -> None:
        self._engine = engine       # Car HAS-A Engine, Car is not an Engine

    def drive(self) -> None:
        self._engine.start()
```

Swapping in `ElectricEngine` costs one line at the construction site. With
inheritance it would cost a new subclass of `Car`.

**In plain words:** inheritance is a promise you can never take back. Composition
is a wire you can unplug.

---

## 4. Polymorphism

**What it is:** one call site, many behaviours, decided at runtime by the actual
object.

**What it defends against:** the `if/elif` chain that grows forever.

### Bad

```python
class NotificationService:
    def send(self, kind: str, to: str, body: str) -> None:
        if kind == "email":
            print(f"emailing {to}")
        elif kind == "sms":
            print(f"texting {to}")
        elif kind == "push":
            print(f"pushing to {to}")
        # every new channel edits this method — forever
```

This method is a magnet for bugs. It grows with every feature, it can't be
tested in isolation, and merge conflicts land here every sprint.

### Good

```python
from abc import ABC, abstractmethod


class Channel(ABC):
    @abstractmethod
    def send(self, to: str, body: str) -> None:
        ...


class Email(Channel):
    def send(self, to: str, body: str) -> None:
        print(f"emailing {to}")


class Sms(Channel):
    def send(self, to: str, body: str) -> None:
        print(f"texting {to}")


class Push(Channel):
    def send(self, to: str, body: str) -> None:
        print(f"pushing to {to}")


class NotificationService:
    def __init__(self, channels: list[Channel]) -> None:
        self._channels = channels

    def broadcast(self, to: str, body: str) -> None:
        for channel in self._channels:      # same call, different behaviour
            channel.send(to, body)
```

Adding WhatsApp = one new file. `NotificationService` never changes again.

**In plain words:** polymorphism is *"I'll call `.send()` and let the object
figure out what that means."*

> **Spotting the opportunity:** whenever you write `if type ==` or `elif kind ==`,
> a polymorphic class hierarchy is hiding there. In an LLD interview, replacing a
> conditional with polymorphism is the single highest-value move you can make.

---

## The compression of all four

| Pillar | One-line job | Smell when missing |
|---|---|---|
| Encapsulation | Guard the data | Public fields, objects in impossible states |
| Abstraction | Guard the caller | Callers break when internals change |
| Inheritance | Share is-a behaviour | Duplicated methods, or a forced "is-a" that lies |
| Polymorphism | Kill the conditional | Long `if/elif` on a type field |

---

## Two more terms interviewers use

**Coupling** — how much one class must know about another. Low is good.
`Checkout` knowing only `PaymentMethod` (not `CardPayment`) is low coupling.

**Cohesion** — how related one class's responsibilities are. High is good. A
`User` class that also sends emails and writes to the DB has low cohesion; split
it.

> *"Low coupling, high cohesion"* is the two-word summary of good LLD. If you say
> nothing else in a design round, say this and then show it.

---

**Next:** [03 — SOLID Principles](03-solid-principles.md)
