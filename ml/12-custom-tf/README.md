# 12. Custom models and training with TensorFlow

Companion notes for **Chapter 12** of *Hands-On Machine Learning with
Scikit-Learn, Keras, and TensorFlow* (2nd edition, Aurélien Géron;
O'Reilly, 2019).

[Chapter 10](../10-anns-keras/) and [chapter 11](../11-training-dnns/)
stay inside Keras's happy path: `Sequential` / Functional API,
`compile`, `fit`. This chapter is the floor those APIs stand on —
tensors, variables, autodiff, and what you write when the loss is
not `y_true` vs `y_pred`. Skip it and you will paste a custom
training loop from a blog, wrap it in `@tf.function`, and spend a
week debugging a Python `print` that only ran on the first step.

The Platform track is a different book. If you need "we wrap a
provider API behind a Model Service," that is
[platform ch. 3](../../platform/3-model-service/). This folder stays
on **your TensorFlow graph and your training step**.

## The mental model

TensorFlow 2.x is eager-first. You write Python that *runs ops now*.
When you need a fused graph (GPU/TPU, export, speed), you ask for a
trace.

```
  Python code
       |
       |  eager (default): each line runs, easy to pdb
       v
  Tensor ops on a device (CPU / GPU / TPU)
       |
       |  @tf.function  -->  AutoGraph  -->  traced graph
       v
  replay the graph with new tensor inputs
       |
       v
  GradientTape records the eager (or graph) ops
       |
       v
  d(loss)/d(Variable)  -->  optimizer.apply_gradients
```

The one sentence to remember a year from now: **Keras is an API;
TensorFlow is a differentiable tensor runtime** — and custom work
is just tensors, variables, and a tape, until `tf.function` makes
the Python control flow a graph.

Two consequences fall out of that diagram. First, if you can write
it with a Keras layer and `compile`, do that; custom loops are how
you own metrics that `fit` will not see. Second, tracing is not
"make it faster." Tracing is "this Python ran once and became a
graph"; side effects, Python `if` on a `float`, and new shapes are
the bugs that follow.

## A TensorFlow 2.x tour

Three ideas, then you can read any custom-training gist.

**Graphs.** TF 1.x made you build a graph, then `Session.run`.
TF 2.x still *has* graphs; it just does not force you to stare at
them. `@tf.function` is the on-ramp. SavedModel export is a graph
story whether you like it or not ([ch. 19](../19-scale-and-deploy/)).

**Devices.** Ops have a device. A tensor lives somewhere. `tf.device`
and the GPU that `nvidia-smi` shows are the same idea: placement.
If an op has no GPU kernel, it silently falls back to CPU and your
"GPU training" is a copy loop.

**Autodiff.** You do not write derivatives. You run a forward
pass under a `GradientTape`, then ask for `tape.gradient(loss,
variables)`. Keras `fit` is that loop with opinions.

**Problem** — People still describe TF as "the session library."

**Solution** — Describe the 2.x mental model: eager for thinking,
graph for shipping, tape for learning.

**Failure mode** — Mixing 1.x snippets (`placeholder`, `tf.Session`)
into a 2.x notebook and concluding TensorFlow is unusable. It is
unusable *that way*.

## TensorFlow like NumPy

A **tf.Tensor** is a multidimensional array with a dtype, a shape,
and a device. Most NumPy elementwise ops have a twin (`tf.add`,
`tf.matmul`, `tf.reduce_mean`, broadcasting rules that will still
surprise you).

```
  ndarray  --tf.constant-->  Tensor (immutable)
  Tensor   --.numpy()----->  ndarray   (copy / sync)

  Variable = a Tensor you are allowed to assign
             (this is where weights live)
```

**Interop.** `tf.constant(np_array)` and `tensor.numpy()` are the
bridge. Going back to NumPy inside a `@tf.function` is how you
accidentally force a sync every step.

**Dtypes.** `float32` is the training default. `float64` is what
NumPy gave you from a Python `float` and then you wondered why the
GPU kernel was missing. `float16` / mixed precision is a later
knob: speed vs stability, not a free lunch. Integers do not get
gradients.

**Variables.** Layers own `tf.Variable`s. You update them with
`assign`, `assign_add`, or an optimizer. A tensor you got from
`tf.matmul` is *not* a variable; trying to apply gradients to it
does nothing useful.

Other structures you should be able to *name* when a dataset or a
model hands you one:

- **SparseTensor** — indices + values + dense shape. One-hot-ish
  and embedding lookups often start here
  ([ch. 13](../13-data-and-preprocessing/)).
- **RaggedTensor** — nested variable-length lists (sentences,
  time series of different lengths). Padding is the alternative;
  ragged is the honest shape.
- **String tensors, queues, lookup tables** — exist; prefer Keras
  preprocessing layers unless you are sure.

**Failure mode** — Treating a ragged batch like a dense `[B, T, D]`
and writing a custom loss that silently indexes the padding. The
structure was trying to tell you the padding was a lie.

## Custom losses

**Problem** — `mse` and `sparse_categorical_crossentropy` are not
the loss you need (Huber, a weighted sum, a penalty on an internal
activation).

**Solution** — A function `loss(y_true, y_pred) -> scalar tensor`,
or a subclass of `keras.losses.Loss` when you need
`reduction` / `get_config`. Keep it in TF ops so the tape can see
it.

Huber is the chapter's running example for a reason: smooth like
MSE near zero, robust like MAE far away. The *pattern* is "write
the math in TF."

**Failure mode** — `float(numpy_mean)` inside the loss. The tape
goes blind; gradients become `None`; Keras may fail loudly or,
worse, skip the weight.

### Saving custom components

Anything you pass to `compile` or put in a layer has to survive
`model.save` / `load_model`.

```
  save  -->  config JSON  +  weights
  load  -->  from_config  +  custom_objects={...}
```

Subclass `Loss`, `Metric`, `Layer`, `Constraint`, `Regularizer`
and implement `get_config` (and `from_config` if you are not
trivial). When you load, pass `custom_objects` or register the
class. A lambda loss will train and then refuse to deserialize.

**Failure mode** — "It works in the notebook." The notebook still
has the class in `__main__`. The serving process does not.

## Custom activations, initializers, regularizers, constraints

Same pattern, smaller objects:

- **Activation** — `tf.nn.*` or a Python function of a tensor.
- **Initializer** — callable on `shape` + `dtype` (He/Glorot live
  here; so does "load this numpy array").
- **Regularizer** — added to the loss from a layer's weights.
- **Constraint** — projected after the update (max-norm from
  [ch. 11](../11-training-dnns/) is a constraint).

**Problem** — You need a one-off nonlinearity or a weight bound
Keras did not list.

**Solution** — Write a tiny TF function; if you will save the
model, make it a class with `get_config`.

**Failure mode** — A NumPy activation. It will run eagerly and
break under `tf.function`.

## Custom metrics

A **metric** is not a loss. It is a streaming statistic: update
from a batch, result for the logs, reset between epochs.

```
  state variables  (true positives, count, ...)
       ^
       |  update_state(y_true, y_pred, sample_weight)
       v
  result()  -->  the number TensorBoard shows
```

**Problem** — You report `precision` as `tp / (tp+fp)` on the
*last batch* and call it an epoch metric.

**Solution** — Subclass `keras.metrics.Metric` (or use the built-in
streaming ones). Precision/recall/AUC are the usual examples of
"must be streaming."

**Failure mode** — Putting a metric in the *loss* so the optimizer
minimizes your dashboard. Metrics can be discrete or
non-differentiable. Losses must be a tape-friendly scalar.

## Custom layers

A layer is `call(inputs)` plus weights you create in `build(shape)`
(or in `__init__` if the shape is known).

```
  __init__   remember hparams
  build      create Variables once, when input shape is known
  call       forward: TF ops only
  get_config so save/load works
```

Dense is a matmul plus a bias plus an activation. Once you have
written that yourself, subclassing for a residual block or a
reparameterization trick stops feeling like magic.

**Problem** — You need a computation Keras did not ship, or you
need to share a weight across two places.

**Solution** — One layer class, weights created once, called
twice. Functional API graphs can reuse the same layer instance;
that is weight sharing.

**Failure mode** — Creating a `tf.Variable` *inside* `call`. You
will grow a new weight every step in eager mode, or fail tracing.
`build` exists so allocation happens once.

## Custom models

`keras.Model` is a layer that you can `compile` and `fit`. The
Subclassing API from [ch. 10](../10-anns-keras/) is this: a
`call` you write by hand.

Use it when the forward pass has real control flow (a loop over
time that is not an RNN layer, a loss that needs an internal
tensor, two heads that are not a tidy Functional graph).

**Failure mode** — Subclassing everything so you lose the
Functional API's model summary, graph inspection, and easy
surgery for transfer. If the graph is static, Functional is
usually the better custom *model*.

## Losses and metrics from internals

Sometimes the loss needs a reconstruction *and* a KL term, or a
feature map, or an auxiliary head
([ch. 11](../11-training-dnns/), [ch. 17](../17-autoencoders-gans/)).
`compile(loss=...)` only sees the model's output.

**Problem** — The scalar you want to minimize is not a function
of `(y_true, y_pred)` alone.

**Solution** — Two honest options:

1. Make the model return extra tensors and write a loss that
   consumes them (or add losses with `self.add_loss` inside
   `call`).
2. Write a custom training step (`train_step` on the Model, or a
   loop) where you can see every intermediate.

**Failure mode** — Logging an internal tensor with Python state
and assuming it was minimized. If it is not in the tape's loss,
it is a spectator.

## Autodiff and custom training loops

The loop you are reinventing:

```
  for batch in dataset:
      with GradientTape() as tape:
          y_hat = model(x, training=True)
          loss = loss_fn(y, y_hat) + sum(model.losses)
      grads = tape.gradient(loss, model.trainable_variables)
      optimizer.apply_gradients(zip(grads, model.trainable_variables))
      metric.update_state(y, y_hat)
```

That is `fit`. Write it yourself when you need gradient clipping
that is not an optimizer kwarg, GAN-style two-player steps, or
losses from internals.

Rules that save the weekend:

- Watch **only the variables you mean**. Non-trainable BN moving
  averages are not in `trainable_variables`; do not apply SGD to
  them.
- `training=True/False` on `model(...)` is how dropout and BN
  behave. Forget it and [ch. 11](../11-training-dnns/) comes back
  as a heisenbug.
- `None` gradients mean the tape never used that variable, or the
  dtype was integer, or you computed the loss outside the `with`.

**Failure mode** — A loop that forgets `model.losses` (regularizers
silently vanish) or that updates metrics but never `reset_states`
between epochs.

Overriding `train_step` on a `keras.Model` is the middle path:
you keep `fit`, callbacks, and `keras.callbacks.Callback`, but
you own the math. Prefer that to a from-scratch `for epoch`
until you cannot.

## `tf.function`, AutoGraph, tracing rules

`@tf.function` traces your Python, AutoGraph rewrites `if` /
`for` / `while` into graph control flow when the condition is a
tensor, and later calls *replay the graph*.

```
  first call  -->  trace (slow, Python runs)
  later calls -->  replay if the *trace key* matches
  mismatch    -->  retrace (new graph)
```

Trace keys include argument **shapes and dtypes**, and Python
*values* that AutoGraph treated as constants.

Rules of thumb:

- Tensor arguments: good. The graph is built to take tensors.
- Python `bool` / `int` hyperparameters: become constants in
  *that* graph. Change them, get a retrace — or worse, *don't*,
  and keep the old constant.
- Side effects (`print`, `list.append`, mutating a Python dict)
  run at trace time, not per step. Use `tf.print`, `tf.Variable`,
  `TensorArray`.
- `input_signature` pins shapes so you do not retrace on every
  new batch length — or so you *fail* instead of silently
  retracing.

**Problem** — Eager custom loops are too slow.

**Solution** — Decorate the *step*, not the whole epoch of Python
I/O. Keep data loading in `tf.data`
([ch. 13](../13-data-and-preprocessing/)).

**Failure mode** — Decorating a function that converts to NumPy
inside, or that branches on `if loss > 1.0` where `loss` is a
0-d tensor you accidentally wrapped with `float()`. AutoGraph
either retraces forever or freezes the first branch.

Debug with `tf.config.run_functions_eagerly(True)` to see if the
bug is your math or your trace. If it only fails under the graph,
it is almost always a side effect or a Python/tensor mixup.

## What aged since 2019

- **TF 2.x eager-first is still the mental model.** You should
  not go back to sessions. TF-Géron 2e was already on this side
  of the break.
- **Keras 3** (multi-backend: TensorFlow, JAX, PyTorch) changed
  where `keras` lives (`keras` vs `tf.keras`) and what a "custom
  component" must look like to be backend-agnostic. The *ideas*
  (Loss, Metric, Layer, `get_config`) survived; copy-paste from a
  2019 gist may import the wrong package.
- **JAX** became a real alternative autodiff runtime (and a Keras
  3 backend). If your team is JAX-native, this chapter's tape
  maps to `jax.grad` / `jit`; do not dual-maintain a TF custom
  loop "because the book did."
- Mixed precision, `tf.distribute` strategies, and
  `model.compile(..., jit_compile=True)` are the grown-up
  versions of "make the step a graph." Distribution details:
  [ch. 19](../19-scale-and-deploy/).
- `tf.GradientTape` is still the right picture even when the
  decorator names change.

## Check yourself

1. In one sentence each: eager mode, a traced graph, a
   `GradientTape`. Which one is Keras `fit` using on a GPU
   today?
2. Why is a `tf.Variable` not just a tensor you keep around? Who
   owns the variables in a Dense layer?
3. You write a Huber loss with NumPy `np.where`. What goes wrong
   under a tape? Rewrite the rule in words using only TF ops.
4. A colleague saves a model with a lambda loss and cannot load
   it in a fresh process. What API did they skip, and where does
   `custom_objects` go?
5. Precision as an epoch metric vs precision on the last batch:
   which objects in Keras implement the streaming version, and
   what three methods do they need?
6. Why must `tf.Variable`s be created in `build` (or `__init__`)
   rather than in `call`? What does that bug look like in eager
   vs under `tf.function`?
7. Your VAE-style loss needs an internal KL term. Name two ways
   to get that scalar into the gradient without pretending it is
   `y_true`/`y_pred`.
8. A custom loop's train loss drops; `model.losses` is empty
   even though you set `kernel_regularizer`. What line is
   missing, and what did the optimizer stop seeing?
9. Give two tracing rules that would make `@tf.function` either
   retrace every batch or freeze a Python `if`. How would you
   confirm the bug is tracing rather than math?
10. Keras 3 + JAX vs this chapter's TF tape: what stays
    (differentiable ops, variables, a step) and what you would
    *not* copy (device of `tf.function`)?

Continue to [Loading and preprocessing data with TensorFlow](../13-data-and-preprocessing/).
