# 10. ANNs with Keras

Companion notes for **Chapter 10** of *Hands-On Machine Learning with
Scikit-Learn, Keras, and TensorFlow* (2nd edition, Aurélien Géron;
O'Reilly, 2019). This is the hinge from Part I's sklearn estimators
to Part II's **stacked differentiable layers**.

Skip it and you will treat a Sequential model as "the neural net
API," copy a Fashion-MNIST tutorial into a regression problem with
softmax on a single price, and then confuse **training a graph of
weights** with **calling a finished model over HTTP**. Depth, width,
learning rate, and batch size will feel like unrelated folklore
instead of one optimization problem.

## The mental model

A neural net in this chapter is not a biological reconstruction and
not a chatbot. It is a **chain (or graph) of tensor ops** with
weights, a **loss** on the output, and **backprop** to move those
weights down the loss.

```
  x
  |
  v
  [ Dense / flatten / ... ] --w,b-->  h1 = act(W1 x + b1)
  |
  v
  [ ... more layers ... ]
  |
  v
  y_hat   -->  LOSS(y_hat, y)
                  |
                  v
           backprop: dLoss/dW for every W
                  |
                  v
           optimizer step (SGD / Adam / ...)
```

The sentence to keep a year from now: **you define a forward graph,
Keras (through TensorFlow in the 2e) defines the backward graph, and
`fit` is a loop of forward → loss → backward → update** —
Sequential, Functional, and Subclassing only change how you *write*
the forward graph.

Two consequences fall straight out of that diagram. First, if the
forward graph cannot represent the task (linear head on a linear
problem is fine; softmax of 10 units on a scalar price is a
mismatch), no amount of TensorBoard will save you. Second, "we use
Keras" does not say whether you can save weights, stop early, or
debug a learning-rate explosion. Callbacks and logs are part of the
model the way `GridSearchCV` was part of sklearn.

## From biological sketch to MLP

History here is a **permission structure**: why a single threshold
unit is not enough, and why stacking + backprop became the default
rather than a museum piece.

A **biological neuron** cartoon: incoming spikes, a combining step,
a fire-or-not. Useful as a metaphor. Dangerous as an architecture
spec. Do not design layers to "be more brain-like" in this track;
design them to be differentiable and wide enough.

A **threshold logic unit / perceptron** unit: weighted sum of inputs
plus bias, then a step (or, later, a smooth activation). sklearn
still ships `Perceptron` — close cousin to SGD linear
classification. A **layer** of such units is a matrix multiply.

The XOR cartoon shows why one TLU (one linear threshold) cannot
separate a dataset that needs a hole or a pair of half-planes.
**Stack** layers: hidden units compute new features; the output unit
combines them. That stack is an **MLP** (multilayer perceptron). The
1980s trick that made stacking trainable is **backpropagation**: run
the chain rule from loss to each weight, then SGD.

Backprop **intuition**, not a derivation dump:

1. **Forward:** cache each layer's pre-activation and output.
2. **Loss:** a scalar how-wrong (MSE, cross-entropy, …).
3. **Backward:** error at the output → multiply by local derivatives
   → error at the previous layer → … down to the first weights.
4. **Update:** `W -= lr * (mean gradient over a batch)`.

```
  forward  x -> h -> y_hat -> L
  backward       dL/dh <- dL/dy_hat
                 dL/dW from those
```

You need a **smooth** activation (sigmoid, tanh, later ReLU) so the
local derivative is not zero almost everywhere. The step function's
gradient is a brick wall. When you stack many layers, products of
those derivatives can **vanish** (or explode). The choice of
activation is part of whether the chain rule has anything useful to
carry.

Explaining a production ReLU stack with a 1940s neuron drawing in a
design review, then being unable to name the loss or the optimizer,
keeps the metaphor and loses the model. Keep the sketch as
*motivation*. Keep the graph as *the model*.

## Regression MLPs vs classification MLPs

Same stack, different **last layer and loss**. Mixing them is the
most expensive beginner bug in this chapter.

**Regression MLP**

- Output units: **one** for a scalar target, or one per dimension
  for multi-output. Activation: **none** (linear) unless the target
  is known positive (ReLU / softplus) or in a band (sigmoid scaled).
- Loss: usually **MSE** (or MAE, Huber).
- Metrics: RMSE, MAE, not accuracy.

**Classification MLP**

- **Binary:** one unit, **sigmoid**, loss **binary cross-entropy**.
- **Multiclass:** one unit per class, **softmax**, loss
  **categorical / sparse categorical cross-entropy**.
- **Multilabel:** independent sigmoids, binary cross-entropy per
  label — softmax forces a single class, so it is the wrong head
  here.

```
  regression head:     Dense(1)                  + mse
  binary head:         Dense(1, sigmoid)         + binary_crossentropy
  multiclass head:     Dense(n_classes, softmax) + sparse_categorical_crossentropy
```

Fashion-MNIST tutorial pasted onto California housing —
`Dense(10, softmax)` and `accuracy` on a price — is the classic
mismatch. Write the **output contract** before any hidden layer:
shape of `y`, range of `y`, discrete vs continuous. Then pick head
and loss as a pair. Softmax + MSE, or linear output + cross-entropy,
will still run. The gradients will mean the wrong thing. TensorBoard
will show a number going somewhere. You will tune learning rate for
a week.

Hidden activations (ReLU in 2019-style MLPs) are shared. Output
activation is **not** a hidden-layer decision.

## Installing TF2 (2019) and Sequential models

The 2e assumes **TensorFlow 2** with **`tf.keras` bundled**. In 2019
that was the pitch: `pip install tensorflow`, `from tensorflow import
keras`, eager execution on by default, Keras as the high-level API.
GPU extras were a separate extra/index. Version pins mattered then
and still do.

You are learning **ideas**. You may run a later Keras; the aged
section names the split. In this folder, code shapes to recognize:

- `keras.models.Sequential([...layers...])`
- `model.compile(loss=..., optimizer=..., metrics=[...])`
- `model.fit(X, y, epochs=..., validation_split=...)`
- `model.evaluate`, `model.predict`

`compile` is not training. It **wires** loss, optimizer, and metrics.
`fit` is the loop. Forgetting `compile` is a runtime error. Passing
the wrong loss is a silent conceptual error (see above).

### Sequential image classifier

Teaching dataset: **Fashion-MNIST** (10 clothing classes, 28×28
grayscale). It is the right size to see **flatten → dense →
softmax** overfit and generalize in a few minutes. It is not a full
vision curriculum; convolutional nets come later.

```
  28x28 uint8
      |
      v
  Flatten        ->  784
  Dense(300, relu)
  Dense(100, relu)
  Dense(10, softmax)
```

What you should actually practice, beyond getting 87%:

- **Scale** pixels ( /255.0 ) as a transformer-like habit, not as a
  magic constant you forget on inference.
- A **validation split** (or better, an explicit validation set)
  every `fit`. Training accuracy alone is the same trap as an
  unbounded tree's train score.
- `softmax` outputs a distribution; `argmax` is the class;
  `predict` vs `predict_classes` naming moved across Keras versions
  — know you need **probabilities and a class**, and look up the
  current call.

`Flatten` omitted, so a Dense layer receives a rank-3 tensor, and
you "fix" it by copying a StackOverflow snippet that changes the
dataset instead of the graph, is a shape bug wearing a data costume.

### Sequential regression

Same Sequential API, **California-housing-shaped** (or any tabular
regression): `Dense(h, relu)` stacks, `Dense(1)` linear head, MSE.
Tabular nets in 2019 were already often beaten by gradient boosting.
You still train one, because Part II is about writing a net, not
about nets always winning.

One-hot or raw categoricals dumped into the first Dense without
scaling numeric columns fails quietly. Nets are **scale-sensitive**
in a way forests are not. Standardize. Pipeline discipline still
applies even when the estimator is Keras.

## Functional API

Sequential is a **list**. Real graphs are not: two inputs (wide
features + deep features), an auxiliary output, a skip, a shared
embedding.

**Functional API:** layers are **callable** on tensors. You name
`inputs`, thread them, build a `Model(inputs=..., outputs=...)`.

```
  input A ----> Dense ----+
                          concat --> Dense --> out
  input B ----> Dense ----+
                     |
                     +--> aux out  (optional extra loss)
```

**Wide & Deep**-style: a linear (wide) path beside a deep MLP,
merged before the head. The point is not the brand name. The point
is **you can add a path without subclassing**.

Use Functional when the graph is static and multiply-connected. You
get a model that still `compile`/`fit`/`save` like Sequential.

Nested Sequential models glued with Python `+` on *arrays* after
`predict`, trained separately, then called "a multi-input net," are
two models with a post-hoc sum. One `Model` with two `Input`s,
trained against a joint loss (or weighted list of losses), is the
real thing. Auxiliary heads exist so **gradients** reach early
layers, not so you can print a second metric.

Silent shape mismatches at concat (forgot to align batch, or
flattened one branch and not the other) waste hours. Print
`model.summary()` and, if needed, a plot of the graph before `fit`.

## Subclassing API

**Subclass `keras.Model`**, write `call(self, inputs, ...)`, use
Python control flow (loops, branches) on tensors. This is the
**dynamic** path: a graph that depends on the data in ways a static
Functional graph would hate.

You gain flexibility. You **pay**:

- `summary()` and some inspectability get harder
- saving can require extra `get_config` / `call` discipline
- bugs hide in `call` instead of in a layer list

**Rule of thumb:** Sequential for a stack, Functional for a static
DAG, Subclassing when you truly need **data-dependent** structure or
a research loop you will rewrite next week. Do not subclass to look
advanced.

Super `__init__` forgotten, or `call` building new layers on every
invocation (weights that never persist), are the classic subclass
bugs. Layers belong in `__init__` (or a `build`). `call` applies
them.

## Save, restore, callbacks, TensorBoard

A net that only lives in a notebook RAM is a demo.

- **Save / restore:** 2019 Keras leaned on **HDF5** (`model.save`,
  `load_model`) for the full model (graph + weights + optimizer
  state, with caveats) and `save_weights` / `load_weights` for
  parameters only. Later TF/Keras pushed **SavedModel** directories.
  The *job* is: you can kill the process and get the same `predict`.
- **`ModelCheckpoint`:** write the best (or latest) weights while
  `fit` runs. Assume a crash.
- **`EarlyStopping`:** halt when a **monitored** validation metric
  stops improving, optionally `restore_best_weights`. This is the
  neural cousin of "do not wait for n_estimators to overfit" in
  boosting.
- Other callbacks: learning-rate schedulers (more depth later),
  custom loggers.

**TensorBoard:** a directory of event files `fit` can write
(`tensorboard --logdir ...`). Curves for loss and metrics, later
histograms of weights, still later embeddings and profiling. Use it
as **eyes on the loop**, not as a substitute for a hold-out.

```
  fit(..., callbacks=[
        EarlyStopping(patience=..., monitor="val_loss"),
        ModelCheckpoint(path, save_best_only=True),
        TensorBoard(logdir)])
```

Ten epochs, no validation, no checkpoint, then "the model" is
whatever happened to be in memory after epoch 10. Every serious
`fit` has a validation signal, a checkpoint, and a stop rule.
TensorBoard is how you *see* whether that rule is sane (train loss
down, val loss up → you are memorizing). Monitoring **training**
loss in EarlyStopping stops when the memorization slows, not when
generalization peaks.

## Fine-tuning the model

Once the graph is the right **shape**, you still have a pile of
knobs. This chapter's practical set:

| Knob | Typical effect if you move it |
|---|---|
| **Depth** (more layers) | More composed features; harder training; vanishing gradients later |
| **Width** (more units) | More capacity per layer; more overfit, more RAM |
| **Learning rate** | Too high: loss explodes or chatters. Too low: crawl. The first knob to plot |
| **Batch size** | Larger: smoother gradients, different effective LR, GPU occupancy. Smaller: noisier, sometimes better generalization |
| **Epochs** | Budget; without early stopping it is just "how long we overfit" |
| **Optimizer** | SGD vs momentum vs Adam — Adam is the 2019 default comfort |
| **Activation** | ReLU stacks vs saturating sigmoids in hidden layers |
| **Initialization** | Mostly "use the library default" here; theory comes with deep training |

Search like sklearn: **random search** over reasonable ranges beats
a hand grid of three depths. In 2019 that was often
`sklearn.model_selection` wrapped around a Keras build function
(`KerasClassifier` / `KerasRegressor` in `tensorflow.keras.wrappers`
or `keras.wrappers.scikit_learn` — those wrappers **aged out**; see
below). Today: Keras Tuner or Optuna, same *idea* (sample
hyperparameters, rebuild, `fit`, score).

Changing five knobs per run because the last run "felt close" is how
you learn nothing. Freeze the graph family. Sweep **learning rate**
first on a short run (a range test). Then depth/width. Then batch.
Log every run (TensorBoard run names, or a tracker). One change per
conclusion. Huge batch, unchanged LR, then declaring Adam broken,
ignores coupling: if you multiply batch size, you often scale LR
(linear or sqrt rules exist; verify, do not tattoo).

Regularization of deep stacks (dropout, batch-norm) belongs with
deep training. Do not steal them here as folklore toppings on a
3-layer MLP until you have seen underfitting vs overfitting on
**this** dataset.

## What aged since 2019

- **Keras 3 (multi-backend).** Keras can run on TensorFlow, JAX, or
  PyTorch. The Sequential / Functional / Subclassing *ideas* held.
  Some `tf.*`-only bits did not. Read the backend paragraph of
  whatever you install.
- **`tf.keras` fused, then split.** The 2e world is "TF2 includes
  Keras." Later, `keras` as a pip package and `tf.keras` as a
  compatibility path diverged. Import paths in old notebooks rust.
  Prefer the Keras 3 docs for new code; keep the 2e *graphs*.
- **sklearn `Perceptron` still exists.** The linear museum piece is
  maintained. It is still not an MLP.
- **TensorBoard is still useful.** It is no longer the only viewer
  (W&B, MLflow, …). A `fit` loop without *some* curve is still
  flying blind.
- **Hyperparameter search.** `keras.wrappers.scikit_learn` is not
  where you should start. **Keras Tuner** and **Optuna** (and
  friends) are the current sockets for the same random/Hyperband
  searches. The knobs in the table above did not retire.
- **SavedModel vs HDF5.** Default save format moved. Write a
  round-trip test (`save`, new process, `predict` equals) instead of
  believing a filename extension.
- **Fashion-MNIST MLPs.** Fine to learn on. Real images: CNNs or a
  pretrained net. Real tabular: still try a histogram gradient
  booster before a 4-layer MLP.

Keep the biological → perceptron → MLP story, backprop as reverse
chain rule, head/loss pairing, three APIs, checkpointing,
TensorBoard as eyes, and LR/depth/width/batch as a coupled search.
Drop 2019 install one-liners when your runtime says so.

## Check yourself

1. A single TLU cannot solve XOR. What does a hidden layer
   *compute* that makes XOR possible, in one sentence that does not
   say "nonlinearity" as a magic word (name a feature the hidden
   units could implement)?
2. Write the four-beat backprop loop (forward, loss, backward,
   update). Where does a **batch** enter, and why is the step
   function a bad hidden activation for that loop?
3. Pick head + loss for: (a) house price, (b) spam vs ham, (c) 10
   exclusive clothing classes, (d) tags that can co-occur. Which
   pairing is the Fashion-MNIST-paste failure?
4. `compile` vs `fit`: which one chooses the optimizer, and which
   one actually walks the dataset? Why can a wrong loss still
   "train"?
5. Sketch a Functional model with two inputs and an auxiliary
   output. What problem is the extra head trying to solve that a
   Python `+` of two separately trained Sequentials does not?
6. When would you subclass `Model` instead of using Functional?
   Name one bug that only shows up in `call` (layers constructed in
   the wrong place).
7. Design a callback set for a net that must survive a killed
   notebook: what do you checkpoint, what do you monitor for early
   stop, and why is monitoring training loss a failure mode?
8. TensorBoard shows train loss falling and val loss rising after
   epoch 4. What is the model doing, and which two knobs from the
   fine-tune table would you touch first (before adding dropout)?
9. You 8× the batch size on GPU and keep the old learning rate.
   What often happens to the optimization, and what coupled change
   would you try?
10. A teammate says "we don't need this chapter; we call a hosted
    LLM over HTTP." In one sentence each, what job does that
    teammate's setup own, and what job does a Keras MLP you `fit`
    yourself still own?
