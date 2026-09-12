# 6. Decision Trees

Companion notes for **Chapter 6** of *Hands-On Machine Learning with
Scikit-Learn, Keras, and TensorFlow* (2nd edition, Aurélien Géron;
O'Reilly, 2019).

Linear models in [ch. 4](../4-training-models/) draw one hyperplane.
SVMs in [ch. 5](../5-svms/) draw a margin around that idea. This chapter
is the first model that is a **flowchart you grew from rows**: nested
tests on one feature at a time, a class (or a number) at each leaf. Skip
it and you will treat "the interpretable model" as automatically honest,
miss that sklearn's default tree will memorize the training set, and then
be surprised when [ch. 7](../7-ensembles/) spends a whole chapter averaging
trees that individually twitch.

See also: this is **not** an agent's planning tree
([agents ch. 5](../../agents/5-reasoning-and-planning/)). Those are
search traces over tools. This folder is a fitted sklearn estimator.

## The mental model

A tree does not score a weighted sum. It asks a sequence of yes/no
questions that were chosen to purify the training rows that fall each
way.

```
  one row x
       |
       v
  [ root:  x_j  ?  threshold t ]
           /                 \
         yes                  no
          /                    \
   [ x_k ? t' ]            [ x_m ? t'' ]
      /      \                /      \
   leaf     leaf           leaf     leaf
   class A  class B        class A  class B
   p = n_A/n_leaf          ...
```

The one sentence to remember a year from now: **a decision tree is a
greedy partition of the feature space into axis-aligned boxes**, each
box labeled with the majority class (or the mean target) of the training
rows that landed there.

Two consequences fall straight out of that diagram. First, you can
*read* a prediction — "petal width ≤ 0.8, then setosa" — which is why
trees show up in credit files and medical notes; that path is not a
causal story and not a calibrated probability. Second, the boxes are
rectangles aligned with the axes, so a 45-degree cluster looks like a
staircase, and a one-row edit near a split can rewrite the whole
flowchart. Forests exist because one tree is both readable and twitchy.

## Train and visualize

**Problem** — A tree in a slide is a cartoon. A tree in a notebook is an
object with hundreds of nodes you cannot keep in your head.

**Solution** — Fit a small `DecisionTreeClassifier` on a dataset whose
features you already understand (Iris is the teaching set for a reason),
then *draw* it. sklearn will emit Graphviz DOT via `export_graphviz`, or,
in later sklearn, `plot_tree` on the matplotlib axis. Restrict
`max_depth` while you are learning to look. The picture is the model.

What you should be able to point at on the drawing:

- the **feature** and **threshold** on every internal node
- the **class counts** (or value vector) on every node
- the **predicted class** of each leaf, which is the majority of the
  training rows that reached it
- **Gini** (or entropy) as a number that should get smaller as you go
  down a pure branch

**Failure mode** — Training an unbounded tree on a messy table and then
"visualizing" a PNG with 800 nodes. That is not interpretation. It is a
wallpaper of overfitting. Shrink the tree until a colleague can walk one
row through it on a whiteboard.

### Making predictions

Prediction is not a matrix multiply. Start at the root. At each node,
compare one feature of the query row to the stored threshold. Go left or
right. Stop at a leaf. Emit that leaf's class.

```
  x  ->  node  ->  ...  ->  leaf  ->  argmax class counts
```

Depth of the path is the number of tests. A balanced tree on *n*
training rows is often around log₂(*n*) tests; a degenerate tree that
always peels off one row is *n* tests. That is why regularization is not
only about generalization — it is also about how long a prediction
takes.

You do **not** look at neighboring leaves. You do **not** blend with a
kernel. The box you landed in *is* the answer. If two rows sit on
opposite sides of a threshold by 1e-9, they can get different classes.
That is the geometry, not a bug in float rounding (though floats can
make it worse).

### Class probabilities

A classifier that only emits a label is hard to threshold later (see
[ch. 3](../3-classification/) on precision/recall). Trees give you a
cheap probability: **the fraction of training rows of each class in the
leaf**.

If a leaf saw 50 training rows, 40 of them class k, then `predict_proba`
is 0.8 for k. That is a **frequency in a box**, not a Bayesian posterior
and not a score that is automatically comparable across leaves of
different sizes.

**Problem** — Product language says "the model is 80% sure."

**Solution** — Say "80% of the training rows that fell in this rectangle
were class k." Then ask whether the leaf has five rows or five thousand.

**Failure mode** — Treating leaf frequencies as calibrated probabilities
and setting a 0.5 threshold as if it were logistic regression. Small
leaves are overconfident. Deep trees manufacture tiny pure leaves on
purpose. If you need a probability for a cutoff, use a hold-out
calibration set, or wait for an ensemble that averages many leaves.

## CART

sklearn's trees are **CART** (Classification and Regression Trees):
**binary** splits, greedy, one feature and one threshold per node.

The training question at a node is: among all features *j* and all split
points *t* that the training rows at this node suggest, which pair
minimizes a **cost** that says "the two children should be purer (or,
for regression, tighter) than we are now"?

```
  for each feature j:
      for each candidate threshold t:
          split the node's rows
          score = (n_left/n) * impurity(left)
                + (n_right/n) * impurity(right)
  pick the (j, t) with the lowest score
  recurse on each child
```

Greedy means: the algorithm does **not** look ahead. A split that looks
mediocre now but unlocks a perfect cut later will be missed. Finding a
globally optimal tree is a hard combinatorial problem; CART does not
try. Live with "good enough partitions," then regularize.

**Failure mode** — Expecting the first split to be "the most important
feature in the domain." It is the feature that most reduced impurity
*given the greedy rule and the sample*. Correlated features steal splits
from each other. A domain expert's favorite variable can sit one level
down and still be the one you should act on.

### Computational complexity

Order-of-magnitude, not a compiler lecture:

- **Training** (sklearn's CART-style implementation): you repeatedly
  sort (or scan sorted) values to try thresholds. A common bound is
  roughly *O(m n log n)* for *n* rows and *m* features, with extra cost
  if you let the tree grow to one row per leaf.
- **Prediction**: walk a path. If the tree is reasonably balanced,
  *O(log n)* comparisons. If you allowed a deep spine, closer to *O(n)*.

That split of costs is why trees are a default when you need **fast
inference** on CPUs and why unbounded depth is a production smell even
before you talk about overfitting.

**Problem** — "Trees are cheap" used as an excuse not to cap depth or
leaf size.

**Solution** — Cheap *per comparison*. A million-node tree is a
million-node memory object and a long path. Set `max_depth`,
`min_samples_leaf`, or `max_leaf_nodes` as part of the model, not as a
later "interpretation" pass.

**Failure mode** — Fitting one huge tree on a wide table in a notebook,
then copying the pickle into a latency-sensitive service. The ensemble
chapter will multiply trees; get one tree's cost picture first.

### Gini vs entropy

Two impurity measures you will see on the drawing and in
`criterion=`:

- **Gini.** Roughly: probability of mislabeling a random training row in
  the node if you labeled it according to the node's class mix. Pure
  node → 0. A 50/50 two-class node is high. Formula you can compute by
  hand: 1 − Σ p_k².
- **Entropy.** Information-theory cousin: −Σ p_k log₂ p_k. Also 0 when
  pure, high when mixed.

They almost always grow **similar trees**. Gini is a bit cheaper (no
log). Entropy is sometimes said to produce slightly more balanced
splits. For this workshop, pick one, keep it fixed while you learn
regularization, and do not retune criterion to chase a 0.3% CV bump.

**Problem** — A review thread that treats Gini vs entropy as the
strategic choice.

**Solution** — The strategic choices are **depth, leaf size, and whether
you should be using one tree at all**. Criterion is a second-order
tweak.

**Failure mode** — Switching criterion to "fix" a tree that is
overfitting because `max_depth=None`. Impurity is how you *score a
split*, not how you *stop memorizing*.

## Regularization hyperparameters

An unconstrained CART tree will keep splitting until leaves are pure (or
until it runs out of distinct rows). On noisy labels that is a
**memorization machine**. Regularization is how you keep the flowchart
smaller than the dataset.

sklearn knobs you should be able to name and predict the direction of:

| Knob | What it does if you tighten it |
|---|---|
| `max_depth` | Shorter paths; coarser boxes |
| `min_samples_split` | Nodes with too few rows cannot split |
| `min_samples_leaf` | Leaves cannot be tiny |
| `min_weight_fraction_leaf` | Same idea, in weight-fraction units |
| `max_leaf_nodes` | Best-first growth up to a leaf budget |
| `max_features` | Each split sees only a random feature subset |

`min_impurity_decrease` (and older `min_impurity_split`) say "do not
bother splitting unless impurity drops enough."

Default sklearn classification trees are **not** conservative. If you
fit with defaults on a dataset of thousands of rows, expect train
accuracy near 1.0 and a validation gap. That gap is the lesson, not a
broken install.

**Problem** — Regularizing by staring at train accuracy.

**Solution** — Treat depth and leaf size as hyperparameters. Use the
same cross-validation discipline as [ch. 2](../2-end-to-end-project/).
Plot train vs validation as you increase `max_depth`: you should see the
classic U of underfit → sweet spot → overfit.

**Failure mode** — Pre-pruning until the tree is a stump because a
stakeholder asked for "something we can put in a PDF," then claiming
trees "don't work" on the problem. Interpretability and capacity are a
trade. If you need both capacity *and* a stable importance story, that
is the next chapter, not a deeper single tree.

A note on **post-pruning**: grow, then cut branches that do not pay
their way on a validation criterion. sklearn grew a
`ccp_alpha` cost-complexity path after the book's 2019 snapshot. The
idea is older than the parameter name. Either pre-prune with the table
above or prune after; do not skip both.

## Regression trees

Same flowchart, different leaf payload and different split cost.

- **Leaf value:** usually the **mean** of the training targets in the
  box (so a piecewise-constant function).
- **Split cost:** typically MSE (or MAE): how much variance (or absolute
  error) would we remove by this cut?

```
  x-axis: one feature
  y-axis: target

  staircase:  ----     ------
                  ----
              [ t1 ] [ t2 ]
```

The prediction surface is a set of **axis-aligned steps**, not a line.
That is excellent when the true function is locally flat and terrible
when it is a gentle slope: the tree approximates the slope with many
small stairs, which is how regression trees overfit noise as "detail."

`DecisionTreeRegressor` takes the same regularization knobs. Unbounded
depth will interpolate the training set (each row can get its own leaf
if features unique). The train MSE can be ~0; the test MSE will not.

**Failure mode** — Using a regression tree because "the plot looked
nonlinear," then comparing it to a linear model on **training** RMSE.
The tree will win the training contest by construction. Hold out data,
or you have learned nothing.

## Instability

Trees look like knowledge. They behave like a brittle partition.

### Rotation

Every split is **orthogonal to a feature axis**. A two-class blob
separated by a diagonal line becomes a staircase of many cuts. Rotate
the same data 45 degrees and CART may need a completely different, often
deeper, tree.

```
  nice axis-aligned blob     same blob, rotated
  one split might suffice    many stair splits
        |                         / / / /
        |                        / / / /
```

**Problem** — A production feature that is a rotation or linear mix of
two sensors (difference, ratio, PCA-unaligned coordinates).

**Solution** — Either engineer a feature that *is* aligned with the
decision you care about, or do not expect a shallow tree to find the
diagonal. PCA as a preprocessor can sometimes rotate variance onto axes
trees like — at the cost of making the splits less readable. That trade
is [ch. 8](../8-dimensionality-reduction/), not a tree hyperparameter.

**Failure mode** — Declaring the domain "not a tree problem" because a
raw (x, y) plot is diagonal. Often it *is* a tree problem after one
feature that names the diagonal.

### Small data changes

CART's first split is a hard argmin over many (feature, threshold)
pairs. Two almost-equal splits: a handful of rows, a resample, a
corrected label, and the winner flips. Every descendant node then sees a
different subset. The whole flowchart can rewrite itself while the
decision boundary barely moves.

That is **high variance** in the sense of [ch. 4](../4-training-models/):
the estimator jumps around the hypothesis space when the sample jumps.

Consequences you should expect:

- Retraining on last month's data plus 50 rows can change which feature
  sits at the root, even if accuracy is the same.
- Feature-importance stories from **one** tree are not a governance
  artifact. They are a sample of a twitchy argmin.
- The fix is not "more max_depth." The fix is **average many twitchy
  trees** that were given different views of the data. That is
  [ch. 7](../7-ensembles/).

**Failure mode** — Shipping a single tree as a "policy" because legal
wanted if-then rules, then discovering after a data refresh that the
rules changed and nobody has a diff of the flowchart. If you must ship
one tree, freeze the training snapshot, version the DOT, and measure how
often the root feature flips under bootstrap of the training set. If it
flips a lot, you do not have a policy. You have a sample path.

## What aged since 2019

The *idea* of CART is older than this book and has not been replaced.
What moved around it:

- **Drawing.** `sklearn.tree.plot_tree` is the default notebook tool.
  Graphviz is still fine; it is no longer the only path.
- **Pruning.** Cost-complexity pruning (`ccp_alpha`) is a first-class
  sklearn knob. Use it. The 2019 text leaned harder on pre-pruning
  hyperparameters.
- **Defaults.** Unconstrained `DecisionTreeClassifier` still overfits.
  Do not wait for a library default to save you.
- **Where trees actually run.** Production tabular work migrated from
  "one CART tree" and even from vanilla `GradientBoosting*` toward
  histogram-boosted successors (see the next folder's aged section). The
  single tree remains the **teaching model**, a **baseline**, and the
  **component** inside forests. It is rarely the champion on a
  leaderboard.
- **Explanation tooling.** SHAP and friends explain *ensembles*. They
  do not make one unstable tree stable. Do not confuse a nicer waterfall
  plot with a nicer estimator.
- **Categorical splits.** HistGradientBoosting and CatBoost-style
  tools handle categoricals more natively than 2019 CART in sklearn.
  For *this* chapter, still encode, then split on numbers.

Keep the mental model. Update the drawing API. Do not ship the unbounded
tree.

## Check yourself

1. Walk one Iris-style row down a depth-2 tree you drew by hand. At
   each node name the feature, the threshold, and which child you take.
   What is the predicted class, and where did that class *come from*?
2. A leaf has 8 training rows, 7 of them class A. What does
   `predict_proba` emit for A? Name one reason you would not treat that
   as "the model is 87.5% sure" in a medical threshold.
3. CART is greedy. Give a two-split cartoon where the globally best
   first cut is *not* the impurity-minimizing first cut. Why does sklearn
   not search for that global tree?
4. Training cost vs prediction cost: which one grows with how *deep*
   you allowed the tree to become, and which one is dominated by trying
   many (feature, threshold) pairs?
5. A teammate wants to switch Gini to entropy to close a 4-point gap
   between train and validation accuracy. What should you look at
   *before* criterion?
6. List three hyperparameters that make leaves larger (or fewer). For
   each, say what happens to bias and variance if you tighten it.
7. Sketch a 1-D regression tree of depth 2 on a noisy line. Why can
   train MSE be near zero at large depth even when the true function is
   a straight line?
8. Rotate a diagonally separable 2-D blob onto the axes and off again.
   Why does the number of splits change? What feature would make the
   tree shallow again without rotating the whole dataset?
9. You retrain the same pipeline on yesterday's data plus 30 cleaned
   rows. The root feature changes; test accuracy does not. Is the model
   "wrong"? What quantity would you bootstrap to see if you should trust
   a single-tree policy document?
10. Someone says "we will use one deep tree so we can explain the
    forest we cannot explain." What two properties of CART make that
    sentence false?

Continue to [Ensemble Learning and Random Forests](../7-ensembles/).
