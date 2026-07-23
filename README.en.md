# How do you take back control of AI-generated code?

> "You won't even have time to gasp before it starts typing all by itself."
>
> — Arkady and Boris Strugatsky, *Tale of the Troika*

*A word for readers who didn't grow up on Soviet science fiction. The Strugatsky brothers were
the genre's great masters east of the Iron Curtain — you may know them as the authors of
"Roadside Picnic," the novel behind Tarkovsky's* Stalker. *"Tale of the Troika" (1968) is their
satire about a commission of bureaucrats solemnly evaluating wonders they don't understand. One
of those wonders is relevant here.*

In *Tale of the Troika*, an inventor brings before the commission his heuristic machine — a
precision electro-mechanical device capable of answering any scientific or practical question.

On closer inspection, the device turns out to be a Remington typewriter, model 1906, fitted
with a wire, a neon bulb, and a fine new toggle switch.

The machine is not yet fully automated, so the questions are typed in by the inventor himself.
The answers — owing to the same unfortunate technical limitation — he also types himself. But
that is temporary. Just seat a man at the apparatus, hand him a few volumes of Brehm's *Life of
Animals*, and wait a little: the analyzer will analyze, the "thinker" will think, and soon
enough the machine will start typing on its own.

Nearly sixty years have passed.

The machine really did start typing on its own.

It writes functions, classes and tests. It builds APIs, lays a project out across files, fixes
compilation errors and cheerfully reports the task complete. Sometimes, in a few minutes, it
produces a volume of code that would once have cost a developer a week.

Except that, along with the middleman, something else quietly left the process.

We understand less and less why this particular class appeared in the project. Where this call
came from. Why the agent picked that dependency. Who allowed it to emit an extra event. On the
strength of which requirement it renamed a field, added a retry, or decided that the sixth file
out of seven wasn't really necessary.

Generated code can look convincing.

It can pass the formatter. It can compile. It can even pass the tests — especially if the tests
were written by the same model, from its own implementation.

And yet a gap opens between human intent and the system that results.

## The problem is not that AI writes bad code

AI often writes perfectly good local code.

Ask a model to implement a sort function, a DTO, a REST controller or a mapper — and the result
will most likely be acceptable. Such tasks have bounded context, a clear input and an easily
checked output.

The trouble begins when the agent is handed not a single function but **a piece of a real
software system**.

Say, a new integration module that must:

* live in a particular architectural layer;
* use only the permitted dependencies;
* consume a message from one specific Kafka topic;
* preserve per-key processing order;
* call one specific external API;
* never retry a non-idempotent operation;
* emit different events for different classes of failure;
* honor a wire schema that is already published;
* create every required file and test scenario;
* change nothing outside its declared boundary.

These rules are almost never gathered in one place.

One lives in an ADR. Another can be reconstructed from a neighboring module. A third exists
only inside a test. A fourth is remembered by a developer who happens to be offline today. A
fifth is expressed as a strange string literal with a typo that must never be fixed, because
external consumers already depend on it.

A person who has worked with the project for years can assemble these fragments in their head.

An LLM receives a handful of files, a task description, and a bounded context window.
Everything missing from that context, it is forced to fill in.

And it does so exactly the way we trained it to: by choosing the most plausible continuation.

For text, that is a useful property.

For the architecture of a payment system — not always.

## Plausibility is not provenance

Imagine the agent added a retry to a handler.

The change looks sensible: the external service is flaky, so the call should be retried. The
code is tidy, the test is green, the error handling is in place.

But the operation creates a financial transaction, and it is not idempotent.

Now a passing network hiccup can charge a customer twice.

Where did the mistake happen?

Did the model write incorrect code?

Not exactly. It applied a common and usually helpful pattern. The mistake happened earlier: the
agent was never given the constraint that forbade this pattern *right here*.

In the same way, a model can:

* invent a missing dependency;
* derive a topic name by analogy;
* "fix" a historical typo in a published value;
* generalize one module's behavior onto the whole family;
* write a test that confirms its own wrong guess;
* skip a file whose existence could not be inferred from the available context.

In every one of these cases the problem is not so much the quality of generation as the absence
of a verifiable chain:

**requirement → architectural rule → contract → implementation → test.**

We see the result, but we cannot reliably say why it has this exact shape.

Git will show which lines changed.

The chat history will sometimes show what the agent was asked.

But neither will prove which rule licensed the existence of a given line, which contract it
realizes, and which check confirms it is correct.

## The black box finally works

The Strugatskys' heuristic machine was funny because there was no "thinker" inside it at all.
The answers were typed by a human middleman, while the commission earnestly debated the
device's scientific merit.

Today the situation is almost exactly reversed.

There really is something inside. The machine really does answer questions and type its own
texts. More than that — it can build working software systems.

But to the observer it has become a black box again — a real one this time.

We hand it a task and receive thousands of lines of code. Between input and output lie
reasoning traces, probabilistic choices, dropped fragments of context and tacit assumptions
that cannot be reconstructed from the result alone.

So the central question of agentic development is no longer:

> Can AI write this code?

It obviously can.

What matters far more is:

> **How do you stay in control of a system when a large share of its code is written by an
> agent?**

Control here does not mean reviewing every line by hand.

That would surrender most of what generation buys you.

Control means that for any significant element of the system you can answer a few questions:

* Why does it exist?
* Which requirement gave rise to it?
* Which contract does it realize?
* Which dependencies and side effects is it allowed?
* Which test checks this exact rule?
* What else must change if the contract changes?
* Where did the agent *derive* — and where was it forced to *guess*?

If the system cannot answer these questions, we are not managing code generation. We are merely
observing its output.

What we need is a layer between human intent and generated code.

A layer the machine can read but is not free to embellish.

A layer that exists before the implementation, bounds the space of admissible solutions, ties
code to tests, and preserves the provenance of every change.

This project began as an attempt to build that layer.

## Contract before code

The beginning was modest: comments.

Not documentation "as it happens to come out," but strict machine-readable blocks right in the
source — before a function, before a class, before a file that does not exist yet:

```kotlin
//@contract: StartCaptureRoute.handle
//@  signature: fun handle(command: StartCapture): CaptureStarted
//@  pre: command.value > 0
//@  post: result.captureId == command.id
//@  calls: [telemetryApi.start]
//@  raises:
//@    PermanentException: provider rejects the capture
//@  assigns: []
```

At first glance this resembles JSDoc, or Eiffel's contracts — an idea pushing fifty. The
difference is the purpose. Classical contracts were written to check code that already existed.
These are written **before the code** — with one hard demand placed on the contract itself: the
block must be sufficient to regenerate an equivalent implementation without inventing anything.

That demand changes everything. If, to write the module, the agent would have to make up a
dependency, a topic or a side effect — the contract is incomplete. The bug is declared one
level earlier: add the missing key, don't improvise in the implementation.

And so a "comment" becomes a boundary. Everything the agent is allowed is enumerated.
Everything not enumerated is forbidden — not by a stern paragraph in the prompt, but by a
mechanical linter that treats any deviation as a violation.

The same move settles the question of tests — perhaps the most underrated question in agentic
development. The usual scheme goes: the model writes the code, then writes a test — and the
test faithfully confirms the code's behavior, bugs included. A closed loop: the examiner
studied under the examinee.

The contract layer breaks the loop with one simple rule: **test assertions are read off the
contract, never off the code**. `pre: command.value > 0` — there is your negative test.
`raises: PermanentException`, plus the declared "on this failure, emit the failed-event" —
there is your failure scenario, and it is mandatory, not optional. Code and tests become two
independent realizations of a single source of truth. For a bug to slip through now, two
mistakes must happen — in different directions, and in agreement with each other.

## Honesty over completeness

The hardest part of this scheme is not teaching the agent to follow contracts. The hardest part
is deciding what it must do when the contracts run out.

And somewhere, they always run out. A real system is richer than any description of it.

An ordinary agent, at that moment, does the very thing we love and fear it for: it fills the
gap plausibly. Ours is obliged to stop. The protocol is simple, almost bureaucratic — and that
is its strength:

1. Stop at this exact spot.
2. Write the gap into a ledger: "the contracts do not cover this."
3. Take the conservative reading — and cite what you leaned on.
4. Leave a marker in the code saying a decision was made here under uncertainty.

```
// TEMPLATE-GAP: helper missing from the template; inlined here
```

Boring? Extremely. But this boredom buys a wonderful thing: **classifiable divergence**. When
the adversarial review arrives after generation and finds the code differing from the
reference, every difference lands in exactly one of three bins: *predicted* (we declared the
deviation up front), *catalogued* (here is the ledger entry), or *candidate defect* (here are
the contracts that should have covered it — and didn't). That third bin is no longer "the AI
generated something weird." It is a work item with an address.

We ran this discipline through six full battle runs: a fresh agent session, a family of several
hundred files, the whole thing rebuilt from contracts alone, then an adversarial diff. The
count of root defect causes fell roughly threefold — and then flattened onto a plateau that
deserves its honest name, the text-iteration asymptote: past that point you are no longer
improving the template, only your luck. Nearly every rule in the current spec exists because
one specific run demonstrated one specific way to ruin everything.

## Asking the system "why"

Contracts, implementations, tests, gap ledgers, markers — these are all links. And links beg
for a graph.

So an index grew up next to the language: a deterministic SQLite database in which every
contract, file, test and recorded gap is a node, and "realizes," "covers," "explains,"
"produces to topic" are edges. An agent connects to it like to any other tool (over MCP) and
gains the ability not to grep text, but to ask.

My favorite question is `lynx_why`: why does this particular line of generated code exist?

The answer is not the model reasoning aloud. The answer is a path through the graph:

```
line of implementation
→ method handle
→ contract StartCaptureRoute.handle
→ rule "never retry non-idempotent calls"
→ requirement from the questionnaire
```

If the path exists, the line has provenance. If it doesn't, this is code the contract layer
knows nothing about — and such code has a name: drift. The `lynx_drift` tool lists it plainly:
methods without contracts, contracts nothing realizes, markers nothing explains. Not "something
diverged somewhere," but item by item, with addresses.

And because the database is deterministic — identical inputs produce a byte-identical index,
with not a single timestamp inside — every state can be kept as a snapshot and compared
**architecturally**, not line by line. The answer to "what changed this week" stops being "600
files touched" and becomes: three new event flows, one published vocabulary widened, one new
cross-layer dependency — and here are the teams it concerns.

## Narrow the rights, don't extend them

And the last piece — possibly the most important one where control is concerned.

Nearly the whole agentic-development industry is currently busy with the question "how do we
give the agent more": more tools, more context, more autonomy. At some point we realized that
for a working system the opposite question is more interesting: **how do you carefully narrow
the agent's rights, and make it explain its changes?**

The graph has exactly one write path. The agent cannot edit a contract directly. It can only
*propose* a change — and the proposal must carry a citation of its grounds, must pass every
mechanical check, and lands in a separate staging copy, touching neither the real contracts nor
the generated code. Accepting it remains a human act.

The usual scheme gets inverted. Not "the agent acts, the human catches up and reviews," but
"the agent proposes something provable, the human decides." Control comes back not by
forbidding the machine, but by a construction in which the machine simply has no way to change
the system silently.

## The machine types by itself. Now you can ask it why

In the Strugatskys' tale, the middleman was exposed — and rightly so: a man sat behind the
typewriter passing his own answers off as the machine's.

Our story ran in the opposite direction, and its moral is opposite too. The machine now
honestly types by itself — and the middleman, the one who knew *why* this particular thing got
typed, is the one we removed from the process. It turns out he wasn't only typing. He was
holding the causality: remembering which requirement gave birth to which line.

We can't sit him back down at the keyboard, and there is no need to. But his job can be brought
back — as a layer the machine reads but may not embellish: contract before code, tests read off
the contract, an honest ledger of gaps, a provenance graph, and a single, jealously guarded
write path.

Then the question "who controls the code?" gets a calm answer.

Still us. Only now — with proof.

---

*The layer this essay describes is assembled in the open: the contract annotation language
[LynxContract](https://github.com/dimaq12/lynxcontract) — the spec, a 21-recipe cookbook, a
deterministic graph index with MCP tools, an LSP for the editor, and agent workflows. Its first
specification is dated July 2025 and [published here](https://github.com/dimaq12/lambda/blob/main/docs/spec.lynx.md).
Everything exemplary in the repository is fictional — and the method carries to any stack.*

---

*Русская версия: [README.md](README.md)*
