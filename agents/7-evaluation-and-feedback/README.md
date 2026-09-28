# 7. Evaluation and feedback

Companion notes for **Chapter 7** of *AI Agents in Action* (2nd edition,
Micheal Lanham; Manning, 2026).

## The mental model

Two timescales, one temptation to merge them and then debug neither.

```
  IN THE LOOP                         AROUND THE LOOP
  (SPAL *learn*, this run)            (you, later, across runs)
  ------------------------            --------------------------
  tool result arrived                 benchmark suite
  draft answer exists                 LLM-as-judge / rubric
  grounding / critic fires            traces in Phoenix
  regenerate, block, or stop          humans annotate spans
                                      datasets, experiments
```

The one sentence to remember a year from now: **eval is how the agent
decides whether to continue, and how you decide whether the agent got
better.**

Two consequences. First, a unit test that asserts `status == 200` does
not know if the refund was allowed. Agents need tests against *behavior*:
answers, tool choices, grounding, rubric scores. Second, a judge model
is another stochastic component. Treat it as instrumentation with
error bars, not as a court of law.

```
  red team / safety probes
           \
  benchmarks (goal attainment)  -->  traces + scores  -->  you change
           /                            ^                    ONE thing
  human thumbs / comments               |
  grounding / critic / eval agents  ----+
```

Feedback either **re-enters the run** (critic tells the agent to try
again) or **lands in a store** (you look next week). Design both paths.
Only the first path makes the agent robust *tonight*. Only the second
path tells you the prompt change helped.

## Why agents need evaluation and feedback

"We'll know it's wrong when users complain" puts the definition of
*wrong* after the damage. Put a definition of *wrong* in the repo before
the persona. Users complain late, loudly, and about a mix of retrieval,
tools, and tone you cannot disentangle after the fact.

Ordinary software tests a function against a return value. Agents
return language, choose tools, and sometimes act. The same agent can
pass a string-equal test on Monday and fail it on Tuesday with no
code change. That is not a reason to skip tests. It is a reason to:

- pin models when you measure,
- run the same case **more than once**,
- score with something other than exact string match as soon as the
  answer is allowed to be a sentence,
- keep a **trace** so you can see whether search ran, what it
  returned, and what the judge said.

### Internal vs external

Internal evaluation is the Learn beat: after an observation, is the
plan still valid? That can be a heuristic (empty hits → refuse) or a
nested agent (grounding, critic). External evaluation is everything
that is *about* the agent rather than *inside* its loop: benchmark
tables, red-team prompts, humans, Phoenix experiments, CI.

You need both. An in-loop grounding check that never logs its
failures will quietly block good answers and you will "fix" the
persona at random. An around-the-loop dashboard that never feeds
the Learn beat will produce beautiful charts of a looping agent.

### Four families of signal

They answer different questions. Mixing the names is how a safety
hole gets filed as "the RAG score dipped."

| Family | Question it answers | Failure if you skip it |
|---|---|---|
| Benchmarks | Did we hit the goal on *these* cases? | You cannot CI a retrieval change |
| Grounding | Is the claim in the context we retrieved? | Fluent lies with citations theater |
| Critics / rubrics | Does open-ended output meet *quality*? | Brand, tone, completeness rot |
| Red team | Can a user make it do something forbidden? | You optimized helpfulness into harm |
| Humans | Does the automatic score match reality? | Judges drift; you automate the bias |

Human feedback (thumbs, comments) is necessary and biased. People
upvote confident tone. Verify a sample. That verification is how
you calibrate judges, not a sign that judges were a mistake.

One "eval agent" asked to do safety, grounding, and style in a single
paragraph of instructions will blur all five families. Split tools and
agents by family. A grounding agent should be almost boring. A red-team
suite should be mean. A rubric critic should hold a spec.

Evaluation will not rescue a missing tool, a poisoned index, or a
persona that orders refunds. If the architecture is wrong, scores
will oscillate while you polish prompts. Fix the layer, then score
again.

## Test-driven agent development (TDAD)

Test-driven development in ordinary code: write a failing test that
states a requirement, write the smallest code that passes, refactor.
Teams skip it under deadline. The *habit* still pays. **TDAD** is
that habit aimed at agents.

The "test" is often a **benchmark plus a rubric**, not `assertEqual`.
You still write it *before* you fall in love with a prompt.

```
  say what good looks like
  (questions, allowed answers, disallowed answers, rubric)
            |
            v
  smallest agent that can be scored
  (persona stub, maybe one tool)
            |
            v
  run the suite  (N times — models wobble)
            |
            v
      fail? --> change ONE of: prompt, tool, retrieval, model
            |
            v
      pass steadily --> refactor (clearer tools, less prompt)
                    --> raise the bar (language, grounding, CI)
```

Building a clever agent, then hunting for a metric that makes the demo
look green, inverts the method. Freeze the metric first, watch it go
red, then earn the green. Initial failure is the point. If version zero
already passes, the benchmark is too kind or the task is not the one you
think.

When two benchmarks fight (format vs completeness, refuse vs
helpful), do not average them in your head. Record the conflict.
Then pick: split into two agents, accept a partial pass rate, or
rewrite the requirement. Silent averaging is how personas become
novels.

### TDAD in practice

Start with a **goal you can fail in public**. Workshop shape: a
tiny knowledge agent over a closed corpus — a fictional handbook,
a catalog, a set of planted facts — plus a table of questions.

Each row should include:

- the question,
- what must appear (key fact, id, number),
- what must *not* appear (a tempting nearby fact, a guess),
- whether "I don't know" is the correct move.

The last row is the one demos omit. An agent that never refuses
will invent.

Architecture for the first slice is deliberately dumb: user
question → maybe search → answer. An evaluator scores against the
table. Accuracy on the table is "% of rows that passed," not a
vibes score from the author of the prompt.

```
  [benchmark row] --> [RAG agent] --> answer
                           ^
                           |
                      search tool
                           |
                      tiny knowledge list / index

  answer + expected --> [evaluator] --> pass/fail (+ later, feedback)
```

Run the table **several times** before you celebrate. A 4/5 that
becomes 1/5 on rerun is not "pretty good." It is an unstable
system. Pin the model. Lower temperature for measurement even if
you raise it later for production charm. Charm is not a benchmark.

Write ordinary TDD for the *boring* pieces: the search function,
the DB connector, the parser. TDAD does not replace pytest for
deterministic code. It covers the parts pytest cannot hash.

### Coding and testing a RAG agent

Minimum viable loop:

1. A knowledge source you control (even a Python list of
   sentences — you can swap in a real retrieval stack later).
2. A tool that searches it. Name and docstring should tell the
   model *when* to call it. That beats a prompt chapter titled
   "Tools You Must Use."
3. A persona that is allowed to be thin: role + "use tools to
   fetch context" + "do not invent."
4. A harness that fires each benchmark row and records the
   answer.

Early evaluation can be **string presence**: the key token must
show up. That is crude and useful. It tells you whether the agent
found "Lumen" when the gold fact is the word Lumen. It will not
survive real sentences. Use it to get the loop running, then
replace it.

Expect version one to fail most rows. Typical failure modes,
worth cataloging in the harness output:

- never called the tool,
- called the tool with a query that cannot hit (too much prose,
  wrong keyword),
- retrieved the nearby wrong sentence,
- retrieved the right sentence and still answered from weights,
- answered with a paragraph when the harness wanted a token
  (harness problem — fix the spec, not only the agent).

Encoding the entire tool manual in the persona because the first run
did not search usually makes the next edit harder. Rename the tool and
rewrite the docstring. If the model still will not call it, *then* add
one line to the persona. Prompt clutter is how TDAD turns into prompt
archaeology.

The persona is role and constraints; tools carry their own contracts.
TDAD makes the rule testable: after a docstring change, the suite
either calls the tool more often or it does not. A prompt change that
"felt clearer" without a score is a diary entry.

### Refactoring the agent

Once the harness runs, refactor like you would any green tests —
except green is a **pass rate band**, not a boolean, until you
pin enough.

Order of cheap moves:

1. **Output contract** — If the benchmark is "the key term
   appears," stop punishing the agent for writing a sentence, *or*
   explicitly demand a one-word form if you truly need it. Do not
   mix those goals in one suite without saying so.
2. **Query shaping** — "Break the question into searchable
   phrases" is a retrieval hint. It is allowed. "Call
   `search_knowledge_by_keyword` exactly twice" is usually a
   smell; put that in a planner or a deterministic wrapper.
3. **Tool rename** — `search` is a puddle. `search_handbook_keywords`
   is a dock.
4. **Model pin** — Changing gpt-x to gpt-y *and* the prompt in
   one step burns the attribution TDAD was for.

After failures, adding every idea to the prompt — few-shot, tool
list, threat of punishment, chain-of-thought, JSON reminder — hides
which change mattered. One change per run of the suite. Record pass
rate. Revert changes that do not move the rate. This is slow in wall
clock and fast in calendar time compared with a 400-line persona
nobody will edit.

When the suite is green enough, *loosen* the output form if
product needs prose, and **upgrade the evaluator** rather than
forcing users to speak in gold tokens. That is the next section,
not a failure of TDAD.

Conflicting rows: one question wants a refusal, another wants a
guessy customer-support tone. That is not a refactor of
temperature. That is two personas or a routing policy. Split the
table.

### An agent evaluator

String match dies when answers are sentences. The next harness
is an **evaluator agent** with a **typed** result: pass/fail,
optionally a score, always a short reason.

The original agent can now speak like a product. The evaluator
looks for the key fact *inside* that language, or applies a
small rubric. Same benchmark table. Different scorer.

```
  RAG agent  -->  natural language answer
                      |
                      v
  evaluator agent  -->  { pass: bool, feedback: str }
                      |
                      v
  harness aggregates  -->  pass rate, failing rows, traces
```

Typed output matters. A paragraph that says "looks good" is not
something CI can gate. A boolean plus feedback is.

The evaluator that sees the gold answer and the candidate will
"helpfully" agree if you let it. Write evaluator instructions like
a pedant: check for the key term or the key proposition; do not
reward style; do not fail for extra true words unless the spec says
so. Sample disagreements by hand. If the judge and you diverge, the
judge is another prompt to TDAD, not an oracle.

Keep the evaluator **dumber than the agent** when you can. A
classifier "is the token present / is the refusal present" is
often enough for RAG fact tables. Save rich judges for summaries
and plans.

Cost: every row is now two model calls (or more). Pin both
models. Log both traces. A flaky judge looks like a flaky agent
if you only store the boolean.

## Grounding, critic, and evaluation agents

Three patterns, three jobs. Using one name for all three is how
a style critic starts blocking true answers because the citation
format was ugly.

```
  GROUNDING     "Is each claim supported by the retrieved context?"
                Best home: RAG, policy Q&A, anything citable.

  CRITIC        "Does this output meet a rubric (style, completeness,
                safety-as-spec)?"  Best home: generation, images,
                long-form, brand.

  EVALUATOR     "Typed score for a suite or a complex artifact."
                Best home: CI, benchmarks, multi-criterion reports.
```

A grounding agent can *act* like a critic (send feedback, demand
regeneration) or like a guardrail (block). An evaluation agent
can wrap either. Start from the **question you need answered**,
then pick the pattern.

### Grounding

Grounding means: the answer is **supported by the context you
claim it came from** — retrieved chunks, citations, tool
payloads — not by the model's prior. It is not "the answer is
true in the world." A grounded answer can still be stale if the
chunk is stale. That is a freshness bug in the retrieval store.
Grounding still caught "the chunk never said that."

Ungrounded is the failure mode users call hallucination when
they trusted your "ask the handbook" banner.

A persona that says "only use the documents" and stops there
still leaks. Give the checker the **same context** the generator
saw. Checkers that cannot see the passages are grading vibes.

Grounding is reusable. A critic can demand sources. A support
agent can ground in ticket fields. The RAG case is the cleanest
teaching example, not the only one.

### Grounding a RAG agent

Wire it so cheating is obvious:

```
  search  -->  context C
                |
      +---------+---------+
      |                   |
      v                   v
  RAG agent            grounding agent
  (question + C)       (question + answer + C)
      |                   |
      v                   v
    answer A            { grounded: bool, feedback }
```

Implementation sketch, without becoming a listing: keep `C` where
the checker can fetch it (a tool `get_last_context`, or pass it
in the checker input). Typed output: `is_answer_grounded`,
`feedback`. Keep the grounding agent **generic** so the next
knowledge agent can reuse it.

After the check you choose a policy:

- **Regenerate** — send feedback in-loop; cap retries.
- **Block** — static "I can't support that from the handbook."
- **Pass through with a flag** — show the user the answer *and*
  the grounding result (honest, sometimes messy).

None of these is free. Regeneration doubles cost. Blocking
angers users if the checker is a false negative (paraphrase the
chunk did contain). Passing through with a flag is an
experimentation tool more than a compliance tool.

False negatives: the answer rewords a sentence in `C` and the
judge wants n-gram overlap. Teach the judge to allow paraphrase
that preserves the proposition. False positives: the answer
splices one true clause from `C` with a invented clause. Teach
the judge to check **claims**, not "was any noun in the
context."

### Grounding as a guardrail

Guardrails are control points on agent output. Here the **output
guardrail** is a grounding agent: if `grounded` is false, trip a
wire, throw, or substitute a safe reply.

That is stronger than a critic comment the generator can ignore.
It is also how you halt a fluent lie before the user sees it.

```
  RAG agent produces AnswerResult
            |
            v
  output guardrail runs grounding agent
            |
      grounded? --yes-->  return answer
            |
           no
            |
            v
      tripwire: block or regenerate (with a retry budget)
```

When the guardrail and generator share a prompt cache of bad
habits, or the guardrail uses a weaker model that rubber-stamps,
the tripwire is theater. Separate agent, typed schema, logged
`output_info` (the boolean and the feedback). Sample blocked
answers weekly. A tripwire that never fires is untested. A
tripwire that always fires is a broken retriever or a sadistic
judge.

Platform-shaped policy (PII, injection, which tools exist) is a
different control plane. This chapter's guardrail is **claim vs
context**.

### Rubrics

Not every output wants a rubric. If the agent emits a label, a
JSON field, or a routing decision, use **accuracy, precision,
recall, F1**. Inventing a five-bullet rubric for a boolean is
ceremony.

Open-ended text (summaries, plans, explanations, images, emails)
does not have one right string. Quality is multi-axis. A
**rubric** is the spec: criteria, levels, what "good enough"
means. Humans can apply it. So can a critic model, with the
caveats already named.

Defaulting every eval to "LLM-as-judge with a vibe" skips the
split. Score the structured parts with numbers. Score the prose
with a rubric. If a criterion cannot be written so two engineers
would agree, it is not a criterion yet.

A usable rubric is short, operational, and hostile to poetry:

- Grounded / not (if retrieval is in play).
- Completeness vs the user ask (addresses X, Y; may skip Z).
- Constraints (length, format, forbidden content).
- Task-specific (brand palette, reading age, "no medical
  advice").

Each criterion needs a **fail example**. Rubrics without fails
grade everything a B+.

TDAD meets rubrics: the rubric *is* the test. Write it before
the pretty samples. If you cannot, you do not know the product.

### A rubric critic

A critic agent holds the rubric and the artifact (text, image
description, tool transcript). It returns structured feedback
and a pass/fail (or per-criterion scores). You can:

- block on fail (guardrail),
- feed comments back for one retry,
- store the critique for humans.

Workshop picture: an image-generation agent with **style
rules** (palette, no photoreal faces, one focal object). The
generator is one agent with an image tool. The critic never
generates; it only judges against the rubric. That split keeps
the generator from grading its own homework.

```
  user brief --> image agent --> artifact
                                    |
                                    v
                              critic + rubric
                                    |
                     pass --> deliver
                     fail --> feedback --> retry or block
```

Putting the style guide only in the generator prompt and skipping
the critic because "the model read it" trusts long constraint lists
that models do not reliably obey. A second pair of weights, with a
rubric and no generation job, catches misses. Measure the critic
against human labels on a handful of artifacts or you will ship a
taste dictator.

Critics have taste drift too. Version the rubric in git.
When marketing changes "navy + cream" to "navy + sand," that
is a spec change; the suite should fail until the critic
prompt (or a retrieved brand doc) updates. That failure is
TDAD working.

## Phoenix for evaluation and feedback

SDK dashboards (vendor trace UIs) are enough for a single
happy path. They fall over when you need **sessions**, custom
metadata, datasets built from real spans, experiments, and
human annotations in one place. **Arize Phoenix** is an open
source (and hosted) collector in that role. It is not the only
observability stack. It is the one this chapter uses as the
agent-level lab.

Without span-level traces, multi-tool and multi-agent runs are
hearsay. You cannot say whether grounding fired, whether search
returned empty, or whether the critic hallucinated a rubric
violation.

```
  agent run (OpenTelemetry spans)
            |
            v
         Phoenix
            |-- traces, tokens, latency, tool calls
            |-- sessions + metadata (env, customer, model)
            |-- datasets (chosen spans)
            |-- evaluators / experiments
            |-- annotations (human labels)
```

Fleet-wide observability — org datasets, cost as a first-class
signal, A/B across services — is a platform concern. Phoenix here
is **the notebook you attach to the Agents SDK (or similar) while
you practice TDAD**.

### Connecting

Phoenix speaks the same OpenTelemetry language many agent
SDKs already emit. Typical shape:

- run a collector (local container or cloud),
- point the process at the collector endpoint,
- **disable duplicate default processors** if the SDK already
  ships traces somewhere else — two processors means two
  half-stories,
- register a tracer provider with a **project name**,
- wrap the run in a named `trace(...)` so the workflow has a
  handle in the UI.

Pain you should expect once: package versions, exporter
env vars, "I ran the agent but the UI is empty." The empty UI
is almost always: wrong port, traces still going to the vendor
default, or the trace context closed before the export flushed.

Name traces after **workflows**, not after "test." Next month
you will grep `refund_flow` and not `Agent`.

### Metadata and session tracking

A trace without identity is a screenshot. Attach:

- **session id** — one user (or one ticket) across multiple
  model calls,
- **metadata** — `env`, `model`, `run_id`, customer tier,
  experiment name, git SHA of the prompt file if you have it.

Put session and metadata **outside** the inner agent trace so
child spans inherit them. Nesting the other way is a common
footgun: you think you tagged the run and only tagged a leaf.

```
  using_session(id)
  using_metadata({env, model, run_id, ...})
      └── trace("handbook_qa")
              └── Runner.run(agent, input)
```

This is how you later ask Phoenix "all `handbook_qa` in prod
on model X last Tuesday" instead of scrolling. It is also how
you avoid mixing local playground noise into a dataset you
meant to be production-like.

Do not put secrets in metadata. Metadata is a log. Treat it
like one.

### Evaluators and experiments

The **outer loop** of TDAD: real traces become a **dataset**,
you run **evaluators** on that dataset, you change the agent,
you run again.

Sketch in the UI (the buttons will move; the idea will not):

1. Select spans that represent the artifact you care about
   (usually the LLM response, sometimes a tool call).
2. Add them to a dataset (give it a name you will still
   understand).
3. Run an experiment: a task (optional replay) plus one or
   more evaluators.
4. Compare runs when you change prompt, model, or retrieval.

Evaluators can be cheap heuristics (regex, contains), LLM
judges, or your grounding/critic agents pointed at stored
inputs. Start with one number you believe. A dashboard with
twelve unevaluated sliders is decoration.

Evaluating every span including "hi" and tool acks makes the
pass rate look like a mood ring. Dataset membership is a product
choice. Prefer spans that are *answers* or *tool decisions*.

This experiment loop is the agent-level cousin of platform
experimentation. Same scientific instinct — change one thing,
keep a holdout — different owner.

### Annotations

Automatic scores drift. **Annotations** are structured human
labels on a span: `incorrect_tool`, `ungrounded`, `good`,
`needs_review`, or a custom tag that matches your rubric.
They live next to the trace, which means they are queryable
instead of living in a spreadsheet named `final_review2`.

Workflow:

1. Define the annotation (name, possible values) once.
2. On the trace UI, mark the span that holds the answer (or
   the bad tool call).
3. Use those labels to calibrate judges, to build a harder
   dataset, or to teach a new evaluator.

Only the original author annotating, and only on days the demo
broke, leaves you with a biased sample. A thin rule: sample N
traces per day, two reviewers when the criterion is subjective.
Disagreement is data. If two humans cannot apply the rubric, the
critic cannot either.

Annotations are how thumbs become a training and eval signal
instead of a support ticket emoji. They do not replace
benchmarks. They catch what the table never asked.

## Check yourself

1. A stakeholder says "we have logs, so we have eval." What
   extra artifacts would you require before you agree the
   *Learn* beat exists, and what extra artifacts before you
   agree *you* can tell if a prompt change helped?
2. Why run the same benchmark row three times in TDAD? What
   do you pin so those three runs mean something?
3. Version-zero RAG agent fails 5/5. List two failures that
   are harness/spec bugs and two that are agent bugs. How
   does mixing them poison the next prompt edit?
4. You improved pass rate by stuffing tool names into the
   persona. What would you try first instead, and how would
   the trace show that it worked?
5. When is string matching a legitimate evaluator, and when
   must you switch to a typed evaluator agent? Give one
   example each from a system you know.
6. Grounding vs truth: a chunk says an outdated fare. The
   agent repeats it. Does a grounding agent pass or fail, and
   what owns the freshness fix?
7. Sketch output-guardrail grounding with a tripwire. What
   do you log when it fires, and how do you catch a judge
   that blocks paraphrases of a real chunk?
8. Name one agent output you would score with F1 and one you
   would score with a rubric. What goes wrong if you swap
   those methods?
9. A critic shares the generator's prompt and also generates
   a "fixed" image. Which TDAD rule did you break, and what
   split would you restore?
10. You dumped every span into a Phoenix dataset and the
    experiment looks noisy. How do you choose spans, what
    metadata must be on them, and when do annotations matter
    more than another LLM judge?
