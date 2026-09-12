# 14. Deep computer vision using CNNs

Companion notes for **Chapter 14** of *Hands-On Machine Learning with
Scikit-Learn, Keras, and TensorFlow* (2nd edition, Aurélien Géron;
O'Reilly, 2019).

[Chapter 11](../11-training-dnns/) taught you to train deep stacks.
This chapter is the stack that *fits images*: convolution, pooling,
the architectures that made ImageNet a solved-enough demo, and the
jump from a class score to a box or a mask. Skip it and you will
flatten a 256×256 photo into a dense MLP, OOM, then download a
vision API and call that "our CNN."

**See also (do not merge):** transfer of a *vision backbone you
fine-tune* is this chapter plus [ch. 11](../11-training-dnns/).
Calling a frozen multimodal API is
[agents ch. 2](../../agents/2-llms-prompting-agents/). Do not
rewrite detection as an agent tool chapter.

## The mental model

A dense layer on pixels treats location as a myth: weight 0,0 is
unrelated to weight 0,1. A conv layer reuses a small filter at
every position. Depth of the stack is how "edge" becomes "object."

```
  image  [H, W, C]
     |
     v
  CONV  (K filters, kxk, stride, pad)  -->  K feature maps
     |
     v
  POOL  (cheap downsample; some translation tolerance)
     |
     v
  stack of CONV/POOL  -->  small grid of rich channels
     |
     +--> GAP / flatten --> Dense --> class scores
     +--> extra head --> box (localization / detection)
     +--> upsample / FCN --> per-pixel classes (segmentation)
```

The one sentence to remember a year from now: **a conv net is
translation-friendly parameter sharing plus a hierarchy of
feature maps** — classification is one head on that stack;
localization, detection, and segmentation are other heads (or
other decoding paths) on the same idea.

Two consequences fall out of that diagram. First, the memory
bill is feature maps, not "I only have 3×3 filters": `H×W×K`
activations can dwarf the filter weights. Second, pretrained
ImageNet backbones are this chapter's transfer story; they are
not a substitute for knowing what stride and padding did to
your spatial grid.

## Visual cortex, sketched

The biological cartoon that motivated conv nets: local receptive
fields, some neurons that fire on oriented edges, a hierarchy
toward more complex patterns, two visual streams (roughly "what"
and "where"). You do not need the neuroscience. You need the
engineering translation:

- **Local receptive field** → a filter that only sees a patch.
- **Same weights everywhere** → one filter slides.
- **Many filters** → many feature maps.
- **Subsampling** → pooling / stride.

**Failure mode** — treating the cartoon as a proof that your
net "sees like a human." It sees like a stack of correlations
trained on your labels.

## Convolutional layers

A **filter** (kernel) is a small tensor `[k, k, C_in]` (or
`[k, k, C_in, C_out]` packed). Slide it with a **stride**.
**Padding** (`valid` vs `same`) decides whether the spatial
grid shrinks.

Each filter produces one **feature map**. A layer with 32
filters produces 32 maps. Later layers take maps as channels:
the "pixels" become "patterns of patterns."

```
  output[h, w, k] = < input patch at (h, w) , filter_k > + b_k
```

**Problem** — A Dense layer on `H*W*C` inputs is too many
weights and ignores that a cat shifted one pixel is still a
cat.

**Solution** — Share the filter. Parameter count is
`k*k*C_in*C_out`, independent of `H` and `W` (until you
flatten).

**Memory.** Weights can be small. Activations are not:

```
  maps ≈ batch * H * W * K * bytes_per_act
```

High-res inputs, large `K`, and naive "same" padding at stride
1 are how a "tiny 3×3 net" fills the GPU. Downsample on
purpose (stride, pooling) when the task allows.

**Failure mode** — Stride and padding you never computed, so a
skip connection in a ResNet-shaped block is 17×17 vs 16×16 and
the add fails. Draw the spatial shape after every layer once.

Channels-last (`NHWC`) vs channels-first (`NCHW`) is a device
convention. Keras hides it until an op or a pretrained weight
file does not.

## Pooling

Pooling downsamples a local window (max or mean), usually
2×2 stride 2.

**Problem** — The grid is still huge; you want some invariance
to a one-pixel shift.

**Solution** — Max-pool is the 2010s default: cheap, keeps the
strongest response. Average-pool (and **global** average pool
before a head) showed up as a way to kill giant Dense layers
on flattened maps.

**Failure mode** — Pooling away the spatial grid and then
trying to localize. Detection and segmentation *need* layout.
Classification can afford to throw it away at the end. Also:
overlapping pool with weird strides as a cargo-cult copy from
AlexNet when your map is already tiny.

Modern stacks often **stride in the conv** instead of a
separate pool. Same job: shrink `H, W`, grow `K`.

## Architectures — what each introduced

Do not memorize paper recaps. Remember the *move* each name
added to the toolkit. You will mix them.

| Name | The move |
|---|---|
| **LeNet-5** | Conv + pool stacks, then Dense, on tiny digits. The template: local features, downsample, classify. |
| **AlexNet** | Depth + ReLU + dropout + GPU scale + heavy aug on ImageNet. "This can actually win." |
| **GoogLeNet / Inception** | Parallel branches (1×1, 3×3, 5×5, pool) in a module; **1×1 bottlenecks** to cut channel cost; global average pool instead of a giant Dense brain. |
| **VGG** | Only 3×3 convs, stacked, with a very regular "double channels when you halve spatial" ladder. Depth by repetition, not by clever branches. |
| **ResNet** | **Residual skip**: learn `F(x)` on top of `x` so a deep stack can default to identity. Makes 34 / 50 / 152 layers trainable. |
| **Xception** | Inception idea pushed to **depthwise separable** convs (spatial then pointwise). Extreme channel/space split; fewer params per effective depth. |
| **SENet** | **Squeeze-excitation**: a tiny MLP on pooled channels that rescales feature maps. Channel-wise attention, not a Transformer. |

**Problem** — A blog says "we used ResNet" and you cannot tell
whether they meant skips, ImageNet weights, or just a 50 in
the filename.

**Solution** — Name the move. If you did not use residual
adds, you did not use ResNet; you used a deep conv stack.

**Failure mode** — Stacking every move at once (SE +
separable + 1×1 + skips) without a reason, then declaring
"architecture search." Ablate from a known backbone.

Inception's 1×1 is not "a useless conv." It is a learned
channel mixer / dimensionality reducer so the 3×3 is affordable.
Depthwise separable is that split taken literally.

## A ResNet-34-shaped idea

You do not need to type 34 layers from memory. You need the
block:

```
  x --> CONV --> BN --> ReLU --> CONV --> BN --> (+) --> ReLU
  |                                           ^
  +---------------- (identity or 1x1 proj) ---+
```

When stride or channel count changes, the skip **projects**
(1×1 conv) so the add is legal. Stages: a conv stem, then
groups of blocks that keep spatial size, then a stride-2
transition, repeat, then global pool and a Dense head.

**Problem** — A 34-layer VGG-shaped stack without skips
trains poorly (ch. 11 vanishing story, plus degradation).

**Solution** — Identity skips. The optimizer can choose "do
nothing" per block.

**Failure mode** — A skip that bypasses BN/ReLU in a way you
did not intend, or a projection skip that is accidentally
the *main* path (huge 1×1) so you no longer have a residual
learning problem.

Once the block is boring, Keras applications (or today's
`timm` / `torchvision` equivalents) are how you get a
ResNet-50 you did not mis-type.

## Pretrained models and transfer

`keras.applications` (2019-shaped) ships ImageNet classifiers
with weights: ResNet, Xception, VGG, Inception, … You strip
the head, optionally freeze, attach a new classifier, and
follow the freeze-then-unfreeze recipe from
[ch. 11](../11-training-dnns/).

**Problem** — Your labeled set is 2,000 medical images.

**Solution** — Start from a backbone that already knows
generic visual statistics. Replace the 1000-way head. Use
the preprocessing the backbone was trained with (mean, scale,
RGB order). That preprocessing is part of the weight file.

**Failure mode** — ImageNet preprocess skipped, so a
"pretrained" net sees pixels in the wrong numeric range.
Or unfreezing the whole backbone at Adam 1e-3 on a tiny set
and erasing the prior in an epoch.

Transfer here is **your labels, your head, their trunk**.
It is not CLIP-as-a-service unless you are explicitly using
that model as a backbone you still train or probe.

## Classification vs localization vs detection vs segmentation

Same backbone, different outputs.

- **Classification** — one (or a few) labels for the whole
  image. Head: pool + Dense / softmax.
- **Localization** — class *plus* a box (usually four
  numbers: center, size, or corners). Two losses: one for
  the class, one for the box. One object assumed.
- **Detection** — *many* boxes, unknown count, plus classes
  (and a "no object" / objectness story). This is a matching
  problem, not a 4-number regression on a single crop.
- **Semantic segmentation** — a class **per pixel**. Layout
  is the output. Instance segmentation (this edition only
  brushes) also wants *which* object.

**Problem** — Product language says "detect" for a single
centered object.

**Solution** — If the count is always one and the crop is
honest, it is localization. If the scene is cluttered, it
is detection. If you need the silhouette, it is
segmentation.

**Failure mode** — A classifier with a sliding window called
"YOLO" because it is fast in a slide deck.

### Detection intuition: FCN and YOLO

**Fully convolutional** — replace Dense with 1×1 convs so
the net runs on a larger map and outputs a grid of scores
(a coarse heatmap). Sliding a classifier becomes *one*
forward pass. FCNs are also the backbone idea for
segmentation (upsample the coarse grid).

**YOLO-shaped** (2016-era intuition, not today's paper):
split the image into a grid; each cell predicts boxes and
class probabilities in one shot. Speed from "one pass, no
separate proposal network." Cost: a matching / assignment
rule at train time, and a struggle with tiny or crowded
objects in the early versions.

You do not need to implement YOLO from scratch in a
workshop. You need to know why detection is **assignment +
box + class + maybe objectness**, not `Dense(4)`.

**Failure mode** — Training detection with classification
cross-entropy only, no box loss, no empty-cell handling.
The net will happily shout the majority class in every
cell.

### Semantic segmentation

Encoder (conv/pool, shrink spatially, grow channels) +
decoder (upsample, skip from encoder so edges come back).
Loss is usually per-pixel cross-entropy (class imbalance
will haunt you: the sky has more pixels than the bicycle).

**Failure mode** — Accuracy as the metric on a 5% foreground
task. Use IoU / Dice-shaped scores. Resize labels with
nearest-neighbor, not bilinear, or class ids become 1.4.

## What aged since 2019

- **Vision Transformers (ViT) and ConvNeXt** reopened the
  "conv vs attention" argument. ConvNets did not die;
  ConvNeXt is a conv net that stole transformer training
  recipes. This chapter's hierarchy-of-maps still describes
  a ResNet; a ViT is a different inductive bias
  ([ch. 16](../16-nlp-attention/) for attention as a *layer*).
- **The YOLO family moved a lot** (v3 → … → whatever number
  is fashionable). One-shot grid detection is still the
  intuition; the matching losses, necks, and backbones are
  not the 2016 diagram. Use a maintained implementation.
- **`keras.applications` is no longer the default zoo** for
  many teams. `torchvision.models` and `timm` (and JAX
  equivalents) are where new weights land first. The
  *transfer recipe* is unchanged: trunk, head, freeze,
  preprocess that matches the weights.
- **Foundation vision models** (CLIP, SAM, SigLIP) changed
  what you fine-tune *for*. They are still pretrained
  backbones plus a head if you train them; they are Agents-
  track APIs if you only call them.
- Depthwise separable convs and residual blocks are still
  how you read a mobile backbone. SE-style channel gates
  evolved into various attention blocks; the original move
  is enough for this chapter.

## Check yourself

1. A Dense net and a conv net see the same 64×64×3 image.
   What is shared in the conv, and why does a one-pixel
   shift hurt the Dense net more?
2. Compute (roughly) activation memory for batch 32, maps
   112×112×256, float32. Why might that dwarf the 3×3
   weights in that layer?
3. Valid vs same padding, stride 2: which one lets you
   predict the output grid on paper, and why do residual
   adds care?
4. When would you refuse pooling (or aggressive stride)
   even though it would save memory?
5. For each of LeNet, AlexNet, Inception, VGG, ResNet,
   Xception, SENet, write *one clause* that is the move.
   Which move are you using if your block is `x + F(x)`?
6. Why does a 1×1 conv exist in GoogLeNet if it does not
   look at neighbors?
7. You load an ImageNet ResNet, swap the head for two
   classes, and val accuracy is chance. Name three
   preprocessing / freeze bugs before you blame Adam.
8. Classification vs localization vs detection: which
   output tensor ranks, and which extra problem does
   detection add that a four-number head does not?
9. In FCN terms, what did you replace, and why does that
   turn sliding windows into one forward pass? What does
   YOLO still have to *assign* at train time?
10. A segmentation IoU is terrible while pixel accuracy
    is 95%. What did the majority class do, and which
    label-resize interpolation would silently corrupt
    the masks?

Continue to [Processing sequences using RNNs and CNNs](../15-sequences-rnns/).
