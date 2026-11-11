# Building robust agents with evaluation and feedback

Evaluation and feedback are how robustness becomes something you can measure and then improve. They will not rescue a muddled design. They show you how the system actually behaves, and they give you a loop for changing it. Put that loop on with the first prompt. Waiting for production is how a user becomes your test suite.

Three checks cover three kinds of correctness. A deterministic test catches a schema, a tool choice, or a known answer. Another model judges language where quality is a matter of degree. A person catches tone, policy, and the edge the other two miss. Around those sit red-team probes for phrases that break the system, benchmarks for whether the goal was met, and ratings from users that need cleaning before they steer a change.

Feedback takes two roads. Inside the run, a critic or a grounding check can change the next action. Outside the run, results land in a store for regressions, dashboards, and a later signal you can learn from.

- **Test-driven agent development.** Write the benchmark first, and include an answer you already know is wrong. Expect a miss, and run the case more than once, because one sample lies. Then change the smallest thing that could pass: a word, a clause, a sentence, a section, a tool, and the model only after the cheaper edits fail. Watch the path as well as the ending. A tidy final sentence reached by a messy tool path still fails when the path was part of the test.
- **Grounding as a guardrail.** Hand a second agent the question, the answer, and the same retrieved context. It returns a typed verdict, supported or not, plus the reason. On a miss, retry with that feedback or block with a fixed reply, and cap the retries. After the ceiling, fail softly for a user and fail loudly for a pipeline. Grounding also covers tool results, memory, and citations, anywhere there is a source to defer to.
- **Critic and rubric agents.** Open-ended work, an image, a report, a plan, needs a purpose, criteria, a scale, and a description of each level. A critic scores against that rubric and sends the misses back until a pass or the attempt limit. Check the rubric on human judgment, or it will be steady and wrong. Give the judge a different model than the worker. Shared weights tend to bless the same mistake, and a green board then becomes false confidence.
- **Phoenix for traces, experiments, and annotations.** Arize Phoenix sits on OpenTelemetry and collects the spans the Agents SDK already emits: projects, sessions, latency, tokens, and cost. Attach metadata so a cohort can be filtered. Build a dataset from spans, run evaluators as experiments, and record human annotations on the spans. Feed those labels back into prompts, tools, and policy.

Because the sampler varies, track stability, latency, and token use alongside accuracy. When two benchmarks fight, choose on purpose: accept a partial target, split the job across specialists, or rewrite the instructions from a blank page.

Log the judges with the workers. An evaluator that passes a bad answer is a silent bug. Leave an agent unpromoted until this loop is actually running.
