# 7. Ensemble Learning and Random Forests

Companion notes for **Chapter 7** of *Hands-On Machine Learning with
Scikit-Learn, Keras, and TensorFlow* (2nd edition, Aurélien Géron;
O'Reilly, 2019).

[Chapter 6](../6-decision-trees/) gave you one twitchy flowchart.
This chapter is what you do when you **refuse to trust one hypothesis**:
grow many imperfect predictors and combine them. Skip it and you will
ship a single tree because it "explains," or you will call every bag of
models a Random Forest, or you will tune AdaBoost as if it were the 2026
default for tabular data. The ideas (diversity, averaging, sequential
residual fitting, a blender on a hold-out) outlive the 2019 class names.

See also: this is still **training estimators on a table**. It is not
[platform ch. 3](../../platform/3-model-service/) routing across
providers, and not a multi-agent "ensemble" of personas.

## The mental model

One accurate-enough model with uncorrelated errors is a miracle. Many
mediocre models whose mistakes **do not line up** are an engineering
plan.

```
  training rows
       |
       +-- view A (sample / features / weights / residuals) --> model 1
       +-- view B                                            --> model 2
       +-- view C                                            --> model 3
       ...
       v
  COMBINE:  vote | average proba | sum of residual trees | blender
       |
       v
  one prediction  (usually lower variance than any one view)
```

The one sentence to remember a year from now: **an ensemble buys a
better bias–variance point by making base learners disagree on their
errors** — through different data views (bagging), different feature
views (patches, random splits), or a conversation with leftover error
(boosting) — then combining them on purpose.

Two consequences fall straight out of that diagram. First, cloning the
same overfit tree ten times and majority-voting is not an ensemble; the
errors are copies. Second, "we used a Random Forest" is a *specific*
recipe (bagged trees plus extra split randomness), not a synonym for
"several models." If you cannot name how the views differ and how they
are combined, you do not have a design. You have a slogan.

## Voting classifiers

**Problem** — You have three classifiers that each sit a little above
chance. You pick the best one on the validation set and throw the
others away.

**Solution** — Keep them, and **vote**. Hard voting: each model emits a
class, majority wins. Soft voting: average predicted probabilities, then
argmax. Soft voting usually wins when the members produce usable
`predict_proba` and are not all the same overconfident leaf-frequency
machine.

The folk theorem: if members are a bit better than random **and** their
mistakes are weakly dependent, the vote concentrates on the truth. That
is a law-of-large-numbers cartoon, not a proof that your three sklearn
objects are independent. Logistic regression, a SVM, and a tree *can* be
diverse because they carve the space differently. Three Random Forests
with different seeds, less so.

```
  model A:  cat
  model B:  dog
  model C:  cat
  hard vote -> cat

  P(cat):  0.40, 0.62, 0.71   average -> cat   (soft can flip hard)
```

**Failure mode** — Soft-voting a mix of a calibrated logistic model and
an unbounded tree whose probabilities are 0.99 because a leaf saw six
rows. The tree will dominate the average. Calibrate, cap tree depth, or
stick to hard votes when probabilities are fiction.

Diversity beats cloning. If you only have one algorithm, the rest of
this chapter is how to *manufacture* diversity from one algorithm.

## Bagging and pasting

Same algorithm, different **training subsets**.

- **Bagging** (bootstrap aggregating): draw *n* rows **with
  replacement**. Some rows repeat; others never appear in that view.
- **Pasting:** draw **without** replacement. Each view is a smaller
  distinct subset.

Train a predictor on each view. For classification, typically vote; for
regression, average. The combination shrinks **variance** if the base
learner was high-variance (unbounded trees are the exhibit). Bias can
rise a little because each view sees less unique data; the vote usually
pays for that.

```
  original set  [ 1 2 3 4 5 ]
  bag 1         [ 2 2 5 1 4 ]   (replacement)
  bag 2         [ 3 1 1 5 2 ]
  paste 1       [ 1 4 5 ]       (no repeats, smaller)
```

**Problem** — "We shuffled `random_state` and trained five full models
on the *same* data." That is not bagging.

**Solution** — Different **rows** (and, later, different **features**).
sklearn: `BaggingClassifier` / `BaggingRegressor` wrapping a base
estimator (`DecisionTreeClassifier` is the usual teaching wrap).
`bootstrap=True` is bagging; `False` is pasting. `n_estimators` is how
many views.

**Failure mode** — Bagging a *stable* learner (a stump, a linear model
on lots of data) and expecting a miracle. Bagging is a variance weapon.
If the base is already low-variance and high-bias, you get a committee
of the same wrong answer.

### Out-of-bag

Each bagged model, by construction, never saw some rows — around 37% of
the training set for ordinary bootstrap of size *n*. Those **out-of-bag
(OOB)** rows are a free validation set *for that model*. Average the OOB
predictions across models and you get an OOB score without peeling off
a hold-out.

```
  row i not in bag k  ->  model k may score row i
  aggregate those leftover scores  ->  OOB estimate
```

Turn on `oob_score=True` when you bag. Use it as you would a validation
metric: to pick `n_estimators` or tree depth **without** burning the
test set.

**Failure mode** — Reporting OOB as if it were a locked test metric
*and* using the test set to early-stop the same search. OOB is for the
training sample's leftover rows. It is not magic against leakage you
introduced before `fit` (target encoding on the full set, scaling on
train+test). The leakage rules of [ch. 2](../2-end-to-end-project/)
still apply.

Pasting has no bootstrap leftovers; do not ask it for OOB.

## Random patches and subspaces

Rows are not the only view.

- **Random subspaces:** each model sees **all rows** (or a large share)
  but only a **random subset of features**.
- **Random patches:** sample **both** rows and features.

sklearn exposes this on the bagging wrappers: `max_samples`,
`max_features`, `bootstrap_features`. High-*m* problems (lots of
columns, not necessarily lots of rows) often gain more from feature
sampling than from another bootstrap of the same collinear columns.

**Problem** — A wide table where three features are the same signal
under different names. Every bagged tree still splits on one of them
first; the committee agrees on a redundant story.

**Solution** — Force `max_features < m` so some members never see the
bully column and have to use the others. That is also the seed of
Random Forests.

**Failure mode** — Setting `max_features` so small that each member is
blind. Diversity is useful; blindness is bias you did not mean to buy.

## Random Forests

A **Random Forest** is bagged **decision trees** plus extra randomness
**at each split**: the tree is not allowed to see every remaining
feature when it picks (j, t). It sees a random subset (often around
√m for classification, m/3 for regression — check the estimator's
default, do not tattoo an old rule).

```
  bootstrap rows
       |
       v
  grow a tree, but at EVERY node:
      draw a feature subset  ->  CART split only among those
       |
       v
  many such trees  ->  vote / average
```

Why this works: trees are high-variance ([ch. 6](../6-decision-trees/)
instability). Bagging decorrelates them a bit; random split features
decorrelate them more. The average of decorrelated high-variance
estimators is the classic variance reduction.

sklearn: `RandomForestClassifier`, `RandomForestRegressor`. You still
own `n_estimators`, `max_depth`, `min_samples_leaf`. More trees almost
monotonically help until they plateaus; they rarely overfit in the
"one more tree ruined us" way. Depth and leaf size still can.

**Failure mode** — Treating `n_estimators=500` as regularization for a
forest of unbounded trees on a tiny noisy set. The average of 500
memorizing trees is a smooth *memorization*. Cap leaves. Measure a
hold-out.

### Extra-Trees

**Extremely Randomized Trees** go further: at each split they also
draw **random thresholds** (among the allowed features) and keep the
best of those random cuts, instead of searching the best CART threshold.
That extra noise:

- often **reduces variance** more, **increases bias** a bit
- **trains faster** (no exhaustive threshold scan)
- can win or lose vs a Random Forest; it is an A/B, not a promotion

`ExtraTreesClassifier` / `ExtraTreesRegressor`. Same forest-shaped API.
Do not call them "the extra Random Forest hyperparameters." They are a
different split proposal.

### Feature importance

Forests give you a cheap ranking: how much **impurity dropped** across
all splits that used feature j, averaged over trees. sklearn exposes
`feature_importances_` that sum to 1.

Use it as a **flashlight**, not as a scientific causal ranking:

- Correlated features share credit or steal it. One of a pair can look
  "unimportant" because its twin always won the split.
- High-cardinality categoricals (after naive encoding) get extra
  chances to split and look important.
- Importances are about **this forest, this sample, this impurity**.

**Problem** — A dashboard titled "drivers of churn" fed directly from
`feature_importances_`.

**Solution** — Treat the ranking as a hypothesis generator. Confirm with
ablation, partial-dependence, or a held-out permutation importance
(shuffle a column, watch the metric fall). Permutation is slower and
still not causal, but it is at least measured in **your** scoring rule.

**Failure mode** — Dropping every feature below 0.01 importance and
refitting until the story is a short list. You may have dropped a
feature that only matters in a leaf the impurity average washed out.

## Boosting

Bagging trains members **in parallel** on resampled views. **Boosting**
trains them **in sequence**, each one asked to fix what the current
committee still gets wrong. Members are usually **weak** (stumps, shallow
trees). The committee can become very strong — and can overfit if you
let the sequence run too long without shrinkage or early stopping.

### AdaBoost

**AdaBoost** (adaptive boosting) reweights **training rows**. Start
equal. Fit a weak learner. Increase the weight of rows it got wrong;
decrease (or hold) the ones it got right. Next learner sees a dataset
that emphasizes yesterday's failures. Combine members with weights that
favor the more accurate ones (SAMME / SAMME.R in sklearn's
`AdaBoostClassifier`).

```
  w_i = 1/n
  for t in 1..T:
      fit weak learner on weighted rows
      raise w_i on mistakes
      store learner + its say  (alpha_t)
  predict = weighted vote of the T learners
```

**Failure mode** — AdaBoost with an already-strong learner (a deep
tree). The first member memorizes; later members have nothing coherent
to reweight toward. Keep the base **weak**. Also: noisy labels get
heavier and heavier weights; AdaBoost will chase outliers unless you
clean or cap.

AdaBoost is the right *mental* picture of "pay attention to mistakes."
It is no longer the default champion on tabular benchmarks. Know it so
you can see Gradient Boosting as a cousin, not as a brand.

### Gradient Boosting

**Gradient Boosting** (often GBRT when the member is a tree) does not
reweight rows. It fits the next tree to the **residual errors** (more
generally, to the negative gradient of the loss). Additive model:

```
  F_0 = constant (mean, or log-odds, ...)
  F_{t} = F_{t-1} + nu * tree_t(x)
          tree_t trained to predict leftover error
          nu = learning_rate  (shrinkage)
```

Small `learning_rate` (`nu`) means each tree is a cautious step. You
then **need more trees** (`n_estimators`) to reach the same fit. That
pair — many small steps — is the usual sweet spot, plus **early
stopping** on a validation set (`staged_predict` in the 2019 sklearn
API, or `n_iter_no_change` in later ones).

sklearn's `GradientBoostingClassifier` / `Regressor` in the 2e era are
the teaching implementations: one split search at a time, no histogram
binning. They are slow on large tables and still the right place to
learn the residual picture.

**Problem** — Setting `n_estimators=1000` and `learning_rate=0.01` and
walking away without a validation curve.

**Solution** — Plot validation loss vs boosting iteration. Stop when it
flattens or rises. Shrinkage without a stop is just a slower overfit.

**Failure mode** — Calling every sequential tree model "a Random
Forest." Forests average **independent** trees. Boosting adds **dependent**
trees that were aimed at leftovers. Diagnostics differ: forests rarely
need early stopping on tree *count*; boosted trees do.

Histogram-based successors and XGBoost-shaped libraries belong in
**What aged**, not in your first residual cartoon. Learn `F_t = F_{t-1}
+ nu * h_t` here.

## Stacking

Voting treats members as equals (or as probability averages).
**Stacking** trains a **blender** (meta-learner) on the members'
outputs.

```
  hold out a blend set
       |
       v
  base models trained on the rest  ->  predict the blend set
       |
       v
  blender trains on those predictions (+ maybe original features)
       |
       v
  at inference: bases first, blender on their outputs
```

If you train the blender on the **same** rows the bases already fit,
the blender learns to trust overfit members. Use a hold-out or
cross-validated out-of-fold predictions. sklearn grew
`StackingClassifier` / `StackingRegressor` after many 2019 readers had
to wire this by hand — the *leakage* trap is the lesson, not the class
name.

The blender can be linear (interpretable mix of experts) or another
tree. Keep it simpler than the bases unless you have a lot of blend
data.

**Failure mode** — Four stacked layers on a 2,000-row table. You are
fitting a committee of committees on noise. Stacking is for when you
already have diverse strong bases and a clean out-of-fold matrix.

## What aged since 2019

- **sklearn stacking.** `StackingClassifier` / `Regressor` with
  cross-validated predictions are built in. Use them instead of a
  hand-rolled leaky `predict` on the training set.
- **Histogram gradient boosting.** `HistGradientBoostingClassifier` /
  `Regressor` are sklearn's answer to "GBRT but for larger data": bin
  features, grow on bins, optional native categoricals. For new tabular
  work *inside sklearn*, start here rather than
  `GradientBoosting*`.
- **XGBoost-shaped successors.** XGBoost, LightGBM, and CatBoost — and
  the histogram idea they popularized — are what most leaderboards
  meant by "gradient boosting" within a few years of this edition.
  The residual-and-shrinkage mental model still applies. The tree
  growing tricks (leaf-wise vs depth-wise, ordered encoding, missing-
  value routing) are why they win on speed and often on accuracy.
- **AdaBoost's job.** Still the right lecture. Rarely the right
  production default vs histogram boosting or a forest baseline.
- **Random Forests.** Still an excellent **baseline**: few knobs, parallel,
  OOB, importances. Not obsolete. Often beaten on raw score by boosted
  trees; often preferred when you want less tuning drama.
- **n_estimators defaults** and `n_jobs` / `n_jobs=-1` habits moved;
  parallelism is assumed. The OOB story is unchanged.

Keep voting, bagging, OOB, forests, boosting-as-residuals, and stacking-
without-leakage. Swap the 2019 GBRT class for a histogram/XGBoost-shaped
tool when the table is real.

## Check yourself

1. You average three copies of the same Decision Tree trained on the
   identical rows. Why is that not the diversity argument for voting?
2. When would you pick **soft** voting over **hard**, and when would
   soft voting be actively misleading?
3. Bagging vs pasting: which one can use OOB, and what does OOB
   *refuse* to catch?
4. A wide dataset has 200 columns, 3 of which are near-duplicates of
   the label-leaky ID. How do random **subspaces** change what the
   committee can memorize compared to row-bootstrap alone?
5. Write the one extra source of randomness that makes a Random Forest
   not "just bagged trees." What happens to correlation of members if
   you set `max_features = m`?
6. Extra-Trees randomize thresholds. Name one cost they pay and one
   they save. How would you decide forest vs Extra-Trees on a problem
   you actually have?
7. `feature_importances_` ranks column A near zero and its near-clone
   column B at the top. What two explanations are still on the table
   besides "A does not matter"?
8. AdaBoost vs Gradient Boosting: one reweights **rows**, one fits
   **residuals**. Give a noisy-label failure that hurts AdaBoost
   especially, and a tuning pair (`learning_rate`, `n_estimators`) you
   must watch for GBRT.
9. Why can adding trees to a Random Forest rarely need early stopping,
   while adding trees to Gradient Boosting often does?
10. Sketch stacking with a hold-out blender. Where exactly does leakage
    creep in if you train the blender on the bases' training-set
    predicted labels?

Continue to [Dimensionality Reduction](../8-dimensionality-reduction/).
