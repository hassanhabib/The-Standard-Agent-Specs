# The Standard for Agents — Specification
**Version 0.1 (draft)**

A **normative, language-neutral** blueprint for building a Tri-Nature agent framework
in any language (JavaScript, .NET, Go, Rust, Python, …). The reference implementation
is `Standard.Agents` (C#). The rationale and philosophy live in the companion theory,
*The Tri-Nature of Agent* (`THE-TRI-NATURE-OF-AGENT.md`); this document is the buildable
contract.

---

## 1. Conformance

The keywords **MUST**, **MUST NOT**, **SHOULD**, **SHOULD NOT**, **MAY** are to be
interpreted as in RFC 2119.

An implementation is **Standard-Agents Conformant** at a given **profile** (§8) if it
satisfies every MUST for that profile. Conformance is about **contracts and behavior**,
not file layout or language idiom.

---

## 2. Notation

Neutral pseudo-types — map each to your language:

| Spec | Meaning |
|---|---|
| `Text` | UTF-8 string |
| `Number` | integer or real |
| `Bool` | boolean |
| `List<T>` | ordered, immutable sequence |
| `Map<K,V>` | key → value |
| `Async<T>` | a value delivered asynchronously (Promise / Future / Task / coroutine) |
| `Void` | no value |
| `enum { … }` | a closed set of named values |

`interface Name { method(params) -> Return }` denotes a **contract** (a role), not a
class. The names are canonical *roles*; adapt casing to your language.

---

## 3. Data Types (MUST)

### 3.1 AgentStatus

```
enum AgentStatus { Working, Responded, AwaitingInput, Refused, Failed }
```

- `Working` **MUST** be the default / initial value.
- The loop (§5) continues while `status == Working` and stops on any other value.
- Implementations **MUST NOT** replace this with a boolean.

### 3.2 AgentContext

The single carrier that threads the loop. **All content is Data**; the regions mark
which nature last wrote each field.

```
record AgentContext {
  prompt        : Text          // the task (input)
  systemPrompt  : Text          // DATA:      written by Recall
  observations  : List<Text>    // DATA:      written by Recall / Act
  intent        : Text          // DECISION:  written by Think
  directionType : Text          // DECISION:  written by Think
  payload       : Text          // DECISION:  written by Think
  rawReply      : Text          // DECISION:  written by Think
  result        : Text          // DIRECTION: written by Act
  status        : AgentStatus   // DIRECTION: written by Act
}
```

- Context updates **MUST** be copy-on-write: a nature returns an updated copy; a nature
  **MUST NOT** mutate a shared instance. (Use records/immutable structs; if unavailable,
  a copy helper or builder.)

---

## 4. Component Contracts

The architecture is four tiers: **Coordination → Orchestration → Foundation → Broker**.
Flow is forward only; a tier **MUST NOT** call a tier above it.

### 4.1 Brokers — thin liaisons

A broker is a liaison to **exactly one** external resource. A broker:
- **MUST** integrate one resource only.
- **MUST NOT** contain business flow control (no branching/looping for business decisions).
- **MUST NOT** author prompts, rules, or rubrics — those are Data.
- **MUST** expose asynchronous methods.

```
interface SkillBroker      { selectSkills() -> Async<List<Skill>> }
interface MemoryBroker     { selectMemories() -> Async<List<Text>>;  insertMemory(m: Text) -> Async<Void> }
interface KnowledgeBroker  { selectKnowledge(query: Text) -> Async<List<Text>> }
interface ClassifierBroker { classify(systemPrompt: Text, input: Text) -> Async<Text> }
interface GeneratorBroker  { generate(systemPrompt: Text, userPrompt: Text) -> Async<Text> }
interface VerifierBroker   { verify(systemPrompt: Text, candidate: Text) -> Async<Text> }   // a verdict: a 0.0–1.0 score and a short reason
interface ToolBroker       { has(name: Text) -> Bool;  run(name: Text, input: Text) -> Async<Text> }
interface McpBroker        { call(name: Text, input: Text) -> Async<Text> }
interface LogBroker        { reset() -> Async<Void>;  write(line: Text) -> Async<Void> }   // support broker
```

A broker's *resource* is an implementation choice, not part of the contract. The same
`KnowledgeBroker` may be backed by a folder of files, a relational database, a key-value cache, or a
vector store; the same `GeneratorBroker` may wrap a remote endpoint or an in-process model. Swapping
the resource behind a broker **MUST NOT** require any change to the services above it — this is the
system's primary extension seam, and the reason the interfaces above are the whole contract.

### 4.2 Foundation Services — one broker each

A foundation wraps one broker, returns **primitives**, and speaks business language.

```
interface SkillService        { retrieveSkills(route: Text) -> Async<Text>;  retrieveSkillCatalog() -> Async<Text> }
interface MemoryService       { recallMemories() -> Async<List<Text>>;  remember(m: Text) -> Async<Void> }
interface KnowledgeService    { retrieveKnowledge(query: Text) -> Async<List<Text>> }
interface GateService         { screen(gatePrompt: Text, input: Text) -> Async<Text> }
interface BrainService        { generate(systemPrompt: Text, userPrompt: Text) -> Async<Text> }
interface JudgeService        { evaluate(judgePrompt: Text, candidate: Text) -> Async<Judgement> }
interface InternalToolService { handles(name: Text) -> Bool;  run(name: Text, input: Text) -> Async<Text> }
interface ExternalToolService { call(name: Text, input: Text) -> Async<Text> }
interface ReturnService       { return(payload: Text) -> Async<Text> }   // NO broker — the dead end
```

`Judgement` is `{ score: Number (0.0–1.0), reason: Text }`. The Judge returns not just a score but a
short reason, so a rejection is **actionable**: the reason is what the next Brain attempt is told to
fix (§4.3). The `VerifierBroker` returns the raw verdict as `Text`; `JudgeService` parses it into the
score and reason and validates the score's range. A `GateService.screen` verdict is likewise `Text` —
a classification (`accept` / `refuse`, and **MAY** additionally `route`) that a rejection carries a
reason on, mirroring the Judge.

`Skill` is `{ name: Text, description: Text, content: Text }`. A skill source yields **discrete
skills** — each with an identity, an optional description, and a body — not one pre-composed blob, so
the service can select and index them. How a broker sources them is an implementation choice (§9): a
file-backed `SkillBroker` **SHOULD** discover skills recursively (a skill **MAY** live in its own
subdirectory) and **MAY** read each skill's `name` / `description` from metadata (e.g. frontmatter),
stripping that metadata from `content`.

`retrieveSkills(route)` composes the applicable skills' bodies into the `systemPrompt`. When `route`
is non-empty (from the Gate's `route` verdict, §4.3) it selects the matching skill(s) by `name`, so a
specialist skill is pulled in only when relevant. `retrieveSkillCatalog()` renders the **index** of
*described* skills — `name` + `description` — the advertisement a skill-index marker expands into,
exactly as tools are advertised (§6.1). A skill is advertised only if it carries a `description` (the
opt-in). Index in context + routing = the model reaches for the skill it needs, the same way it
reaches for a tool.

### 4.3 Orchestration Services — one per nature

Each coordinates its nature's foundations and returns an **updated AgentContext**.

```
interface DataOrchestration      { recall(ctx: AgentContext) -> Async<AgentContext> }
interface DecisionOrchestration  { think(ctx: AgentContext)  -> Async<AgentContext> }
interface DirectionOrchestration { act(ctx: AgentContext)    -> Async<AgentContext> }
```

Required behavior:
- **recall** — **MUST** set `systemPrompt` from `SkillService`; **MAY** seed `observations`
  from Memory/Knowledge. It refreshes what the agent HAS.
- **think** — **MUST**: (a) Gate screens the input, (b) Brain generates, (c) interpret the
  reply into `intent`/`directionType`/`payload`/`rawReply`. It **MAY** run Judge on a final
  answer and loop instead of returning (reflective judgment); when it does and the Judge
  **rejects**, the Judge's **reason MUST be fed back into `observations`** so the next Brain
  attempt revises against that feedback rather than blind. It **MUST NOT** author prompt text.
- **act** — **MUST** route by `directionType`: **Return** (terminal), **Internal** (a local
  tool), or **External** (out across the boundary). A non-terminal result **MUST** be
  appended to `observations`, and its `status` **MUST** be `Working`.

### 4.4 Coordination — the Agent

```
interface AgentCoordination { processPrompt(prompt: Text) -> Async<Text> }
```

It runs the loop (§5) and **MUST** hold no nature logic beyond sequencing and observing
(logging). An `AgentCoordination` **SHOULD** also satisfy the tool contract (§6) so an
agent can be nested as a tool of another agent (the fractal).

### 4.5 Guardian Rubric Composition

The `gatePrompt` and `judgePrompt` a guardian receives are **assembled**, not hardcoded,
from up to three ordered layers:

1. **Constitution** (optional): an ethical constitution, prepended above everything. When
   absent it contributes nothing.
2. **Policy**: what the guardian screens or scores. A default policy is built in; a host
   **MAY** replace it with a consumption skill.
3. **Contract**: the output protocol the broker parses (the Gate's classification line, the
   Judge's score-and-reason). It is framework-owned.

Rules:

- The layers **MUST** be assembled in the order constitution, policy, contract.
- The contract **MUST** always be present and **MUST NOT** be replaceable by a supplied
  policy, so a replacement policy can never break the broker's parsing.
- The constitution and the consumption policy, when supplied, **MUST** be loaded from Data
  (§7.2), never hardcoded.
- Absent a constitution and a consumption policy, a guardian **MUST** behave exactly as it
  does with its built-in policy and contract.

### 4.6 Boundary Protections (Full, OPTIONAL capabilities)

Two optional perimeter controls harden where the agent meets an untrusted outside: the Brain
(which **MAY** be a remote host) and the tools it can drive. Each is off unless configured, each
is Data-driven, and neither adds a nature. They are enforcement at an existing boundary, not new
reasoning.

**Redaction at the Brain boundary.** An implementation **MAY** redact sensitive values from the
text sent to the Brain. When redaction is configured:

- Every configured pattern **MUST** be replaced, in the `systemPrompt` and `userPrompt` alike,
  with an opaque, reversible placeholder before the `GeneratorBroker` is called.
- The Brain, and any remote host serving it, **MUST NOT** receive the sensitive value in the clear.
- The mapping from placeholder to original value **MUST** stay inside the agent and **MUST NOT**
  cross to the Brain. The Brain's reply **MUST** be rehydrated, each placeholder restored to its
  original, before the value is returned to the caller or written to Data.
- The redaction rules (label and pattern) are Data (§7.2) and **MUST NOT** be hardcoded in a broker.
- Redaction is a confidentiality control at the boundary (Invariant 8). It **MUST NOT** alter the
  loop, the reply protocol, or any verdict.

**Least-privilege tool allow-list.** An implementation **MAY** restrict which tools may actually
run to a configured allow-list. When an allow-list is configured:

- The Brain **MAY** still propose any tool. The allow-list constrains execution, not proposal.
- A proposed tool not on the allow-list **MUST** be denied at Direction (§4.3) before it executes,
  never partially run and then rolled back (Invariant 7).
- A denial **MUST** be non-terminal: it is appended to `observations` with `status` `Working`, so
  the agent can choose a permitted path on the next turn (§5), exactly as it recovers from a
  malformed call (§6.1).
- Matching **MUST** be over the tool name and **SHOULD** be case-insensitive. The allow-list is Data.
- Absent an allow-list, every registered tool is runnable. The control is opt-in and its absence
  changes nothing.

---

## 5. The Loop (normative algorithm)

```
processPrompt(prompt):
    logBroker.reset()
    context := AgentContext { prompt: prompt }        // status defaults to Working
    for turn in 1 .. MAX_TURNS:                        // MAX_TURNS MUST be finite; SHOULD be small
        context := dataOrchestration.recall(context)
        context := decisionOrchestration.think(context)
        context := directionOrchestration.act(context)
        if context.status != Working: break
    return context.result
```

- The loop **MUST** cap iterations (`MAX_TURNS`).
- Direction **MUST** feed non-terminal results back into `observations`, and the next
  Decision **MUST** be able to read them (this is how tool results and fetched data return
  to the Brain).

---

## 6. The Tool / Reply Protocol

The Brain expresses its choice as text (**reference protocol**, SHOULD support), as a
structured tool-call emitted as text (§6.1, **MAY**), or via a provider-native tool-call
mechanism (**MAY**).

### 6.0 Reference protocol (text)

- To act: a single line — `ACTION: <toolName>: <input>`
- To answer: `FINAL: <answer>`

Parsing (MUST):
- A tool call **MUST** be read from the **first line only** (models often emit extra lines).
- A final answer **MAY** span multiple lines.
- `directionType == "ReturnResponse"` is terminal → `Responded`.
- `directionType == "Refuse"` is terminal → `Refused`.
- Any other `directionType` is a tool name, routed by Direction (§4.3).

### 6.1 Structured tool-calls (Full — MAY)

A Full implementation **MAY** additionally accept **structured tool-calls** — the same choice
expressed as typed data rather than a text line. The structured call is emitted **as text and
parsed**, so it is portable to any endpoint and requires no provider-native tool API.

**Tool contract.** A tool offered for structured calling declares three things:
- `name` — the identifier used to route the call.
- `description` — what the tool does and when to use it.
- `parameters` — a schema of typed inputs (JSON Schema or equivalent).

**Advertisement.** The offered tools — name, description, parameters — **MUST** be presented to
the Brain as Data (§7.2): rendered into the developer's Data at a location the developer controls,
never force-injected. The Brain **MUST NOT** be expected to use a tool it was not shown.

Advertisement is **under the developer's control**, because which tools a Brain may reach for is a
safety boundary, not a convenience:
- A tool is advertised only if it carries a **description** — the description is the opt-in. A tool
  with no description is **callable but not advertised**: still reachable if the Brain names it, but
  the implementation does not offer it.
- The developer controls *whether*, *where*, and with *what framing* the catalog appears. An
  implementation **MUST NOT** inject the catalog unbidden. The reference implementation expands a
  `{{tools}}` marker in Data into the catalog of described tools; absent the marker, nothing is
  advertised.
- The catalog itself is **derived from the tools** (name/description/parameters), so it cannot drift
  from what is actually registered.

**The call.** To act, the Brain emits a single line:

```
TOOL: { "tool": "<name>", "arguments": { ... } }
```

- The object **MUST** carry a string `tool` and an object `arguments`.
- `arguments` **SHOULD** conform to the tool's advertised `parameters`.
- To answer, `FINAL: <answer>` — unchanged.

**Parsing (MUST).**
- The tool call **MUST** be read from the **first line only**, as in §6.0.
- An implementation supporting both forms **MUST** accept `ACTION: <tool>: <input>` (§6.0) and
  `TOOL: { … }` interchangeably.
- A malformed `TOOL:` line (unparseable JSON, missing `tool`) **MUST NOT** end the loop as a
  fault; it is a non-terminal turn the agent recovers from (§5), exactly as a nameless
  `ACTION:` is treated as an answer.

**Direction.** The routed tool receives the structured `arguments`; a tool that consumes a single
string **MAY** receive the raw `arguments` JSON. Where a sub-agent is the tool, its handoff
(Data) wraps the arguments into the sub-agent's prompt.

Structured tool-calls are **additive**: the text reference protocol remains the Core contract
(§8.1), and Core conformance is unaffected.

---

## 7. Invariants (MUST / MUST NOT)

1. **Everything is Data.** Produced content (a reply, a tool result, a fetched document)
   **MUST** be treated as Data once produced.
2. **Prompts, rules, and rubrics live in Data.** They **MUST** be loaded from skills/config
   and **MUST NOT** be authored in Decision, in brokers, or hardcoded.
3. **Brokers are thin.** A broker **MUST NOT** contain business flow control.
4. **The agent is ephemeral.** An instance **MUST** be stateless across prompts. Persistent
   memory **MUST** live outside the agent — recalled by Data, written by Direction.
5. **A draft is not a commitment.** Where a guardian applies, output **MUST NOT** cross a
   boundary un-vetted.
6. **No self-certification.** A guardian (Gate/Judge) **MUST NOT** be the Brain. A faculty
   **MUST NOT** certify its own trustworthiness. Guardian output that attempts to *answer* or
   *act* (rather than screen or score) **MUST** be neutralized — treated as a non-authoritative
   classification, never executed as the Brain's — and **SHOULD** be recorded.
7. **Irreversible before, not after.** An irreversible action **MUST** be authorized before
   execution by a trusted guardian (deterministic rule and/or human), never after.
8. **The boundary.** External state **MUST** enter only as Data (via a Direction that reached
   out) and effects **MUST** leave only via Direction. The three natures are the agent's
   interior.
9. **Configuration is not a nature.** It **MUST** be consumed by brokers to function, not
   reasoned over by Decision.

---

## 8. Conformance Profiles

### 8.1 Core — the minimal viable agent (MUST)

- `AgentContext`, `AgentStatus` (§3).
- The Coordination loop (§5).
- **Data:** `SkillService` + `SkillBroker`.
- **Decision:** `BrainService` + `GeneratorBroker`. Gate and Judge **MAY** be pass-through.
- **Direction:** `InternalToolService` + `ToolBroker`, and `ReturnService`.
- The reply protocol (§6).
- Invariants 1–4, 8, 9.

### 8.2 Full — adds

- `MemoryService`, `KnowledgeService` backed by real stores.
- `GateService` and `JudgeService` as **real guardians, distinct from the Brain**.
- `ExternalToolService` (MCP / remote).
- **Structured tool-calls (§6.1):** tools carry `description` + `parameters`, advertised to the
  Brain, which **MAY** emit typed `TOOL:` calls.
- **Boundary protections (§4.6):** redaction at the Brain boundary and a least-privilege tool
  allow-list, both OPTIONAL; when either is provided it **MUST** conform to §4.6.
- Invariants 5–7 (guardian & safety).

---

## 9. Porting Notes

- **Async:** use the language's native model; every contract method is asynchronous.
- **Immutability:** records/immutable structs with copy-update. If unavailable, a builder
  or an explicit `copyWith`.
- **Naming:** adapt case (PascalCase / camelCase / snake_case). The identifiers here are
  canonical roles, not literal names.
- **DI:** OPTIONAL. A hand-wired composition root is fully conformant.
- **Tiers:** the four tiers **SHOULD** be preserved as module boundaries. A small-scale
  implementation **MAY** collapse a tier that adds no value (e.g. back the three Decision
  brokers with one inference endpoint), provided the contracts and invariants still hold.
- **Inference:** `GeneratorBroker` **SHOULD** target an OpenAI-compatible
  `POST /v1/chat/completions` endpoint for interoperability, but **MAY** wrap any model.
- **Data stores are backend-neutral.** `MemoryBroker` and `KnowledgeBroker` define *operations*,
  not storage. An implementation **MAY** back them with files, a SQL database, a cache, or a
  document / vector store — the contract is the boundary. Retrieval quality (substring, full-text,
  semantic) is the broker's concern and **MAY** improve without changing the interface or any
  service above it.
- **Store scope is fixed at construction.** A shared store may serve many agents, users, or
  sessions. Which partition a broker reads and writes — a path, a key, a table — **MUST** be bound
  when the broker is constructed, so the broker contract stays scope-free. One broker instance
  serves one scope; multi-tenancy is achieved by constructing one broker per scope, not by widening
  the interface.
- **The Decision substrate is location-neutral.** `GeneratorBroker`, `ClassifierBroker`, and
  `VerifierBroker` each **MAY** run in-process (a local model) or across the network (a remote
  endpoint), and independently of one another. One model **MAY** back all three; because each is
  driven by its own Data-supplied rubric, Invariant 6 (no self-certification) still holds — the
  substrate is shared, the consciences are not.
- **The guardian substrate is model-optional.** A Gate (`ClassifierBroker`) or Judge
  (`VerifierBroker`) **MAY** be backed by a deterministic rule (a fixed refuse-on-match screen, or
  a required-content check) instead of a model. The service contract (§4.2) is unchanged: the loop
  still calls `screen` / `evaluate` and reads the same verdict. A deterministic guardian trivially
  satisfies Invariant 6 (it is not the Brain), and is the preferred substrate wherever the rule is
  stateable, because compliance is then reproducible rather than probabilistic (Invariant 7).
- **Observability is layered and optional.** The support `LogBroker` (§4.1) records the run. An
  implementation **MAY** offer a human-readable trace organized as Turn, Step, Process with
  selectable verbosity (outcomes only; per-nature; or every step), and **MAY** additionally emit a
  machine-readable audit, one structured record per event, to a separate sink for a SIEM or
  telemetry pipeline. Observability **MUST** be side-effect-free with respect to the loop: enabling
  it or changing its verbosity **MUST NOT** change any verdict, route, or result.

---

## 10. Conformance Checklist

- [ ] `AgentContext` and `AgentStatus` as specified (§3)
- [ ] Loop caps turns; non-terminal results feed back into observations (§5)
- [ ] Brokers thin — one resource, no flow control, no authored prompts (§4.1, §7.3)
- [ ] Broker resources swappable behind their contracts — Data stores and the inference substrate
      MAY be file / DB / cache / local / remote, scope bound at construction, without changing any
      service above (§4.1, §9)
- [ ] Prompts / rules / rubrics loaded from Data, never hardcoded (§7.2)
- [ ] Skills are discrete entries (name / description / content); a file source discovers them
      recursively; described skills are advertised as an index for model-driven routing (§4.1–§4.2)
- [ ] Agent instance stateless across prompts; persistent memory external (§7.4)
- [ ] Reply protocol: first-line ACTION, terminal ReturnResponse / Refuse (§6)
- [ ] **Full:** guardians (Gate/Judge) distinct from the Brain; guardian output that answers or
      acts is neutralized, not obeyed (§7.6)
- [ ] **Full:** a rejecting Judge returns a reason, fed back into observations as revision
      feedback (§4.2, §4.3)
- [ ] **Full:** irreversible actions authorized before execution (§7.7)
- [ ] **Full:** structured tool-calls — tools advertise name/description/parameters; `TOOL:`
      calls parsed from the first line; malformed calls recover, not crash (§6.1)
- [ ] **Full (optional):** boundary redaction hides configured values from the Brain and
      rehydrates the reply; the Brain never sees them in the clear (§4.6)
- [ ] **Full (optional):** a tool allow-list denies disallowed tools at Direction before
      execution, non-terminally (§4.6)
- [ ] **Optional:** a guardian MAY be a deterministic rule; observability MAY add trace verbosity
      and a machine-readable audit, both side-effect-free (§9)

---

*Specification v0.1 — evolves with the theory. Reference implementation: `Standard.Agents` (C#).
Rationale: `THE-TRI-NATURE-OF-AGENT.md`.*
