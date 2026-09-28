# 8. Dimensionality Reduction

Companion notes for **Chapter 8** of *Hands-On Machine Learning with
Scikit-Learn, Keras, and TensorFlow* (2nd edition, Aurélien Géron;
O'Reilly, 2019).

Part I so far fitted models in the feature space you were given.
This chapter is about not using all those axes. Skip it and you will
either drown a distance-based model in empty high-d space, or you
will run t-SNE on 200 columns and call the 2-D plot "the data."
Projection and manifold methods are different bets. PCA is a
variance machine. LLE unrolls. Neither one is a classifier by
itself.

## The mental model

High dimension is not "more information." It is a geometry where
distances, densities, and "nearest" stop meaning what they meant in
2-D cartoons.

```
  x in R^m   (m large)
       |
       |  BET: the useful signal lives in a smaller structure
       v
  +------------------+     +------------------+
  | LINEAR SUBSPACE  |     | NONLINEAR SHEET  |
  | (projection /    |     | (manifold: Swiss |
  |  PCA axes)       |     |  roll, LLE,     |
  |                  |     |  kernel PCA...) |
  +--------+---------+     +--------+---------+
           |                        |
           v                        v
        z in R^d                 z in R^d
        d << m                   d << m
           |                        |
           +----->  model, plot, compress, invert?
```

The sentence to keep a year from now: **dimensionality reduction is
a bet that a low-d structure carries the signal** — a flat subspace
(PCA) or a curved sheet (manifold methods) — and every method is
allowed to destroy a kind of structure it did not bet on.

Two consequences fall straight out of that diagram. First, "we
reduced to 2-D" is not a model; it is a **lossy view**. t-SNE's view
is for eyeballs; PCA's view can be inverted and used as features.
Second, if the bet is wrong (classes sit in directions of *small*
variance, or the roll needs an unroller and you used a projection),
you will fit a clean model on the wrong object and call the domain
impossible.

## Curse of dimensionality

A pipeline that worked in 8 features is pointed at 800. k-NN,
clustering, and "distance to centroid" quietly rot. You add
regularization and still watch metrics fall. Name the geometry, then
either get more rows (rarely enough), engineer, or reduce.

Pictures that should live in your head:

- A unit hypercube's **volume concentrates near the surface** as *m*
  grows. Most points are "on the edge," not in a cozy interior.
- Random points' **distances concentrate**: nearest and farthest
  neighbor look similar. "Nearest" becomes a fragile ranking.
- To keep the same sample density you need a number of rows that
  grows **exponentially** in *m*. You will not get that dump.

This is why high-d models prefer **structure** (sparsity, linear
subspaces, smoothness, a manifold) over "the space of all pixels."
It is also why adding a pile of noisy columns is not free: you did
not only add signal. You added empty volume.

Standardizing 500 columns does not solve the curse. Scaling puts
features on one numeric range. It does not put mass back in the
middle of the cube. The curse is the *reason this chapter exists*.
It is not a reason to PCA every table by reflex. Regularized linear
models and forests that sample features are other answers. Reduce
when you need a smaller representation, a plot, compression, or a
distance that works again.

## Projection vs manifold

Two different bets. Mixing their names is how a Swiss roll gets
flattened through the roll instead of unrolled.

**Projection.** Assume the data sit near a **linear subspace**: a
plane in 3-D, a 40-d hyperplane in 784-d pixels. Drop the axes of
little variance (or little signal, if you have a supervised cousin
like LDA). PCA is the workhorse. Fast, invertible (with
reconstruction error), linear.

**Manifold.** Assume the data sit near a **nonlinear low-d sheet**
embedded in high-d space. The Swiss roll is the teaching toy: 2-D
neighborhoods, rolled into 3-D. A linear projection through the roll
*can squash two coils onto each other*. You wanted an unroll, not a
shadow.

```
  Swiss roll (side view)     projection (bad bet)    unroll (manifold)
      @@@@@@@@                 ####                  --------
     @        @                ####                  --------
      @@@@@@@@                 ####                  --------
```

"Manifold learning" is sometimes used as a synonym for any
`n_components=2`. Ask instead: if I rotate and slice with a
hyperplane, do I keep the thing I care about? If yes, try PCA
first. If the structure is a roll, a ring, a bent surface, you need
a neighborhood-preserving method (LLE, Isomap, kernel PCA with the
right kernel) — and you should expect pain: slower, fussier, often
not invertible.

Running a manifold method because the 2-D plot looks cooler, then
feeding those 2 coordinates into a production classifier with no
inverse, no stable out-of-sample map, and hyperparameters you
re-tune by staring, confuses two jobs. Visualization and **feature
extraction for a model** are different. PCA (and sometimes kernel
PCA) can do both. t-SNE is almost only the first.

## PCA

**Principal Component Analysis** finds an orthogonal set of axes
(principal components) aligned with **variance** of the centered
data. PC1: direction of greatest variance. PC2: greatest remaining
variance orthogonal to PC1. And so on.

```
  center X  (subtract mean)
       |
       v
  SVD / eigen of covariance  ->  components  c1, c2, ..., c_m
       |
       v
  z = X_centered · W_d     W_d = first d components
```

Variance is a **stand-in for signal**. That stand-in is excellent
when noise is isotropic and the interesting spread is large. It is a
betrayal when the class boundary lives in a low-variance direction
(a thin gap between two pancakes). PCA will happily keep the pancake
and discard the gap.

### Components and projection to d

sklearn: `PCA(n_components=d).fit_transform(X)`. After fit you have:

- `components_` — the axes (rows are components)
- `explained_variance_` / `explained_variance_ratio_` — how much of
  the centered variance each axis holds
- `mean_` — what you must subtract (and add back to invert)

Projecting to *d* dimensions: keep the first *d* components, drop
the tail. Reconstructing: map back with `inverse_transform` (or the
corresponding matrix product) and you get a **lossy** version of X.
The dropped variance is the reconstruction error's cousin.

Fitting PCA on train+test together "because it is unsupervised" is
still leakage. It is a transformer. `fit` on the training fold only,
`transform` the rest. Otherwise the PCs peek at hold-out variance.
Pipe it (`Pipeline([("pca", PCA(...)), ("model", ...)])`).

Forgetting to center matters for hand-rolled SVD (sklearn PCA
centers for you). PCA on unscaled features with incompatible units
(meters vs milliseconds) is a variance-of-the-unit contest. Scale
when units are arbitrary. Do not scale when the unit *is* the signal
(e.g. you want a high-variance pixel to dominate).

### Explained variance ratio and choosing d

`explained_variance_ratio_` is the fraction of **total centered
variance** along each PC. The cumulative sum is the usual plot:

```
  cumulative variance
  1.0 |                ________
      |         ______/
  0.95|    ____/
      | __/
      +---------------- d
         elbow?   95%?
```

Ways to pick *d* that are not "2 because we have a screen":

- **Variance budget** — keep enough PCs to retain 95% (or 99%) of
  variance. A default, not a law.
- **Elbow** — where extra PCs stop buying variance. Subjective,
  often fine for visualization.
- **Downstream metric** — grid-search `n_components` inside a
  pipeline against the **task** loss. This is the grown-up choice
  when PCA is a preprocessor, not a plot.

Keeping 95% variance and then being shocked that a rare-class signal
died is a common surprise. Rare is often low-variance. If you need
that class, PCA-as-compression is the wrong bet, or you need a
supervised reduction (LDA, or "choose *d* by F1").

### Compression

PCA as a **codec**: store `mean_`, `W_d`, and the *d* coordinates
per row instead of *m*. Reconstruct when you need an approximate
original. On images this is the blurry-but-recognizable gallery.
Reconstruction MSE is a measure of **codec loss**, not of
classification quality.

Use compression when storage, RAM, or a linear model's *m* is the
constraint. Do not use it as a ritual before every forest: trees
already sample features, and a rotation can hurt axis-aligned
splits. PCA-before-trees is a hypothesis, not a default.

### Randomized PCA and incremental PCA

Full SVD on a huge dense `X` is the bill you notice.

- **Randomized PCA** (`svd_solver="randomized"`): approximate the
  first *d* components when *d* is much smaller than *m* and *n* is
  large. Teaching point: you do not always need the exact tail of
  the spectrum to keep the head. sklearn may pick randomized for you
  on large problems; know that it exists so a slightly different
  `components_` is not a "bug."
- **Incremental PCA**: `IncrementalPCA` + `partial_fit` (or
  `np.array_split` mini-batches). For data that **does not fit in
  RAM**. You trade a bit of exactness and a batch-size knob for a
  PCA you can actually run. This is mini-batch learning applied to a
  variance decomposition.

Incremental PCA with a tiny batch, then interpreting the last PCs as
if they were a full SVD, overclaims. Use it to get a working *d*-d
representation. Do not write a paper on PC-400 from a streaming
approximation unless you checked a subsampled full SVD.

## Kernel PCA

Linear PCA cannot unroll a roll. **Kernel PCA** does PCA in a
**feature space** induced by a kernel (linear, RBF, sigmoid, …)
without building that space explicitly — the same kernel trick as
SVMs, now unsupervised.

RBF kernel PCA is the usual teaching demo: a circle or roll that
becomes linearly spread in the kernel space, so the **top kernel
PCs** separate what a line in the original coordinates could not.

**Selecting the kernel and its hyperparameters** (γ for RBF,
`n_components`) is the hard part because there is no obvious
unsupervised loss that matches your *task*:

- If PCA is a **preprocessor for a labeled task**, grid-search
  kernel + γ + *d* against that task's CV metric. Honest and often
  best.
- Reconstruction error is trickier than linear PCA: you need a
  **pre-image** (approximate inverse) to say "how far is the
  reconstruction in original space." sklearn's
  `KernelPCA(fit_inverse_transform=True)` is an extra, imperfect
  model. Treat that error as a proxy, not as MSE from linear
  `inverse_transform`.

Picking γ by which 2-D plot looks like blobs is a weak selection
rule. Freeze a downstream protocol (pipeline + CV). Plots are for
sanity ("did we actually unroll") not for γ. Kernel PCA on *n* in
the hundreds of thousands is painful: the kernel matrix is *n × n*
in spirit. This is a medium-data tool, or a subsampled one. For
large nonlinear compression, autoencoders are the industrial path.

## LLE

**Locally Linear Embedding** is a manifold method that does not go
through a kernel matrix in the SVM sense. The bet:

1. Each point is a **linear combination of a few neighbors** in the
   original space.
2. Find low-d coordinates where those **same local weights** still
   reconstruct each point.

```
  for each x_i:
      find k neighbors
      fit weights w that rebuild x_i from them
  find z_i in R^d that keep those w's
```

LLE is good at **unrolling** a roll when neighborhoods stay on the
sheet (choose *k* with care: too small is a shattered graph; too
large jumps off the roll and "sees" the chord). It is not great at
projecting new points unless you add an out-of-sample trick;
sklearn's `LocallyLinearEmbedding` can transform new data with an
approximate map — check it, do not assume it is as cheap and stable
as PCA's `transform`.

LLE as a production feature step on a stream of new rows, without
checking that the out-of-sample map still unrolls, is a surprise
waiting to happen. For training-set visualization, LLE is a
legitimate tool. For a live pipeline, PCA, kernel PCA, or
autoencoders are the less surprising sockets.

## Other techniques surveyed

Names and **jobs**, not derivations. You should know which drawer to
open.

| Method | Job |
|---|---|
| **t-SNE** | Visualize: keep **local** neighborhoods, make clusters pop in 2-D/3-D. Distances between far clusters are not a metric you should report. Hyperparameters (perplexity) change the story. Not a general-purpose compressor. |
| **MDS** | Place points in low-d so **pairwise distances** stay close to the originals (classical MDS is cousin to PCA on a distance matrix). |
| **Isomap** | Manifold: replace Euclidean chords with **geodesic** (shortest path on a neighborhood graph), then MDS-like embedding. Unrolls when the graph stays on the sheet. |
| **LDA** | **Supervised** linear projection: axes that separate **labeled classes**, not axes of variance. Use as a reducer when labels exist and class-Gaussian-ish is not a terrible cartoon. It is also a classifier. Do not call it unsupervised PCA. |

UMAP is the post-2019 visualization default in many labs; it lives
in the aged section, not in the 2e's drawer of names.

A paper-style figure of t-SNE blobs used as evidence that "the model
will generalize" overclaims. t-SNE is allowed to pull a continuous
cloud into islands. Use it to **look**, then measure on a task. If
you need a reducer as a transformer, prefer PCA / kernel PCA / LDA
(when labeled) / later autoencoders. Feeding t-SNE coordinates into
a classifier in production is a demo, not a feature store: there is
no cheap, stable `transform` that matches the plot you stared at,
and the embedding can change with perplexity and random init.

## What aged since 2019

- **PCA APIs.** Randomized solver selection, `PCA(n_components=0.95)`
  as a variance budget, and IncrementalPCA are still the right three
  knobs. GPU / out-of-core stacks exist; the math you learned did
  not flip.
- **Visualization.** **UMAP** (McInnes et al.) largely took t-SNE's
  job as the default 2-D look at high-d data: faster, often stabler
  global layout, still **not** a production feature map. sklearn
  still ships t-SNE; UMAP is typically `umap-learn`.
- **Nonlinear compression at scale.** Autoencoders and friends ate
  the "kernel PCA on a million images" fantasy. Kernel PCA remains a
  medium-*n* teaching and tabular tool.
- **Supervised and hybrid reducers.** Target-aware methods and
  representation learning (contrastive pretraining) often beat
  unsupervised PCA on downstream accuracy. PCA remains the **first**
  linear tool and the **invertible** compressor.
- **Trees vs PCA.** Histogram boosting and forests still do not
  *need* PCA; rotation can still hurt axis-aligned splits. Do not
  prepend PCA as folklore.

Keep the curse, the two bets (plane vs sheet), PCA's variance axes,
how to pick *d*, randomized/incremental variants, kernel PCA as the
nonlinear cousin, LLE as local-weight unroll, and the survey table
as a drawer of jobs.

## Check yourself

1. In a high-d cube, why do k-NN and "distance to mean" degrade even
   if you have scaled every column to zero mean unit variance?
2. Draw a Swiss roll. Show a linear projection that overlays two
   coils and a manifold unroll that does not. Which class of methods
   is each?
3. PCA keeps high-variance directions. Invent a 2-D cartoon where
   the **class boundary is the low-variance axis**. What happens if
   you keep only PC1?
4. Why must `PCA.fit` live inside the training fold of a pipeline
   even though PCA does not use *y*?
5. You plot cumulative explained variance and keep 95%. Name one
   downstream disaster that variance budget cannot see, and what you
   would optimize instead if PCA is a preprocessor.
6. When do you standardize before PCA, and when is that a mistake?
7. Randomized PCA vs IncrementalPCA: one is for **approximate first
   components on a big matrix you can still slice**, one is for
   **streaming / out of core**. Assign the jobs. What must you not
   over-interpret in the streaming case?
8. Kernel PCA needs a kernel and γ. Why is "the 2-D plot looks nice"
   a weak selection rule, and what protocol would you use if a
   classifier sits downstream?
9. LLE stores local reconstruction **weights**. What goes wrong if
   *k* is so large that neighbors jump off the manifold's sheet?
10. Pick t-SNE, MDS, Isomap, or LDA for each job: (a) 2-D slide of
    MNIST neighborhoods, (b) preserve original distances, (c) unroll
    using graph geodesics, (d) project with **labels** to separate
    classes. Which of these would you *not* ship as a live
    `transform`?
