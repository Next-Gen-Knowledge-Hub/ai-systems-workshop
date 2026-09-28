# 11. Training deep neural networks

Companion notes for **Chapter 11** of *Hands-On Machine Learning with
Scikit-Learn, Keras, and TensorFlow* (2nd edition, Aurélien Géron;
O'Reilly, 2019).

Assembling a Keras net and calling `fit` is the easy half. This chapter
is why a deeper net often refuses to train, and what you change —
initialization, activations, normalization, optimizer, schedule,
regularizer — before you blame the architecture. Skip it and you will
stack more layers, watch the loss NaN or stall, then reach for a
pretrained API and call that transfer learning. Transfer of a Keras
graph you own is this chapter. Sampling a frozen vendor model is a
different job.

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

Two consequences fall out of that diagram. First, adding layers without
He or Glorot init, a non-saturating activation, and usually batch-norm
is how you buy a deeper untrainable net. Second, regularizers (dropout,
ℓ1/ℓ2, max-norm) are how you stop a now-trainable deep net from
memorizing the train set.

## Vanishing and exploding gradients

Sigmoid and tanh squash. Deep stacks of them send activations toward 0
or ±1, and send gradients toward 0. Unbounded weights send the other
way: exploding updates. Think of backprop as a product of many numbers
along the chain. If most of those numbers are less than one, the product
shrinks toward zero at the early layers — those layers stop learning
even though the loss still complains. If most are greater than one, the
product blows up and a single step can NaN the weights.

You match three knobs so the variance of activations and of gradients
stays roughly constant across depth:

1. **Initialization** that respects fan-in / fan-out.
2. **Activations** that do not saturate in the typical operating range.
3. **Normalization** (next section) so each layer sees a stable input
   distribution.

Swapping only the optimizer does not fix this. Exploding and vanishing
are geometry of the chain. If early-layer histograms are dead and the
loss is NaN, fix the chain before you tune Adam's β₂.

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
silent mismatch. Copy-pasting `he_normal` onto an output softmax
because "we use He now" is another: output layers still want a small,
balanced init; the last nonlinearity is not ReLU.

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

Every activation is a hyperparameter people treat as a religion. Pick a
default for the *block* you are in (He+ReLU+BN for a generic MLP/CNN;
SELU only if you will keep the contract), then ablate one change at a
time. Mixing SELU, batch-norm, and regular dropout in the same stack
"because each is good" breaks self-normalization: that regime is a
contract, not a garnish.

## Batch Normalization

Batch-norm is a layer that, during training, standardizes each
mini-batch (per channel / per feature), then learns a scale and a
shift. At inference it uses moving averages instead of the batch.

```
  z  -->  (z - mean_B) / sqrt(var_B + eps)  -->  gamma * . + beta
                 train: batch stats
                 test:  running mean/var
```

Internal covariate shift is the 2015 slogan; the practical problem is
that each layer's input distribution keeps moving, so you need tiny
learning rates and careful init. Normalize inside the net and you
usually get higher learning rates, less sensitivity to init, a mild
regularizing effect from batch noise, and the option to saturate less.

Where to put it is a holy war. Common 2019 Keras pattern: Dense/Conv
→ BN → activation. Some residual blocks do BN before the weight.
Pick one convention per codebase and stick to it.

Batch-norm with batch size 2, or with a train/serve skew you never
measured, is a trap. Tiny batches make the batch mean a bad estimate;
frozen moving averages that never saw production-like inputs make
serve-time activations alien. For RNNs, batch-norm is awkward;
layer-norm and recurrent variants show up more often there.

Use `training=True/False` correctly when you call a BN model by hand.
A custom loop that forgets to pass the training flag will update (or
fail to update) moving averages and you will debug "the metric only
works inside `fit()`."

## Gradient clipping

When the chain still explodes — recurrent nets are the classic case —
clip the gradient tensor.

- **Clip by value** — every component into `[-c, c]`. Simple, can
  change direction a lot.
- **Clip by norm** — scale the whole vector if its global norm
  exceeds `c`. Direction preserved; this is the usual default.

A single huge minibatch gradient can nuke the weights. Set `clipnorm`
or `clipvalue` on the optimizer (Keras) so every `apply_gradients` is
bounded. Clipping a *vanishing* net does nothing useful — clipping is
a fuse, not a substitute for He+ReLU+BN. A clip norm so tight that
every step is rescaled is just a disguised tiny learning rate.

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

1. Load the source model (or a public Keras application for vision).
2. Strip or replace the head.
3. **Freeze** the reused stack; train the new head.
4. Optionally **unfreeze** some upper reused layers and continue at
   a *much* smaller learning rate.

Your target set is often too small to train 50 layers. Steal features:
the stolen layers are a prior. Fine-tuning the whole net at the
original learning rate can wipe that prior in a few hundred steps.
Freezing forever when the domains differ (medical vs ImageNet, ticks
vs words) leaves a mismatch you never adapted.

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
tool call.

### Unsupervised pretraining and auxiliary tasks

When labeled target data is scarce but *unlabeled* data is not:

- **Unsupervised / self-supervised pretraining** — train a stack to
  reconstruct, predict a held-out patch, or contrast two views, then
  attach a supervised head. Autoencoders are the 2019-shaped version;
  today's vision/language pretraining is the same *job* with fancier
  pretext tasks.
- **Auxiliary output** — a side head that must predict something
  easier or related (a coarser label, a reconstruction). It injects
  extra gradient into the middle of a deep stack so early layers
  hear a loss.

Pretraining on a corpus whose support does not overlap the target
(different sensors, different language, different class prior) and
then declaring "unsupervised learning didn't work" misses the point:
the prior has to be a prior *on this input*.

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

Loss surfaces are long valleys. SGD zigzags. Momentum smooths the
path. Adaptive methods stretch the step per coordinate so rare
features still move.

Treating Adam as "no learning-rate to set" is a mistake. You still set
`learning_rate`, and you still often decay it. Adaptive does not mean
schedule-free. Switching optimizer every time the loss hiccups means
you never know which *other* knob was wrong.

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

A constant LR is either too big to converge or too small to leave the
initial plateau. Prefer a schedule you can plot next to the loss.
Keras callbacks and `tf.keras.optimizers.schedules` both exist; pick
one so you do not decay twice. Decay on *train* loss noise, or decay
because epoch 10 felt late, is cargo-cult. Tie decay to a val curve or
to a planned step count. A schedule copied from ImageNet onto a
3-epoch fine-tune will hit the drop after you already stopped.

## Regularization

A trainable deep net overfits. You need a budget for capacity.

### ℓ1 and ℓ2

Penalize large weights in the loss (`kernel_regularizer`). ℓ2 is the
usual smoothness prior. ℓ1 drives sparsity. Combined (elastic-net
style) shows up less in modern conv nets than in linear models, but
the idea is the same: constrain the hypothesis class. Huge ℓ2 *and*
dropout *and* aggressive augmentation, then wondering why the train
loss never drops, is stacking every brake at once.

### Dropout and MC dropout

Dropout zeros a random fraction of units during training (and scales
at train or test so the expected activation matches). It is an
ensemble of subnetworks that share weights. Units that memorize in
cliques (co-adaptation) get broken apart. Typical rates: 0.2–0.5 on
dense heads; lower (or none) on already-noisy conv stacks that use BN.

**MC dropout** — leave dropout *on* at inference and average T
stochastic forward passes. You get a cheap uncertainty sketch, not
a calibrated Bayesian posterior. Useful as "does this input make
the subnetworks disagree?"

Dropout on the output of a batch-norm layer as a reflex, or MC
dropout with T=1 (that is just noisy inference), wastes the idea.
SELU nets want *alpha dropout*, not vanilla dropout.

### Max-norm

Clip each layer's weight vector to a maximum ℓ2 norm after the
update. Bounds the step without putting a penalty in the loss.
Pairs well with dropout in older MLP recipes. A max-norm so small
that the layer cannot represent the function looks like underfitting
and gets "fixed" by adding layers.

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
  multi-backend story. Read the version you run, not a 2019
  screenshot.
- Calling a closed model is still not this chapter.

## Check yourself

1. Draw the product-of-Jacobians picture for a 20-layer MLP. Where
   would you look first if the loss is NaN versus if it is flat?
2. Why does He init pair with ReLU and Glorot with tanh? What
   happens if you swap them and keep everything else?
3. A teammate says dead ReLUs mean "use Adam." What would you
   inspect in the activation histograms before you agree?
4. Explain train-time vs serve-time batch-norm in one diagram.
   What breaks if `fit` saw batch size 128 and your serving path
   calls the model on a single example *in training mode*?
5. You clone a Keras model, freeze the trunk, add a head, and the
   frozen layers still change. Name two Keras mechanics that could
   cause that.
6. How is unsupervised pretraining different from calling a closed
   language model? Give one case where a pretext task would help
   your labeled set, and one where it would not.
7. AdaGrad vs RMSProp vs Adam: which accumulator dies, which
   decays, and which also keeps a velocity? When might you still
   pick SGD with momentum?
8. Dropout at train, MC dropout at serve: what is the extra
   average buying you, and what is it not?
9. Using the guidelines table, pick a first stack for tabular
   data with 8k rows. Which rows of the table would you skip if
   you copied an ImageNet recipe?
10. A val curve is great, production is not. Which of this
    chapter's tools (BN moving averages, dropout train/serve,
    frozen vs unfrozen BN) is your first suspect?
