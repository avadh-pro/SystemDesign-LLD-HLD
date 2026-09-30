# 1. LLD vs HLD

> **Verdict:** HLD decides *which boxes exist and how they talk*. LLD decides *what is inside one box*. Same system, two zoom levels.

---

## The one-paragraph version

Imagine you are asked to design Instagram. If you draw a diagram with an API
gateway, a post service, a feed service, S3 for images, Kafka between them and
Cassandra underneath — that is **High Level Design**. If instead you are asked to
write the classes that let a user upload a post, apply a filter, tag friends and
publish it — `Post`, `Filter`, `FilterStrategy`, `PostBuilder`, `NotificationObserver` —
that is **Low Level Design**.

Nobody asks you to do both in one 45-minute round. Know which room you are in.

---

## Side by side

| | High Level Design | Low Level Design |
|---|---|---|
| **Unit of thought** | Service, database, queue, cache | Class, interface, method, field |
| **Question it answers** | Will this hold 10M users? | Can a new developer extend this without breaking it? |
| **Typical artifact** | Box-and-arrow architecture diagram | UML class diagram + working code |
| **Vocabulary** | Sharding, replication, CAP, consistency, load balancing, CDN | SOLID, design patterns, composition, polymorphism, interfaces |
| **Failure you are graded on** | Single point of failure, no bottleneck analysis | God class, `if/elif` chains, no abstraction |
| **Time in interview** | 45–60 min, whiteboard, no code | 45–90 min, often live code that must run |
| **Round name you'll see** | "System Design", "Architecture" | "Machine Coding", "Object Modelling", "LLD" |

---

## The same problem at both altitudes

**Problem:** users can pay for an order by card, UPI or wallet.

### HLD answer

```
[Client] → [API Gateway] → [Order Service] → [Payment Service] → [Card Gateway   ]
                                   │                │           → [UPI Provider   ]
                                   │                │           → [Wallet Service ]
                                   ▼                ▼
                             [Orders DB]      [Payments DB]
                                                    │
                                                    ▼
                                              [Kafka: payment.events]
                                                    │
                                                    ▼
                                            [Notification Service]
```

The discussion here: how do we avoid double-charging if the Payment Service
retries? Do we need idempotency keys? Is the payment write synchronous? What
happens when the card gateway times out?

### LLD answer

Same problem, zoomed all the way in:

```python
from abc import ABC, abstractmethod
from decimal import Decimal


class PaymentMethod(ABC):
    """One way to move money. Add a new one by adding a class, not an if-branch."""

    @abstractmethod
    def pay(self, amount: Decimal) -> "PaymentResult":
        ...


class CardPayment(PaymentMethod):
    def __init__(self, card_number: str, cvv: str) -> None:
        self._card_number = card_number
        self._cvv = cvv

    def pay(self, amount: Decimal) -> "PaymentResult":
        return PaymentResult(success=True, reference=f"CARD-{amount}")


class UpiPayment(PaymentMethod):
    def __init__(self, vpa: str) -> None:
        self._vpa = vpa

    def pay(self, amount: Decimal) -> "PaymentResult":
        return PaymentResult(success=True, reference=f"UPI-{amount}")


class Checkout:
    """Knows nothing about cards or UPI. That is the whole point."""

    def __init__(self, method: PaymentMethod) -> None:
        self._method = method

    def complete(self, amount: Decimal) -> "PaymentResult":
        return self._method.pay(amount)
```

The discussion here: where does retry logic live? Should `PaymentResult` be
immutable? If we add crypto tomorrow, how many existing files change? (Correct
answer: one — the file that picks the method.)

**In plain words:** HLD asks *"will it survive Black Friday?"*. LLD asks *"will
it survive the next feature request?"*

---

## How to tell which round you are in

The interviewer's first sentence gives it away.

| They say | You are in |
|---|---|
| "Design a URL shortener for 100M URLs" | HLD |
| "Design a parking lot" | LLD |
| "How would you scale this to 10x traffic?" | HLD |
| "Now add a new vehicle type" | LLD |
| "What database would you pick?" | HLD |
| "What classes would you need?" | LLD |
| "Walk me through a request end to end" | HLD |
| "Write the code, it should compile" | LLD |

**Trap:** an LLD problem often *sounds* like a product feature ("design a
Splitwise", "design an elevator system"). Small, physical, bounded scope with no
mention of scale = LLD. If they mention users-per-second, it is HLD.

**If you genuinely cannot tell, ask.** "Should I focus on the class design, or
the service architecture and scale?" is a question interviewers respect — it
shows you know the two are different.

---

## Where they meet

One HLD box becomes one LLD problem. The Payment Service in the diagram above is
*exactly* the `PaymentMethod` hierarchy below it. Senior candidates are the ones
who can zoom between the two without losing the thread:

> "I'd put payments in its own service because it needs PCI isolation — and
> inside it, each provider is a strategy behind a common interface, so onboarding
> a new provider is a deploy of one service, not a rewrite."

That single sentence covers both altitudes. That is the target.

---

**Next:** [02 — OOP Foundations](02-oop-foundations.md)
