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

### 1.1 Claiming conformance

A conformance claim that cannot be checked is marketing. An implementation claiming a profile:

- **MUST** state which profile it claims, and **MUST** state it per released version. "Conformant"
  without a profile and a version says nothing.
- **MUST** make the claim **reproducible by a third party**: the evidence is a runnable
  certification over the vectors, and running it **MUST** be possible without access to anything
  the implementer holds privately.
- **MUST NOT** claim a profile whose evidence it has not run. A profile whose requirements are
  merely believed to hold is not claimed, it is assumed.
- **SHOULD** publish what a consumer needs in order to depend on it: which contracts are stable,
  what a deprecation looks like and how long one lasts, and how to move between versions.

These are obligations on the *claim*, not on the agent. An implementation may be excellent and
claim nothing; what it may not do is claim a level it cannot demonstrate.

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
enum AgentStatus { Working, Responded, AwaitingInput, AwaitingApproval, Refused, Failed }
```

- `Working` **MUST** be the default / initial value.
- `AwaitingApproval` is terminal for the turn and means an effect is proposed but not performed
  (§4.9). It is distinct from `AwaitingInput`: the agent is not asking the user a question, it is
  asking an authority for permission.
- The loop (§5) continues while `status == Working` and stops on any other value.
- Implementations **MUST NOT** replace this with a boolean.

### 3.2 AgentContext

The single carrier that threads the loop. **All content is Data**; the regions mark
which nature last wrote each field.

```
record AgentContext {
  prompt        : Text          // the task (input)
  sessionId     : Text?         // DATA:      the conversation this prompt belongs to (§4.11)
  history       : List<Turn>    // DATA:      what was said before, oldest first (§4.11)
  systemPrompt  : Text          // DATA:      written by Recall
  observations  : List<Text>    // DATA:      written by Recall / Act
  intent        : Text          // DECISION:  written by Think
  directionType : Text          // DECISION:  written by Think
  payload       : Text          // DECISION:  written by Think
  rawReply      : Text          // DECISION:  written by Think
  result        : Text          // DIRECTION: written by Act
  status        : AgentStatus   // DIRECTION: written by Act
}

record Turn {
  prompt : Text
  answer : Text
}
```

- Context updates **MUST** be copy-on-write: a nature returns an updated copy; a nature
  **MUST NOT** mutate a shared instance. (Use records/immutable structs; if unavailable,
  a copy helper or builder.)

### 3.3 AgentEffect (OPTIONAL capability — §4.9)

A proposed act, described. Direction routes a tool call today as a bare name and payload; an
implementation offering the perimeter controls of §4.9 **MUST** carry them as an effect instead,
because authorization, approval and run-once are all judgments about *an act*, and an act with no
identity cannot be judged twice the same way.

```
enum RiskLevel { Safe, Sensitive, Irreversible }

record AgentEffect {
  runId               : Text        // the run proposing it (§4.4)
  principal           : Text?       // on whose behalf, when known
  toolName            : Text
  arguments           : Text
  idempotencyKey      : Text        // derived, not supplied — see §4.9
  riskLevel           : RiskLevel   // defaults to Safe
  approvalRequirement : Bool        // defaults to false
}
```

- `riskLevel` **MUST** default to `Safe`, so an implementation that adopts effects without
  classifying its tools behaves exactly as it did before.

### 3.4 Usage (OPTIONAL capability — §4.10)

What a model call actually cost. Reported by the provider where it can be, so a budget bounds
real consumption rather than an estimate of it.

```
record Usage {
  promptTokens     : Number
  completionTokens : Number
}
```

- An implementation **MUST NOT** present an estimate as a measurement. Where a provider reports
  usage, the reported value **MUST** be used; where it does not, the absence **MUST** be
  distinguishable from zero.

### 3.5 AuditRecord (OPTIONAL capability — §4.7)

One structured event in the agent's **decision log**. Present only when a host configures a
decision log; the type is specified here so that log is portable across implementations.

```
enum AuditKind { Run, Turn, Step, Process, Outcome, Error }

record AuditRecord {
  runId        : Text          // identifies exactly one processPrompt invocation
  sequence     : Number        // monotonic within the run, starting at its first record
  timestamp    : Text          // the instant, UTC, in a fixed machine-readable format
  kind         : AuditKind
  actor        : Text          // which tier or nature produced it
  message      : Text
  principal    : Text?         // on whose behalf the run executes; absent when unattributed
  previousHash : Text?         // tamper-evidence chain (§4.7)
}
```

- `runId` **MUST** be unique per invocation and **MUST NOT** be reused across prompts.
- `sequence` **MUST** be per-run, not global, so a run remains reconstructible from an
  interleaved sink.

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
interface VerifierBroker   { verify(systemPrompt: Text, task: Text, candidate: Text) -> Async<Text> }   // a verdict: a 0.0–1.0 score and a short reason
interface ToolBroker       { has(name: Text) -> Bool;  run(name: Text, input: Text) -> Async<Text> }
interface McpBroker        { call(name: Text, input: Text) -> Async<Text> }
interface LogBroker        { reset() -> Async<Void>;  write(line: Text) -> Async<Void> }   // support broker
interface AuditBroker      { write(record: AuditRecord) -> Async<Void> }                   // support broker, OPTIONAL (§4.7)
interface PolicyBroker     { authorize(effect: AgentEffect) -> Async<AuthorizationDecision> }  // support broker, OPTIONAL (§4.9)
interface ApprovalBroker   { request(effect: AgentEffect) -> Async<ApprovalDecision> }         // support broker, OPTIONAL (§4.9)
interface ResilienceBroker { execute<T>(operation: () -> Async<T>) -> Async<T> }               // support broker, OPTIONAL (§4.10)
interface SessionBroker    { selectSession(id: Text) -> Async<Session?>;  upsertSession(s: Session) -> Async<Void> }  // OPTIONAL (§4.11)
```

`AuthorizationDecision` is `{ permitted: Bool, reason: Text }` and `ApprovalDecision` is
`enum { Approved, Denied, Pending }`. A decision **MUST** carry its reason: a denial with no
reason cannot be audited, cannot be appealed, and cannot be told apart from a fault.

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
interface JudgeService        { evaluate(judgePrompt: Text, task: Text, candidate: Text) -> Async<Judgement> }
interface InternalToolService { handles(name: Text) -> Bool;  run(name: Text, input: Text) -> Async<Text> }
interface ExternalToolService { call(name: Text, input: Text) -> Async<Text> }
interface ReturnService       { return(payload: Text) -> Async<Text> }   // NO broker — the dead end
```

`Judgement` is `{ score: Number (0.0–1.0), reason: Text }`. The Judge returns not just a score but a
short reason, so a rejection is **actionable**: the reason is what the next Brain attempt is told to
fix (§4.3). The `VerifierBroker` returns the raw verdict as `Text`; `JudgeService` parses it into the
score and reason and validates the score's range.

**The Judge MUST receive the task.** An answer is not good or bad on its own — it is good or bad
*for a question*. Correctness, completeness and relevance are all judgments about the fit between a
task and an answer, so a Judge given only the candidate is being asked to score a fit it cannot
see, and its verdict is noise dressed as a number. The task **MUST** therefore be passed to
`evaluate` alongside the candidate and reach the `VerifierBroker`. This is a **MUST** and not a
**SHOULD** because the failure is silent: a blind Judge still returns a plausible score, still
rejects, and still spends a turn of the loop's budget on a revision it could not have reasoned
about. A `GateService.screen` verdict is likewise `Text` —
a classification (`accept` / `refuse`, and **MAY** additionally `route`) that a rejection carries a
reason on, mirroring the Judge.

`Skill` is `{ name: Text, description: Text, content: Text }`. A skill source yields **discrete
skills** — each with an identity, an optional description, and a body — not one pre-composed blob, so
the service can select and index them. How a broker sources them is an implementation choice (§9): a
file-backed `SkillBroker` **SHOULD** discover skills recursively (a skill **MAY** live in its own
subdirectory) and **MAY** read each skill's `name` / `description` from metadata (e.g. frontmatter),
stripping that metadata from `content`.

**Retrieval is ranked, and what Recall injects is bounded.** `retrieveKnowledge` and
`recallMemories` answer *what is relevant to this task*, not *what exists*.

- Retrieval **MUST** rank candidates by relevance to the query and return the best, not the first
  found. Matching the whole query literally against whole documents is not retrieval: a natural
  question will not appear verbatim inside an answer, so such an implementation returns nothing
  for exactly the queries a user asks.
- A result **SHOULD** be the relevant passage rather than the whole document, so a large source
  costs proportionally to what it contributed.
- How relevance is computed — term overlap, lexical scoring, embeddings — is the broker's concern
  and **MAY** improve without changing any contract above it (§9).
- What Recall injects **MAY** be bounded. When it is, the bound applies across skills, memories
  and knowledge together, and the highest-ranked content **MUST** be kept. Unbounded recall makes
  every prompt cost whatever has accumulated, which is a bill that grows on its own.
- Memory **MAY** be filtered by relevance and age. Forgetting is a feature: a memory store that
  only grows eventually poisons every prompt it is injected into.

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

**Concurrent invocation.** Invariant 4 says an agent instance is stateless across prompts. Its
direct consequence is that one instance **MUST** be able to serve many prompts at once:

- A single `AgentCoordination` instance **MUST** support concurrent `processPrompt` invocations.
  Two prompts in flight **MUST NOT** observe or overwrite each other's state, and the result of
  each **MUST** be identical to running it alone.
- Each invocation **MUST** establish its own **run identity**, and everything the invocation
  records — its turns, its steps, its decision-log records (§4.7) — **MUST** be attributed to
  that run and to no other.
- Run state is therefore **per invocation, never per instance**. An implementation that keeps a
  run's identity, counters, or timing in fields shared by the instance is **not** conformant,
  because a second concurrent prompt silently corrupts the first's record.
- This is a **statement about state, not about parallelism.** An implementation **MAY** serialize
  work internally; what it **MUST NOT** do is let one run's bookkeeping leak into another's.

How run identity is carried is an implementation choice (§9) — a parameter, a context field, or
an ambient per-flow value — provided the guarantee above holds.

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

**Redaction at the model boundary.** An implementation **MAY** redact sensitive values from the
text sent to a model. When redaction is configured:

- Every configured pattern **MUST** be replaced, in the `systemPrompt` and `userPrompt` alike,
  with an opaque, reversible placeholder before the `GeneratorBroker` is called.
- Redaction **MUST** cover **every** model the agent drives, not only the Brain. The Gate
  (`ClassifierBroker`) screens the raw task and the Judge (`VerifierBroker`) reads the task and the
  drafted answer, so both see exactly the values redaction exists to hide. A guardian **MAY** run on
  a different host than the Brain, which makes an unredacted guardian a *wider* exposure than an
  unredacted Brain, not a narrower one. An implementation that redacts one model call and not the
  others does not satisfy this section.
- No model, and no remote host serving one, **MUST** receive the sensitive value in the clear.
- The mapping from placeholder to original value **MUST** stay inside the agent and **MUST NOT**
  cross to any model. A model's reply **MUST** be rehydrated, each placeholder restored to its
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

### 4.7 The Decision Log (Full, OPTIONAL capability)

The human-readable trace (`LogBroker`, §4.1) exists to be *read by a person during development*.
A **decision log** is a different artifact with a different reader: it is the machine-readable,
durable record of what the agent decided and why, written for an incident review, a regulator, or
a security pipeline. An implementation **MAY** offer one; when it does, the rules below apply,
because a log that can be silently lost is worse than no log — it invites reliance it cannot bear.

The sink is a broker (`AuditBroker`, §4.1): a file, an append-only store, a telemetry collector, a
SIEM. Which one is an implementation choice and **MUST** be swappable without changing anything
above it (§4.1).

**Durability.**
- The decision log **MUST** be append-only. An implementation **MUST NOT** truncate, clear,
  overwrite, or otherwise discard previously written records when a new run begins, when the agent
  is reconfigured, or when the process restarts.
- Beginning a run **MUST NOT** be a destructive operation on the sink. Where the trace (§4.1)
  defines `reset()`, that reset applies to the human-readable trace only; it **MUST NOT** propagate
  to the decision log.

**Attribution and order.**
- Every record **MUST** carry `runId`, `sequence`, `timestamp`, and `kind` (§3.5).
- Records from concurrent runs **MAY** interleave in the sink. Each record **MUST** remain
  attributable to exactly one run, and a reader **MUST** be able to reconstruct any run's records,
  in order, by selecting on `runId` and ordering by `sequence`.
- A record **MUST NOT** be written in a form that can be corrupted by a concurrent write to the
  same sink. Interleaving *between* records is permitted; interleaving *within* a record is not.
- When a `principal` is configured, it **MUST** be recorded on every record of that run, so the log
  answers *who* as well as *what*.

**Tamper-evidence.**
- An implementation **SHOULD** chain records: `previousHash` carries a cryptographic hash of the
  preceding record's canonical form. Given the chain, a reader **MUST** be able to detect that a
  record was altered or removed.
- Tamper-evidence is a detection control, not a prevention control. It does **NOT** replace the
  sink's own access controls.

**Neutrality.**
- The decision log **MUST** be side-effect-free with respect to the loop: enabling it, disabling
  it, changing its sink, or failing to write **MUST NOT** change any verdict, route, or result.
- Absent configuration, no decision log is emitted, no sink is touched, and behavior is exactly as
  if this section did not exist.

### 4.8 Capability Access — Local, External, Custom

Every capability is reached through the implementation's **host-facing composition surface** (the
builder, client, or equivalent — §9). This section is about that surface, and it is normative
because the reach of a capability is what decides whether the same agent definition survives
growth: a capability offered only one way forces a rewrite the day the host outgrows it.

For **each** capability it exposes — Skills, Memory, Knowledge, Brain, Gate, Judge, Tools, the
trace, and any capability the implementation adds — the surface **MUST** offer three modes:

| Mode | Satisfied by | Configured with |
|---|---|---|
| **Local** | the implementation's own package | a resource the host already has: a path, a folder, a literal rule |
| **External** | a provider outside that package | a broker (§4.1), supplied by the host |
| **Custom** | host-authored code | a function, where authoring a whole broker would be disproportionate |

Rules:

- **Local MUST NOT require an additional dependency**, and **MUST NOT** require a network. It is
  what makes the first line of an agent work with nothing installed.
- **External MUST** be reached through the capability's broker contract exactly as §4.1 defines it,
  and **MUST NOT** require modifying the implementation. This is the seam that lets a provider ship
  independently.
- **Custom MUST NOT** require the host to implement more than the capability's own contract; where
  a single function suffices, a function **MUST** be accepted.
- The three modes **MUST** be distinguishable by name at the surface, and the naming **SHOULD** be
  uniform across capabilities, so a host that has learned one capability has learned them all. A
  mode's name **MUST** describe *where the capability comes from*, never merely how it is passed —
  in particular, host-authored code is **Custom**, not Local, whatever the transport.
- Where a mode is genuinely meaningless for a capability, it **MAY** be omitted, and the omission
  **MUST** be documented with its reason. Silence is not an omission; it is a defect.
- A capability offered with fewer than three modes, absent such a documented reason, is
  **incomplete**, and an implementation **SHOULD** be able to detect that mechanically rather than
  by review.

### 4.9 The Perimeter (Full, OPTIONAL capabilities)

§4.6 hardens what the agent *sends*. This section hardens what the agent *does* and what it
*believes*. Each control is off unless configured, each is Data-driven, and none adds a nature —
Direction already owns the boundary, and these are enforcement at it.

**Effects.** When any control in this section is configured, a proposed tool call **MUST** be
carried as an `AgentEffect` (§3.3) rather than a bare name and payload, because authorization,
approval and run-once are all judgments about an act, and an act with no identity cannot be judged
the same way twice.

Direction **MUST** apply the following order, and **MUST NOT** reorder it:

1. **Authorize** — `PolicyBroker.authorize`. A denial is non-terminal: appended to `observations`
   with `status` `Working`, so the agent can choose a permitted path, exactly as §4.6 requires of
   the allow-list. The decision's reason **MUST** be recorded.
2. **Record the intent** — before the effect runs, never after. An effect that executed but was
   never recorded is indistinguishable from one that never ran.
3. **Approve, if required** — `ApprovalBroker.request`. `Pending` **MUST** stop the turn with
   `AwaitingApproval` (§3.1) and **MUST NOT** execute. `Denied` is non-terminal, like a denial.
   An act that was **not** approved **MUST** have its run-once claim released, because a held act
   is one that still has to be able to happen. An implementation that leaves the claim standing
   has built an approval that can only ever be granted too late: the authority says yes, the
   resumed run proposes the act, and the ledger reports it as already done.
4. **Execute at most once** — see below.
5. **Record the outcome** — before the loop advances, so a resumed or retried run can find it.

**Run-once.** An effect **MUST NOT** execute more than once for the same `idempotencyKey`.

- The key **MUST** be *derived* from the effect — at minimum the run, the tool name, and a
  canonical form of the arguments — and **MUST NOT** be supplied by the caller or by the Brain. A
  key the model can choose is a key the model can vary, and run-once becomes advisory.
- On a repeat key the implementation **MUST** return the first outcome and **MUST NOT** call the
  tool again.
- A claim with no outcome recorded against it means the act is **in flight**, which is
  deliberately not the same as *never happened*: a run that died between step 4 and step 5 left
  an effect in the world and no record of what it produced, and reporting that as "never
  happened" would run it again. It is also not the same as *finished*, so an implementation
  **MUST** distinguish the two rather than collapsing them.
- This matters most where it is least visible: retries (§9) and resumption both exist to run
  something *again*, so an implementation offering either **MUST** offer run-once for
  `Irreversible` effects, or it has built two ways to pay a wire transfer twice.

**Compensation.** Run-once makes an effect safe to *propose* twice. It does nothing for the effects
that cannot be made idempotent at all — a payment sent, a message delivered, a record filed with a
third party — where the only way back is a second, opposite act. An implementation **MAY** offer
compensation. When it does:

- A tool **MUST** be able to declare how it is undone, given the same arguments it was called with
  and the outcome it produced. Both are required: an undo that only knows the arguments cannot
  cancel the specific booking that was made.
- A tool that declares nothing **MUST** report that the effect stands, and **MUST NOT** be treated
  as having been undone. Silence is not reversal, and a run that reports itself cleanly unwound
  when it was not is worse than one that never offered compensation.
- Compensation **MUST** unwind in reverse order of execution. A later effect may depend on an
  earlier one, and undoing the booking before the payment that bought it leaves the payment
  attached to nothing.
- It **MUST** operate on what the run actually *performed* — not on what was proposed, not on what
  was authorized. An effect denied by policy, held for approval, or replayed from the ledger was
  never performed by this run and **MUST NOT** be compensated.
- Compensation is itself a set of effects and **MUST NOT** be exempt from the perimeter's record
  keeping: each attempt and its result **MUST** be recorded like any other act.
- A failed compensation **MUST NOT** stop the remaining ones. The unwind is best-effort per effect,
  and abandoning the rest because the first refused would leave more standing, not less.

**Untrusted inbound.** Text that entered as Data from outside the agent — a tool result, an
external call's response, a retrieved document — **MAY** be screened before it reaches the Brain.
When screening is configured:

- The screen **MUST** run before the text is appended to `observations`.
- A refusal **MUST** be non-terminal and **MUST NOT** silently drop the text; the agent is told the
  content was refused, so it can proceed differently.
- Screening **MUST** reuse the Gate (§4.2) rather than introduce a second guardian. Instructions
  arriving inside data are the same category of thing as instructions arriving in a prompt.

Absent every control in this section, behavior is exactly as if the section did not exist.

---

### 4.10 Resilience and Budget (Full, OPTIONAL capabilities)

An agent that cannot be stopped, cannot survive a transient failure, and cannot say what it spent
is not something an enterprise can operate — however correct each decision is.

**Cancellation.** `processPrompt` **SHOULD** accept a cancellation signal.

- When cancelled, the loop **MUST** stop at the next turn boundary at the latest, and **MUST NOT**
  begin a new turn.
- Cancellation **MUST NOT** be reported as success. A cancelled run's result is not an answer.
- An in-flight *effect* (§4.9) **MUST NOT** be abandoned half-recorded: its outcome is written
  before the loop notices the cancellation, or the effect never began.

**Retry.** A transient failure **MAY** be retried.

- What is retryable is decided by the **error's category, not its text**. A dependency failure is
  retryable; a validation failure never is, because retrying it will fail identically and only
  spends the budget.
- Retries **MUST** be bounded and **SHOULD** back off, with jitter where many agents share a
  provider.
- A retried call is still one *turn*: retrying **MUST NOT** consume the turn budget (§5), because
  a turn is a unit of the agent's reasoning, not of the network's luck.
- **Retrying an effect is subject to run-once** (§4.9). This is the point where retry and
  Invariant 7 meet, and where an implementation that added retries without a ledger has silently
  built a way to pay twice.

**Budget.** An implementation **MAY** bound what one prompt may consume — tokens, cost, or wall
clock.

- A budget **MUST** be checked between turns and **MUST** stop the loop when exhausted.
- Exhaustion **MUST** be reported distinguishably: it is not a refusal and not an answer. A caller
  that cannot tell "I will not" from "I ran out" cannot decide whether to retry.
- Where a provider reports usage (§3.4), a budget **MUST** be measured against reported usage
  rather than an estimate.
- `MAX_TURNS` (§5) is a budget of the same family and **MUST** continue to apply.

**Degradation before failure.** An implementation **MAY** track a provider's health and stop
calling one that is failing.

- When a provider is judged unhealthy, calls **SHOULD** be routed to a configured alternative
  rather than failed outright. A degraded answer is worth more than no answer.
- An implementation with no alternative configured **MUST** fail rather than pretend: silently
  returning an empty or fabricated result is worse than an error, because the caller cannot tell.
- Health tracking **MUST NOT** change any verdict or result while the primary is healthy.

**Guardian efficiency.** Screening the same unchanged input on every turn is waste, not safety.

- An implementation **MAY** screen a prompt once per prompt rather than once per turn, provided
  the input being screened has not changed.
- This **MUST NOT** weaken any guardian guarantee: untrusted inbound (§4.9) changes every turn and
  is therefore screened every time it appears.

### 4.11 Sessions and Conversation (Full, OPTIONAL capability)

Invariant 4 says the agent instance is stateless across prompts. It does **not** say the
*conversation* is. A session is that distinction made real: the continuity lives outside the agent,
in a broker, and is recalled into a prompt the same way memory and knowledge are.

Without one, every prompt starts from nothing. *"And what about Paris?"* has no idea what came
before, and `AwaitingInput` (§3.1) is a dead end — the agent asks a clarifying question and then
discards the context needed to use the answer. An implementation offering `AwaitingInput` or
`AwaitingApproval` without sessions has built a pause it cannot resume from.

```
record Session {
  id      : Text
  history : List<Turn>
  status  : AgentStatus      // what the session was left in the middle of
  pending : AgentEffect?     // an effect awaiting approval, if any (§4.9)
}
```

**Continuity.**
- When a `sessionId` is supplied, Recall **MUST** load that session's history into the context
  before Decision runs, so the Brain sees what was said before.
- History **MUST** be ordered oldest first and **MUST** be bounded — by the context budget (§4.2)
  or by its own limit. An unbounded history makes every prompt in a long conversation cost more
  than the last, without limit.
- A completed prompt **MUST** be appended to the session before the call returns, so the next
  prompt sees it. A prompt that failed or was cancelled **MUST NOT** be recorded as an answer.

**Resumption.**
- A session left in `AwaitingInput` or `AwaitingApproval` **MUST** be resumable: a later prompt
  carrying the same `sessionId` continues from that point rather than starting over.
- Resumption **MUST** work from a *different process*. The session is the state; the agent
  instance is not, and an implementation whose resumption depends on the original instance still
  being alive has not implemented this.
- A resumed effect **MUST** be subject to run-once (§4.9). Resumption exists to continue work
  that may have partly happened, which is precisely when running it twice is possible.

**Neutrality.** Absent a `sessionId`, behavior is exactly as if this section did not exist: no
session is loaded, none is written, and the agent is stateless prompt to prompt.

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
   memory **MUST** live outside the agent — recalled by Data, written by Direction. Because it
   holds no state between prompts, one instance **MUST** also serve prompts *concurrently*
   without runs observing or overwriting one another (§4.4).
5. **A draft is not a commitment.** Where a guardian applies, output **MUST NOT** cross a
   boundary un-vetted.
6. **No self-certification.** A guardian (Gate/Judge) **MUST NOT** be the Brain. A faculty
   **MUST NOT** certify its own trustworthiness. Guardian output that attempts to *answer* or
   *act* (rather than screen or score) **MUST** be neutralized — treated as a non-authoritative
   classification, never executed as the Brain's — and **SHOULD** be recorded. Neutralization
   **MUST** hold on every path by which a prompt can be processed; an implementation offering
   both a batched and a streamed loop **MUST** enforce it in both, since a guardian that can
   answer on one path is a guardian that can answer.
7. **Irreversible before, not after.** An irreversible action **MUST** be authorized before
   execution by a trusted guardian (deterministic rule and/or human), never after. And it
   **MUST** happen at most once: an implementation offering retries or resumption offers two
   mechanisms whose whole purpose is to run something again, so it **MUST** also offer run-once
   for irreversible effects (§4.9). Authorizing an act correctly and then performing it twice
   fails this invariant just as surely as never authorizing it. Where an act cannot be made
   idempotent, an implementation **MAY** offer compensation (§4.9) — and if it does, it **MUST**
   report the effects it could not undo rather than reporting the run cleanly unwound.
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
  machine-readable **decision log**, one structured record per event, to a separate sink for a SIEM
  or telemetry pipeline. The two layers are not interchangeable and have different readers: the
  trace is transient and may be reset; the decision log is durable and **MUST NOT** be
  (§4.7). Observability **MUST** be side-effect-free with respect to the loop: enabling it or
  changing its verbosity **MUST NOT** change any verdict, route, or result.
- **The composition surface is part of the contract.** §4.8 constrains how a host reaches a
  capability — Local, External, Custom — but not what the surface is called or how it is spelled. A
  fluent builder, a constructor with options, a configuration object, or a module of factory
  functions are all conformant, provided every capability is reachable all three ways and the mode
  names say where a capability comes from.

---

## 10. Conformance Checklist

- [ ] A conformance claim names its profile and its version, and its evidence is runnable by a
      third party (§1.1)
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
- [ ] **Full (optional):** a session carries conversation across prompts, is bounded, records only
      completed prompts, and resumes an AwaitingInput or AwaitingApproval pause from a DIFFERENT
      process; a resumed effect is subject to run-once (§3.2, §4.11, §4.9)
- [ ] One instance serves concurrent prompts; run identity and counters are per invocation, never
      per instance, so no run corrupts another's record (§4.4, §7.4)
- [ ] Reply protocol: first-line ACTION, terminal ReturnResponse / Refuse (§6)
- [ ] **Full:** guardians (Gate/Judge) distinct from the Brain; guardian output that answers or
      acts is neutralized, not obeyed (§7.6)
- [ ] **Full:** a rejecting Judge returns a reason, fed back into observations as revision
      feedback (§4.2, §4.3)
- [ ] **Full:** the Judge receives the task it is scoring against, not the candidate alone (§4.2)
- [ ] **Full:** guardian overreach is neutralized on every processing path, batched and streamed
      alike (§7.6)
- [ ] **Full:** irreversible actions authorized before execution (§7.7)
- [ ] **Full (optional):** a proposed act is carried as an AgentEffect, and Direction authorizes,
      records intent, approves, executes and records the outcome in that order (§3.3, §4.9)
- [ ] **Full (optional):** an effect executes at most once per derived idempotency key; the key is
      never supplied by the caller or the Brain (§4.9, §7.7)
- [ ] **Full (optional):** a Pending approval stops the turn with AwaitingApproval and executes
      nothing; a denial is non-terminal and carries its reason (§3.1, §4.9)
- [ ] **Full (optional):** untrusted inbound text is screened by the Gate before it reaches
      observations, and a refusal is non-terminal rather than silently dropped (§4.9)
- [ ] **Full (optional):** a cancelled run stops at the next turn boundary, is not reported as
      success, and leaves no effect half-recorded (§4.10)
- [ ] **Full (optional):** retries are bounded, chosen by error category rather than message, do
      not consume the turn budget, and are subject to run-once (§4.10, §7.7)
- [ ] **Full (optional):** a budget stops the loop between turns, is measured against reported
      usage rather than an estimate, and exhaustion is distinguishable from a refusal (§3.4, §4.10)
- [ ] **Full (optional):** an unhealthy provider degrades to a configured alternative rather than
      failing, and an implementation with no alternative fails rather than fabricates (§4.10)
- [ ] Retrieval ranks by relevance and returns the best rather than the first found; a natural
      question retrieves the passage that answers it (§4.2)
- [ ] **Full (optional):** what Recall injects is bounded across skills, memories and knowledge
      together, keeping the highest-ranked content; memory may be filtered by relevance and age (§4.2)
- [ ] **Full:** structured tool-calls — tools advertise name/description/parameters; `TOOL:`
      calls parsed from the first line; malformed calls recover, not crash (§6.1)
- [ ] **Full (optional):** boundary redaction hides configured values from **every** model —
      Brain, Gate and Judge alike — and rehydrates the reply; none sees them in the clear (§4.6)
- [ ] **Full (optional):** a tool allow-list denies disallowed tools at Direction before
      execution, non-terminally (§4.6)
- [ ] **Optional:** a guardian MAY be a deterministic rule; observability MAY add trace verbosity
      and a machine-readable decision log, both side-effect-free (§9)
- [ ] Every capability reachable Local, External and Custom; mode names say where the capability
      comes from; any omitted mode documented with its reason (§4.8)
- [ ] **Full (optional):** the decision log is append-only — beginning a run never discards prior
      records — and every record carries runId / sequence / timestamp / kind (§3.5, §4.7)
- [ ] **Full (optional):** concurrent runs stay attributable; a run is reconstructible by runId
      ordered by sequence; no record is corrupted by a concurrent write (§4.7)
- [ ] **Full (optional):** the decision log SHOULD be hash-chained so alteration or removal of a
      record is detectable (§4.7)

---

*Specification v0.1 — evolves with the theory. Reference implementation: `Standard.Agents` (C#).
Rationale: `THE-TRI-NATURE-OF-AGENT.md`.*
