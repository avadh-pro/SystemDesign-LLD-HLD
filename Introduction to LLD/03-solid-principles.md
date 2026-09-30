# 3. SOLID Principles

> **Verdict:** Five rules that all push the same way — *make change cheap*. You will be graded on them whether or not the interviewer says the word "SOLID".

---

## The five, in one table

| | Name | One line |
|---|---|---|
| **S** | Single Responsibility | A class has one reason to change |
| **O** | Open/Closed | Add behaviour by adding code, not editing code |
| **L** | Liskov Substitution | A subclass must work anywhere its parent works |
| **I** | Interface Segregation | Don't force a class to implement methods it doesn't use |
| **D** | Dependency Inversion | Depend on abstractions, not concrete classes |

Read that table before an interview. Read the rest of this chapter to actually
use them.

---

## S — Single Responsibility Principle

> A class should have **one reason to change**.

Note the wording: *one reason to change*, not "one method" or "one job". Two
things belong together if they always change together.

### Bad

```python
class Invoice:
    def __init__(self, items: list[dict]) -> None:
        self.items = items

    def total(self) -> float:
        return sum(i["price"] * i["qty"] for i in self.items)

    def save_to_db(self) -> None:                 # reason to change #2
        print("INSERT INTO invoices ...")

    def send_email(self, to: str) -> None:        # reason to change #3
        print(f"SMTP → {to}")

    def to_pdf(self) -> bytes:                    # reason to change #4
        return b"%PDF ..."
```

This class changes when tax rules change, **and** when you migrate Postgres →
DynamoDB, **and** when you switch SMTP → SendGrid, **and** when marketing wants a
new invoice layout. Four teams edit one file.

### Good

```python
class Invoice:
    """Business rules only."""

    def __init__(self, items: list[dict]) -> None:
        self._items = items

    def total(self) -> float:
        return sum(i["price"] * i["qty"] for i in self._items)


class InvoiceRepository:
    def save(self, invoice: Invoice) -> None:
        print("INSERT INTO invoices ...")


class InvoiceMailer:
    def send(self, invoice: Invoice, to: str) -> None:
        print(f"SMTP → {to}")


class InvoicePdfRenderer:
    def render(self, invoice: Invoice) -> bytes:
        return b"%PDF ..."
```

**In plain words:** if you describe a class and have to use the word "and", it
probably has two responsibilities.

---

## O — Open/Closed Principle

> Open for extension, **closed for modification**. New behaviour = new code, not
> edited code.

### Bad

```python
class DiscountCalculator:
    def apply(self, customer_type: str, amount: float) -> float:
        if customer_type == "regular":
            return amount
        elif customer_type == "premium":
            return amount * 0.9
        elif customer_type == "employee":
            return amount * 0.7
        # every new tier reopens and edits this tested method
```

Editing tested code risks breaking what already worked. That is the whole
objection.

### Good

```python
from abc import ABC, abstractmethod


class DiscountPolicy(ABC):
    @abstractmethod
    def apply(self, amount: float) -> float:
        ...


class NoDiscount(DiscountPolicy):
    def apply(self, amount: float) -> float:
        return amount


class PremiumDiscount(DiscountPolicy):
    def apply(self, amount: float) -> float:
        return amount * 0.9


class EmployeeDiscount(DiscountPolicy):
    def apply(self, amount: float) -> float:
        return amount * 0.7


class DiscountCalculator:
    def __init__(self, policy: DiscountPolicy) -> None:
        self._policy = policy

    def final_price(self, amount: float) -> float:
        return self._policy.apply(amount)
```

A Diwali discount is now a new file. `DiscountCalculator` and its tests never
move.

**In plain words:** you should be able to add a feature without opening the file
that already works.

> **The honest caveat:** something has to pick the policy. That factory *does*
> change. OCP moves the change to one small, obvious place instead of scattering
> it through business logic — it doesn't make change free.

---

## L — Liskov Substitution Principle

> Anywhere the parent works, the child must work too — without the caller
> knowing the difference.

This is the one candidates state correctly and violate anyway.

### Bad

```python
class Bird:
    def fly(self) -> None:
        print("flying")


class Penguin(Bird):
    def fly(self) -> None:
        raise NotImplementedError("penguins can't fly")   # broke the contract


def migrate(birds: list[Bird]) -> None:
    for b in birds:
        b.fly()            # explodes the day someone adds a Penguin
```

`migrate` was correct. Adding a subclass broke it. That is the LSP violation:
the child *narrowed* what the parent promised.

### Good

```python
from abc import ABC, abstractmethod


class Bird(ABC):
    @abstractmethod
    def move(self) -> None:
        ...


class FlyingBird(Bird):
    def move(self) -> None:
        print("flying")


class Penguin(Bird):
    def move(self) -> None:
        print("waddling")


def migrate(birds: list[Bird]) -> None:
    for b in birds:
        b.move()           # true for every bird, forever
```

**The three tests for a violation** — a subclass must not:
1. **Throw** where the parent didn't (`NotImplementedError` in an override is the
   loudest possible alarm).
2. **Demand more** than the parent (parent accepts any int, child requires
   positive).
3. **Promise less** than the parent (parent returns a sorted list, child returns
   unsorted).

The `Square`/`Rectangle` example from [chapter 2](02-oop-foundations.md) is
violation type 3 — `Square.set_width` silently changes height, breaking the
guarantee that width and height are independent.

**In plain words:** a subclass may do *more*, never *less*.

---

## I — Interface Segregation Principle

> Many small interfaces beat one fat one. Don't make a class implement methods it
> doesn't need.

### Bad

```python
from abc import ABC, abstractmethod


class Worker(ABC):
    @abstractmethod
    def work(self) -> None: ...

    @abstractmethod
    def eat(self) -> None: ...

    @abstractmethod
    def sleep(self) -> None: ...


class RobotWorker(Worker):
    def work(self) -> None:
        print("assembling")

    def eat(self) -> None:
        raise NotImplementedError    # robots don't eat

    def sleep(self) -> None:
        raise NotImplementedError    # or sleep
```

Two thirds of this class is dead weight that also violates Liskov. The interface
was too greedy.

### Good

```python
from abc import ABC, abstractmethod


class Workable(ABC):
    @abstractmethod
    def work(self) -> None: ...


class Feedable(ABC):
    @abstractmethod
    def eat(self) -> None: ...


class HumanWorker(Workable, Feedable):
    def work(self) -> None:
        print("coding")

    def eat(self) -> None:
        print("lunch")


class RobotWorker(Workable):
    def work(self) -> None:
        print("assembling")
```

Each class implements exactly what it means. No stubs, no exceptions.

**In plain words:** don't make people sign a contract with clauses that don't
apply to them.

**The smell:** a method body that is `pass`, `raise NotImplementedError`, or
`return None  # not applicable`. That is ISP telling you to split.

---

## D — Dependency Inversion Principle

> High-level code should depend on **abstractions**, not on low-level concrete
> classes. Both should depend on the interface.

### Bad

```python
class MySqlDatabase:
    def save(self, data: dict) -> None:
        print("INSERT INTO ...")


class UserService:
    def __init__(self) -> None:
        self._db = MySqlDatabase()      # welded to MySQL, forever

    def register(self, user: dict) -> None:
        self._db.save(user)
```

Two consequences, and interviewers will probe both:
1. Migrating to Postgres means editing `UserService` — business logic changed for
   an infrastructure reason.
2. You cannot unit-test `register` without a live MySQL. There is no seam.

### Good

```python
from abc import ABC, abstractmethod


class UserRepository(ABC):
    @abstractmethod
    def save(self, user: dict) -> None:
        ...


class MySqlUserRepository(UserRepository):
    def save(self, user: dict) -> None:
        print("INSERT INTO ...")


class InMemoryUserRepository(UserRepository):
    """Used by tests. No database required."""

    def __init__(self) -> None:
        self.saved: list[dict] = []

    def save(self, user: dict) -> None:
        self.saved.append(user)


class UserService:
    def __init__(self, repo: UserRepository) -> None:   # injected, not created
        self._repo = repo

    def register(self, user: dict) -> None:
        self._repo.save(user)
```

```python
# the test writes itself now
def test_register_saves_the_user() -> None:
    repo = InMemoryUserRepository()
    UserService(repo).register({"name": "Ram"})
    assert repo.saved == [{"name": "Ram"}]
```

**In plain words:** a class should be *handed* its tools, not go shopping for
them. If a constructor contains `new`/`SomeClass()`, that's a dependency you can
no longer swap.

> **Say this in the interview:** "I'm injecting the repository so the service is
> testable without a database." Testability is the argument that lands — it's
> concrete, and it's the reason DIP exists.

---

## Skim cheat sheet

| Letter | Smell that flags it | Fix |
|---|---|---|
| **S** | Class description needs "and" | Split by reason-to-change |
| **O** | Editing a tested `if/elif` to add a feature | Strategy class behind an interface |
| **L** | `raise NotImplementedError` in an override | Re-model the hierarchy around what's actually common |
| **I** | Fat interface with `pass` stubs | Split into small role interfaces |
| **D** | `self._db = MySqlDatabase()` in `__init__` | Inject the abstraction |

**The unifying idea:** all five answer *"when the requirement changes next month,
how many files do I touch?"* Good LLD keeps that number at one.

---

## Honest limits

SOLID is a set of heuristics, not laws. Applied without judgement it produces
twelve interfaces for a script that should have been twelve lines. The real skill
— and what separates a mid from a senior in the interview — is knowing *which
axis is likely to change* and putting the abstraction only there.

If nothing about payments will ever change, `if method == "card"` is fine. Say
that out loud when it's true; interviewers respect a candidate who can defend
*not* abstracting as much as one who abstracts everything.

---

**Next:** [04 — UML for Interviews](04-uml-for-interviews.md)
