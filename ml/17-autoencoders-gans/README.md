# 17. Autoencoders and GANs

Companion notes for **Chapter 17** of *Hands-On Machine Learning with
Scikit-Learn, Keras, and TensorFlow* (2nd edition, Aurélien Géron;
O'Reilly, 2019).

Sequence models map tokens to tokens. This chapter is **unsupervised
representation and generation**: squeeze an input through a
bottleneck, reconstruct it, optionally sample new inputs from a latent
space, or pit two nets against each other until one of them fakes the
data distribution. Skip it and you will call every unlabeled net "a
GAN," use a linear autoencoder and think you invented PCA, or ship a
generator whose loss went down while the pictures went to noise.

## The mental model

Two families share a chapter because both learn without class labels.
They do not share a training loop.

```
  AUTOENCODER                         GAN
  -----------                         ---
  x --> [encoder] --> z               noise z --> [generator] --> x_fake
           |                                              |
           v                                              v
       bottleneck                              [discriminator]
           |                                   real x vs x_fake
           v                                          |
    [decoder] --> x_hat                                 v
       |                                        "real or fake?"
       v
  L(x, x_hat)  (+ KL if VAE)

  undercomplete z: must throw information away
  VAE z: a *distribution* you can sample
  GAN z: a handle the generator invents; no decoder of x
```

The one sentence to remember a year from now: **an autoencoder is a
compressor you train with a reconstruction (and maybe a prior); a GAN
is a two-player game that does not reconstruct — it tries to match a
distribution.**

If you need a code you can **index, threshold, or sample**, start with
autoencoders (and VAEs). If you need **samples that look like the
training set** and you are in 2019, the chapter's answer is GANs. If
you are in 2026, read **What aged** before you pick a generator.

## Efficient representations

Raw inputs (pixels, readings, bag-of-words counts) are wide,
redundant, and unlabeled. You still want a shorter vector that
preserves what matters.

Build a mapping `x → z → x̂` and train so `x̂` is close to `x`. If `z`
is lower-dimensional than `x` (**undercomplete**), the net cannot
copy. It has to throw away noise and keep structure. That `z` is the
representation: downstream clustering, visualization, compression,
anomaly scores (`||x - x̂||` is large when `x` is weird).

This is the unsupervised twin of "learn features, then classify."
Earlier dimensionality-reduction chapters gave you PCA and manifolds
as linear/kernel geometry. Here the encoder and decoder can be deep
and nonlinear.

A bottleneck that is still wide enough to memorize produces perfect
reconstructions and a useless `z`. Using the reconstruction loss as a
quality score for *generation* is another trap: a sharp reconstruction
of the training set is not a diverse sample of the data manifold.

## Linear undercomplete autoencoders ≈ PCA

If the encoder and decoder are **linear** and you minimize mean squared
error with an undercomplete `z`, the space you get is the PCA subspace
(same span; the actual axes may be rotated). That is the point of the
exercise.

People open Keras, stack Dense layers with linear activations, and
announce a deep representation. Treat the linear undercomplete AE as a
**unit test of your understanding**. If you cannot recover PCA-like
reconstructions, your training loop is wrong before you add ReLUs.

On linear Gaussian-ish data, PCA is cheaper, unique (up to sign), and
easier to explain. Use a deep AE when the manifold is bent.

## Stacked autoencoders

One linear map cannot untangle a bent manifold. You want depth. Stack
nonlinear layers in the encoder, mirror them in the decoder, train
end-to-end with a reconstruction loss. "Stacked" in this chapter means
**deep**, not a new algorithm. You inspect:

- **Reconstructions.** Do hold-out images look like the inputs, or
  like a blurry average? Blurry means the bottleneck or the decoder
  lacks capacity; exact copies on train plus garbage on hold-out means
  memorization.
- **The latent plane.** If you squeeze `z` to 2-D, you can scatter
  the dataset. Fashion-MNIST-style clothes should clump. If they do
  not, the representation is not using the two dimensions you paid for.

### Unsupervised pretraining

In the era this book records, deep supervised nets were hungry for
labels. A stacked AE gave you encoder weights **trained on unlabeled
`x`**. You then chopped the decoder, stuck a classifier on `z`, and
fine-tuned. Earlier training chapters already named this pattern.

Running greedy pretraining in 2026 as a first choice is usually the
wrong default. Residual nets, better init, batch-norm, and huge labeled
(or weakly labeled) sets made the trick mostly historical for vision
classifiers. The *idea* — use unlabeled data to get a representation —
did not die; it moved (contrastive learning, masked autoencoders,
generative pretraining on text).

### Tying weights

The decoder matrix can be the transpose of the encoder matrix (per
layer). Fewer parameters, a hard-coded "undo," less overfitting on
small sets.

Tying weights and then changing encoder/decoder widths so the
transpose is the wrong shape breaks the constraint. Tying is a
constraint, not a free regularization you sprinkle after a random
architecture.

### Greedy layer-wise training (historical)

Train a tiny AE on `x`, freeze it, train the next AE on that code,
stack, then optionally fine-tune. This is how deep AEs were once made
trainable. You should recognize the name so old papers do not look
like magic. You should not start a new stack this way without a
reason: end-to-end training with modern init usually wins.

Stopping after layer-wise training and never fine-tuning the stack
leaves each layer optimized for a local reconstruction that can be a
poor global code.

## Convolutional, recurrent, denoising, sparse

The bottleneck idea is architecture-agnostic. Match the encoder to
the data, as in the rest of Part II.

- **Convolutional AE.** Conv encoder, conv-transpose (or upsample +
  conv) decoder. Images should not be flattened into a Dense soup if
  you care about locality.
- **Recurrent AE.** Sequence in, a code (last state or a pooled
  sequence), sequence out. Related to encoder-decoder translation, but
  the target is the **input sequence** (or a reconstruction of it).
- **Denoising AE.** Corrupt `x` (noise, dropout of pixels, masking)
  and ask the decoder to recover the **clean** `x`. The code cannot
  store a pixel-perfect copy of the input it saw; it has to store
  structure. This is the ancestor of a lot of later "mask and
  predict" work.
- **Sparse AE.** Keep `z` wide (even overcomplete) but penalize
  activations so only a few units fire. Sparsity is another way to
  force an information bottleneck when you do not want a tiny `z`.

A vanilla AE copies `x` through a too-wide `z`. Add a constraint that
is not just "small `z`": noise, sparsity, a variational prior, or a
tiny code. Pick one constraint you can measure.

Denoising with so much noise that the only survivable reconstruction
is the dataset mean, or sparse penalties so strong that `z` is all
zeros and the decoder learns a constant, both destroy the
representation. Always look at reconstructions **and** at the
distribution of `z`, not just the scalar loss.

## Variational autoencoders

A plain AE maps `x` to a point `z`. If you want to **generate**, you
need to sample `z` from somewhere the decoder understands. Nothing
forces the point codes of a plain AE to fill a nice region; they can
be a spiky cloud with holes. Sampling between two codes then decodes
as garbage.

You want a decoder that defines a distribution over `x`, and a latent
space you can sample. The encoder outputs a **mean and a log-variance**
per example, not a point. You sample `z` with the reparameterization
trick (`z = μ + σ ⊙ ε`, `ε ~ N(0, I)`) so gradients flow. The loss is
reconstruction (how well `x` is decoded from that `z`) plus a KL term
that pushes `(μ, σ)` toward a simple prior, usually `N(0, I)`.

At generation time you skip the encoder: draw `z` from the prior, run
the decoder. At representation time you can use `μ` as the code.

Watch the two loss terms separately. KL that dies (`σ → 0`, encoder
"cheats" back toward a point AE) or KL that dominates (decoder ignores
`z`, reconstructions are bland averages — posterior collapse) are the
usual failures. Treating VAE samples as a fidelity competition against
GANs misses the point: classic VAEs were blurrier; they bought a
**structured latent** and a proper probabilistic story.

The reparameterization trick is the implementation hinge. If you sample
`z` with a non-differentiable draw and no path around it, the encoder
does not learn.

## Generative adversarial networks

You want samples from `p_data` and you do not want to write `p_data`.
Reconstruction losses tend to average; images look blurry.

Two nets. The **generator** maps noise `z` to `x_fake`. The
**discriminator** sees real `x` and `x_fake` and tries to tell them
apart. The generator is trained to fool the discriminator. There is
no decoder of a given `x` and no explicit reconstruction of training
rows.

```
  z ~ noise  -->  G  -->  x_fake  --+
                                    +-->  D  -->  real / fake
  x ~ data   ---------------------+
```

Nash equilibrium, in the toy analysis, is when `G` matches the data
distribution and `D` is reduced to chance. You will not sit at that
equilibrium in a notebook. You will oscillate around it.

### Training difficulties

GANs are famous because they fail in characteristic ways:

- **Mode collapse.** `G` finds one image (or a few) that fools `D` and
  parks there. Loss can look "solved." Diversity is dead. Always look
  at a *grid* of samples, not one pretty cherry-pick.
- **Vanishing generator gradient.** If `D` becomes perfect too early,
  `G` gets no useful gradient. The game has to stay slightly unfair
  in both directions.
- **Unstable hyperparameters.** Learning rates, update ratios (`k`
  discriminator steps per generator step), and saturation of the
  discriminator loss all move the same needle. A trick that stabilizes
  one dataset will destabilize another.
- **No unique scalar that means "good."** Unlike AE reconstruction,
  you cannot stop at "loss < ε." You need samples, maybe an FID-class
  score (later than some 2019 notebooks), and domain checks.

Reporting generator loss going down as success is unreliable. The
generator loss going down can mean `D` got worse. Trust images and
coverage of modes.

### DCGAN

Deep Convolutional GANs put conv / conv-transpose stacks, batch-norm,
and specific activations into `G` and `D` so image GANs have a chance
of training. Treat DCGAN as the **2016–2018 baseline recipe** this
chapter expects you to recognize: fully connected GANs on pixels were
already known to be miserable.

### Progressive growing and StyleGAN (2019 snapshot)

- **Progressive growing.** Start at 4×4, train until that game is
  stable, fade in the next resolution, repeat up to 1024×1024-class
  faces. The generator is not asked to invent a megapixel face on
  day one.
- **StyleGAN (2019).** A mapping network sends `z` through an MLP
  into an intermediate **style** space `w`. Styles modulate layers
  of the synthesis network (AdaIN-class affine scales). Extra noise
  injects stochastic detail (hair, pores). Style mixing (different
  `w` at different resolutions) is how the paper showed coarse vs
  fine control. This is the photorealistic-face story the 2e book
  can close on.

Implementing "StyleGAN" as "a DCGAN with a prettier name" skips the
design: mapping network, style space, and noise inputs. Also: training
these recipes from scratch is a research compute budget, not a weekend
homework, once you leave tiny toy images.

## What aged since 2019

- **Diffusion (and flow-matching) models took the photorealistic
  generation crown** for images, and a large share of audio/video.
  The 2020s default for "make me a picture" is a diffusion-class
  model. If you are choosing a generator in 2026, start from that
  fact, then read the next bullets before you delete GANs from your
  brain.
- **GANs are not dead.** They still show up where you want a fast
  forward pass (one shot, no 50-step denoising), some graphics and
  compression hybrids, certain small-data domains, and as
  discriminators inside other systems. StyleGAN-class work continued
  after 2019; it is no longer the default news.
- **Autoencoders are still useful** for compression, denoising,
  anomaly/novelty scores, and as **encoders of a latent space**.
  Modern latent diffusion *uses* a VAE-class encoder so the diffusion
  model can run in a cheaper `z`. The AE changed jobs; it did not
  disappear.
- **Greedy layer-wise AE pretraining** is a historical algorithm for
  deep nets. Unsupervised / self-supervised pretraining is very much
  alive; the recipe is different.
- **VAE research moved** (hierarchical latents, better priors, VQ-VAEs
  for discrete codes). The μ / σ / KL picture in this chapter is still
  the one you should be able to draw from memory.
- **Evaluation.** Inception-ish scores and FID became table stakes
  after many 2016 GAN papers. They are still gameable. Domain metrics
  beat a leaderboard number you cannot explain.

Learn reconstruction, latents, and adversarial games here. Pick the
2026 generator from a current survey, not from this chapter's last
section alone.

## Check yourself

1. Why does an undercomplete bottleneck *have* to throw information
   away, and what goes wrong if you widen `z` until reconstructions
   are perfect?
2. When is a linear autoencoder allowed to be "just PCA"? When would
   you still prefer PCA?
3. You plot reconstructions and they look great on the training set
   only. Is the representation good? What else would you plot?
4. Weight tying: what is the constraint, and when is it the wrong
   shape?
5. Greedy layer-wise training: what problem did it solve in the
   2000s, and why is it usually the wrong default now?
6. Denoising vs sparse vs undercomplete: three ways to stop copying.
   Pick one and name a failure if you turn that knob too far.
7. A VAE encoder emits `μ` and `log σ²`. Why do you sample with
   `μ + σ ⊙ ε` instead of drawing `z ~ N(μ, σ²)` as a black box?
   What does the KL term buy you at *generation* time?
8. Posterior collapse vs a collapsed `σ → 0`: which loss term won,
   and what do samples / reconstructions look like?
9. GAN mode collapse vs a perfect discriminator: which failure is
   "pretty but identical faces," and which is "the generator cannot
   learn"? Why is generator loss a bad dashboard by itself?
10. DCGAN vs progressive growing vs StyleGAN: what problem does each
    recipe add on top of "two nets in a loop"? What took the
    photorealistic crown after this edition?
