# The Standard for Agents — Specs

The **theory, specification, and architecture** for building AI agents according to the
Tri-Nature Theory. Language-agnostic — build a conformant framework in any language
(JavaScript, Go, Rust, Python, .NET, …).

```
Agent = Orchestration(Data, Decision, Direction)
```

- **Data** — what the agent *has* (skills, memory, knowledge)
- **Decision** — what the agent *thinks* (one brain, wrapped in a Gate and a Judge)
- **Direction** — what the agent *does* (act internally, act externally, or return)

## Contents

| File | What it is |
|---|---|
| [`THE-TRI-NATURE-OF-AGENT.md`](THE-TRI-NATURE-OF-AGENT.md) | **The theory** — the model, the fractal, lifecycle & memory, safety (the privilege boundary), and governance (jurisdiction). *Why.* |
| [`SPEC.md`](SPEC.md) | **The specification** — a normative, RFC-2119, language-neutral contract: all component contracts, the loop algorithm, the reply protocol, the invariants, and **Core** vs **Full** conformance profiles. *What to build.* |
| [`architecture.svg`](architecture.svg) | The 1·3·9 tier architecture (`Coordination → Orchestration → Foundation → Broker`). |

## Reference implementation

The C# reference implementation — and a runnable conformance suite — live in
**[hassanhabib/The-Standard-Agent](https://github.com/hassanhabib/The-Standard-Agent)**.

## Building your own

Read [`SPEC.md`](SPEC.md). Implement the contracts and the loop in your language, honor
the nine MUST/MUST-NOT invariants, and certify against the conformance vectors. A **Core**
implementation is a minimal viable agent; a **Full** implementation adds real memory,
knowledge, guardian gate/judge, and external tools.

## License

The Standard Software License (TSSL) — see [LICENSE](LICENSE).
