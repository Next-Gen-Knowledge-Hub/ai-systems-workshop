# 5. Support Vector Machines

Companion notes for **Chapter 5** of *Hands-On Machine Learning with
Scikit-Learn, Keras, and TensorFlow* (2nd edition, Aurélien Géron;
O'Reilly, 2019).

This chapter is a different linear (and kernelized) machine: not "fit
the average row" but **fit the widest street between classes**. Skip
it and "SVM" stays a checkbox on a 2016 slide — you will not know why
it hates unscaled features, why a soft margin is a product choice, or
when a kernel is cheaper than exploding polynomial columns. You do
not need the appendix-C derivations to use the estimator. You do need
the geometry, the knobs, and the complexity story.

## The mental model

```
           class −                    class +
              o  o                      x
           o        o               x      x
              o  |      STREET      |  x
                 |<---- widest ---->|
                 |    margin        |
                 o   support        x
                     vectors
                     (the only rows
                      that hold the
                      fences up)
```

The one sentence to remember a year from now: **an SVM classifies by
a large-margin boundary held up by a few support vectors, and kernels
are a way to get a curved boundary without baking a huge feature
map yourself.**

Two consequences. First, most training rows **do not matter** once
the street is set; a few awkward points (and your slack penalty C)
do. Second, if you do not scale features, "widest" is in the units of
whatever column is in milliseconds vs whatever column is in
billions — the street is a lie.

## Linear SVM classification

In the linearly separable cartoon, many hyperplanes separate the two
classes. The SVM picks the one with the **largest margin**: distance
from the plane to the nearest points on each side. Those nearest
points are the **support vectors**. Move a point that is not a
support vector, and the solution does not budge. Move a support
vector, and the street tilts.

This is a different objective from logistic regression. Logistic cares
about every row's log loss. A hard-margin SVM cares about the
worst-placed rows *on the frontier*. That is why outliers are so loud
here.

A single mislabeled or extreme point pinches the margin to a slit, or
makes "perfect separation" impossible. **Soft margin** (next section)
is almost always what you run. An SVM with no scaling on mixed-unit
tables lies about width. Celebrating a linear SVM on a XOR-ish pair of
features and then "proving SVMs don't work" usually means you needed a
kernel or an explicit map.

```python
from sklearn.pipeline import Pipeline
from sklearn.preprocessing import StandardScaler
from sklearn.svm import LinearSVC, SVC

linear = Pipeline([
    ("scale", StandardScaler()),
    ("svm", LinearSVC(C=1.0, dual="auto")),
])
linear.fit(X_train, y_train)
```

`StandardScaler` puts features on a common scale so margin width is
meaningful. `LinearSVC` then finds a large-margin linear separator;
`C` is the soft-margin tax (below). `LinearSVC` (or
`SGDClassifier(loss="hinge")`) is the scalable linear path.
`SVC(kernel="linear")` is the same geometry with a different solver,
often slower on large m.

## Soft margin

Hard margin: no point may sit inside the street or on the wrong
side. Real labels are noisy. Hard margin either **fails to solve** or
**overfits the noise**.

Soft margin introduces **slack**: some points may violate the
margin, at a cost. The knob is **C** (sklearn):

- **Large C** — little slack. Narrower street, fewer violations,
  closer to hard margin. Sensitive to outliers. Can overfit.
- **Small C** — more slack. Wider street, more points allowed
  inside / wrong side. Smoother, can underfit.

```
  high C:  skinny street, hugs the messy frontier
  low  C:  fat street, tolerates a few trespassers
```

Walk a numeric intuition. Suppose a fraud table where one mis-keyed
amount is a million times larger than typical rows. At high C the
optimizer spends the street's width trying to keep that outlier on the
correct side of the margin — the boundary tilts toward noise. At low C
the model accepts a few margin violations; the street stays wide; the
outlier becomes one of a handful of support vectors that are *allowed*
to trespass. C is not "regularization α" from Ridge, but it plays a
similar social role: **how expensive is complexity / fussiness.** Tune
it with CV on a metric that matches the decision. Grid-searching C on
accuracy with a 1% positive class picks a street that never flags the
rare class.

Class weight (`class_weight="balanced"`) is sometimes the more
honest lever than cranking C when priors are ugly.

## Nonlinear SVMs

When the boundary is a curve in the original x, you have three
practical moves. They are not the same bill.

### Polynomial features, then a linear SVM

Explicitly add powers and cross terms (`PolynomialFeatures`), scale,
`LinearSVC`. You **see** the map. You also **pay** for it: degree and
n explode together. Fine for a few columns. Hopeless as a default on
a wide table.

### Polynomial kernel

A **kernel** computes what the dot product *would have been* in that
expanded space, without allocating the expanded columns. `SVC` with
`kernel="poly"` has degree, `C`, and `coef0` (how much the high
degree is allowed to dominate).

Degree 8 "because it can" still overfits; it just overfits in a
different memory envelope than an explicit map. Start low. Watch val
error. Explicit degree-8 expansion on 40 raw columns, RAM death, then
a story about "SVMs need Spark" usually means you needed a kernel or a
different family.

### Similarity features and the RBF kernel

Another explicit idea: pick landmarks, measure **how similar** each
row is to each landmark (a bell around the landmark). That feature
space makes "blobs" linearly separable. The common bell is the
**Gaussian RBF**. The common kernel is **RBF**: it is the similarity
trick without you placing landmarks by hand.

RBF knobs:

- **γ (gamma)** — how *local* a landmark is. High γ: each point's
  influence is a spike; decision surface gets busy (overfit). Low γ:
  wide bells; surface gets mushy (underfit).
- **C** — still the slack tax, now in the kernelized space.

```
  γ high, C high   -->  islands around individual points
  γ low,  C low    -->  a blunt, possibly useless blob
```

```python
rbf = Pipeline([
    ("scale", StandardScaler()),
    ("svm", SVC(kernel="rbf", C=1.0, gamma="scale")),
])
```

`SVC(kernel="rbf")` fits a kernel SVM whose similarity is a Gaussian
bell; `gamma="scale"` sets γ from feature variance (a sane default,
not a law). Grid **log-spaced** C and γ. They interact. A 2D heatmap
of CV scores on (C, γ) is more honest than two independent 1D sweeps.

### Complexity (practical, not appendix C)

- **Linear SVM**: scales roughly like a linear classifier. `LinearSVC`
  is the one you try on large m. Training cost grows with m and n,
  without a kernel matrix.
- **Kernel SVM**: you (implicitly) deal with an **m × m** similarity
  structure. Training often grows **worse than quadratic** in m.
  Fine for thousands of rows. Painful for hundreds of thousands,
  unless you approximate (linear, Nystroem, or trees /
  HistGradientBoosting).
- **Prediction**: kernel SVM scores against **support vectors**. Many
  support vectors ⇒ slower predict. A tiny C / smoother γ can reduce
  that count; it can also underfit.

`SVC(kernel="rbf")` on a million rows because it won a 5,000-row
notebook did not "fail." You left its complexity class.

## SVM regression

Turn the picture inside out. Instead of the widest street *between*
classes, fit a tube of width ε around the targets: **points inside
the tube are fine; points outside become support vectors** and pull.

- **ε large** — a fat tube, fewer support vectors, a coarser fit.
- **ε small** — a skinny tube, more vectors, a wigglier fit.
- **C** still taxes slack outside the tube.

```
  y
  |     x        x
  |   ----- ε tube -----     (points inside do not pull)
  |     x        x     x
  +-------------------- x
```

Use `LinearSVR` / `SVR` with the same scaling rule. Regression SVMs
are less fashionable than gradient boosting on tables, but the
**ε-insensitive** loss is a real product choice: you may not care
about errors smaller than a sensor's noise floor. Tuning ε on RMSE
until the tube is zero buys a very expensive, kernelized, "care about
every residual" model — SVM prices for a job Ridge might have done.

```python
from sklearn.svm import LinearSVR, SVR

svr = Pipeline([
    ("scale", StandardScaler()),
    ("est", SVR(kernel="rbf", C=1.0, epsilon=0.2, gamma="scale")),
])
```

`SVR` fits a tube of width `epsilon` in the RBF feature space; points
inside the tube do not contribute to the loss.

## Under the hood (intuition only)

Stay at the level you can draw. Leave the KKT wall of Greek letters
in the book's appendix.

### Decision function

The model is still a **score** w·x + b (in the linear case). Sign
gives the class; magnitude is distance from the street's center
line. Thresholding that score is allowed — the operating point is
still a product choice. `decision_function` is the object behind
`predict`.

Platt-style probabilities (`probability=True` on `SVC`) are an
**extra** logistic fit on those scores. They cost time and are not
the SVM's native output. Do not turn them on "for free" in a hot
loop.

### The dual

The **primal** talks about w (one weight per feature). The **dual**
talks about one coefficient per **training row**, most of them zero
(the non-support-vectors). When n is huge and m is modest, or when
you want a kernel, the dual is the natural home: you never write w
in the expanded space; you write a weighted sum of kernels against
support vectors.

You do not need to solve the dual by hand. You need to remember
**why predict time depends on the support-vector count**, and why
linear `LinearSVC` can stay in the primal and scale to larger m.

### The kernel trick

A kernel k(x, x') is a similarity that equals a dot product
φ(x)·φ(x') for some map φ you refuse to materialize. Polynomial and
RBF are the two you will actually type. The trick is **legal
laziness**: same geometry as "map, then linear SVM," cheaper when φ
would be wide or infinite (RBF).

A kernel on **unscaled** x, or a custom kernel that is not actually a
valid similarity, produces curved garbage.

### Online SVMs

Classic kernel SVM is a **batch** quadratic program: it wants the
set. For streams, people use **linear** approximations: hinge loss +
SGD (`SGDClassifier(loss="hinge")`) is the chapter's practical
"online SVM." You get a linear street that **updates**, not a kernel
matrix that grows forever.

"We need online RBF SVM" as a requirement usually means either a
periodically refit kernel model on a window, or a different family
that was born incremental. Hinge+SGD is the honest online cousin.

```
  LinearSVC / hinge SGD     primal, large m, linear street
  SVC kernel                dual, medium m, curved street
  hinge SGD                 the streaming linear compromise
```

## What aged since 2019

- `LinearSVC`, `SVC`, `LinearSVR`, `SVR`, and hinge `SGDClassifier`
  are still the same knobs (C, γ, ε, kernel). `dual="auto"` is the
  modern way to avoid the old dual/primal footgun on `LinearSVC`.
- On **tabular** problems, histogram gradient boosting is often
  stronger out of the box than a kernel SVM, with kinder scaling in
  m. SVMs remain relevant for **medium, well-scaled, not-huge-m**
  problems, text with linear hash/TF-IDF, and as a clean large-margin
  baseline.
- Kernel approximations (`Nystroem`, random Fourier features) are
  more approachable in sklearn than they felt in many 2019
  notebooks. They sit between "linear" and "full RBF."

## Check yourself

1. In a 2-class problem you know (fraud vs not, defect vs not),
   what would a "widest street" *mean*, and which rows would you
   expect to become support vectors?
2. You forgot `StandardScaler` before `LinearSVC`. What does "margin"
   become in that system's units, and which column would dominate?
3. Soft-margin C: pick an outlier policy from a real table (a few
   mis-keyed sensors, a celebrity customer). What happens at high C
   vs low C, and which metric would lie if you only watched
   accuracy?
4. Explicit polynomial features vs poly kernel: for the width of a
   table you have used, which one blows memory first, and what
   failure looks like in a training job?
5. RBF γ: describe an overfit surface vs an underfit one on a
   domain you know (not moons-in-a-textbook). What would CV vs train
   error do in each case?
6. Why is `SVC(kernel="rbf")` a bad default on a million-row
   warehouse extract? What linear or tree-shaped alternative would
   you try first, and what would you lose?
7. SVM regression: when is an ε-tube a better *product* loss than
   RMSE (sensor noise, pricing ticks)? What happens if you shrink ε
   until it vanishes?
8. Primal vs dual, in one sentence each, tied to a predict-time
   incident: "why is scoring slow?" What would you inspect (support
   vector count vs feature count)?
9. `probability=True` on `SVC`: what extra machinery did you turn
   on, and when has a "probability" from a margin model misled a
   threshold in a system you know?
10. Online hinge-SGD vs kernel SVM: which one can follow a stream,
    and what confusion with "online" meaning session memory would you
    shut down in a design review?
