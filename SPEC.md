# The Standard for Agents — Specification
**Version 1.4**

A **normative, language-neutral** blueprint for building a Tri-Nature agent framework
in any language (JavaScript, .NET, Go, Rust, Python, …). The reference implementation
is `Standard.Agents` (C#). The rationale and philosophy live in the companion theory,
*The Tri-Nature of Agent* (`THE-TRI-NATURE-OF-AGENT.md`); this document is the buildable
contract.

Version 1.0 settled it. Everything below has been built at least once and certified against the
conformance vectors, so nothing here is a shape nobody has tried. The test of this document is not
whether it reads well: it is whether someone who has never seen the reference implementation can
build one that passes the same vectors. Where a requirement was reachable only by a mechanism the
text left unsaid, that mechanism is now said — §4.11's run continuity is the clearest example, and
it was found by building the thing rather than by rereading the prose.

**Version 1.1** closes the first gap found by that test being applied in the other direction —
not "can someone else build this?" but "does the reference implementation actually do what this
says?" It did not. §4.10's budget was enforced only where a provider volunteered its usage
numbers, which is to say not on the text protocol at all, and the document permitted that by
saying nothing about the case where there is none. A specification that is silent where an
implementer will guess has not settled the question; it has moved it. §3.4 and §4.10 now say it
outright, and the conformance vectors can express it, which they previously could not.

**Version 1.2** closes the same kind of gap in §4.9, found the same way. Run-once was correctly
specified and its *boundary* was never stated, so an implementer reading it would reasonably
conclude the guarantee is broader than it is. It is scoped to a run: a redelivered trigger starts a
new run, derives a new key, and performs the effect again. That is the correct behaviour — an agent
must be able to make a second payment when asked — but a caller who has not been told will discover
it as a duplicate transaction rather than as a design decision. The boundary is now written down,
along with the obligation it places on the caller. Nothing about the mechanism changed.

**Version 1.3** says what permission *is*. §4.9 could name a tool and nothing else, so "may write
files" could not be told from "may write files under /project" — an authorization model able to
describe a banking agent and not a file-touching one, which is most of the agents anyone is
building now. The effect now carries the **scope** it is about to touch, supplied by the tool
because only the tool knows what its own arguments mean. Two things follow that were previously
inexpressible: an allow-list can say *where*, and a **mode** can answer for the acts nothing named —
the only case that matters for an agent whose targets cannot be enumerated at composition. And one
`MUST NOT`, because the reference implementation got this wrong and nothing caught it: risk **must
not** be derived from whether an act requires approval, or the middle level is one the
implementation is structurally incapable of reaching.

**Version 1.4** adds two things the reference implementation built and this document had no words
for. §4.8 gains **composition from data**: the same surface the three modes define, offered as a
single structured document, so a platform that can produce JSON can define an agent — with the
one rule that makes it safe, that an entry the surface does not know **must** refuse to compose
with the entry named, because a control the host believes is on and is not is worse than an error.
And §4.12 names **telemetry** as a capability: observation of the loop's own boundaries, under the
same honesty obligations as everything else that reports — the recorded outcome is the truth the
caller was told, never a success the run did not earn.

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

**Readiness levels.** §8 defines two conformance profiles — Core and Full — and Full is a wide door:
it spans an agent with a Judge and an agent that can move money. An implementation **MAY** therefore
publish finer *readiness levels* within Full, and where it does, each level **MUST** name the
evidence it requires rather than describing itself in prose. A level defined by adjectives cannot be
failed. The reference implementation publishes four — Core, Reliable, Enterprise, Critical — as
machine-readable lists of required vectors, which is what makes `--profile Critical` answerable with
an exit code rather than an opinion.

Levels are the implementation's to name; the obligation this section imposes is only that a claimed
level be checkable by someone who does not trust the claimant.

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
  principal           : Principal?  // on whose behalf, when known
  toolName            : Text
  arguments           : Text
  idempotencyKey      : Text        // derived, not supplied — see §4.9
  riskLevel           : RiskLevel   // defaults to Safe
  approvalRequirement : Bool        // defaults to false
  scope               : Text        // what the act touches; "" when nothing addressable
}

record Principal {
  id           : Text
  tenantId     : Text?      // where one agent serves several tenants
  jurisdiction : Text?      // the regime this act falls under
  delegatedBy  : Text?      // who this principal is acting FOR, when delegated
}
```

- `riskLevel` **MUST** default to `Safe`, so an implementation that adopts effects without
  classifying its tools behaves exactly as it did before.
- `riskLevel` **MUST NOT** be derived from whether an act requires approval. They answer different
  questions — *how consequential is this* and *must a human permit it* — and an implementation that
  makes one the other's predicate can produce only the two extremes, leaving `Sensitive` a level it
  is structurally incapable of reaching. A tool **SHOULD** declare its own risk, because the tool is
  the only thing that knows what it does; a host **MAY** override that for tools it did not write,
  and the host's word wins, because the host is accountable for the deployment.
- `scope` is **what the act touches** — a path, a host, an account — and **MUST** be supplied by the
  tool rather than parsed out of `arguments` by the framework. Only the tool knows what its own
  arguments mean; a framework that parsed them would be guessing, and a host forced to parse them
  inside a policy delegate reinvents it once per deployment with nothing checking any of them.

> **Permission is what AND where.** "May write files" is not "may write files under /project", and
> an authorization model that can only name a tool cannot express the difference — which leaves an
> agent with hands permitted everywhere it is permitted anywhere. That distinction is the whole
> difference between a model that can describe a file-touching agent and one that can only describe
> a banking one.
- The principal **MUST** be carried on the effect itself, so it reaches the authorization decision
  and not merely the record written afterwards. An implementation that names the caller in its
  decision log while authorizing without them has not implemented identity-aware authorization; it
  has implemented identity-aware *reporting* (§4.9).
- It **MUST** be resolved per act rather than fixed at construction. One instance serves many
  callers (§4.4), so an identity captured once is the wrong identity for every prompt but the first.
- Every field beyond `id` is **OPTIONAL**, and an implementation offering only `id` is conformant.
  They exist so a host that knows more is not forced to encode it into a string. A policy is
  routinely written as *this principal, in this tenant, under this jurisdiction*, and a delegated
  act is a different question from the same service acting for itself.
- An implementation **MUST NOT** mint a principal. Establishing identity — tokens, sign-in,
  credential lifetime — is the host's; the framework consumes what it is given, and inventing an
  identity where none was supplied would make an authorization decision about a fiction.

### 3.4 Usage (OPTIONAL capability — §4.10)

What a model call actually cost. Reported by the provider where it can be, counted locally where
it cannot, and **always marked as one or the other** — because a bound *enforced* on an estimate
and a bound *reconciled* against an invoice are different claims.

```
record Usage {
  promptTokens     : Number
  completionTokens : Number
  isEstimated      : Boolean
}
```

- Where a provider reports usage, the reported value **MUST** be used, with `isEstimated` false.
  It is what the invoice will be drawn from.
- Where a provider reports nothing, an implementation **MUST** count what it sent and received
  rather than treat the call as free, and **MUST** set `isEstimated` true. Not every protocol
  volunteers its numbers; the ones that do not are not thereby unbounded (§4.10).
- An implementation **MUST NOT** present an estimate as a measurement. `isEstimated` is how that
  obligation is met in a form a caller, a trace, and an audit record can all read — it is not
  advisory metadata, and dropping it makes every downstream consumer guess.

> Version 1.0 required a reported value and said nothing about the case where there is none. That
> silence was read as permission to contribute zero, which made the bound in §4.10 inert on any
> text-only protocol while an implementation still conformed. Stated positively here.

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
- An entry **MAY** additionally constrain the effect's `scope` (§3.3). "May write files" is not
  "may write files under /project", and a list that can only name a tool leaves an agent permitted
  everywhere it is permitted anywhere.
- Absent an allow-list, every registered tool is runnable. The control is opt-in and its absence
  changes nothing.

**The disposition toward what nothing permitted.** Explicit permissions answer for the acts they
name. An implementation **MAY** offer a mode that answers for the rest.

- The default **MUST** be to permit, so an implementation adopting this changes nothing for an
  existing deployment.
- A mode **MUST NOT** override an explicit permission. An allow-list that names the act, or a
  policy that permitted it, has already answered; asking anyway would make the list meaningless.
- Where the mode is to **ask**, an unpermitted act **MUST** be treated exactly as one requiring
  approval (§4.9) — held, not failed, and non-terminal.

> This is the case an enumerated model cannot reach, and the reason it matters is not
> completeness. **An agent that touches files cannot have its targets listed at composition**, so
> the only acts that can be enumerated are the ones nobody was worried about. Before a disposition
> exists, everything unlisted is silently permitted — which is the correct default for an agent
> that only talks, and the wrong one for an agent with hands.

**Remembering a grant.** An implementation **MAY** remember that an authority permitted an act,
so the same question is not asked twice.

- A remembered grant **MUST** be scoped to the tool **and** the effect's `scope`. Approving a write
  to one file is not approving writes to every file, and a grant broader than what was asked for is
  a permission the authority never gave.
- It **MUST NOT** outlive the run unless the authority itself says otherwise. An approval broker
  that wants a longer grant already has the means: it answers the next request without asking
  anyone, which keeps the decision where the accountability is.

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

**Composition from data.** The surface above is code. An implementation **MAY** additionally
offer the same surface as a single structured document (JSON or equivalent) — one entry per
capability, named for the surface's own verbs — so that any platform able to produce the
document can define an agent without authoring code. An implementation offering this:

- **MUST** refuse to compose from a document containing an entry it does not know, or an entry of
  the wrong shape, and the refusal **MUST** name the entry. A misspelled budget silently ignored
  is a bound the host believes is holding and is not; the failure mode of configuration is never
  silence.
- **MUST** scope the document to what is data. A capability satisfied by host code — a tool, a
  Custom function, an External broker instance — is outside the document, and the implementation
  **MUST** allow code to compose alongside the document rather than forcing a choice between them.
  External tools reached by address (§4.2's remote tools) **MAY** appear in the document, because
  an address is data.
- **SHOULD** keep the document surface mechanically checkable against the capability surface, the
  same way the three modes are — a capability added to the code surface and absent from the
  document surface is the same invisible erosion §4.8 exists to prevent.

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

**The scope of run-once, stated because it is where implementers assume more than is offered.** The
key is derived from the run, so the guarantee has a boundary, and the boundary has three cases:

- **Within a run — protected.** A retry, or a Brain re-proposing the same act, derives the same key
  and is replayed rather than performed.
- **Across an interrupted run, resumed in the same session — protected**, by run continuity
  (§4.11). The resumed run keeps the interrupted run's identity, so the key still matches.
- **Across a *completed* run — NOT protected, and MUST NOT be.** A later run proposing an identical
  act derives a different key and **MUST** perform it. An agent asked to send a second reminder, or
  to make a second payment of the same amount to the same payee, has to be able to. A run-once that
  spanned completed runs would make the second request impossible to express.

It follows — and this is the part no implementer should have to infer — that **run-once does not
span triggers**. Where the thing that starts a run may start it more than once for the same work
(an at-least-once message bus, a webhook retry, a scheduler that fires twice, a user who submits
twice), a second delivery is a second run with a second key, and the effect **WILL** be performed
again.

- A caller whose delivery may repeat **MUST** deduplicate at the trigger boundary. An
  implementation **MUST NOT** be read as providing that, and **SHOULD** say so where it documents
  run-once.
- An implementation **MUST NOT** close this by admitting a caller-supplied key: that is the same
  door §4.9 already shuts, and it cannot be opened for a trusted caller without also opening it for
  an untrusted one, since nothing at the seam can tell them apart.

Delivery semantics belong to the host for the same reason identity does (§3.3): a framework that
invented them would be deciding something only the host can know. What the specification owes is
the boundary, not the mechanism.

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
- **A budget MUST bound the run on every protocol the implementation supports.** Where a provider
  reports no usage, the implementation **MUST** count (§3.4) rather than accrue zero. A bound that
  applies only where a provider volunteers its numbers is not a bound; it is a setting that
  happens to work on some deployments and silently does nothing on others, with no signal telling
  the operator which one they have.
- A cost bound is priced off the token count, so it **MUST** hold wherever the token count does.
  The two fail independently: an implementation can carry a count and still never price it.
- `MAX_TURNS` (§5) is a budget of the same family and **MUST** continue to apply.

> This is the requirement Version 1.0 left unsaid, and the cost of leaving it unsaid was not
> theoretical. The reference implementation set usage only on the native tool-calling path, where
> the provider volunteers it; on the text protocol every turn accrued zero, `maxTokens` and
> `maxCostUsd` never tripped, and it passed every conformance vector and claimed the Enterprise
> profile — which lists budgets by name — for eight releases. The vectors could not catch it
> because the only budget they could express was wall clock. Both gaps are closed together: the
> vectors `budget-bounds-tokens-on-any-protocol` and `budget-bounds-cost-on-any-protocol` now
> certify this requirement, and both were proven able to fail against the old behaviour.

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
  runId   : Text             // the run that last worked this session (see Run continuity)
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
- A session left awaiting approval **MUST** carry the effect it is waiting on, not merely the fact
  that something is waiting. A process that picks the session up has to be able to show an
  authority *what* it is permitting, and an approval that arrives cannot be checked against the act
  it was granted for unless that act's `idempotencyKey` (§4.9) travelled with the pause.

**Run continuity.** The requirement above — that a resumed effect is subject to run-once — is not
reachable unless the resumed run keeps the interrupted run's identity, because the idempotency key
is *derived* from the run (§4.9). A fresh run id produces a fresh key, the ledger recognises
nothing, and the act goes out a second time. Therefore:

- A session **MUST** record the run working it, and **MUST** record it when the run *starts*, not
  when it finishes. A crash means nothing at the end runs at all; an identity written only on
  success is one the failure case can never use. This write carries no answer — a prompt that has
  not been answered **MUST NOT** be recorded as though it had.
- A prompt continuing a session that **did not deliver** — anything other than a conclusion the
  caller received, so cancelled, exhausted, faulted, or awaiting — **MUST** adopt the recorded run
  identity rather than beginning a new run.
- A prompt continuing a session that *did* deliver **MUST** begin a new run. A finished conversation
  is not an interrupted one, and reusing its identity would make the next prompt's acts collide with
  the last prompt's ledger entries.
- An implementation whose run identity cannot outlive the process has not implemented resumption,
  whatever else it persists.

**Neutrality.** Absent a `sessionId`, behavior is exactly as if this section did not exist: no
session is loaded, none is written, and the agent is stateless prompt to prompt.

### 4.12 Telemetry (Full, OPTIONAL capability)

The trace (§4.7) narrates for a person and the decision log records for an auditor. Telemetry is
the third voice: spans and metrics for a collector, emitted at the loop's own boundaries — a run
opens a scope, each turn opens a scope inside it, each turn's usage is recorded, and the run's
outcome closes it.

An implementation offering telemetry:

- **MUST NOT** let it change what the agent decides or does. Telemetry observes the loop; a signal
  that can steer the run is a control and belongs to another section.
- **MUST** record the run's outcome as the same truth reported to the caller. A run that stopped
  at its turn cap is recorded as stopped mid-work; a run stopped by budget or cancellation is
  recorded as such — never as a success the run did not earn. Telemetry that flatters is worse
  than none, because someone will alert on it.
- **MUST** carry whether a usage number was reported by the provider or estimated locally (§3.4),
  so a dashboard never presents an estimate as a measurement.
- **SHOULD** name spans, attributes and metrics by the prevailing open convention for generative
  systems (at this writing, the OpenTelemetry GenAI semantic conventions), so a collector that
  already understands agents understands this one without translation.
- **SHOULD** cost nothing when nothing is observing. §4.8 applies in full: the Local mode **MUST
  NOT** require a dependency, which is achievable wherever the platform ships an instrumentation
  primitive in its standard library.

**Neutrality.** An agent composed without telemetry behaves exactly as if this section did not
exist.

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

### 6.2 Provider-native tool-calls (Full — MAY)

§6.1 keeps the call in the text and parses it, which is what makes it portable. This section is for
implementations that instead speak a provider's own tool-calling API — where the choice never
becomes text at all, and the provider returns it as typed data.

The reason to do so is not elegance. A model asked to emit `ACTION: calculator: 47*89` is imitating
a format; the same model emitting a tool call is doing what it was trained on. What the text
protocol cannot express *at any length* is **which call a result answers**, and that is the whole
subject of this section.

An implementation offering this **MUST NOT** replace §6.0. The text protocol remains the Core
contract, because it is the one that works against every endpoint — including the small local
models that follow a format more reliably than they emit schema-valid arguments.

The contract below is the seam, not the wire. Providers disagree about the bytes — one carries
the choice as `tool_calls` on a message, another as `tool_use` content blocks answered by
`tool_result` blocks with its own turn-alternation rules — and an implementation supporting more
than one **MUST** map each onto this one contract rather than surfacing a second contract per
provider. The loop above this seam never learns which wire shape answered.

**The exchange.** A native generation contract carries a conversation rather than two strings:

```
enum  MessageRole   { System, User, Assistant, Tool }

record Message {
  role       : MessageRole
  content    : Text
  toolCalls  : List<ToolCall>   // set when the assistant asked for calls
  toolCallId : Text?            // set on a Tool message — which call it answers
}

record ToolCall       { id : Text, name : Text, arguments : Text }   // arguments as JSON
record ToolDefinition { name : Text, description : Text, parameters : Text }
record Generation     { content : Text, toolCalls : List<ToolCall>, usage : Usage }
```

**Advertisement** is unchanged from §6.1: tools are offered as Data, a description is the opt-in,
and a tool without one is callable but not advertised. Presenting them as typed definitions rather
than as prose does not widen what the Brain may reach for.

**Attribution (MUST).** This is the requirement that distinguishes this section from §6.1:

- When a tool call has been performed, the next generation **MUST** include the assistant's request
  carrying that call's `id`, followed by a `Tool` message whose `toolCallId` is that same `id` and
  whose content is the result.
- **Every** call **MUST** be answered, whatever the outcome. A policy denial, a withheld result
  (§4.9) and a failure are answers; a call left unanswered strands the conversation, and providers
  are entitled to reject one whose tool call has no matching tool message.
- Narrating results back as prose instead — *"calculator: 4183"* in an assistant message — does
  **not** satisfy this. It leaves the model matching answers to questions by reading, which is the
  guessing native tool calling exists to remove.
- Ids are the provider's or the implementation's to mint, and are **not** the idempotency key of
  §4.9. A model may reuse or vary an id; run-once **MUST NOT** depend on one.

**One act per turn.** A provider may return several calls at once. Direction performs acts one at a
time, because authorization, approval and run-once are judgments about a *single* act (§4.9). An
implementation **MAY** carry the remainder forward, and **SHOULD** rely on the model re-proposing
them — run-once makes a repeat of an act already performed cost nothing.

**Nothing else changes.** Adopting native calls changes how a choice is *read*. Interpretation is
the only part that differs: the guardians, the perimeter, budgets, redaction (§4.6 applies to every
message going out and every reply coming back, exactly as on the text path) and the loop itself
**MUST** behave identically. An implementation where a control holds on one protocol and not the
other has not implemented this section; it has forked its agent.

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
- [ ] **Full (optional):** a session records the run working it, written when the run STARTS; a
      prompt continuing a session that did not deliver adopts that run identity, so a resumed act
      derives the same idempotency key and is replayed rather than performed twice (§4.11, §4.9)
- [ ] **Full (optional):** a session awaiting approval carries the effect it is waiting on, so the
      act can be shown to an authority and checked against the approval that arrives (§4.11)
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
- [ ] **Full (optional):** run-once holds within a run and across an interrupted run resumed in the
      same session, and an identical act proposed in a later *completed* run performs again — the
      boundary is stated rather than left to be assumed, and callers whose delivery may repeat are
      told they must deduplicate at the trigger (§4.9, §4.11)
- [ ] **Full (optional):** a Pending approval stops the turn with AwaitingApproval and executes
      nothing; a denial is non-terminal and carries its reason (§3.1, §4.9)
- [ ] **Full (optional):** untrusted inbound text is screened by the Gate before it reaches
      observations, and a refusal is non-terminal rather than silently dropped (§4.9)
- [ ] **Full (optional):** the principal reaches the authorization decision on the effect, resolved
      per act; a policy can refuse on identity alone; absent an identity the effect claims none
      rather than inventing one (§3.3, §4.9)
- [ ] **Full (optional):** an act that was denied or held has its run-once claim released, so an
      approval that arrives later can still be used; a claim with no outcome reads as in flight,
      which is neither "never happened" nor "finished" (§4.9)
- [ ] **Full (optional):** compensation unwinds only what the run performed, in reverse order, best
      effort per effect; a tool that declares no way back is reported as an effect that stands
      rather than counted as undone (§4.9, §7.7)
- [ ] **Full (optional):** a cancelled run stops at the next turn boundary, is not reported as
      success, and leaves no effect half-recorded (§4.10)
- [ ] **Full (optional):** retries are bounded, chosen by error category rather than message, do
      not consume the turn budget, and are subject to run-once (§4.10, §7.7)
- [ ] **Full (optional):** a budget stops the loop between turns, is measured against reported
      usage where there is any and a local count where there is none, bounds the run on **every**
      protocol rather than only the ones that volunteer their numbers, marks which of the two a
      number was, and reports exhaustion distinguishably from a refusal (§3.4, §4.10)
- [ ] **Full (optional):** an unhealthy provider degrades to a configured alternative rather than
      failing, and an implementation with no alternative fails rather than fabricates (§4.10)
- [ ] Retrieval ranks by relevance and returns the best rather than the first found; a natural
      question retrieves the passage that answers it (§4.2)
- [ ] **Full (optional):** what Recall injects is bounded across skills, memories and knowledge
      together, keeping the highest-ranked content; memory may be filtered by relevance and age (§4.2)
- [ ] **Full:** structured tool-calls — tools advertise name/description/parameters; `TOOL:`
      calls parsed from the first line; malformed calls recover, not crash (§6.1)
- [ ] **Full (optional):** provider-native tool-calls round-trip — the next generation carries the
      assistant's request with its call id and a Tool message answering that id; every call is
      answered, denials and withheld results included; the text protocol still works and every
      control behaves identically on both (§6.2)
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
- [ ] **Full (optional):** telemetry observes and never steers; the recorded outcome is the truth
      the caller was told; estimated usage never presents as reported (§4.12)
- [ ] **Optional:** composition from a data document refuses an unknown or wrong-shaped entry with
      the entry named; code composes alongside the document; the document surface is mechanically
      checkable against the capability surface (§4.8)

---

*Specification v0.1 — evolves with the theory. Reference implementation: `Standard.Agents` (C#).
Rationale: `THE-TRI-NATURE-OF-AGENT.md`.*
