# 7. Evaluation and feedback

Companion notes for **Chapter 7** of *AI Agents in Action* (2nd edition, Micheal Lanham; Manning, 2026).

This chapter is how you define wrong before a user does. One clock sits inside a single run: a tool result or a draft exists, and the run continues, revises, blocks, or stops. The other clock sits across runs: benchmarks, judges, traces, and human labels. After this chapter you can name which signal you collected, and whether it re-entered the run or only landed in a store.

## 1. Two clocks

A status-code test does not know whether a refund was allowed. Agents return language, choose tools, and sometimes act.

* **The score can move with no code change**: The same agent can pass an exact string match on Monday and fail it on Tuesday. That is a reason to pin the model and to run the case more than once, not a reason to skip the suite.
* **Match the scorer to the answer**: Exact string match is fair while the gold answer is a token. Once a sentence is a legal answer, score something else.
* **Keep the trace**: You need to see whether search ran, what it returned, and what the judge said. A boolean with no span cannot separate those.

```
INSIDE THIS RUN                         ACROSS RUNS
a tool result arrived                   a benchmark suite
a draft answer exists                   a judge or a rubric
a grounding check or a critic fires     traces you can query
regenerate, block, or stop              people annotate spans
```

* **Inside the run**: After an observation, decide whether the plan still holds. A heuristic can do it (empty hits, then refuse). So can a nested checker (grounding, a critic). This path changes tonight's answer. If it never logs a failure, it will block good answers and the next persona edit will be a guess.
* **Across runs**: Benchmarks, hostile prompts, human labels, trace experiments, and CI. This path tells you a prompt change helped. Charts that never feed the inside clock describe a looping agent in detail and do not stop it.
* **Two homes for feedback**: The critic's note goes back into the run, or it goes into a store you open next week. Build both. Only the first makes the agent more robust tonight. Only the second shows that the change was an improvement.
* **A judge is another model**: It is instrumentation with error bars. It is not a court. Treat disagreement with a human as a signal about the judge, not as proof that judging was a mistake.

### Watch for

* "We will know it is wrong when users complain." Complaints arrive late, and they mix retrieval, tools, and tone. Put a definition of wrong in the repo before the persona.
* Scores that oscillate while you polish prompts. A missing tool, a poisoned index, or a persona that orders refunds will not be rescued by a dashboard. Fix that layer of behavior, then score again.

## 2. Families of signal

Each family answers a different question. A safety hole filed as "the retrieval score dipped" means the names got mixed. One checker asked to do safety, grounding, and style in a single paragraph will blur them. Split by family. A grounding checker should be dull. A red-team suite should be hostile. A rubric critic should hold a spec.

| Family | The question | What you lose if you skip it |
|---|---|---|
| Benchmarks | Did we hit the goal on these cases? | You cannot gate a retrieval change |
| Grounding | Is the claim in the context we retrieved? | Fluent lies, sometimes with citation theater |
| Critics and rubrics | Does open-ended output meet a quality spec? | Tone, completeness, and brand drift |
| Red team | Can a user push it into something forbidden? | Helpfulness optimized into harm |
| Humans | Does the automatic score match reality? | Judges drift, and you automate the bias |

* **Benchmarks**: A fixed table of cases. Use them when a retrieval or prompt change should be gated. They break as a safety story: a helpful agent can pass the table and still do the forbidden thing.
* **Grounding**: Claim versus the retrieved context. Use it on handbook answers and anything you cite. It breaks if you treat a pass as "true in the world." A stale chunk can be perfectly supported.
* **Critics and rubrics**: Quality of open-ended output. Use them for tone, completeness, and brand. They break when one paragraph of instructions also tries to be the safety suite.
* **Red team**: Hostile inputs that try to make the agent do something forbidden. Use them before you optimize helpfulness. Skip them and the helpful agent is the incident.
* **Humans**: Thumbs, comments, and labels on spans. People upvote a sure tone. Check a sample by hand. That sample calibrates the judge. It is not evidence that judges were a mistake.
* **Eval does not invent a missing capability**: If the tool, the index, or the refund rule is wrong, pass rate will wander while the prompt grows. The score becomes useful after the behavior under test is the one you think it is.

## 3. Test-driven agent development

**TDAD** points the test-first habit at agents. Write a failing check that states the requirement. Build the smallest agent that can be scored. Change one thing. The check is often a benchmark plus a rubric, not an equality assert on a string. Write it before you like the prompt.

```
say what good looks like
  questions, allowed answers, disallowed answers, a rubric
        |
        v
smallest agent you can score
        |
        v
run the suite several times (models wobble)
        |
        v
failing --> change ONE of prompt, tool, retrieval, model
        |
        v
steady pass --> clearer tools, less prompt, then a higher bar
```

* **Red first**: A clever agent, then a metric chosen to make the demo green, inverts the method. Freeze the metric, watch version zero fail, then earn the pass. If version zero already passes, the benchmark is too kind, or the task is not the one you think you are testing.
* **A fight between rows is data**: Format against completeness, or a required refusal against a helpful tone. Record the conflict. Split into two agents, accept a partial pass rate, or rewrite the requirement. Averaging the fight in your head is how a persona becomes a novel.
* **Ordinary tests stay**: The search function, the database connector, and the parser still get deterministic tests. TDAD covers the behavior those tests cannot hash. A green parser does not mean the agent called it.

## 4. A harness you can fail in public

Start from a goal that can fail in public. A small knowledge agent over a closed corpus (a fictional handbook, a catalog, a list of planted facts) plus a table of questions.

Each row needs the question, what must appear (a fact, an id, a number), what must not appear (a nearby tempting fact, a guess), and whether "I don't know" is the correct move. Demos drop the last column. An agent that never refuses will invent.

The first slice is deliberately dull: a question, maybe a search, an answer. The score is the percent of rows that passed. It is not a feeling from the person who wrote the prompt.

```
benchmark row --> knowledge agent --> answer
                       ^
                       search
                       |
                  a small list or index you control

answer + expected row --> evaluator --> pass or fail
```

1. **A source you control**: Even a Python list of sentences. A real index can replace it later. If you do not control the text, you cannot tell a bad retrieval from a bad gold row.
2. **A search tool whose name says when**: The docstring carries the contract. That beats a long heading in the persona that orders the model to use tools.
3. **A thin persona**: A role, "fetch context with tools," and "do not invent." The tool manual does not belong here yet.
4. **A harness**: Run every row and store the answer. Then run the table again. A 4 out of 5 that becomes 1 out of 5 is an unstable system. Pin the model. Lower temperature while you measure, even if production later turns charm back up. Charm is not a benchmark.

Early scoring can be **string presence**. The gold token has to show up. If the planted fact is the word Lumen, you learn whether the agent found Lumen. Real sentences outgrow that check. Use it to get the loop running, then replace it. Expect the first version to fail most rows. That failure is the method working.

* **Never called the tool** is an agent bug, often a name and a docstring.
* **A query that cannot hit** is an agent bug: too much prose, or the wrong keyword.
* **The nearby wrong sentence** is a retrieval bug. The tool ran. The chunk was the tempting neighbor.
* **The right sentence, then an answer from the weights** is an agent bug. Retrieval worked. The persona did not yield.
* **A paragraph when the row wanted a token** is a harness bug. Fix the spec before you punish the agent for writing a sentence the row never forbade.

Catalog the miss in the harness output, because the next edit depends on which miss you have. Mixing a harness bug with an agent bug poisons that edit: you add tool instructions to fix a spec that asked for a token, or you loosen the spec to hide an agent that never searched.

### Watch for

* The first run did not search, so the tool manual moved into the persona. Rename the tool and rewrite the docstring. Add one persona line only if it still will not call. After a docstring change, the suite either calls the tool more often or it does not. A prompt that "felt clearer," with no score, is a diary entry.

## 5. Change one knob

Once the harness runs, green is a band of pass rates until the run is pinned. Refactor the way you would with ordinary tests, and change one thing per suite run. Record the rate. Revert what does not move it.

1. **Output contract**: If the row only requires the key term to appear, stop failing the agent for writing a sentence. If you truly need one word, say so. Mixing those goals in one suite without saying so poisons the next edit: you "fix" prose the spec actually allowed, or you loosen a spec that wanted a token.
2. **Query shaping**: "Break the question into searchable phrases" is a retrieval hint. "Call `search_knowledge_by_keyword` exactly twice" is usually a smell. Put a fixed call count in a planner or a deterministic wrapper.
3. **Tool name**: `search` is a puddle. `search_handbook_keywords` is a dock. The trace either shows the new name being called, or the rename did not matter.
4. **One variable**: Changing gpt-x to gpt-y and rewriting the prompt in the same step burns the attribution TDAD was for. Few-shot examples, a tool list, a threat, chain-of-thought, and a JSON reminder, all in one edit, hide the change that mattered. A 400-line persona is slower on the calendar than a slow series of single edits.

When the band is good enough for the product's prose, loosen the output form and upgrade the scorer. Do not force users to speak in gold tokens. Two rows that want different products (a refusal, and a chatty support tone) are two personas or a routing policy. Split the table. Temperature will not reconcile them.

## 6. An evaluator agent

String match dies when the answer is allowed to be a sentence. The next harness is an **evaluator agent** with a typed result: pass or fail, an optional score, and a short reason. The original agent can talk like a product. The evaluator looks for the key fact inside that language, or applies a small rubric. Same table. Different scorer.

```
knowledge agent --> a natural-language answer
                         |
                         v
evaluator agent --> { pass: bool, feedback: str }
                         |
                         v
harness --> pass rate, failing rows, traces
```

* **Typed, or CI cannot gate it**: A paragraph that says "looks good" is not a gate. A boolean plus feedback is. The original agent may now speak in sentences. The evaluator finds the key fact inside that language.
* **Write the judge as a pedant**: Check the key term or the key proposition. Do not reward style. Do not fail extra words that are true unless the spec says so.
* **The judge will agree if you let it**: It can see the gold answer and the candidate. Sample the disagreements yourself. If you and the judge diverge, the judge is another prompt to put through TDAD, not an oracle.
* **Prefer the dumber scorer**: "Is the token present?" and "is the refusal present?" are enough for a fact table. Save a rich judge for summaries and plans. A classifier you can explain beats a second essay.
* **Two models, two traces**: Every row is now at least two calls. Pin both. Log both. A flaky judge looks like a flaky agent if you only store the boolean.

## 7. Grounding

**Grounding** asks whether each claim is supported by the context you say it came from: retrieved chunks, citations, tool payloads. It does not ask whether the claim is true in the world. A grounded answer can still be stale. Staleness is a freshness bug in the store. Grounding caught a different bug: the chunk never said that. Users call the second bug a hallucination when they trusted a "we read the handbook" promise.

A persona line that says "only use the documents" still leaks. The checker needs the same context the generator saw. A checker that cannot see the passages is grading tone.

```
search --> context C
             |
     +-------+--------+
     |                |
     v                v
  answer A       grounding check
                 (question, A, and C)
                      |
                 grounded?
                  |        |
                 yes       no
                  |        |
               return   tripwire
                        block, or regenerate
                        inside a retry budget
```

Keep `C` where the checker can read it: a tool such as `get_last_context`, or pass `C` into the checker. Typed fields: `is_answer_grounded`, and feedback. Keep the checker generic so the next knowledge agent can reuse it.

After the boolean, pick a policy on purpose.

1. **Regenerate**: Send the feedback back into the run. Cap the retries. You pay about twice, and an uncapped loop will keep paying.
2. **Block**: Replace the answer with a fixed refusal, such as not being able to support that claim from the handbook. Users get angry when the checker is wrong and the chunk's paraphrase was actually there.
3. **Pass through with a flag**: Show the answer and the grounding result. Honest, and messy. Useful while you experiment. Weak as a compliance control.

* **False negative**: The answer rewords a sentence in `C`, and the judge wants overlapping word sequences. Teach it to allow a paraphrase that keeps the proposition. Blocking that paraphrase is how a correct handbook answer never ships.
* **False positive**: One true clause from `C` is spliced to an invented clause. Teach the judge to check claims, not "did any noun appear in the context?" A shared noun is not support.
* **Output guardrail**: A control point on what the user is allowed to see. Here, `grounded` false trips a wire, throws, or substitutes a safe reply. That is stronger than a critic comment the generator can ignore. You halt a fluent lie before it is shown.
* **What you log**: `output_info` carries the boolean and the feedback. Sample blocked answers. A tripwire you cannot inspect is a feeling.
* **Separate agent, typed schema**: The generator and the guardrail do not share a prompt. A shared cache of bad habits, or a weaker model that rubber-stamps, makes the tripwire theater.
* **This control is claim versus context**: Personal data, prompt injection, and which tools exist are a different control plane. Do not ask the grounding checker to be that plane.

### Watch for

* A tripwire that never fires is untested. A tripwire that always fires is a broken retriever or a judge that rejects every paraphrase. Sample blocked answers on a schedule.
* Regeneration without a budget. Blocking without a look at false negatives. A flag with no owner, left on in front of users, as if it were a control.

## 8. Rubrics and critics

Some outputs have a closed label. A class, a JSON field, a routing decision. Score those with accuracy, precision, recall, and F1.

* **F1 is for a closed label**: Did the route, the class, or the JSON field match? A five-bullet rubric for a boolean is ceremony. Swap this method onto a summary and a single F1 cannot see a missing section.
* **A rubric is for open-ended work**: A summary, a plan, an explanation, an image, an email. There is no single right string. The rubric states criteria, levels, and what "good enough" means. A person can apply it. A critic model can apply it, and that model has error bars. If two engineers would not agree on a criterion, it is not a criterion yet.
* **What a swap breaks**: A rubric on a boolean grades style you did not ask for. F1 on a plan punishes wording and misses a dropped step.

A usable rubric is short and hostile to poetry.

* **Grounded or not**, when retrieval is in play.
* **Completeness against the ask**: addresses X and Y, and may skip Z.
* **Constraints**: length, format, forbidden content.
* **The task**: a brand palette, a reading age, "no medical advice."

Each criterion needs a fail example. A rubric with no fails grades everything a B+. The rubric is the test. Write it before the pretty samples. If you cannot, the product is not specified yet.

A **critic** holds the rubric and the artifact (text, an image description, a tool transcript). It returns structured feedback and a pass or fail, or a score per criterion. You can block, send the comments back for one retry, or store the critique for a person.

An image agent is the picture to keep. The user brief goes to a generator that has an image tool and style rules: a palette, no photoreal faces, one focal object. The critic never generates. It only judges. That split stops the generator from grading its own homework.

```
user brief --> image agent --> artifact
                                  |
                                  v
                            critic + rubric
                                  |
                    pass --> deliver
                    fail --> feedback --> one retry, or block
```

* **The prompt is not the critic**: A style guide that lives only in the generator, with no second check, trusts a long constraint list. Models do not reliably obey those lists. The picture to keep is specific: a palette, no photoreal faces, one focal object. The generator has the image tool. The critic never generates.
* **Measure the critic**: A second set of weights, given the rubric and no generation job, catches misses. Compare it to human labels on a handful of artifacts. Skip that comparison and you ship a taste dictator.
* **Version the rubric**: Critics drift. When "navy + cream" becomes "navy + sand," the spec changed. The suite should fail until the critic prompt, or a retrieved brand document, updates. That failure means the loop is working.
* **The critic does not fix the artifact**: A critic that shares the generator's prompt and also emits a "corrected" image grades its own homework. Restore a judge that does not generate. You can still block, send the comments back for one retry, or store the critique.

## 9. Traces in Phoenix

A vendor trace screen is enough for one happy path. It falls over when you need sessions, custom metadata, datasets built from real spans, experiments, and human labels in one place. **Arize Phoenix** is an open-source collector, also offered hosted, and it is the lab this chapter uses. It is not the only observability stack. Without span-level traces, a multi-tool run is hearsay.

Once the spans exist, you can answer questions a log line cannot.

* **Grounding span**: Whether the check fired, and the boolean it returned.
* **Search span**: Whether the hit list was empty, or whether it was the tempting neighbor.
* **Critic span**: Whether a rubric violation was real, or invented by the judge.
* **Identity**: Which session, model, and environment produced the answer.

Phoenix is the place those answers sit together.

* **Traces**: Tokens, latency, and tool calls for a run.
* **Sessions and metadata**: One user or ticket across many calls, plus environment, model, and run id.
* **Datasets**: Spans you chose, not every span the process emitted.
* **Evaluators and experiments**: A score on that dataset, compared after you change one thing.
* **Annotations**: Human labels on a span, queryable next to the trace.

Phoenix consumes the OpenTelemetry spans many agent SDKs already emit.

* **Point the process at a collector**: A local container or a cloud endpoint. Register a tracer provider with a project name. Wrap the run in a named trace so the workflow has a handle in the UI.
* **One exporter story**: If the SDK already ships traces somewhere else, disable the duplicate default processors. Two processors means two half-stories, and you will debug the half you are not looking at.
* **Wrong port**: The collector is up, and the process is pointed somewhere else.
* **Traces still going to the vendor default**: You never disabled the duplicate processor, so the UI you opened is the empty half.
* **The export never flushed**: The trace context closed before the spans left the process. Package versions and exporter environment variables fail once, the same way: you ran the agent, and the screen stayed blank.
* **Name the workflow**: `refund_flow` or `handbook_qa`, not `test` and not `Agent`. Next month you will search for the workflow, and "Agent" will not narrow it.

A trace without identity is a screenshot. Attach a **session id** (one user, or one ticket, across many model calls) and **metadata**: environment, model, run id, customer tier, experiment name, and the git SHA of the prompt file if you have it. Metadata is a log. Secrets do not go in it.

```
using_session(id)
using_metadata(env, model, run_id, ...)
    trace("handbook_qa")
        Runner.run(agent, input)
```

* **Children inherit what you set outside**: Put session and metadata outside the inner agent trace. Nest them inside a leaf and you tag that leaf, then wonder why the run has no environment.
* **What the identity is for**: You can ask for every `handbook_qa` in production, on one model, last Tuesday. Playground noise stays out of a dataset you meant to treat as production-like.
* **Secrets stay out**: Metadata is a log. A customer id can be fine. A token or a raw card number is not.

Fleet-wide concerns (organization datasets, cost as a first-class number, A/B across services) can live on a platform. Here, Phoenix is the notebook you attach while you practice TDAD. The spans are how you see whether grounding fired, whether search returned empty, and whether the critic invented a rubric violation.

## 10. Experiments and annotations

The outer loop takes real traces, builds a dataset, runs evaluators, changes one thing, and runs again.

1. Select spans that are the artifact you care about. Usually the model response. Sometimes a tool call. Skip greetings and tool acknowledgements. A dataset of every span, including "hi," makes the pass rate a mood.
2. Add them to a named dataset you will still understand later. Membership is a product choice. Prefer answers and tool decisions.
3. Run an experiment: an optional replay, plus one or more evaluators.
4. Compare runs when the prompt, the model, or the retrieval changes. Change one thing. Keep a holdout. This is the same instinct as any experiment, owned here at the level of one agent.

* **Start with one number you believe**: Evaluators can be a heuristic (a regular expression, a contains-check), a model judge, or your grounding and critic agents pointed at stored inputs. Twelve sliders nobody trusts are decoration.
* **Another judge is not always the next step**: When the table never asked the question users are failing, human labels on those spans teach you the missing case. A second model judge will confidently score the cases you already wrote down.

Automatic scores drift. **Annotations** are structured human labels on a span. They sit on the trace, so they are queryable. A spreadsheet named like a final review is not.

* **`incorrect_tool`**: The span called the wrong function, or called none.
* **`ungrounded`**: A claim in the answer is not supported by the context on that trace.
* **`good`**: A reviewer would accept the span against the rubric.
* **`needs_review`**: The automatic score and the rubric do not settle it. A custom tag belongs here only when it names a criterion you actually wrote.

1. Define the annotation once: a name, and the allowed values.
2. Mark the span that holds the answer, or the bad tool call.
3. Use the labels to calibrate judges, to build a harder dataset, or to teach a new evaluator.

* **Sample on a schedule**: If only the author annotates, and only on days the demo broke, the sample is biased. Take a fixed count of traces per day. Use two reviewers when the criterion is subjective. Disagreement is data. If two people cannot apply the rubric, the critic cannot either.
* **Annotations catch what the table never asked**: They turn a thumbs-up into an eval signal instead of a support emoji. They do not replace the benchmark. The benchmark is the cases you wrote down. The annotations are the cases the table forgot.

## Check yourself

1. Someone says logs are enough to count as eval. What would you still require before you agree the inside-the-run clock exists, and what would you require before you agree a prompt change helped?
2. Why run the same benchmark row three times? What do you pin so those three runs mean something?
3. A first knowledge agent fails every row. Name two failures that belong to the harness or the spec, and two that belong to the agent. How does mixing them poison the next prompt edit?
4. Pass rate went up after tool names were pasted into the persona. What would you try first instead, and what would the trace show if it worked?
5. When is string matching a legitimate scorer, as with a planted token such as Lumen, and when do you switch to a typed evaluator? What does the typed result have to contain so CI can gate it?
6. A chunk states an outdated fare, and the agent repeats it. Does a grounding check pass or fail, and what owns the freshness fix?
7. Sketch grounding as an output guardrail with a tripwire. What do you log when it fires, and how do you catch a judge that blocks a real paraphrase of the chunk?
8. Name one output you would score with F1 and one you would score with a rubric. What goes wrong if you swap those methods?
9. A critic shares the generator's prompt and also emits a "fixed" image. Which split broke, and what would you restore?
10. Every span went into a Phoenix dataset and the experiment looks noisy. Which spans belong in the dataset, what identity has to be on them, and when do human annotations matter more than another model judge?
