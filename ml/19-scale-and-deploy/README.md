# 19. Training and Deploying TensorFlow Models at Scale

Companion notes for **Chapter 19** of *Hands-On Machine Learning with
Scikit-Learn, Keras, and TensorFlow* (2nd edition, Aurélien Géron;
O'Reilly, 2019).

[Chapter 18](../18-reinforcement-learning/) fitted a policy in a
simulator. This chapter is what you do when the artifact is **your
graph**: export it, serve it, squeeze it onto a phone, and — if
training still hurts — put it on GPUs and more than one of them.
Skip it and you will wrap `model.predict` in Flask on a laptop, call
that "MLOps," and then be surprised when a second GPU sits idle, a
mobile build is 400 MB, or a teammate "deploys" by pointing the app
at a vendor chat API and using this chapter's words.

**See also (do not merge):** [platform ch. 3](../../platform/3-model-service/)
is a **gateway to provider APIs** (routing, cache, fallbacks, cost).
[Agents ch. 8](../../agents/8-deploying-agents/) ships the **agent
runtime** (tools, secrets, budgets, threat model).
[Platform ch. 8](../../platform/8-workflow-service/) turns a decorated
workflow into an HTTP service with health checks. This folder stays on
**your weights, your devices, your serving binary**. Rows:
[`TRADEOFFS.md`](../../TRADEOFFS.md) ("Serving weights vs a Model
Service vs deploying an agent").

## The mental model

Three artifacts people keep calling "the model in production." They
do not share a deploy path.

```
  THIS CHAPTER                         NOT THIS CHAPTER
  ------------                         ----------------
  data -> train -> SavedModel          POST /chat  (provider weights)
       |                                    |
       v                                    v
  TF Serving / TF Lite / TF.js         Platform Model Service  [P3]
       |                                    |
       v                                    v
  scores for *your* task               tokens for *their* LLM
                                            |
                                            v
                                       Agent runtime loop     [A8]
                                       Workflow container     [P8]
```

The one sentence to remember a year from now: **TF Serving ships a
SavedModel you trained; a platform Model Service is an adapter to
someone else's API; agent deploy ships a loop.**

If you cannot point at a SavedModel (or ONNX, or a TF Lite flatbuffer)
that your org compiled, you are not in this chapter.

## Serving a TensorFlow Model

**Problem** — A Keras/TF net in a notebook is not an endpoint. Ad-hoc
Python processes do not version graphs, batch incoming requests, or
let you roll back to yesterday's weights without a Git archaeology
session.

**Solution** — Export a **SavedModel** (the TF 2 unit of exchange:
graph + weights + signatures). Run a **serving binary** that loads it
and exposes gRPC and/or REST. TensorFlow Serving is the 2019-shaped
answer in this book: model versions on disk, a config that points at
the bundle, warm-up, batching, the ability to serve v12 while you
load v13.

```
  trainer  -->  SavedModel/  -->  TF Serving  -->  client
                  |                  |
               variables          gRPC / REST
               signatures         version policy
```

Clients send tensors that match the signature, not "a prompt." The
contract is **your** input pipeline's contract: same preprocessing
or you have trained/serving skew (the silent killer from
[ch. 2](../2-end-to-end-project/) with a new costume).

**Failure mode** — Training preprocessing in `tf.data` and then
reimplementing it in the client "for speed." One extra normalization
and the production AUC is fiction. Prefer exporting preprocessing
*inside* the SavedModel signature, or a shared transform graph
([ch. 13](../13-data-and-preprocessing/) TF Transform-shaped).

A second failure: one process, one model file, no versions. You cannot
canary, you cannot roll back, and a bad export is an outage.

### 2019 GCP AI Platform (prediction)

The book also shows a **managed** prediction service on then-current
Google Cloud AI Platform: upload the SavedModel, get an HTTPS endpoint,
let the cloud own replicas. Same artifact, rented ops.

That product line was renamed and folded toward **Vertex AI**. The
pattern remains: managed serving of *your* graph, not a Chat Completions
wrapper. Details in **What aged**.

**Failure mode** — Assuming "we use GCP" means you deployed this
chapter. Calling Gemini through a Model Service is [P3](../../platform/3-model-service/).
Hosting a SavedModel on Vertex endpoints is this chapter's grandchild.

## Mobile and embedded (TF Lite / TF.js era names)

**Problem** — The server has a GPU and a network. The user has a phone,
a browser, or a microcontroller. You cannot round-trip every frame, or
you must not send the pixels off-device.

**Solution** — Convert the SavedModel to a **smaller, stricter runtime**:

- **TensorFlow Lite** (2019 name): flatten the graph, quantize
  (float32 → float16 / int8), prune ops the mobile interpreter does
  not have. Inference on CPU/GPU/NPU of the device.
- **TensorFlow.js**: run in the browser or Node, WebGL/WASM backends
  of that era. Weights ship to the client; privacy and latency change
  shape.

This is still **your** net, shrunk. It is not an LLM in the browser
unless you actually converted one (you probably did not, in this
book's examples).

**Failure mode** — Converting without a device-side metric. Quantization
can drop a class you care about. Always re-evaluate on a hold-out
*after* conversion, not only on the GPU trainer. Another: shipping an
unquantized 200 MB conv net and blaming TF Lite. The converter is not
a magic compressor; you chose the architecture in
[ch. 14](../14-cnns/).

Edge also means **fixed input sizes, fewer ops, no Python**. Custom
layers from [ch. 12](../12-custom-tf/) that do not convert are a brick
wall. Prefer stock ops if mobile is a real requirement.

## Using GPUs to speed up training

**Problem** — Dense linear algebra on a CPU is how you wait. Conv nets,
Transformers from [ch. 16](../16-nlp-attention/), GAN training from
[ch. 17](../17-autoencoders-gans/) want a device that multiplies
matrices in parallel.

**Solution** — Put the hot kernels on a GPU. Three common ways to *get*
one in this edition's world:

- **Your own card.** Driver, CUDA/cuDNN matching the TF wheel, heat,
  and a machine you maintain. Best iteration loop if it works.
- **A GPU VM in the cloud.** Rent hours. Watch idle time; the meter
  does not care that you are debugging a shape error.
- **Colaboratory-class hosted notebooks.** A free or cheap GPU with
  session limits. Fine for this book's exercises. Not a training
  cluster.

### GPU RAM

The scarce resource is often **device memory**, not FLOPs. A batch that
fits in host RAM will OOM the card. You manage that by:

- shrinking batch size (and often adjusting learning rate / batch-norm
  behaviour),
- mixed precision (later than some 2019 defaults, now table stakes),
- telling TF whether to **pre-allocate all GPU RAM** or grow as needed
  (the 2019 knobs: allow-growth vs a fraction of the card).

**Failure mode** — Two Python processes on one GPU, both trying to grab
the full card. One dies, or both thrash. If you share a box, cap memory
per process or use one process.

### Placing ops and variables

TF can pin a variable or an op to `/GPU:0` or `/CPU:0`. Data
preprocessing often stays on CPU; matmuls go to GPU. The book wants
you to know **placement is a choice**, not magic: put a huge embedding
table on GPU without thinking and you have spent the card before the
first conv.

**Failure mode** — Forcing every op onto GPU including things that are
slower there (some string, some small sequential work). Profile once
instead of guessing. Host↔device copies in a tight loop will also
eat the win: feed the GPU in batches, prefetch ([ch. 13](../13-data-and-preprocessing/)).

### Parallel devices

Two GPUs are not "twice as fast" unless you have a strategy for
**what each one holds**. That is the next section. A single stream of
tiny ops will not fill one GPU, let alone four. Increase batch size
until you are memory-bound or throughput plateaus, then talk about
distribution.

## Training across multiple devices

Two different ways to cut the net. Mixing their names is how a cluster
job does nothing twice.

```
  MODEL PARALLELISM                 DATA PARALLELISM
  (split the *graph*)               (split the *batch*)

  GPU0: layers 1-10                 GPU0: batch slice 0  \  same weights
  GPU1: layers 11-20                GPU1: batch slice 1  /  gradients merge
        pipeline bubbles                  allreduce or parameter server
```

### Model parallelism vs data parallelism

**Model parallelism** — different layers (or different heads, or
tensor shards) live on different devices. You need it when **one**
replica does not fit in GPU RAM (giant embeddings, giant Transformers).
You pay pipeline bubbles, cross-device traffic, and a harder debugger.

**Data parallelism** — every device has a replica of the whole model.
Each step, each device sees a different minibatch, computes
gradients, then **aggregates** (average) so replicas stay in sync.
This is the default win when the model **does** fit.

**Failure mode** — Splitting layers across GPUs because you have two
cards and a small CNN. You wanted data parallelism. Model parallelism
on a small graph is extra latency. The other failure: data parallelism
with a batch so small per GPU that batch-norm statistics are noise
and you "distributed" your way into worse accuracy.

### Distribution strategies

TF 2's answer in this edition is **`tf.distribute.Strategy`**: write
the model once, wrap the training loop (or `model.fit`) in a strategy
scope so variables and gradients land correctly.

Mental catalog, 2019 names:

- **MirroredStrategy.** One machine, several GPUs, all-reduce, replica
  variables mirrored. The first thing to try on a multi-GPU box.
- **MultiWorkerMirrored.** Several machines, still synchronous
  all-reduce. Now you have networking, clocks, and a worker that can
  die.
- **Parameter-server strategy.** Some tasks hold variables, workers
  compute gradients (async or sync depending on setup). Classic
  cluster shape; stale gradients if async.
- **TPUStrategy.** Same idea, TPU mesh instead of GPUs. Step structure
  is stricter (static shapes help).

The API details moved (see **What aged**). The split — **mirror the
whole net vs shard it vs parameter servers** — did not.

### TensorFlow clusters

A **cluster spec** is a JSON-ish map of jobs (`worker`, `ps`,
sometimes `chief`) to `host:port` lists. Each process is a **task**
with a type and an index. They rendezvous, then run the distributed
graph.

**Failure mode** — Firewalls, mismatched TF versions, and a chief that
starts before workers and deadlocks. Distributed TF fails as
*distributed systems* fail. Log rank, hostname, and "did N workers
join?" before you debug the loss. Another: forgetting that **learning
rate and effective batch size** scaled with the number of replicas
and then declaring the algorithm broken.

### Large jobs on the cloud

When the box on your desk is not enough, you submit a **training job**
to a cloud AI platform of 2019: a container or a package, a GPU/TPU
machine type, a bucket for data and checkpoints, logs you read in a
browser. The chapter's point is not the click-path. It is: **training
at scale is a job scheduler plus artifacts in object storage**, not a
longer Colab session.

**Failure mode** — No checkpointing. Preemptible / spot machines will
kill you. No data locality: streaming millions of tiny files from a
bucket into one worker is how you train the network of your cloud
bill. Use TFRecords / sharded files from [ch. 13](../13-data-and-preprocessing/).

### Black-box hyperparameter tuning on AI Platform

Grid search from [ch. 2](../2-end-to-end-project/) does not love a
fleet of GPU jobs. Managed **black-box / Bayesian-class** tuning
(the book's AI Platform hyperparameter service) treats each training
run as a trial: you expose a metric, the service proposes the next
learning rate / depth / dropout.

**Failure mode** — Tuning on the test set, or tuning 40 knobs with 15
trials. The service is not a substitute for a validation split or for
thinking. Also: this is **experimentation on training metrics**, not
platform A/B of an LLM feature ([P7](../../platform/7-observability/)).
Different loop, different folder.

## What aged since 2019

- **AI Platform → Vertex AI.** Same cloud, new product name, broader
  surface (pipelines, feature store, endpoints). When you read the
  book, mentally substitute Vertex for "AI Platform" and then check
  today's docs; click-paths in a 2019 chapter are fossils.
- **Keras 3.** Keras can run on TensorFlow, JAX, or PyTorch backends.
  The *ideas* in Part II (layers, `fit`, callbacks) survived. A TF-only
  custom training loop from [ch. 12](../12-custom-tf/) may not be the
  2026 default.
- **Research often prefers PyTorch or JAX.** That does not make TF
  Serving a museum piece. If your org's production graph is TF, Serving
  (or TorchServe, or NVIDIA Triton, or a Vertex/SageMaker endpoint)
  is still the right *pattern*: freeze an artifact, version it, score
  tensors.
- **TF Serving is still a real pattern for in-house graphs.** gRPC,
  model versions, batching, a dedicated binary — teams reinvent this
  poorly in FastAPI every year. Use a serving stack. It does not
  become a Model Service by adding a chat JSON wrapper.
- **Lite runtime names moved** (TF Lite continued; mobile runtimes
  keep being rebranded). Quantization and on-device eval did not
  expire. TF.js still exists; WebGPU joined the backend list.
- **Colab still exists**; so do Kaggle notebooks, GPU VMs, and
  reserved clusters. Session limits are still how students lose
  checkpoints. Save to a bucket.
- **Distribution.** `tf.distribute` evolved; JAX `pmap`/`pjit` and
  PyTorch DDP/FSDP are the papers you will actually read for giant
  language models. Data vs model vs pipeline parallelism is the
  transferable vocabulary.
- **Hugging Face + vendor endpoints** made "deploy" mean "call an
  API" for many NLP apps. That is the Agents/Platform collision this
  workshop keeps drawing. Those apps still do not replace a fraud
  model you trained on *your* table and must serve at 5 ms.

## Check yourself

1. You have a Keras classifier and a chatbot. Which one belongs on
   TF Serving, and which one belongs on a Model Service? Name the
   artifact each ships.
2. Why is a SavedModel more than "the `.h5` file in my home directory"?
   What breaks if the client preprocesses differently from training?
3. Managed AI Platform / Vertex endpoints vs TF Serving on your VMs:
   what do you gain, and what do you still have to get right about
   versions and signatures?
4. Quantize to int8, skip the hold-out eval, ship to phones. What
   failure mode did you choose? Which metric would have caught it?
5. Two processes, one GPU, default memory growth off. What happens,
   and which knob exists so they can coexist?
6. You have two GPUs and a 5 M parameter CNN that already fits on
   one. Model or data parallelism? Why would the other one be a
   waste?
7. MirroredStrategy vs a parameter-server cluster: which one is
   "one box, synchronous replicas," and what new failure appears
   when you add machines?
8. Effective batch size grew 8× with 8 replicas. What else might
   you scale, and what metric tells you the run is now
   communication-bound rather than compute-bound?
9. Black-box HP tuning on the cloud vs grid search in sklearn vs
   platform A/B on live LLM traffic: three loops. Which folder owns
   each, and what is the unit of a "trial" here?
10. A teammate says "we deployed the model" after wrapping an agent
    in Docker. Which of [A8](../../agents/8-deploying-agents/),
    [P8](../../platform/8-workflow-service/), and this chapter did
    they actually do — and what SavedModel question would you ask
    to find out?

Continue to [the project checklist](../appendix-b-project-checklist/).
