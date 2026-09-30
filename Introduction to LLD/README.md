# Introduction to LLD

The foundations you need before any Low Level Design round: what LLD is, the OOP
it's built on, the five principles you're graded against, the three diagrams
worth drawing, and how the 45 minutes actually go.

Code examples are Python.

## Chapters

| # | Chapter | Verdict line |
|---|---|---|
| 1 | [LLD vs HLD](01-lld-vs-hld.md) | HLD decides which boxes exist; LLD decides what's inside one box |
| 2 | [OOP Foundations](02-oop-foundations.md) | Each pillar is a defence against a specific way code rots |
| 3 | [SOLID Principles](03-solid-principles.md) | Five rules, one goal — make change cheap |
| 4 | [UML for Interviews](04-uml-for-interviews.md) | Three diagrams and six symbols; the rest is noise |
| 5 | [The LLD Interview](05-the-lld-interview.md) | You're graded on absorbing a new requirement, not on finishing |

---

## Skim cheat sheet

Two minutes, the night before.

**LLD vs HLD**
- HLD = services, DBs, queues, scale. LLD = classes, interfaces, methods.
- Small bounded problem, no traffic numbers mentioned → it's LLD.
- If unsure which round you're in, ask. It's a respected question.

**The four pillars**
- **Encapsulation** — one door to the data, and I guard it.
- **Abstraction** — you get a steering wheel, not the engine.
- **Inheritance** — inherit for "is-a", compose for "has-a".
- **Polymorphism** — kills the `if/elif` chain.
- Target state: **low coupling, high cohesion.**

**SOLID**
| | Smell | Fix |
|---|---|---|
| S | Description needs "and" | Split by reason-to-change |
| O | Editing tested `if/elif` to add a feature | Strategy behind an interface |
| L | `NotImplementedError` in an override | Re-model around what's truly common |
| I | Fat interface with `pass` stubs | Split into small role interfaces |
| D | `self._db = MySql()` in `__init__` | Inject the abstraction |

**UML**
- `+` public · `-` private · `#` protected
- `▷` is-a · `◆` owns, dies together · `◇` has, survives · `-->` knows · `..>` uses
- Multiplicity on the line: `1`, `0..1`, `*`, `1..*`
- Unsure of a symbol? Label the line in English. Never guess a diamond.

**The 45 minutes**
- Clarify 5 · entities 12 · diagram 22 · code 38 · **extension 45**
- Nouns → classes, verbs → methods.
- Interface goes on the axis you expect to change — and name that axis.
- Narrate everything. Name your design's weakness before they do.
- The extension question is the exam: a good answer adds a **file**, a bad
  answer adds an **`elif`**.

---

## Worth knowing

These chapters teach the foundation, not the catalogue. Design patterns
(Strategy, Factory, Observer, Builder, Singleton, Decorator…) and worked
machine-coding problems are separate topics — they belong on their own branches
once this one is solid.
