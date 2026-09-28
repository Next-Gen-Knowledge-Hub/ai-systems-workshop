# AI Systems Workshop

A workshop for learning **how models are trained, how agents are built, and
how organizations platform both**: the difference between fitting a
classifier on a table and calling an LLM in a loop, the five layers that
make an agent more than a prompt, and the production services (model,
session, data, tools, guardrails, observability, workflows) that stop every
team from reinventing the same 98% of the stack.

We are guiding this workshop with three books. They are kept on **separate
tracks**. Each chapter note teaches only that chapter. It does not point
at the others.

| Track | Folder | Book |
|---|---|---|
| **Agents** | [`agents/`](./agents/) | Micheal Lanham, *AI Agents in Action*, 2nd edition (Manning, 2026). Subtitle: *Intelligent workflows with LLMs, MCP, A2A, and more*. |
| **Platform** | [`platform/`](./platform/) | Suhas Suresha and Dewang Sultania, *Designing AI Systems* (Manning MEAP, 9 chapters). Subtitle: *A guide to production-ready platforms*. |
| **ML** | [`ml/`](./ml/) | Aurélien Géron, *Hands-On Machine Learning with Scikit-Learn, Keras, and TensorFlow*, 2nd edition (O'Reilly, 2019). Subtitle: *Concepts, tools, and techniques to build intelligent systems*. |

Each numbered folder is one chapter of that book: the topics from the chapter,
rewritten as a human-friendly companion you can read after (or alongside) the
book.

The notes here are **original study material**. They do not reproduce the
books' text. Read the chapter, then use the matching folder to lock in the
vocabulary, the trade-offs, the failure mode, and one production example.
Keep the PDFs next to your other books; they are not committed in this git
repo.

The topic index for all three books lives in [`INDEX.md`](./INDEX.md).
Cross-cutting choices (agent vs flow, train vs call, MCP vs native tools,
truncation vs RAG memory, sync vs stream) live in
[`TRADEOFFS.md`](./TRADEOFFS.md).

## What this workshop assumes

You can write Python. For Agents and Platform you have called an LLM API
once. For ML you are willing to run notebooks against a table of numbers.
You do not need prior agent-framework experience, MCP, RAG internals,
platform design, or a stats degree — that is what the folders are for.
Where a chapter needs a concept, it teaches that concept in the note
itself. The topic index and the trade-offs sheet are separate maps if
you want to look a word up later.

## How the three books divide the work

Lanham's book is **how an agent is put together**: persona, tools, MCP,
multi-agent patterns, reasoning, memory/RAG, evaluation, deployment, the
agentic loop, and cognition. You finish it able to *build* an agent.

Suresha and Sultania's book is **how an organization stops rebuilding that
agent's plumbing**: a Model Service, Session Service, Data Service, Tool and
Guardrails services, observability and experiments, and a Workflow Service.
You finish it able to *platform* agents so the next team does not copy-paste
the same Redis session store.

Géron's book is **how a model is learned from data**: the landscape
(supervised / unsupervised / batch / online), an end-to-end sklearn
project, classifiers and their metrics, training linear models, trees and
ensembles, dimensionality reduction, clustering, then Keras/TensorFlow
networks, CNNs, RNNs, attention, autoencoders/GANs, reinforcement learning,
and serving those weights at scale. You finish it able to *train and
evaluate* a model. That is not the same job as wrapping `chat.completions`
in a loop, and it is not the same job as a Model Service that *calls* a
provider.

Read the ML track first if you have never fit a model and words like
"gradient, overfit, precision" are still fog. Read the Agents track first
if you have never built an agent. Read the Platform track first if you
already ship LLM features and keep hitting cost, memory, and sprawl. Any
order works; the index is the map. A shared word ("embedding", "agent",
"deploy", "evaluate") still means a different job in each book, and each
folder keeps its own chapter.

## This repository contains the following topics

### Track A — Agents (*AI Agents in Action*, 2e)

**Part A1 — What an agent is**

1. [The rise of AI agents](./agents/1-rise-of-ai-agents/) — AIA ch. 1
    - Agent vs assistant vs raw LLM; sense–plan–act–learn
    - MCP as the standard tool socket; five functional layers
    - Multi-agent shapes: flow, hub-and-spoke, collaboration
2. [LLMs, prompting, and agents](./agents/2-llms-prompting-agents/) — AIA ch. 2
    - Tokens, temperature, top-p; persona as the system prompt
    - A minimal OpenAI Agents SDK agent; tools and tracing
3. [Actions with MCP](./agents/3-mcp/) — AIA ch. 3
    - Clients, servers, tools/resources/prompts; STDIO vs SSE
    - Using and building MCP servers for agents

**Part A2 — Multi-agent, reasoning, memory**

4. [Multi-agent systems](./agents/4-multi-agent-systems/) — AIA ch. 4
    - Control patterns, shared memory vs messages vs MCP
    - Flows, handoffs, input/output guardrails
5. [Reasoning and planning](./agents/5-reasoning-and-planning/) — AIA ch. 5
    - Chain-of-thought, ReAct, tree-of-thought, Reflexion
    - Sequential thinking as an MCP server
6. [Memory and RAG](./agents/6-memory-and-rag/) — AIA ch. 6
    - Embeddings, vector search, hybrid RAG agents
    - Graph/hybrid memory over MCP; compression and forgetting

**Part A3 — Hardening and shipping**

7. [Evaluation and feedback](./agents/7-evaluation-and-feedback/) — AIA ch. 7
    - Test-driven agent development; grounding and critic agents
    - Phoenix traces, evaluators, annotations
8. [Deploying agents](./agents/8-deploying-agents/) — AIA ch. 8
    - Voice, API, Docker Compose; edge vs API vs event-driven
    - Reliability, cost routing, threat model, prompt injection
9. [The agentic loop](./agents/9-agentic-loop/) — AIA ch. 9
    - Inner SPAL loop, task loop, meta loop
    - Deep research agent; orchestration and collaboration loops
10. [Cognitive agents](./agents/10-cognitive-agents/) — AIA ch. 10
    - Cognition and metacognition as engineering, not metaphor
    - Workspace, attention, confidence gates, stagnation
11. [Field tips](./agents/11-field-tips/) — AIA ch. 11
    - Tips by the five layers; support, RAG, and research blueprints

Appendices: [sample code setup](./agents/appendix-a-sample-code/) (AIA A),
[Node.js for local MCP](./agents/appendix-b-nodejs-mcp/) (AIA B).

### Track B — Platform (*Designing AI Systems*, MEAP)

**Part B1 — Why a platform, and how you talk to it**

1. [Why your AI projects need a platform](./platform/1-why-a-platform/) — DAS ch. 1
    - Prototype vs production; AI sprawl; the model is ~2% of the system
    - Platform blueprint: the services the rest of the book builds
2. [SDK and API design](./platform/2-sdk-and-api/) — DAS ch. 2
    - Developer experience contract; code-to-container
    - Sync / stream / async APIs; gateway and SDK

**Part B2 — The core services**

3. [The Model Service](./platform/3-model-service/) — DAS ch. 3
    - Provider adapters, streaming, retries, fallbacks, routing, cache
4. [The Session Service](./platform/4-session-service/) — DAS ch. 4
    - Conversation history, storage backends, token-budget strategies
5. [The Data Service](./platform/5-data-service/) — DAS ch. 5
    - Indexes, ingestion, embeddings, hybrid search, RRF
6. [Tools and guardrails](./platform/6-tools-and-guardrails/) — DAS ch. 6
    - Tool registry, credentials, MCP on the platform, execution policies

**Part B3 — Seeing, shipping, assembling**

7. [Observability and experimentation](./platform/7-observability/) — DAS ch. 7
    - Traces, scores, cost; offline/online eval; A/B
8. [The Workflow Service](./platform/8-workflow-service/) — DAS ch. 8
    - Decorated functions as HTTP; jobs, health, composition, deploy
9. [Building an AI assistant](./platform/9-building-an-assistant/) — DAS ch. 9
    - Putting every service into one assistant (memory, RAG, tools, safety)

### Track C — ML (*Hands-On Machine Learning*, 2e)

**Part C1 — The fundamentals (sklearn)**

1. [The ML landscape](./ml/1-ml-landscape/) — HOML ch. 1
    - What "learning from data" means; why not hard-code the rules
    - Supervised / unsupervised / semisupervised / reinforcement
    - Batch vs online; instance-based vs model-based
    - Overfit, underfit, test/validation, data mismatch
2. [End-to-end ML project](./ml/2-end-to-end-project/) — HOML ch. 2
    - Frame the problem, pick a metric, hold out a test set
    - Explore, clean, pipelines, cross-validation, grid/random search
    - Launch, monitor, maintain a trained system
3. [Classification](./ml/3-classification/) — HOML ch. 3
    - Binary classifiers; accuracy is a trap
    - Confusion matrix, precision, recall, PR curve, ROC/AUC
    - Multiclass, multilabel, multioutput
4. [Training models](./ml/4-training-models/) — HOML ch. 4
    - Linear regression, the normal equation, gradient descent (batch / SGD / mini-batch)
    - Polynomial features, learning curves, ridge / lasso / elastic net
    - Logistic and softmax regression
5. [Support vector machines](./ml/5-svms/) — HOML ch. 5
    - Soft-margin classification; kernels (poly, RBF); SVM regression
6. [Decision trees](./ml/6-decision-trees/) — HOML ch. 6
    - CART, Gini vs entropy, regularization, tree regression, instability
7. [Ensembles and random forests](./ml/7-ensembles/) — HOML ch. 7
    - Voting, bagging/pasting, random forests, extra-trees
    - AdaBoost, gradient boosting, stacking
8. [Dimensionality reduction](./ml/8-dimensionality-reduction/) — HOML ch. 8
    - Curse of dimensionality; PCA, kernel PCA, LLE
9. [Unsupervised learning](./ml/9-unsupervised/) — HOML ch. 9
    - K-Means, DBSCAN; Gaussian mixtures; anomaly and novelty detection

**Part C2 — Neural nets and deep learning (Keras / TensorFlow)**

10. [ANNs with Keras](./ml/10-anns-keras/) — HOML ch. 10
    - Perceptrons, MLPs, backprop; Sequential / Functional / Subclassing APIs
    - Callbacks, TensorBoard, hyperparameter search
11. [Training deep nets](./ml/11-training-dnns/) — HOML ch. 11
    - Vanishing/exploding gradients; init, activations, batch-norm, clipping
    - Transfer learning; Adam and friends; dropout and other regularizers
12. [Custom models and training](./ml/12-custom-tf/) — HOML ch. 12
    - TF as a numpy-like runtime; custom losses, layers, training loops; Autograph
13. [Data and preprocessing in TF](./ml/13-data-and-preprocessing/) — HOML ch. 13
    - `tf.data`, TFRecord, one-hot vs embeddings as *model features*
14. [CNNs for vision](./ml/14-cnns/) — HOML ch. 14
    - Convolution, pooling; classic architectures; transfer; detection and segmentation
15. [Sequences with RNNs and CNNs](./ml/15-sequences-rnns/) — HOML ch. 15
    - Recurrent cells; forecasting; LSTM/GRU; 1D conv for sequences
16. [NLP with RNNs and attention](./ml/16-nlp-attention/) — HOML ch. 16
    - Char-RNN, sentiment, encoder–decoder, attention, the Transformer
17. [Autoencoders and GANs](./ml/17-autoencoders-gans/) — HOML ch. 17
    - Undercomplete and variational autoencoders; GAN training pathologies
18. [Reinforcement learning](./ml/18-reinforcement-learning/) — HOML ch. 18
    - Rewards, MDPs, Q-learning, DQN; policy gradients
19. [Scale and deploy](./ml/19-scale-and-deploy/) — HOML ch. 19
    - TensorFlow Serving; GPUs; data vs model parallelism

Appendix: [ML project checklist](./ml/appendix-b-project-checklist/) (HOML B).
Exercise solutions and the math appendices (C–G) stay in the book.

Cross-cutting: [topic index](./INDEX.md) · [trade-offs cheat sheet](./TRADEOFFS.md).

## How to use this workshop

1. Read the book chapter. The book is the source of truth.
2. Read the matching folder here. Headings follow the book's topics, so you
   can move between the two without losing your place.
3. Answer the **Check yourself** questions in your own words. A good answer
   has three parts: the takeaway, the failure mode it prevents, and one
   example from a system you have actually worked on.

Work through each track in order the first time. MCP (Agents 3) without the
five layers (Agents 1) is a protocol with nowhere to plug in. The Model
Service (Platform 3) without "why a platform" (Platform 1) looks like
needless abstraction. The assistant chapter (Platform 9) assumes every
service in 3–8 exists. Fine-tuning a Keras net (ML 11) without the test-set
discipline (ML 2–3) is how you ship an overfit demo.

## Try it locally

You do not need the full platform to learn the notes. For the Agents track,
the book's samples live at
[cxbxmxcx/AI-Agent-Workflows](https://github.com/cxbxmxcx/AI-Agent-Workflows)
and need Python, an OpenAI (or compatible) API key, and Node.js/`npx` for
local MCP servers — see [appendix A](./agents/appendix-a-sample-code/) and
[appendix B](./agents/appendix-b-nodejs-mcp/).

For the ML track, Géron's notebooks live at
[ageron/handson-ml2](https://github.com/ageron/handson-ml2)
(2nd edition; Python, scikit-learn, TensorFlow 2). A CPU is enough for
chapters 1–10. GPUs help from CNNs onward; they are not required to *read*
the notes. Pin a venv; do not mix the Agents companion repo with this one.

For the Platform track, think in services, not notebooks: one process per
workflow, a gateway in front, Postgres (or similar) for sessions and
pgvector for indexes. A scratch stack that matches the mental model:

```bash
docker run --name aisys-pg \
  -e POSTGRES_PASSWORD=secret \
  -p 5432:5432 -d pgvector/pgvector:pg16
```

Use that for session rows and hybrid search labs. Point model calls at
whatever provider you already have; the Model Service chapter is about the
*adapter*, not a particular vendor.

Feel free to use and make any change ;)
