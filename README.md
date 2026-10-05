# SystemDesign-LLD-HLD

Design-interview notes: **Low Level Design (LLD)** and **High Level Design (HLD)**.
Each topic lives on its own branch so you can read one thing at a time.

| Branch | Topics |
|---|---|
| [introduction-to-LLD](../../tree/introduction-to-LLD) | LLD vs HLD, OOP foundations, SOLID principles, UML for interviews, the LLD interview playbook |
| [design-patterns](../../tree/design-patterns) | Gang of Four patterns by category. Creational: Singleton |

```bash
git clone https://github.com/avadh-pro/SystemDesign-LLD-HLD.git
cd SystemDesign-LLD-HLD
git checkout introduction-to-LLD   # or design-patterns
```

## How to read this repo

Every chapter opens with a **verdict line** — the one sentence you'd keep if you
forgot everything else — then the explanation, then a concrete example.
Chapter READMEs carry a **skim cheat sheet** so you can revise a whole topic in
two minutes the night before an interview.

Code examples are **Python** on `introduction-to-LLD` and **Java** on `design-patterns`
(the language each branch's material is normally taught in).

## The two altitudes, in one line

- **HLD** — which boxes exist and how they talk. Services, databases, queues, caches.
- **LLD** — what is inside one box. Classes, methods, interfaces, data structures.

An interview loop usually tests both, in separate rounds, with different rubrics.
