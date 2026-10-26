# 1. Why your AI projects need a platform

Companion notes for **Chapter 1** of *Designing AI Systems* (Suhas Suresha and Dewang Sultania; Manning MEAP).

This chapter is the map. A prototype is a model call that wins a demo. Production is the sessions, budgets, guardrails, retrieval, traces, and deploy path around that call. Afterward you can name which service owns a production failure, and you can see why four teams with four session stores is sprawl rather than four clever choices. The model call is a leaf. The platform is the tree.

## 1. The model call is about 2%

```
                         +------------------ 98% ------------------+
                         | gateway, identity, quotas, deploy       |
                         | model adapters, routing, cache          |
                         | sessions + long-term memories           |
                         | indexes, chunk, embed, hybrid search    |
                         | tools, credentials, MCP, guardrails     |
                         | traces, scores, experiments             |
                         | workflows as services                   |
                         +-----------------------------------------+
                                           ^
                                           |  ~2%
                                      provider.chat()
```

* **The leaf**: `provider.chat()` is real and small. A diagram that is one box labeled "LLM" is the demo, not the system.
* **The 2015 version of the same point**: Sculley et al., "Hidden Technical Debt in Machine Learning Systems," argued that model code was a sliver of an ML system. A 2015-shaped support classifier loads weights, encodes a ticket, and takes an argmax. Maybe 5–10% of the repo is the model. The rest is pipelines, training cadence, serving, and monitoring.
* **What GenAI added**: Conversational state, token cost, tool isolation, retrieval, and judges. A modern support bot can be three lines to `chat.completions.create`. The other 98% is still the product: ingestion, safety, sessions, credentials, traces, routing, and deploy.

The invoice is not a quality report. Tokens, retries, and a chatty persona show up as dollars. None of those numbers say whether the answer was right.

## 2. Sam's production wake-up

The running story is Sam. Two weeks to a dazzling support-bot demo, then production. The failures are boring, and they repeat.

1. **Latency with no trace**: A request stalls. Sam cannot say whether the model, the retrieval, or a lock in his own code did it.
2. **Concurrency with no limit**: A marketing blast arrives. The laptop script has no queue, no cap, and no degradation path.
3. **Cost with no owner**: The invoice jumps. Nobody can name the feature or the tenant.
4. **Safety that lived in a paragraph**: Users ask for financial advice, attempt a jailbreak, or paste PII. A prompt-only policy loses.
5. **Knowledge that rots**: FAQs hardcoded for the demo are wrong the week legal updates the PDF.
6. **No experiment**: The next prompt change is a hope dressed as a release.

Each item already has an owner once you name the service: traces, limits, cost meters, guardrails, retrieval, evaluation. Sam did not hit a mysterious AI problem. He hit missing platform services.

* **Prototype**: Tolerable for five people. A date you can demo.
* **Production**: SLOs, concurrent users, audit, cost alerts, and a knowledge base that updates without a deploy. If you cannot fill that column, you are not ready to promise a date. You are ready to list the services you would have to invent.

### Watch for

* Calling the missing services "the app." Once they have names, "we will glue Redis again" is sprawl, not pragmatism.
* A notebook with a public URL treated as production. The wake-up list still applies.
* Buying a framework and skipping the names. The framework does not answer which team owns the session store.

## 3. AI sprawl

Sam's mess multiplies. Four teams, four session stores (Redis, Postgres, files), four ways of holding keys, four log formats.

* **The security question nobody can answer**: Which bots see customer data?
* **The learning that cannot spread**: A good prompt or a good chunk size stays inside the team that found it. The next feature takes months because navigating the maze is the work.
* **The cause**: Uncoordinated progress, not foolish engineers. Each team solved yesterday's demo. None of them shared a default path.
* **What a platform changes**: The secure, observable path becomes the one people take without a ticket. Shared session and cost code is how a policy and a lesson leave the team that discovered them. Teams still ship. They stop rediscovering the store.

## 4. What an application actually needs

LangChain, LlamaIndex, Bedrock, Azure OpenAI, and similar products are real. You still want the patterns, so you can debug, avoid accidental lock-in, and fill a gap. The platform in this book is a teaching implementation and a blueprint you can subset.

Test a design against five needs.

1. **Context-aware intelligence**: Organization knowledge and conversational state are two jobs. Stuffing both into one blob is how "as I mentioned yesterday" disappears into a PDF index.
2. **Orchestration**: More than one model or tool hop, with a lifecycle you can observe and interrupt.
3. **Safe action**: Tools with isolation and policy. A raw key in the prompt is a credential leak waiting for a jailbreak.
4. **Visible quality and cost**: Traces you can join to scores and to dollars.
5. **A developer path**: An SDK and one deploy story, so the fifth team does not invent a fifth HTTP wrapper.

"We use LangChain" names a library. It does not name who owns sessions, keys, or the trace. Sprawl survives a shared import.

## 5. The blueprint

One service per concern. Each can be deployed on its own. They share contracts, adapters, and telemetry.

```
  App / SDK  -->  API gateway  -->  Workflow (the actual assistant)
                      |                    |
                      |         +----------+-----------+
                      |         v          v           v
                      |     Model      Session        Data
                      |     Service    Service        Service
                      |         |          |            |
                      |         +------+---+------------+
                      |                v
                      |         Tools + Guardrails
                      |                v
                      +----->  Observability + Experimentation
```

* **Model Service**: Vendors behind one contract. Streaming, retries, routing, cache.
* **Session Service**: History and memories under a token budget.
* **Data Service**: Indexes and hybrid search.
* **Tools and guardrails**: Credentials, isolation, and policy checks.
* **Observability and experimentation**: Traces, scores, and comparisons.
* **Workflow Service**: The unit you deploy. The assistant is a service.
* **SDK and API**: How a developer feels all of the above without holding every knob.

You can buy a vendor platform. You cannot skip naming these capabilities. Skip the names and Sam's story repeats with better slides: another demo, another wake-up, another estate of four session stores.

## 6. One question, five owners

Maria asks, "What is the return policy for electronics?" Do not collapse that into `openai.chat()`.

1. **Gateway**: Authenticates Maria and starts a trace.
2. **Session**: Loads yesterday's laptop conversation.
3. **Data**: Retrieves the current policy chunk, not last year's demo FAQ.
4. **Model**: Generates with that context.
5. **Tools and guardrails**: If she asks to start a return, tools run under credentials the workflow never sees. Guardrails check the input, the output, and the tool arguments.
6. **Observability**: Records tokens, latency, and a quality score. Cost is attributed to the support product, not to a bucket labeled "misc AI."

A rate limit belongs on the gateway and on the workflow, not only inside a provider SDK. The provider limit protects the vendor. Your limit protects the tenant, the budget, and the rest of the estate when one feature loops.

Two rings a sklearn classifier did not have, and a GenAI assistant cannot skip: multi-turn session state, and tool credentials that must never enter the prompt. Retrieval that updates without a redeploy, and a judge attached to a trace, are the same kind of ring.

## Check yourself

1. List Sam's six production failures. For each, name the service in the blueprint that would have owned it.
2. Why is "we use LangChain" not an answer to sprawl by itself?
3. Explain the 2% / 98% split to a manager who only sees the model invoice. What on that invoice is not model quality?
4. A user says "as I mentioned yesterday, our enterprise discount." Which service should answer, and what goes wrong if that fact lives only in a vector index of PDFs?
5. Four teams, four session stores. Name one security question leadership cannot answer, and one lesson that cannot spread. How does a platform change both without blocking the teams?
6. Draw Maria's question through at least five components. Where does a rate limit go, and why not only in the provider SDK?
7. A colleague wants to productionize the notebook. What is the first non-model component you would demand, and which failure does it prevent?
8. Name two infrastructure rings a sklearn classifier did not have, that now dominate the design of a support assistant.
9. When would you not build an internal platform, and instead buy one or stay on a single script? Give a criterion from this chapter.
10. In one sentence, what does this chapter teach that a single `provider.chat()` call cannot?
