# 5. The LLD Interview

> **Verdict:** You are not graded on finishing. You are graded on whether your design absorbs a new requirement without a rewrite — so budget time to *be given one*.

---

## The 45-minute shape

| Minutes | Phase | What you produce |
|---|---|---|
| 0–5 | **Clarify** | An agreed, written list of in-scope features |
| 5–12 | **Identify entities** | Nouns → candidate classes, with responsibilities |
| 12–22 | **Class diagram** | Boxes, relationships, key methods |
| 22–38 | **Code the core** | The 2–3 classes that carry the design |
| 38–45 | **Extend** | They add a requirement; you absorb it |

The last row is the exam. Everything before it is setup.

---

## Phase 1 — Clarify (0–5 min)

Do **not** start drawing. Ask, and write the answers where they can see them.

The four questions that always pay:

1. **Scope** — "Should I handle payments, or assume that's another service?"
2. **Actors** — "Who uses this? Just customers, or admins too?"
3. **Scale** — "Single machine in-memory, or does this need persistence and
   concurrency?" *(This is the LLD version of the scale question — it decides
   whether you need locks, not whether you need shards.)*
4. **Priority** — "If I run short on time, which part matters most to you?"

Then read the scope back:

> "So: park and unpark a vehicle, three vehicle sizes, hourly pricing, in-memory,
> single-threaded to start. Payments and reservations are out. Correct?"

**Why this matters more than it looks:** half of LLD failures are candidates
solving a bigger problem than they were asked, running out of time, and shipping
nothing that works.

---

## Phase 2 — Entities (5–12 min)

Pull the nouns out of the problem statement. They are your candidate classes.

> *"A **parking lot** has multiple **floors**. Each floor has **spots** of
> different **sizes**. A **vehicle** takes a spot and gets a **ticket**. On exit
> the **ticket** is used to compute a **fee**."*

Candidates: `ParkingLot`, `ParkingFloor`, `ParkingSpot`, `VehicleSize`,
`Vehicle`, `Ticket`, `Fee`.

Now prune and assign one responsibility each — say it out loud:

| Class | Its one job |
|---|---|
| `ParkingLot` | Entry point; find a spot, issue a ticket |
| `ParkingFloor` | Own its spots; find a free one of a size |
| `ParkingSpot` | Know its size and whether it's occupied |
| `Vehicle` | Know its plate and size |
| `Ticket` | Record spot + entry time |
| `PricingStrategy` | Turn duration into money |

**Verbs become methods. Nouns become classes.** That's the heuristic.

`Fee` was a noun but became a return value — pruning like that, out loud, is a
positive signal.

---

## Phase 3 — Class diagram (12–22 min)

Draw it ([chapter 4](04-uml-for-interviews.md)). Narrate while you draw — silence
is the enemy in this round.

Three decisions to make explicitly, because they are what gets probed:

1. **Where does each piece of state live?** (Who owns the list of spots — the lot
   or the floor?)
2. **What is behind an interface?** Pick the axis you believe will change.
   Pricing and vehicle type are the usual answers.
3. **What is a value object?** Immutable, no identity — `VehicleSize`, `Money`,
   `TimeRange`. Enums and frozen dataclasses. Mentioning immutability here is
   cheap and scores well.

---

## Phase 4 — Code the core (22–38 min)

You will not finish the whole system. **Say which parts you're skipping and why.**

Code the classes that carry the design — the interface, one or two
implementations, and the orchestrator. Stub the rest with a one-line body.

```python
from abc import ABC, abstractmethod
from dataclasses import dataclass
from datetime import datetime
from enum import Enum


class Size(Enum):
    SMALL = 1
    MEDIUM = 2
    LARGE = 3


@dataclass(frozen=True)
class Vehicle:
    plate: str
    size: Size


class ParkingSpot:
    def __init__(self, spot_id: str, size: Size) -> None:
        self._id = spot_id
        self._size = size
        self._vehicle: Vehicle | None = None

    @property
    def is_free(self) -> bool:
        return self._vehicle is None

    def fits(self, vehicle: Vehicle) -> bool:
        return self.is_free and self._size.value >= vehicle.size.value

    def occupy(self, vehicle: Vehicle) -> None:
        if not self.fits(vehicle):
            raise ValueError(f"spot {self._id} cannot take {vehicle.plate}")
        self._vehicle = vehicle

    def release(self) -> None:
        self._vehicle = None


@dataclass(frozen=True)
class Ticket:
    ticket_id: str
    spot: ParkingSpot
    entered_at: datetime


class ParkingFloor:
    def __init__(self, level: int, spots: list[ParkingSpot]) -> None:
        self._level = level
        self._spots = spots

    def find_spot(self, vehicle: Vehicle) -> ParkingSpot | None:
        return next((s for s in self._spots if s.fits(vehicle)), None)


class PricingStrategy(ABC):
    @abstractmethod
    def fee(self, hours: float) -> float:
        ...


class HourlyPricing(PricingStrategy):
    def __init__(self, rate: float) -> None:
        self._rate = rate

    def fee(self, hours: float) -> float:
        return round(hours * self._rate, 2)


class LotFullError(Exception):
    pass


class ParkingLot:
    def __init__(self, floors: list[ParkingFloor], pricing: PricingStrategy) -> None:
        self._floors = floors
        self._pricing = pricing          # injected — swap without touching this class
        self._counter = 0

    def park(self, vehicle: Vehicle) -> Ticket:
        for floor in self._floors:
            spot = floor.find_spot(vehicle)
            if spot is not None:
                spot.occupy(vehicle)
                self._counter += 1
                return Ticket(f"T{self._counter}", spot, datetime.now())
        raise LotFullError(f"no spot for {vehicle.size.name}")

    def unpark(self, ticket: Ticket, now: datetime | None = None) -> float:
        now = now or datetime.now()
        hours = (now - ticket.entered_at).total_seconds() / 3600
        ticket.spot.release()
        return self._pricing.fee(hours)
```

Habits that score while you type:
- **Name things fully.** `find_spot`, not `fs`.
- **Raise real exceptions.** `LotFullError`, not `return None`.
- **Type hints.** Free credibility in Python.
- **Say the tradeoff.** *"Linear scan across floors — O(floors × spots). If this
  needed to scale I'd keep a free-spot heap per size."* Naming the weakness
  before they do is a senior signal.

---

## Phase 5 — The extension (38–45 min)

This is the real question. It always comes, and it always sounds casual:

> "Nice. Now add electric vehicles that need charging spots."
> "Now add weekend pricing."
> "Now handle two cars arriving at the same instant."

**If your design is good, you answer in files, not edits:**

> "`ELECTRIC` joins the `Size` enum — actually no, charging is a *feature*, not a
> size. I'd add a `SpotFeature` set on `ParkingSpot` and widen `fits()` to check
> required features. `ParkingLot` doesn't change at all."

> "Weekend pricing is a new `PricingStrategy` implementation. One new class,
> nothing else moves — that's why pricing is behind an interface."

**If your answer is "I'd add an `elif` in `park()`", you have just failed the
round.** That is exactly the signal Open/Closed exists to produce.

**If the design genuinely can't absorb it, say so.** "My current model puts size
on the spot, so features don't fit cleanly — I'd refactor `ParkingSpot` to hold a
capability set." Recognising the limit is worth far more than pretending it fits.

---

## The rubric they're marking against

| Signal | What it looks like |
|---|---|
| **Requirements** | Clarified scope before coding; didn't over-build |
| **Modelling** | Classes map to real concepts; one responsibility each |
| **Abstraction** | Interfaces on the axes that actually vary |
| **Extensibility** | New requirement = new class |
| **Code quality** | Readable names, real exceptions, no dead code |
| **Communication** | Narrated decisions and tradeoffs throughout |

Note what's absent: cleverness, completeness, and knowing all 23 GoF patterns.

---

## Failure modes, ranked by how often they happen

1. **Coding before clarifying.** You build the wrong thing beautifully.
2. **Analysis paralysis.** Twenty minutes of diagram, no code. Get to code by
   minute 22 even if the diagram is unfinished.
3. **Pattern-dumping.** Forcing Singleton + Factory + Observer into a parking
   lot. Use a pattern when it removes a problem, and *name the problem* it
   removes.
4. **The god class.** `ParkingLot` doing finding, pricing, ticketing and
   persistence. Interviewers look for this specifically.
5. **Silence.** They cannot grade thinking they can't hear. Narrate.
6. **`if/elif` on a type.** The single loudest negative signal in LLD. Any time
   you branch on what something *is*, that's polymorphism asking to be used.
7. **Defending a bad call.** When they push back, they're usually hinting.
   "That's fair — the cleaner split would be…" costs nothing and scores well.

---

## The problems that actually get asked

Warm-ups: parking lot · vending machine · elevator · ATM · tic-tac-toe

Mid: Splitwise · BookMyShow · library management · snake & ladders · chess

Harder: rate limiter · LRU cache · logging framework · notification service ·
in-memory key-value store with TTL

Almost all of them reduce to: *a small set of entities, one thing that varies
behind an interface, and one orchestrator.* If you can do the parking lot
cleanly, you can do most of the list.

---

## Skim cheat sheet

- **Clarify 5 min. Diagram by 22. Code by 38. Save 7 for the extension.**
- Nouns → classes. Verbs → methods. Prune out loud.
- Put an interface on the axis you expect to change. Name that axis.
- Inject dependencies; never construct them inside a constructor.
- Narrate every decision, including the ones you're unsure about.
- Name your own design's weakness before the interviewer does.
- The extension question is the exam. A good answer adds a file. A bad answer
  adds an `elif`.

---

**Back to:** [Chapter index](README.md)
