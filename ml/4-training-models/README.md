# 4. Training models

Companion notes for **Chapter 4** of *Hands-On Machine Learning with
Scikit-Learn, Keras, and TensorFlow* (2nd edition, Aurélien Géron;
O'Reilly, 2019).

This chapter opens the box: **how parameters are found**. Linear
regression is the smallest interesting machine — a weighted sum, a
loss, a closed form or a walk downhill. Skip it and later nets, SVMs,
and "we used Adam" are folklore. You will not know why batch size
changes the path, why a polynomial exploded, or why ℓ2 is not a moral
preference. Chapters 1–3 told you *what* to measure and *how not to
leak*. This one tells you *what `fit` is doing*.

**See also (do not merge).** A platform Model Service is an adapter
to a **provider API** —
[platform ch. 3](../../platform/3-model-service/). This folder trains
**your** weights. Do not merge "the model" across those jobs
([`TRADEOFFS.md`](../../TRADEOFFS.md)).

## The mental model

```
  hypothesis  ŷ = f(x; θ)
       |
       v
  loss  L(θ)  on a batch of rows
       |
       v
  +---- closed form, if the math is kind (normal equation)
  |
  +---- walk  θ ← θ − η ∇L
              batch / stochastic / mini-batch
       |
       v
  regularize so θ cannot use every wiggle
  (Ridge / Lasso / Elastic Net / stop early)
```

The one sentence to remember a year from now: **training is
optimization of a loss on parameters, and generalization is a fight
against the extra capacity you did not pay for with data.**

Two consequences. First, a learning-rate or a polynomial degree is
not "tuning flavor"; it changes whether you converge, oscillate, or
memorize. Second, if train error and val error tell different
stories, you already know overfit vs underfit — you do not need a
more exotic estimator yet.

## Linear regression

A linear model predicts a number as a **weighted sum** of features,
plus a bias:

```
  ŷ = θ₀ + θ₁ x₁ + θ₂ x₂ + … + θₙ xₙ
```

Training usually minimizes **mean squared error** (MSE) between ŷ
and y. That choice matches RMSE-as-selector from
[ch. 2](../2-end-to-end-project/) and makes the math cooperative:
the loss is a bowl (convex) in θ if features are fixed.

**Problem** — "Linear" sounds like the world must be a straight line
in the raw columns.

**Solution** — Linearity is in **parameters**, not in your domain
story. You can feed ratios, logs, and polynomial expansions of x.
The model is still linear in θ. The curve lives in the features.

**Failure mode to recognise** — Interpreting θᵢ as "the causal
effect of column i" on a table full of collinear census fields. You
have a predictor, not a policy simulation.

### The normal equation

For MSE linear regression, there is a **closed-form** θ: invert a
matrix built from the feature matrix X (with a bias column) and
multiply by Xᵀy. When that inverse exists, you are done in one shot.
When features are redundant, the matrix is singular and you need a
pseudoinverse (or regularization, below).

You do not have to derive it in a meeting. You do have to know **why
it stops being the default**: matrix inversion / factorization grows
nasty as the number of features climbs. Thousands of columns is a
different budget than five.

**Complexity, intuition only.** Let m be rows and n be features.
Building XᵀX is about features × features, with a pass over rows.
Solving that system is roughly cubic in n for a naive invert, much
better with a proper factorization, still **painful in n**. Prediction
is cheap: a dot product. So: closed form loves **wide-but-not-insane
n**, huge m is mostly a pass over data. Gradient descent (next) loves
**streaming m** and stays linear in n per step.

```python
from sklearn.linear_model import LinearRegression, Ridge

lin = LinearRegression()          # factorization under the hood
lin.fit(X_train, y_train)
```

sklearn will not make you write the inverse. It will still punish you
if you one-hot a million ids and ask for a dense closed form.

## Gradient descent: batch, stochastic, mini-batch

When a closed form is absent (logistic, nets) or too expensive, you
**walk downhill**.

```
  θ_next = θ − η * gradient_of_loss(θ)
```

- **η (learning rate)** too small: you crawl. Too large: you jump
  over the bowl and diverge (loss goes NaN or oscillates).
- **Feature scaling** ([ch. 2](../2-end-to-end-project/)) is not
  optional here. An unscaled column stretches the bowl into a ravine.
  GD zigzags. The closed form cares less; GD looks broken.

Three ways to estimate the gradient:

| Flavor | Gradient from | You gain | You pay |
|---|---|---|---|
| **Batch** | all m rows | stable direction; true convex bowl | every step waits on the whole set |
| **Stochastic (SGD)** | one row | fast updates; can escape shallow trouble | noisy path; never quite sits still |
| **Mini-batch** | a slice (e.g. 32–256) | vectorized hardware; in-between noise | one more knob (batch size) |

```
  batch:     one precise arrow from the whole cloud
  SGD:       a jittery path, cheap per step
  mini-batch: a thicker noisy arrow  <-- default in later deep nets
```

**Problem** — "SGD" in a doc can mean the algorithm, the sklearn
estimator, or "we trained a net."

**Solution** — Name the **gradient estimator**. Mini-batch is what
almost everyone runs in Part II. This chapter's SGDClassifier /
SGDRegressor are the linear, streaming version of the same idea.

**Failure mode to recognise** — Learning rate and scaling left on
defaults, loss exploding, "linear models don't work on our data."
The bowl was a ravine. Another: declaring convergence on train loss
while val loss already turned up (you needed early stopping).

```python
from sklearn.linear_model import SGDRegressor
from sklearn.pipeline import Pipeline
from sklearn.preprocessing import StandardScaler

pipe = Pipeline([
    ("scale", StandardScaler()),
    ("sgd", SGDRegressor(max_iter=1000, eta0=0.01,
                         learning_rate="invscaling", random_state=0)),
])
pipe.fit(X_train, y_train)
```

Schedule `eta0` and `max_iter` like you mean it. Tiny original
sketch, not a copied listing: if the scale of x changes, `eta0`
that used to work will lie.

## Polynomial features and learning curves

To fit a bend with a linear-in-θ model, **expand** x: add x², x³,
cross terms. Degree 2 is often a useful extra; degree 30 on one
column is a memorization machine.

```python
from sklearn.preprocessing import PolynomialFeatures

poly = Pipeline([
    ("expand", PolynomialFeatures(degree=2, include_bias=False)),
    ("scale", StandardScaler()),
    ("lin", LinearRegression()),
])
```

The expansion lives **inside** the pipeline so CV cannot leak, same
ethic as [ch. 2](../2-end-to-end-project/).

### Learning curves as diagnosis

Plot **train error** and **val error** against **training-set size**
(or against degree, or against epochs). The shapes are the
vocabulary:

```
  UNDERFIT                         OVERFIT
  --------                         -------
  train error HIGH                 train error LOW
  val error HIGH, close to train   val error HIGH, a GAP
  more data barely helps           more data narrows the gap
  need a richer hypothesis         need less capacity / more data
                                   / regularization
```

**Problem** — The team argues "more trees" vs "more rows" with no
curve.

**Solution** — If both curves are bad and together, **underfit**. If
train is great and val is not, **overfit**. If val is still falling
when you add rows, go collect. If val has flattened with a gap,
collecting the same distribution will help slowly; shrinking capacity
helps now.

**Failure mode to recognise** — A validation curve computed with the
test set. That is chapter 1's peeking, drawn as a line chart. Another:
polynomial degree chosen on the same val you will report, ten times,
until the curve looks "nice."

## Regularized linear models

Regularization is **taxing θ** so the model cannot spend capacity on
noise. It is the everyday answer to the overfit side of the curves.

### Ridge (ℓ2)

Add λ Σ θᵢ² to the loss (usually not the bias). Large weights get
expensive. The solution **shrinks** coefficients, keeps them dense.
A closed form still exists (the matrix gets a ridge down the
diagonal — that is the name). λ → 0 is ordinary least squares. λ →
large is "almost a constant predictor."

**Failure mode to recognise** — Ridge without scaling. The tax falls
on whatever column happens to be in small units. You regularized
units, not complexity.

### Lasso (ℓ1)

Add λ Σ |θᵢ|. The geometry **drives some weights to exact zero**.
Lasso is a feature selector in disguise. Useful when you believe
few columns matter. Unstable when columns are clones of each other
(it picks one of a correlated pack arbitrarily).

**Failure mode to recognise** — Treating Lasso zeros as "we proved
this sensor is irrelevant" in a collinear plant. It proved the
optimizer picked a representative.

### Elastic Net

A mix of ℓ1 and ℓ2. Default grown-up choice when you want sparsity
*and* a little sharing among correlated columns. You now have two
knobs (overall strength, mix). Grid or random search from chapter 2.

```python
from sklearn.linear_model import Ridge, Lasso, ElasticNet

ridge = Ridge(alpha=1.0)
lasso = Lasso(alpha=0.01, max_iter=5000)
enet = ElasticNet(alpha=0.01, l1_ratio=0.2, max_iter=5000)
```

`alpha` is sklearn's λ. Start on a log grid. Always scale first.

### Early stopping

For iterative solvers (SGD, later nets): watch **val error**. Save
the θ where val was best, stop when it stops improving. That snapshot
is a regularizer: you refused to take the extra steps that only fit
train.

**Failure mode to recognise** — Early stopping on *train* loss. You
stopped when you were still underfitting, or you never stopped
because train keeps falling. Val is the signal. A tiny val split that
you also used for model selection is a weak signal — prefer a
protocol you could defend in chapter 2.

```
  epochs -->
  train loss \\\\\\\\________
  val loss   \____/  <-- stop near the trough, not at the right edge
```

## Logistic regression and softmax

Classification with a **linear score** squashed into a probability.

**Logistic** (binary): z = θ·x, p = σ(z) = 1 / (1 + e^{−z}). Train
by minimizing log loss (cross-entropy), not MSE. MSE on probabilities
is a poor bowl here; log loss is the one that matches Bernoulli
likelihood and keeps GD well-behaved.

The **decision boundary** is where z = 0 (p = 0.5 if you threshold
there): a hyperplane in feature space. Polynomial / nonlinear
features bend the boundary the same way they bent regression.

**Threshold is still a product choice** from
[ch. 3](../3-classification/). Logistic gives you a score with a
probabilistic *interpretation* if it is calibrated. It does not force
you to use 0.5.

```python
from sklearn.linear_model import LogisticRegression

log = Pipeline([
    ("scale", StandardScaler()),
    ("clf", LogisticRegression(C=1.0, max_iter=200)),
])
# C is 1 / regularization strength. Small C = stronger tax.
```

**Softmax** (multinomial): several class scores z_k, converted to a
distribution that **sums to 1**. Predict argmax. Train with
multiclass cross-entropy. This is the clean OvR alternative when you
want **one** model and calibrated-ish class probabilities. Trees and
later nets will do their own versions; the geometry started here.

**Problem** — Softmax outputs get treated as "the model is 91% sure."

**Solution** — Check calibration on a holdout. Linear softmax on
unscaled, collinear features is a ranking tool with a confidence
costume.

**Failure mode to recognise** — Using softmax as if labels could
overlap (multilabel tags). Softmax **fights** itself: raising one
class's p lowers the others. Overlapping tags want independent
sigmoids (chapter 3's multilabel story), not a single softmax.

```
  binary logistic     p(y=1|x) = σ(θ·x)
  softmax             p(y=k|x) = exp(z_k) / Σ exp(z_j)
  decision            hyperplane(s) in x; curves if x was expanded
```

GD vs closed form: logistic / softmax **need** iterative solvers.
That is why this chapter taught you η, scaling, and early stopping
before Part II exists.

## What aged since 2019

- `LinearRegression`, `Ridge`, `Lasso`, `ElasticNet`,
  `LogisticRegression`, `SGDRegressor` / `SGDClassifier` are stable
  APIs. Solvers rotated (`lbfgs`, `saga`); you pick one that matches
  n, sparsity, and multinomial.
- `PoissonRegressor` and other GLMs joined sklearn after this
  edition's center of gravity. Same idea: linear in θ, different
  loss. Not required to understand this chapter.
- For tabular accuracy, histogram gradient boosting often beats a
  hand-tuned polynomial + Elastic Net. Use that fact in
  [ch. 7](../7-ensembles/), not as a reason to skip *why* ℓ2 and
  learning curves exist. Boosting still overfits; the curves still
  tell you.
- Neural training in Part II is this chapter's mini-batch GD with
  more layers. If this folder is fog, stop and rerun it before
  Keras.

## Check yourself

Good answers need the takeaway, the failure mode, and a numeric
system you have actually fit or reviewed.

1. Write the linear hypothesis for a prediction you have shipped
   (or almost shipped). What would a large θ on a collinear pair
   *not* mean in the business?
2. When would you still want a closed-form linear fit, and when is
   n too wide (or the loss no longer MSE) so you must walk downhill?
   Point at a table you know.
3. Batch vs SGD vs mini-batch: which one were you running in spirit
   (nightly full fit vs streaming updates vs default net trainer),
   and what fails if you confuse them with "online" in chapter 1?
4. Unscaled features + GD: what does the path look like, and which
   column in a real dataset would have been the ravine?
5. You added polynomial degree 8 because degree 2 "looked underfit."
   What should learning curves have shown before you did that, and
   what production miss does a high-degree expansion create?
6. Ridge vs Lasso on a sensor pack that moves together: which one
   zeros, which one shares, and what false story does a Lasso zero
   tell a plant engineer?
7. Early stopping: what signal do you watch, and how is watching
   *train* loss a failure mode you have seen (or will now recognise)?
8. Why is log loss the training objective for logistic, not MSE on
   0/1? Give a badly calibrated "probability" from a system you
   know.
9. A 0.5 threshold on logistic output: when is that defensible, and
   when should chapter 3's PR curve pick `t` instead? Use a rare
   positive from your work.
10. Softmax vs independent sigmoids: pick a labeling scheme you have
    seen (intents vs tags). What goes wrong if you use the wrong
    head?

Continue to [Support Vector Machines](../5-svms/).
