# Topic index

Three books, three tracks. This page is the **index of topics as the books
organize them**, plus a cross-book map so you do not merge chapters that
happen to share a word.

- **AIA** = *AI Agents in Action*, 2e (Lanham) → [`agents/`](./agents/)
- **DAS** = *Designing AI Systems*, MEAP (Suresha & Sultania) → [`platform/`](./platform/)
- **HOML** = *Hands-On Machine Learning*, 2e (Géron) → [`ml/`](./ml/)

When a term appears in more than one book, pick the job you actually have:

- Agents folder — *how the agent uses it*
- Platform folder — *how the organization serves it*
- ML folder — *how a model is trained and measured from data*

Follow the link you need; do not collapse the folders into one note.

---

## Cross-book map (same word, different job)

Agents vs Platform (unchanged jobs):

| Topic | Agents track (how you build it) | Platform track (how you serve it) |
|---|---|---|
| Why agents / why a platform | [A1 five layers](./agents/1-rise-of-ai-agents/) | [P1 iceberg and sprawl](./platform/1-why-a-platform/) |
| Prompts / persona / context engineering | [A2 persona](./agents/2-llms-prompting-agents/), [A11](./agents/11-field-tips/) | [P9 context assembly](./platform/9-building-an-assistant/) |
| Models, routing, cost | [A2 sampling](./agents/2-llms-prompting-agents/), [A8 routing](./agents/8-deploying-agents/) | [P3 Model Service](./platform/3-model-service/) |
| Tools | [A2 tools](./agents/2-llms-prompting-agents/), [A3 MCP](./agents/3-mcp/) | [P6 Tool Service](./platform/6-tools-and-guardrails/) |
| MCP | [A3 protocol, STDIO/SSE, servers](./agents/3-mcp/) | [P6 MCP as platform interoperability](./platform/6-tools-and-guardrails/) |
| Guardrails | [A4 input/output/agent-as-guardrail](./agents/4-multi-agent-systems/), [A7 grounding](./agents/7-evaluation-and-feedback/) | [P6 input/output/behavioral policies](./platform/6-tools-and-guardrails/) |
| Multi-agent / workflows | [A4 flow, hub, team](./agents/4-multi-agent-systems/), [A9 loops](./agents/9-agentic-loop/) | [P8 Workflow Service](./platform/8-workflow-service/) |
| Memory (conversation) | [A6 memory forms, MCP memory](./agents/6-memory-and-rag/) | [P4 Session Service, token budget](./platform/4-session-service/) |
| Knowledge / RAG | [A6 vector + hybrid RAG agent](./agents/6-memory-and-rag/) | [P5 Data Service, indexes, RRF](./platform/5-data-service/) |
| Evaluation | [A7 TDAD, critic, Phoenix](./agents/7-evaluation-and-feedback/) | [P7 scores, datasets, A/B](./platform/7-observability/) |
| Observability | [A2 tracing, A7 Phoenix, A8](./agents/8-deploying-agents/) | [P7 traces, logs, cost](./platform/7-observability/) |
| Deployment | [A8 Docker, runtimes, threat model](./agents/8-deploying-agents/) | [P2 SDK/API, P8 deploy pipeline](./platform/8-workflow-service/) |
| End-to-end assistant | [A11 support / RAG / research](./agents/11-field-tips/) | [P9 Claw, the assembled assistant](./platform/9-building-an-assistant/) |

Where Géron shares a *word* with the other two, the job is still
**training and measuring a model**. Do not paste these rows into an Agents
or Platform note.

| Topic | ML track (how you train it) | Do not confuse with |
|---|---|---|
| What "learning" is | [M1 landscape](./ml/1-ml-landscape/) (fit parameters from a dataset) | [A1 agency](./agents/1-rise-of-ai-agents/) (a runtime that chooses tools) |
| End-to-end project / launch | [M2 California housing pipeline](./ml/2-end-to-end-project/), [M-B checklist](./ml/appendix-b-project-checklist/) | [P1 sprawl](./platform/1-why-a-platform/), [P9 assistant](./platform/9-building-an-assistant/) |
| Evaluation metrics | [M3 precision/recall/ROC](./ml/3-classification/), hold-out + CV in [M2](./ml/2-end-to-end-project/) | [A7 traces and judges](./agents/7-evaluation-and-feedback/), [P7 scores/A/B](./platform/7-observability/) |
| "The model" | Weights you fit ([M4](./ml/4-training-models/)–[M19](./ml/19-scale-and-deploy/)) | [P3](./platform/3-model-service/) adapter to a *provider API* |
| Embeddings | Learned lookup tables inside a net ([M13](./ml/13-data-and-preprocessing/), [M16](./ml/16-nlp-attention/)) | Retrieval vectors in [A6](./agents/6-memory-and-rag/) / [P5](./platform/5-data-service/) |
| Attention / Transformer | Architecture you train ([M16](./ml/16-nlp-attention/)) | Sampling an already-trained LLM ([A2](./agents/2-llms-prompting-agents/)) |
| Agent (RL) | Policy that maximises reward ([M18](./ml/18-reinforcement-learning/)) | LLM sense–plan–act–learn ([A1](./agents/1-rise-of-ai-agents/), [A9](./agents/9-agentic-loop/)) |
| Deploy / serve | TF Serving of *your* graph ([M19](./ml/19-scale-and-deploy/)) | [A8](./agents/8-deploying-agents/) agent runtime, [P3](./platform/3-model-service/) provider gateway |
| Experimentation | Grid/random search, learning curves ([M2](./ml/2-end-to-end-project/), [M10](./ml/10-anns-keras/)) | Platform A/B and annotation queues ([P7](./platform/7-observability/)) |

---

## Track A — *AI Agents in Action* (by chapter)

### AIA 1 — The rise of AI agents
[`agents/1-rise-of-ai-agents`](./agents/1-rise-of-ai-agents/)

- Agent, assistant, LLM (direct chat) as three interaction patterns
- Agency; sense–plan–act–learn (SPAL)
- Tools as registered functions (JSON schema, failures)
- Model Context Protocol (introduction only)
- Five layers: persona · tools/actions · reasoning/planning · knowledge/memory · evaluation/feedback
- Multi-agent: assembly-line **flow**, **hub-and-spoke** orchestration, **collaboration** (teams)

### AIA 2 — Core components: LLMs, prompting, agents
[`agents/2-llms-prompting-agents`](./agents/2-llms-prompting-agents/)

- LLM as probabilistic token machine; tokens
- Temperature, top-p, and related sampling knobs
- Prompt engineering as the **persona** layer
- OpenAI Agents SDK: minimal agent, model params, typed I/O, tracing
- Tool integration and tracing tool calls

### AIA 3 — Actions with MCP
[`agents/3-mcp`](./agents/3-mcp/)

- Why a standard (N×M tool integrations)
- Architecture: host / client / server; tools, resources, prompts
- Deployment patterns; how MCP feeds the five layers
- Writing a server, MCP Inspector, transports (STDIO, SSE)
- Desktop MCP vs agent MCP
- Agents consuming local/remote servers; converting tools into a server

### AIA 4 — Multi-agent systems
[`agents/4-multi-agent-systems`](./agents/4-multi-agent-systems/)

- Decision-making and control patterns
- Communication: shared memory, message passing, MCP
- Coordination strategies
- Agent flows vs autonomous agents; A2A flows; agency inside a flow
- Handoffs: implementing, visualizing, monitoring
- Guardrails: input, output, agents-as-guardrails, pass-off flows

### AIA 5 — Reasoning and planning
[`agents/5-reasoning-and-planning`](./agents/5-reasoning-and-planning/)

- Chain-of-thought; ReAct (reason–act–observe); planning with LLMs
- Applying CoT and ReAct to agents
- Tree-of-thought; Reflexion; choosing a pattern
- Sequential thinking MCP server

### AIA 6 — Memory and knowledge RAG
[`agents/6-memory-and-rag`](./agents/6-memory-and-rag/)

- RAG basics; semantic search; indexing; vector similarity
- Embeddings; querying (Chroma as the book's example store)
- Vector-search RAG agent; hybrid-search RAG agent
- Memory forms; graph memory via MCP; hybrid memory
- Semantic / episodic / procedural memory
- Compression and forgetting

### AIA 7 — Evaluation and feedback
[`agents/7-evaluation-and-feedback`](./agents/7-evaluation-and-feedback/)

- Why agents need eval, not just unit tests
- Test-driven agent development (TDAD); testing and refactoring a RAG agent
- Agent evaluator; grounding agent; grounding as guardrail
- Rubrics; critic agent
- Phoenix: connection, metadata/sessions, evaluators, annotations

### AIA 8 — Deploying agents
[`agents/8-deploying-agents`](./agents/8-deploying-agents/)

- Consuming agents: voice in a web app, API hosting, web client
- Dockerizing a microservice; Compose; exposing local services
- Runtime: edge vs API vs event-driven; three communication wires
- Topologies; state, memory, idempotency
- Release engineering (prompts, tools, models); observability
- Timeouts, fallbacks, budgets; cost and model routing
- Threat model; IAM for people/services/agents; secrets
- Tool sandboxing and egress; prompt injection and exfiltration; policy

### AIA 9 — The agentic loop
[`agents/9-agentic-loop`](./agents/9-agentic-loop/)

- Layer 1 inner loop (SPAL); layer 2 task loop; layer 3 meta loop
- Deep research agent: state, tools, iteration output, termination gate, synthesis
- When a loop is the right shape; repetitive task loops
- Multi-agent orchestration loops; collaborative loops

### AIA 10 — Cognitive agents
[`agents/10-cognitive-agents`](./agents/10-cognitive-agents/)

- Cognition and metacognition as engineering terms
- Failure modes of capable-but-not-cognitive agents
- Architecture: workspace, perception, planning, execution, evaluation, attention, memory
- Cognitive loop; MCP memory; confidence gates; stagnation; knowledge boundaries
- Cognitive efficiency metrics

### AIA 11 — Field tips
[`agents/11-field-tips`](./agents/11-field-tips/)

- Tips by the five layers
- Customer support agent; RAG agent system; deep research blueprint

### AIA appendices
- [A — sample repository setup](./agents/appendix-a-sample-code/)
- [B — Node.js / npx for local MCP servers](./agents/appendix-b-nodejs-mcp/)

---

## Track B — *Designing AI Systems* (by chapter)

The MEAP ships a **single-level TOC** (nine chapter titles). Section headings
below are the topics those chapters actually cover, so the index matches how
you will study, not only the one-line contents page.

### DAS 1 — Why your AI projects need a platform
[`platform/1-why-a-platform`](./platform/1-why-a-platform/)

- Prototype vs production; the production wake-up call
- AI sprawl (duplicated session stores, keys, logging)
- Hidden Technical Debt (2015) → GenAI iceberg (~2% model, ~98% platform)
- First-principles needs; platform blueprint; a request traced through services

### DAS 2 — SDK and API design
[`platform/2-sdk-and-api`](./platform/2-sdk-and-api/)

- Developer experience: setup, memory, org knowledge, safety, tools, optimization, multi-step
- Code to container: process isolation, one workflow / one service
- Exposing workflows: API sprawl vs unified exposure; sync / stream / async
- Gateway vs internal RPC; SDK (gateway, service clients, deploy metadata)

### DAS 3 — The Model Service
[`platform/3-model-service`](./platform/3-model-service/)

- Contract: generate, list models, system prompts, custom models; gRPC messages
- Provider adapters; OpenAI message shape as the platform standard
- Streaming chunks; retries and fallbacks
- Routing: cost, load, features; rate limits
- Two-level cache; cost metrics into observability; SDK `ModelClient`

### DAS 4 — The Session Service
[`platform/4-session-service`](./platform/4-session-service/)

- What a session contains; service contract; storage abstraction
- Postgres (or similar) backend; SDK integration
- Model-managed memories (facts that outlive a transcript)
- Context window: truncation, summarization, hierarchical memory, retrieval-augmented memory

### DAS 5 — The Data Service
[`platform/5-data-service`](./platform/5-data-service/)

- Documents → searchable knowledge; isolated indexes
- Ingestion: formats, extract, metadata, chunk, embed, lifecycle, async
- Vector store interface; pgvector example; search orchestration
- Hybrid search + Reciprocal Rank Fusion; full retrieval flow

### DAS 6 — Tools and guardrails
[`platform/6-tools-and-guardrails`](./platform/6-tools-and-guardrails/)

- Tools as platform capabilities; registry, versions, capability discovery
- Adapters, credential isolation, sync vs async execution
- MCP: N×M → N+M → M; hosts/clients/servers; what MCP does **not** cover
- Resource limits, circuit breakers
- Guardrails as execution policy (input, output, behavioral); policy as config

### DAS 7 — Observability and experimentation
[`platform/7-observability`](./platform/7-observability/)

- Quality, not only uptime; cross-service correlation; cost as a signal
- Data model: sessions, traces, spans, generations, scores
- Logs, metrics, distributed tracing; how services report telemetry
- Quality scores and cost attribution
- Experimentation service; datasets; offline vs online eval; annotation queues; A/B

### DAS 8 — The Workflow Service
[`platform/8-workflow-service`](./platform/8-workflow-service/)

- `@workflow` decorator → HTTP; sync, streaming, async modes
- Job lifecycle, polling, progress
- Concurrency, retries, health
- Composing workflows (including parallel)
- Registry and deploy; gateway routes; runtime management

### DAS 9 — Building an AI assistant
[`platform/9-building-an-assistant`](./platform/9-building-an-assistant/)

- Blueprint; model call in a workflow; session memory; long-term memory
- Agentic RAG; tools and the agent loop; guardrails at each step
- Context engineering; streaming; observability; experiments; deploy
- What the platform gave you that a script did not

---

## Track C — *Hands-On Machine Learning* (by chapter)

Section headings follow the 2nd-edition TOC. The notes are companions, not
a substitute for the book or for [ageron/handson-ml2](https://github.com/ageron/handson-ml2).

### HOML 1 — The Machine Learning landscape
[`ml/1-ml-landscape`](./ml/1-ml-landscape/)

- What ML is; why write a learner instead of rules
- Application sketches (the book's survey, not a product list)
- Supervised, unsupervised, semisupervised, reinforcement
- Batch vs online learning
- Instance-based vs model-based learning
- Challenges: too little data, nonrepresentative data, poor quality, irrelevant features, overfit, underfit
- Testing, validation, hyperparameter tuning, data mismatch

### HOML 2 — End-to-end Machine Learning project
[`ml/2-end-to-end-project`](./ml/2-end-to-end-project/)

- Frame the problem; pick a performance measure; check assumptions
- Get the data; train/test split (including stratified); no snooping
- Explore (geo plots, correlations, attribute combos)
- Prepare: cleaning, categoricals, custom transformers, scaling, `Pipeline`
- Train, cross-validate, grid/random search, ensembles, test-set evaluation
- Launch, monitor, maintain

### HOML 3 — Classification
[`ml/3-classification`](./ml/3-classification/)

- MNIST as the running set; binary classifier
- Why accuracy lies; confusion matrix; precision, recall, F1
- Precision/recall trade-off; ROC and AUC
- Multiclass (OvR / OvO); error analysis
- Multilabel and multioutput classification

### HOML 4 — Training models
[`ml/4-training-models`](./ml/4-training-models/)

- Linear regression; normal equation; computational complexity
- Batch / stochastic / mini-batch gradient descent
- Polynomial regression; learning curves
- Ridge, Lasso, Elastic Net, early stopping
- Logistic regression; softmax

### HOML 5 — Support Vector Machines
[`ml/5-svms`](./ml/5-svms/)

- Linear SVM; soft margin
- Polynomial and RBF kernels; similarity features; complexity
- SVM regression
- Decision function, dual problem, kernel trick, online SVMs (intuition)

### HOML 6 — Decision Trees
[`ml/6-decision-trees`](./ml/6-decision-trees/)

- Training and visualizing; CART; Gini vs entropy
- Complexity; regularization hyperparameters
- Regression trees; instability

### HOML 7 — Ensemble Learning and Random Forests
[`ml/7-ensembles`](./ml/7-ensembles/)

- Voting classifiers; bagging and pasting; OOB evaluation
- Random patches/subspaces; Random Forests; Extra-Trees; feature importance
- AdaBoost; Gradient Boosting; stacking

### HOML 8 — Dimensionality Reduction
[`ml/8-dimensionality-reduction`](./ml/8-dimensionality-reduction/)

- Curse of dimensionality; projection vs manifold
- PCA (variance, components, choosing *d*, compression, randomized, incremental)
- Kernel PCA; LLE; other techniques the chapter surveys

### HOML 9 — Unsupervised Learning Techniques
[`ml/9-unsupervised`](./ml/9-unsupervised/)

- K-Means (limits, segmentation, preprocessing, semi-supervised)
- DBSCAN and other clustering algorithms
- Gaussian mixtures; anomaly/novelty detection

### HOML 10 — Introduction to ANNs with Keras
[`ml/10-anns-keras`](./ml/10-anns-keras/)

- From biological neuron to MLP and backprop
- Regression and classification MLPs
- Sequential, Functional, Subclassing APIs; save/restore; callbacks; TensorBoard
- Fine-tuning depth, width, learning rate, batch size

### HOML 11 — Training Deep Neural Networks
[`ml/11-training-dnns`](./ml/11-training-dnns/)

- Vanishing/exploding gradients; Glorot/He; activations; batch-norm; clipping
- Reusing pretrained layers; unsupervised and auxiliary pretraining
- Momentum, Nesterov, AdaGrad, RMSProp, Adam/Nadam; LR schedules
- ℓ1/ℓ2, dropout, MC dropout, max-norm; practical guidelines

### HOML 12 — Custom Models and Training with TensorFlow
[`ml/12-custom-tf`](./ml/12-custom-tf/)

- TF as distributed numpy; tensors, variables
- Custom losses, metrics, layers, models, training loops; autodiff
- `tf.function`, AutoGraph, tracing rules

### HOML 13 — Loading and Preprocessing Data with TensorFlow
[`ml/13-data-and-preprocessing`](./ml/13-data-and-preprocessing/)

- `tf.data` pipeline: shuffle, map, prefetch
- TFRecord and protobufs
- One-hot vs **embedding tables** (features of *this* network, not a RAG index)
- Keras preprocessing; TF Transform; TFDS

### HOML 14 — Deep Computer Vision Using CNNs
[`ml/14-cnns`](./ml/14-cnns/)

- Cortex sketch; convolution and pooling
- LeNet, AlexNet, GoogLeNet, VGG, ResNet, Xception, SENet
- Pretrained models; localization, detection (FCN, YOLO), semantic segmentation

### HOML 15 — Processing Sequences Using RNNs and CNNs
[`ml/15-sequences-rnns`](./ml/15-sequences-rnns/)

- Recurrent neurons; sequence-to-vector / vector-to-sequence / seq2seq
- Time-series forecasting; deep RNNs
- Unstable gradients; LSTM/GRU; 1D convolutions for sequences

### HOML 16 — NLP with RNNs and Attention
[`ml/16-nlp-attention`](./ml/16-nlp-attention/)

- Character RNN; stateful RNN
- Sentiment; masking; pretrained embeddings
- Encoder–decoder NMT; bidirectional RNNs; beam search
- Attention; Transformer ("Attention Is All You Need")
- 2019-era language-model notes (what aged is called out in the folder)

### HOML 17 — Autoencoders and GANs
[`ml/17-autoencoders-gans`](./ml/17-autoencoders-gans/)

- Undercomplete linear AE ≡ PCA; stacked, convolutional, recurrent, denoising, sparse
- VAEs; GANs, DCGAN, progressive growing, StyleGAN (2019 snapshot)

### HOML 18 — Reinforcement Learning
[`ml/18-reinforcement-learning`](./ml/18-reinforcement-learning/)

- Rewards and policy search; OpenAI Gym (vintage)
- Credit assignment; policy gradients; MDPs; TD; Q-learning; DQN variants
- TF-Agents sketch; survey of other algorithms
- Not the LLM agentic loop — see [A9](./agents/9-agentic-loop/)

### HOML 19 — Training and Deploying TensorFlow Models at Scale
[`ml/19-scale-and-deploy`](./ml/19-scale-and-deploy/)

- TensorFlow Serving; 2019 GCP AI Platform (now Vertex-shaped)
- Mobile/embedded; GPU placement; model vs data parallelism; `tf.distribute`
- Not a provider Model Service — see [P3](./platform/3-model-service/)

### HOML appendices
- [B — Machine Learning project checklist](./ml/appendix-b-project-checklist/)
- A (exercise solutions) and C–G (SVM dual, autodiff, extra ANN architectures, TF data structures, TF graphs) stay in the book

---

## Alphabetical index

Use this when you remember a **word**, not a chapter number.

| Term | Where |
|---|---|
| A2A (agent-to-agent flow) | [A4](./agents/4-multi-agent-systems/) |
| A/B testing | [P7](./platform/7-observability/) |
| AdaBoost | [M7](./ml/7-ensembles/) |
| Adapter (model provider) | [P3](./platform/3-model-service/) |
| Adam / Nadam | [M11](./ml/11-training-dnns/) |
| Adapter (tool) | [P6](./platform/6-tools-and-guardrails/) |
| Agency / agentic | [A1](./agents/1-rise-of-ai-agents/) (not RL: [M18](./ml/18-reinforcement-learning/)) |
| Agentic loop (inner / task / meta) | [A9](./agents/9-agentic-loop/) |
| AI sprawl | [P1](./platform/1-why-a-platform/) |
| Annotation queue | [P7](./platform/7-observability/) |
| API gateway | [P2](./platform/2-sdk-and-api/), [P8](./platform/8-workflow-service/) |
| Assistant (vs agent vs LLM) | [A1](./agents/1-rise-of-ai-agents/) |
| Attention (Transformer) | [M16](./ml/16-nlp-attention/) |
| Attention module (cognitive) | [A10](./agents/10-cognitive-agents/) |
| AUC / ROC | [M3](./ml/3-classification/) |
| Autoencoder / VAE / GAN | [M17](./ml/17-autoencoders-gans/) |
| Autodiff | [M12](./ml/12-custom-tf/) |
| Backpropagation | [M10](./ml/10-anns-keras/) |
| Bagging / pasting | [M7](./ml/7-ensembles/) |
| Batch Normalization | [M11](./ml/11-training-dnns/) |
| Batch vs online learning | [M1](./ml/1-ml-landscape/) |
| Beam search | [M16](./ml/16-nlp-attention/) |
| Blackboard (shared workspace) | [A1](./agents/1-rise-of-ai-agents/), [A4](./agents/4-multi-agent-systems/) |
| Budget (tokens / dollars) | [A8](./agents/8-deploying-agents/), [P3](./platform/3-model-service/), [P7](./platform/7-observability/) |
| Caching (model responses) | [P3](./platform/3-model-service/) |
| Chain-of-thought (CoT) | [A5](./agents/5-reasoning-and-planning/) |
| Chunking | [P5](./platform/5-data-service/), [A6](./agents/6-memory-and-rag/) |
| Circuit breaker (tools) | [P6](./platform/6-tools-and-guardrails/) |
| CNN | [M14](./ml/14-cnns/) |
| CART | [M6](./ml/6-decision-trees/) |
| Cross-validation | [M2](./ml/2-end-to-end-project/), [M3](./ml/3-classification/) |
| Cognitive workspace | [A10](./agents/10-cognitive-agents/) |
| Collaboration (agent team) | [A1](./agents/1-rise-of-ai-agents/), [A4](./agents/4-multi-agent-systems/), [A9](./agents/9-agentic-loop/) |
| Confidence-gated execution | [A10](./agents/10-cognitive-agents/) |
| Context engineering | [P9](./platform/9-building-an-assistant/), [A2](./agents/2-llms-prompting-agents/) |
| Context window / token budget | [P4](./platform/4-session-service/), [A6](./agents/6-memory-and-rag/) |
| Cost-aware routing | [P3](./platform/3-model-service/), [A8](./agents/8-deploying-agents/) |
| Credential store | [P6](./platform/6-tools-and-guardrails/) |
| Critic agent | [A7](./agents/7-evaluation-and-feedback/) |
| Customer support agent | [A11](./agents/11-field-tips/), [P1](./platform/1-why-a-platform/) (Sam's story) |
| Decision tree | [M6](./ml/6-decision-trees/) |
| Dropout / MC Dropout | [M11](./ml/11-training-dnns/) |
| DQN / Q-learning | [M18](./ml/18-reinforcement-learning/) |
| Data Service | [P5](./platform/5-data-service/) |
| Deep research agent | [A9](./agents/9-agentic-loop/), [A11](./agents/11-field-tips/) |
| Deployment pipeline | [P8](./platform/8-workflow-service/), [A8](./agents/8-deploying-agents/) |
| Deploy trained weights (TF Serving) | [M19](./ml/19-scale-and-deploy/) |
| Docker / Compose | [A8](./agents/8-deploying-agents/), [P8](./platform/8-workflow-service/) |
| Embeddings | [A6](./agents/6-memory-and-rag/), [P5](./platform/5-data-service/) (retrieval); [M13](./ml/13-data-and-preprocessing/), [M16](./ml/16-nlp-attention/) (tables inside a net) |
| Episodic / semantic / procedural memory | [A6](./agents/6-memory-and-rag/) |
| Evaluation (offline / online) | [P7](./platform/7-observability/), [A7](./agents/7-evaluation-and-feedback/) |
| Evaluation (precision / recall / ROC) | [M3](./ml/3-classification/) |
| Experimentation service | [P7](./platform/7-observability/) |
| Fallback (models) | [P3](./platform/3-model-service/), [A8](./agents/8-deploying-agents/) |
| Five functional layers | [A1](./agents/1-rise-of-ai-agents/), [A11](./agents/11-field-tips/) |
| Flow (assembly line) | [A1](./agents/1-rise-of-ai-agents/), [A4](./agents/4-multi-agent-systems/) |
| Forgetting / compression | [A6](./agents/6-memory-and-rag/), [P4](./platform/4-session-service/) (summarization) |
| Gradient boosting | [M7](./ml/7-ensembles/) |
| Gradient descent (batch / SGD / mini-batch) | [M4](./ml/4-training-models/) |
| Grid search / randomized search | [M2](./ml/2-end-to-end-project/) |
| gRPC contracts | [P3](./platform/3-model-service/)–[P8](./platform/8-workflow-service/) |
| Graph memory | [A6](./agents/6-memory-and-rag/) |
| Grounding | [A7](./agents/7-evaluation-and-feedback/), [P9](./platform/9-building-an-assistant/) |
| Guardrails | [A4](./agents/4-multi-agent-systems/), [P6](./platform/6-tools-and-guardrails/) |
| Handoff | [A4](./agents/4-multi-agent-systems/) |
| Health endpoints | [P8](./platform/8-workflow-service/) |
| Hierarchical memory | [P4](./platform/4-session-service/) |
| Hub-and-spoke orchestration | [A1](./agents/1-rise-of-ai-agents/), [A4](./agents/4-multi-agent-systems/) |
| Hybrid search | [A6](./agents/6-memory-and-rag/), [P5](./platform/5-data-service/) |
| Hybrid memory | [A6](./agents/6-memory-and-rag/) |
| Iceberg (2% model) | [P1](./platform/1-why-a-platform/) |
| Idempotency | [A8](./agents/8-deploying-agents/) |
| Index (knowledge isolation) | [P5](./platform/5-data-service/) |
| Ingestion pipeline | [P5](./platform/5-data-service/) |
| Inspector (MCP) | [A3](./agents/3-mcp/) |
| JSON-RPC | [A3](./agents/3-mcp/) |
| Keras | [M10](./ml/10-anns-keras/) |
| K-Means / DBSCAN | [M9](./ml/9-unsupervised/) |
| Kernel (SVM / PCA) | [M5](./ml/5-svms/), [M8](./ml/8-dimensionality-reduction/) |
| LLM-as-judge | [A7](./agents/7-evaluation-and-feedback/), [P7](./platform/7-observability/) |
| Load-based routing | [P3](./platform/3-model-service/) |
| Long-term memory | [A6](./agents/6-memory-and-rag/), [P4](./platform/4-session-service/), [P9](./platform/9-building-an-assistant/) |
| MCP | [A3](./agents/3-mcp/), intro [A1](./agents/1-rise-of-ai-agents/), platform [P6](./platform/6-tools-and-guardrails/) |
| Memory (conversational) | [A6](./agents/6-memory-and-rag/), [P4](./platform/4-session-service/) |
| Metacognition | [A10](./agents/10-cognitive-agents/) |
| Model Service | [P3](./platform/3-model-service/) |
| MLP / perceptron | [M10](./ml/10-anns-keras/) |
| Overfitting / underfitting | [M1](./ml/1-ml-landscape/), [M4](./ml/4-training-models/) learning curves |
| PCA / LLE | [M8](./ml/8-dimensionality-reduction/) |
| Pipeline (sklearn) | [M2](./ml/2-end-to-end-project/) |
| Precision / recall / F1 | [M3](./ml/3-classification/) |
| Policy gradient (RL) | [M18](./ml/18-reinforcement-learning/) |
| npx / Node for MCP | [A appendix B](./agents/appendix-b-nodejs-mcp/) |
| Observability | [P7](./platform/7-observability/), [A7](./agents/7-evaluation-and-feedback/) Phoenix, [A8](./agents/8-deploying-agents/) |
| OpenAI Agents SDK | [A2](./agents/2-llms-prompting-agents/) |
| Orchestration | [A4](./agents/4-multi-agent-systems/), [A9](./agents/9-agentic-loop/), [P8](./platform/8-workflow-service/) |
| Persona | [A1](./agents/1-rise-of-ai-agents/), [A2](./agents/2-llms-prompting-agents/), [A11](./agents/11-field-tips/) |
| pgvector | [P5](./platform/5-data-service/) |
| Phoenix (Arize) | [A7](./agents/7-evaluation-and-feedback/) |
| Policy as configuration | [P6](./platform/6-tools-and-guardrails/) |
| Prompt injection | [A8](./agents/8-deploying-agents/), [P6](./platform/6-tools-and-guardrails/) |
| Provider abstraction | [P3](./platform/3-model-service/) |
| RAG | [A6](./agents/6-memory-and-rag/), [P5](./platform/5-data-service/), [P9](./platform/9-building-an-assistant/) |
| Rate limiting | [P3](./platform/3-model-service/) |
| Random Forest / Extra-Trees | [M7](./ml/7-ensembles/) |
| Ridge / Lasso / Elastic Net | [M4](./ml/4-training-models/) |
| RNN / LSTM / GRU | [M15](./ml/15-sequences-rnns/) |
| ReAct | [A5](./agents/5-reasoning-and-planning/) |
| Reciprocal Rank Fusion (RRF) | [P5](./platform/5-data-service/) |
| Reflexion | [A5](./agents/5-reasoning-and-planning/) |
| Release engineering (prompts/tools/models) | [A8](./agents/8-deploying-agents/) |
| Retrieval-augmented memory | [P4](./platform/4-session-service/), [A6](./agents/6-memory-and-rag/) |
| Retry / fallback | [P3](./platform/3-model-service/), [A8](./agents/8-deploying-agents/) |
| Rubric | [A7](./agents/7-evaluation-and-feedback/) |
| Sandboxing (tools) | [A8](./agents/8-deploying-agents/), [P6](./platform/6-tools-and-guardrails/) |
| SDK | [P2](./platform/2-sdk-and-api/) |
| Sense–plan–act–learn | [A1](./agents/1-rise-of-ai-agents/), [A9](./agents/9-agentic-loop/) |
| Sequential thinking (MCP) | [A5](./agents/5-reasoning-and-planning/) |
| Session Service | [P4](./platform/4-session-service/) |
| SSE vs STDIO (MCP transport) | [A3](./agents/3-mcp/) |
| Stagnation detection | [A10](./agents/10-cognitive-agents/) |
| Streaming | [P2](./platform/2-sdk-and-api/), [P3](./platform/3-model-service/), [P8](./platform/8-workflow-service/), [P9](./platform/9-building-an-assistant/) |
| Summarization (history) | [P4](./platform/4-session-service/) |
| System prompt (managed) | [P3](./platform/3-model-service/), [A2](./agents/2-llms-prompting-agents/) |
| Softmax | [M4](./ml/4-training-models/), [M10](./ml/10-anns-keras/) |
| SVM | [M5](./ml/5-svms/) |
| TensorFlow Serving | [M19](./ml/19-scale-and-deploy/) |
| `tf.data` / TFRecord | [M13](./ml/13-data-and-preprocessing/) |
| Transformer | [M16](./ml/16-nlp-attention/) |
| Transfer learning | [M11](./ml/11-training-dnns/), [M14](./ml/14-cnns/) |
| Temperature / top-p | [A2](./agents/2-llms-prompting-agents/) |
| Termination gate | [A9](./agents/9-agentic-loop/) |
| Test-driven agent development | [A7](./agents/7-evaluation-and-feedback/) |
| Threat model | [A8](./agents/8-deploying-agents/) |
| Token | [A2](./agents/2-llms-prompting-agents/) |
| Tool registry | [P6](./platform/6-tools-and-guardrails/) |
| Tracing (distributed) | [P7](./platform/7-observability/), [A2](./agents/2-llms-prompting-agents/) |
| Tree-of-thought | [A5](./agents/5-reasoning-and-planning/) |
| Truncation | [P4](./platform/4-session-service/) |
| Typed outputs | [A2](./agents/2-llms-prompting-agents/) |
| Vanishing / exploding gradients | [M11](./ml/11-training-dnns/) |
| Vector database | [A6](./agents/6-memory-and-rag/), [P5](./platform/5-data-service/) |
| Voice agents | [A8](./agents/8-deploying-agents/) |
| Workflow Service | [P8](./platform/8-workflow-service/) |
| Workflow decorator | [P8](./platform/8-workflow-service/) |

---

## Suggested paths (not a merge)

**Path 1 — "I have never built an agent."**  
A1 → A2 → A3 → A4 → A5 → A6 → A7 → A8. Peek at P1 when A8 starts talking
about cost and sprawl. Use the index, not a combined chapter.

**Path 2 — "We already call OpenAI in production and it hurts."**  
P1 → P2 → P3 → P4 → P5 → P6 → P7 → P8 → P9. Peek at A3 when P6 hits MCP,
and at A6 when P4/P5 hit memory and RAG.

**Path 3 — "I need one topic, both (or all three) angles."**  
Use the cross-book map at the top. Read the folder that matches the *job*
(train / build the agent / serve it). Do not copy paragraphs from one into
the other.

**Path 4 — "I have never fitted a model."**  
M1 → M2 → M3 → M4, then pick trees/ensembles (M6–M7) or jump to Keras
(M10) if your data is images or sequences. Peek at A2 only when M16 hits
the Transformer — that chapter *trains* attention; Agents *call* an API.
