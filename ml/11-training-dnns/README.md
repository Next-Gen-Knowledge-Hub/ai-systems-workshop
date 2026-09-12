# 11. Training deep neural networks

Companion notes for **Chapter 11** of *Hands-On Machine Learning with
Scikit-Learn, Keras, and TensorFlow* (2nd edition, Aurélien Géron;
O'Reilly, 2019).

[Chapter 10](../10-anns-keras/) taught you to *assemble* a Keras net and
fit it. This chapter is why a deeper net often *refuses* to train, and
what you change — initialization, activations, normalization, optimizer,
schedule, regularizer — before you blame the architecture. Skip it and
you will stack more layers, watch the loss NaN or stall, then "fix" it
by calling a chat API and calling that transfer learning.

**See also (do not merge):** reusing pretrained *weights you own or
download as a Keras graph* is this chapter. Sampling a frozen vendor
model is [agents ch. 2](../../agents/2-llms-prompting-agents/). A
platform Model Service that *routes* those API calls is
[platform ch. 3](../../platform/3-model-service/). Transfer ≠ calling
GPT.

## The mental model

A deep net is a long chain of multiplies. Training is a long chain of
derivatives of those multiplies. Either chain can die.

```
  x --> [W1] --> [act] --> [W2] --> [act] --> ... --> [WL] --> loss
         ^                                              |
         |         backprop: product of Jacobians       |
         +------------- dL/dW1  <-----------------------+

  |product| -> 0     vanishing: early layers freeze
  |product| -> inf   exploding: NaNs, huge steps
```

The one sentence to remember a year from now: **depth is only useful if
signal and gradient both survive the trip**, and you buy that survival
with init, activations, normalization, and an optimizer that does not
fight you.

Two consequences fall out of that diagram. First, "add more layers"
without He/Glorot, a non-saturating activation, and usually batch-norm
is how you buy a deeper *untrainable* net. Second, regularizers
(dropout, ℓ1/ℓ2, max-norm) are not a moral extra — they are how you
stop a now-trainable deep net from memorizing the train set.

## Vanishing and exploding gradients

**Problem** — Sigmoid and tanh squash. Deep stacks of them send
activations toward 0 or ±1, and send gradients toward 0. Unbounded
weights send the other way: exploding updates.

**Solution** — Match three knobs so the variance of activations and of
gradients stays roughly constant across depth:

1. **Initialization** that respects fan-in / fan-out.
2. **Activations** that do not saturate in the typical operating range.
3. **Normalization** (next section) so each layer sees a stable input
   distribution.

**Failure mode** — You swap only the optimizer and call the net
"unstable." Exploding and vanishing are *geometry of the chain*, not
Adam vs SGD. If early-layer histograms are dead and the loss is NaN,
fix the chain before you tune β₂.

### Glorot, He, and friends

The idea is not a magic constant. You want the forward variance and the
backward variance to neither collapse nor explode when you multiply by
`W` then by an activation slope.

| Family | Typical pairing | Fan rule (intuition) |
|---|---|---|
| Glorot / Xavier | tanh, sigmoid, softmax | Balance fan-in and fan-out |
| He | ReLU, leaky ReLU, ELU | Scale for ReLU's half-rectified variance |
| LeCun | SELU (self-normalizing nets) | Fan-in, with the SELU assumptions |

Keras lets you set `kernel_initializer` per layer. Default initializers
have changed across Keras versions; **name the one you mean** when you
change the activation. A He-uniform dense stack behind a sigmoid is a
silent mismatch.

**Failure mode** — Copy-pasting `he_normal` onto an output softmax
because "we use He now." Output layers still want a small, balanced
init; the last nonlinearity is not ReLU.

### Nonsaturating activations

Sigmoid was a biological story and a gradient trap. The ReLU family
keeps a non-zero slope on a large region of the real line.

- **ReLU** — cheap, sparse, default for a reason. Dies if a unit is
  pushed into the negative side and the gradient never rescues it.
- **Leaky ReLU / PReLU** — a small negative slope (fixed or learned)
  so "dead" is less permanent.
- **ELU** — smooth negative side; closer to zero-mean activations;
  slower than ReLU.
- **SELU** — ELU-shaped with a specific scale so a *dense* stack can
  self-normalize *if* you honor LeCun init, the right dropout variant,
  and no batch-norm fighting it.
- **GELU / Swish / SiLU / Mish** (later than this edition's center of
  gravity) — smooth, used in a lot of post-2019 transformers. Same
  job: do not saturate.

**Problem** — Every activation is a hyperparameter people treat as a
religion.

**Solution** — Pick a default for the *block* you are in (He+ReLU+BN
for a generic MLP/CNN; SELU only if you will keep the contract), then
ablate one change at a time.

**Failure mode** — Mixing SELU, batch-norm, and regular dropout in the
same stack "because each is good." Self-normalization is a *contract*,
not a garnish.

## Batch Normalization

Batch-norm is a layer that, during training, standardizes each
mini-batch (per channel / per feature), then learns a scale and a
shift. At inference it uses moving averages instead of the batch.

```
  z  -->  (z - mean_B) / sqrt(var_B + eps)  -->  gamma * . + beta
                 train: batch stats
                 test:  running mean/var
```

**Problem** — Internal covariate shift is the 2015 slogan; the
practical problem is that each layer's input distribution keeps
moving, so you need tiny learning rates and careful init.

**Solution** — Normalize inside the net. You usually get: higher
learning rates, less sensitivity to init, a mild regularizing effect
from batch noise, and the option to saturate less.

Where to put it is a holy war. Common 2019 Keras pattern: Dense/Conv
→ BN → activation. Some residual blocks do BN before the weight.
Pick one convention per codebase and stick to it.

**Failure mode** — Batch-norm with batch size 2, or with a train/serve
skew you never measured. Tiny batches make the batch mean a bad
estimate; frozen moving averages that never saw production-like
inputs make serve-time activations alien. For RNNs, batch-norm is
awkward; layer-norm / recurrent variants show up more often there
(see [ch. 15](../15-sequences-rnns/)).

Use `training=True/False` correctly when you call a BN model by hand.
A custom loop that forgets to pass the training flag will update
(or fail to update) moving averages and you will debug "the metric
only works inside `fit()`."

## Gradient clipping

When the chain still explodes — recurrent nets are the classic case —
clip the gradient tensor.

- **Clip by value** — every component into `[-c, c]`. Simple, can
  change direction a lot.
- **Clip by norm** — scale the whole vector if its global norm
  exceeds `c`. Direction preserved; this is the usual default.

**Problem** — A single huge minibatch gradient nukes the weights.

**Solution** — `clipnorm` / `clipvalue` on the optimizer (Keras) so
every `apply_gradients` is bounded.

**Failure mode** — Clipping a *vanishing* net. If gradients are ~0,
clipping does nothing useful. Clipping is a fuse, not a substitute
for He+ReLU+BN. Also: a clip norm so tight that every step is
rescaled is just a disguised tiny learning rate.

## Reusing pretrained layers

You rarely train a deep net from random weights when a related task
already has a trained stack.

```
  source task                    target task
  [low] [mid] [high] [head]      [low] [mid] [new head]
    |     |      |                  |     |
    +-----+------+  copy / freeze   +-----+  then maybe unfreeze
```

Lower layers tend to be generic (edges, counts, local patterns).
Upper layers are task-specific. The usual recipe:

1. Load the source model (or a public Keras application —
   [ch. 14](../14-cnns/) for vision).
2. Strip or replace the head.
3. **Freeze** the reused stack; train the new head.
4. Optionally **unfreeze** some upper reused layers and continue at
   a *much* smaller learning rate.

**Problem** — Your target set is too small to train 50 layers.

**Solution** — Steal features. The stolen layers are a prior.

**Failure mode** — Fine-tuning the whole net at the original learning
rate and wiping the prior in a few hundred steps. Or freezing
forever when the domains differ (medical vs ImageNet, ticks vs
words) so the "pretrained" stack is a mismatch you never adapted.

### Transfer with Keras

Mechanically this is `clone_model` / load weights, `trainable = False`
on a slice, a new `Dense` head, `compile`, `fit`, then unfreeze and
`compile` again (so the optimizer knows which variables exist).

Two Keras gotchas that eat evenings:

- **Frozen BN** — a frozen batch-norm layer still has a training-mode
  vs inference-mode question. If you unfreeze a BN stack and keep
  tiny batches, you can wreck the moving statistics. Often you keep
  BN in inference mode while fine-tuning.
- **Recompile after toggling `trainable`** — otherwise the optimizer
  may not pick up the newly trainable weights.

This is still *your graph, your loss, your labels*. It is not an LLM
tool call. If the sentence you want to write is "we transferred GPT,"
you are in the Agents track.

### Unsupervised pretraining and auxiliary tasks

When labeled target data is scarce but *unlabeled* data is not:

- **Unsupervised / self-supervised pretraining** — train a stack to
  reconstruct, predict a held-out patch, or contrast two views, then
  attach a supervised head. Autoencoders in
  [ch. 17](../17-autoencoders-gans/) are the 2019-shaped version;
  today's vision/language pretraining is the same *job* with fancier
  pretext tasks.
- **Auxiliary output** — a side head that must predict something
  easier or related (a coarser label, a reconstruction). It injects
  extra gradient into the middle of a deep stack so early layers
  hear a loss.

**Failure mode** — Pretraining on a corpus whose support does not
overlap the target (different sensors, different language, different
class prior) and then declaring "unsupervised learning didn't work."
The prior has to be a prior *on this input*.

## Faster optimizers

Plain SGD is a noisy estimator of the true gradient. The 2019 toolkit
adds momentum and/or per-parameter scaling.

| Optimizer | What it adds | Watch for |
|---|---|---|
| SGD | Baseline; you understand the step | Slow in ravines |
| Momentum | Velocity: keep going if gradients agree | Overshoot if momentum is huge |
| Nesterov | Look-ahead momentum | Usually a cheap upgrade on momentum |
| AdaGrad | Per-weight scaling by accumulated sq. grad | Learning rate dies; rare as a final choice |
| RMSProp | AdaGrad with a decaying accumulator | Solid default for some recurrent nets |
| Adam | Momentum + RMSProp-style scaling | Default for many nets; can generalize worse than tuned SGD |
| Nadam | Adam + Nesterov | Géron's 2019 "try this first" for many Keras nets |

**Problem** — Loss surfaces are long valleys. SGD zigzags.

**Solution** — Momentum smooths the path. Adaptive methods stretch
the step per coordinate so rare features still move.

**Failure mode** — Treating Adam as "no learning-rate to set." You
still set `learning_rate`, and you still often decay it. Adaptive
does not mean schedule-free. A second failure: switching optimizer
every time the loss hiccups, so you never know which *other* knob
was wrong.

Weight decay vs ℓ2: in SGD they can coincide; in Adam, "ℓ2 in the
loss" and "decoupled weight decay" (AdamW, after this edition) are
not the same update. If you reproduce a modern paper, check which
one they meant.

## Learning rate scheduling

The learning rate is usually the first knob that actually moves the
curve.

Common shapes (names vary by API):

- **Power / inverse-time** — gradually shrink.
- **Exponential decay** — multiply by a factor every N steps.
- **Piecewise constant** — drop at milestones (the "reduce by 10
  at epoch 80" schedule).
- **Performance scheduling** — decay when a val metric plateaus
  (`ReduceLROnPlateau`).
- **1cycle / cyclical** (popularized around this era) — go *up*
  then down. Can allow a large max LR if the rest of the stack is
  stable.

**Problem** — A constant LR is either too big to converge or too
small to leave the initial plateau.

**Solution** — Schedule. Prefer a schedule you can plot next to the
loss. Keras callbacks and `tf.keras.optimizers.schedules` both exist;
pick one so you do not decay twice.

**Failure mode** — Decay on *train* loss noise, or decay because
epoch 10 felt late. Tie decay to a val curve or to a planned
step count. Also: a schedule copied from ImageNet onto a 3-epoch
fine-tune will hit the drop after you already stopped.

## Regularization

A trainable deep net overfits. You need a budget for capacity.

### ℓ1 and ℓ2

Penalize large weights in the loss (`kernel_regularizer`). ℓ2 is the
usual smoothness prior. ℓ1 drives sparsity. Combined (elastic-net
style) shows up less in modern conv nets than in linear models
([ch. 4](../4-training-models/)), but the idea is the same: constrain
the hypothesis class.

**Failure mode** — Huge ℓ2 *and* dropout *and* aggressive
augmentation, then wondering why the train loss never drops.

### Dropout and MC dropout

Dropout zeros a random fraction of units during training (and scales
at train or test so the expected activation matches). It is an
ensemble of subnetworks that share weights.

**Problem** — Co-adaptation: units memorize in cliques.

**Solution** — Break the clique. Typical rates: 0.2–0.5 on dense
heads; lower (or none) on already-noisy conv stacks that use BN.

**MC dropout** — leave dropout *on* at inference and average T
stochastic forward passes. You get a cheap uncertainty sketch, not
a calibrated Bayesian posterior. Useful as "does this input make
the subnetworks disagree?"

**Failure mode** — Dropout on the output of a batch-norm layer as a
reflex, or MC dropout with T=1 (that is just noisy inference).
SELU nets want *alpha dropout*, not vanilla dropout.

### Max-norm

Clip each layer's weight vector to a maximum ℓ2 norm after the
update. Bounds the step without putting a penalty in the loss.
Pairs well with dropout in older MLP recipes.

**Failure mode** — A max-norm so small that the layer cannot
represent the function, which looks like underfitting and gets
"fixed" by adding layers.

## Practical guidelines (start here, then measure)

This is a *workshop default*, not a copied cookbook. Change one
row when the failure mode in the right-hand column shows up.

| Knob | Sensible 2019-shaped start | Change it when |
|---|---|---|
| Depth / width | Deep enough to overfit slightly, then regularize | Train loss will not drop (underfit) or val never tracks (overfit) |
| Init | He for ReLU family; Glorot for tanh/sigmoid | Activations saturate or blow up in the first steps |
| Activation | ReLU (hidden), softmax/sigmoid/linear (head) | Dead ReLUs, or you are honoring a SELU contract |
| Normalization | Batch-norm after linear/conv in MLPs/CNNs | Tiny batches, RNNs, or a SELU stack |
| Optimizer | Nadam or Adam; SGD+momentum if you will tune | Adam generalizes worse than a tuned SGD on this task |
| Learning rate | Find the highest stable LR, then schedule down | Loss diverges, or it crawls after a short drop |
| Regularizer | Light ℓ2 and/or dropout on the head | Val gap is large; do not stack every regularizer |
| Transfer | Frozen trunk, trained head, then gentle unfreeze | Domains differ; then unfreeze more with a tiny LR |
| Gradient clip | Off for BN-ReLU CNNs; on for RNNs | You see NaNs despite sane init |
| Batch size | Largest that fits *and* still regularizes | BN stats are garbage (too small) or you need to retune LR (too big) |

If two of these are on fire, **init/activation/BN first**, optimizer
second, regularizer third. Regularizing an untrainable net produces
beautiful, useless curves.

## What aged since 2019

- **AdamW, cosine, warmup** are the default language of many
  training recipes. The *job* (adaptive step + schedule) is this
  chapter; the paper names moved.
- **GELU / SiLU** displaced ELU/SELU in a lot of transformer and
  conv literature. ReLU is still everywhere in vision backbones.
- **Layer-norm / RMSNorm** dominate language models; batch-norm
  still dominates many conv nets. The "always BN" row in the table
  is vision/MLP-shaped, not LLM-shaped.
- **Self-supervised pretraining** (SimCLR, MAE, masked language
  modeling) ate a lot of what this chapter called unsupervised
  pretraining. The transfer *recipe* (freeze, new head, unfreeze)
  did not change.
- **Keras 3** moved some initializer/activation defaults and the
  multi-backend story ([ch. 12](../12-custom-tf/)). Read the
  version you run, not a 2019 screenshot.
- Calling a closed model is still not this chapter.

## Check yourself

1. Draw the product-of-Jacobians picture for a 20-layer MLP. Where
   would you look first if the loss is NaN versus if it is flat?
2. Why does He init pair with ReLU and Glorot with tanh? What
   failure do you get if you swap them and keep everything else?
3. A teammate says dead ReLUs mean "use Adam." What would you
   inspect in the activation histograms before you agree?
4. Explain train-time vs serve-time batch-norm in one diagram.
   What breaks if `fit` saw batch size 128 and your serving path
   calls the model on a single example *in training mode*?
5. You clone a Keras model, freeze the trunk, add a head, and the
   frozen layers still change. Name two Keras mechanics that could
   cause that.
6. How is unsupervised pretraining different from "we call GPT"?
   Give one case where a pretext task *would* help your labeled
   set, and one where it would not.
7. AdaGrad vs RMSProp vs Adam: which accumulator dies, which
   decays, and which also keeps a velocity? When might you still
   pick SGD with momentum?
8. Dropout at train, MC dropout at serve: what is the extra
   average buying you, and what is it *not*?
9. Using the guidelines table, pick a first stack for tabular
   data with 8k rows. Which rows of the table would you *not*
   copy from an ImageNet recipe?
10. A val curve is great, production is not. Which of this
    chapter's tools (BN moving averages, dropout train/serve,
    frozen vs unfrozen BN) is your first suspect — and why is
    that a *training* bug, not an Agents-track bug?

Continue to [Custom models and training with TensorFlow](../12-custom-tf/).
