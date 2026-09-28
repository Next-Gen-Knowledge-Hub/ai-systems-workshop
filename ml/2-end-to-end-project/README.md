# 2. End-to-end Machine Learning project

Companion notes for **Chapter 2** of *Hands-On Machine Learning with
Scikit-Learn, Keras, and TensorFlow* (2nd edition, Aurélien Géron;
O'Reilly, 2019).

This chapter is the **whole loop** on one tabular problem, shaped like
district-level California housing: frame, split, explore, transform,
fit, select, and only then look at the test set. Skip it and you will
know the names of estimators from later chapters and still leak the
target into the features, tune on the test slice, and call a notebook
"production."

## The mental model

```
  FRAME  what is the decision? what number would change it?
    |
    v
  LOCK A TEST SET   (stratify if a slice must not vanish)
    |               wall: no peeking, no transform fit
    v
  EXPLORE TRAIN ONLY   plots, correlations, combo ideas
    |
    v
  PIPELINE             clean, encode, scale, estimate
    |                  .fit on train folds only
    v
  SELECT               CV, grid / random search, maybe a blend
    |
    v
  CONFIRM              one test-set number, then stop tuning
    |
    v
  LAUNCH / MONITOR / RETRAIN   the model is a living snapshot
```

The one sentence to remember a year from now: **the pipeline is the
model**, and the test set is a **one-shot audit**.

Two consequences. First, a heroic RMSE from a notebook that scaled on
all rows is a leak. Second, launching without a monitor for *input
drift and metric drift* is how last quarter's winner becomes this
quarter's silent wrong prices.

## Frame the problem

Write the decision before you write `read_csv`. The book's running
example predicts a district's median house value from census-style
features (income, rooms, location, ocean proximity). That grain —
district, not individual listing — decides everything downstream.

- Who consumes the prediction (a pricing tool, a human reviewer, a
  downstream optimizer)?
- Is the job **supervised regression** (a number), classification, or
  ranking? Housing-shaped work is usually "predict a typical value for
  a district," which is regression. If the business actually wants
  "above/below a cutoff," that is a different metric and a different
  chapter.
- Batch or online? Most first projects are **batch**: train on a
  snapshot, score a file or a nightly job. Real-time scoring is still
  batch *learning* if the weights are frozen.
- Is there an existing baseline (district median, last year's price,
  a vendor AVM)? If you cannot beat it on a fair split, you do not have
  a project. You have a plot.

The ticket says "predict house prices" and everyone imagines a
different output grain (listing vs district vs buyer). Freeze grain,
target, and the **action** the number triggers. "We will feed ŷ into a
downstream allocation" is a different product from "an analyst glances
at a chart." Training on listing-level rows, then reporting
district-level error (or the reverse), optimizes a number nobody uses.

### Performance measure: RMSE vs MAE

A performance measure is the loss you are willing to **select
models** on.

Say a district's true median value is \$200,000. One model predicts
\$210,000 (error \$10k). Another predicts \$350,000 (error \$150k).
**Mean absolute error (MAE)** averages the absolute misses: both errors
enter linearly, so ten \$10k mistakes look like one \$100k mistake.
**Root mean squared error (RMSE)** squares each miss before averaging
and taking the square root, so the \$150k disaster dominates the
score. In symbols, for *m* districts:

```
  MAE  = (1/m) Σ |ŷᵢ − yᵢ|
  RMSE = √( (1/m) Σ (ŷᵢ − yᵢ)² )
```

If a few disastrous districts dominate the business pain, RMSE is the
more honest selector. If the typical miss matters and outliers are
either noise or someone else's problem, MAE is kinder — and less
hijacked by one luxury tail.

Neither is "the scientific one." Map the measure to the **cost of
being wrong**. A 2× error on a cheap district may be cheaper than a
0.3× error on a dense one; if that is true, you may need a weighted
metric. Selecting on RMSE all week, then telling the business "median
error" on Friday because RMSE looked less flattering, trains one thing
and sells another.

### Check assumptions

List the stories the pipeline is allowed to believe: the target is
the right grain; features will be available *at score time* (no
future leakage); categories are stable; the geographic coverage of
train is the coverage of prod. A feature "sale_closed_at" that exists
only after the event you are predicting will look like the best column
you have ever seen.

## Workspace, data, and the first look

You need a repeatable environment (same sklearn, same pandas, pinned
once), a way to **fetch** a snapshot (not a mystery download from
someone's laptop), and a habit of looking at **structure** before
algorithms: row count, dtypes, missingness, obvious sentinels, whether
an id is unique, whether the target is already in the file under
another name.

This is not EDA-as-art. You are hunting for **lies in the table**:
duplicate districts, a column that is a function of the target, a
categorical with 8,000 levels that will explode a one-hot.

```python
# Structure, not modeling. Do this on a working copy, not the test set.
frame.info()
frame.isna().mean().sort_values(ascending=False).head()
frame.describe(include="all").T
```

`info` prints dtypes and non-null counts. The `isna().mean()` chain
ranks columns by missing fraction. `describe` summarizes numeric (and
optionally categorical) ranges. If `head()` looks clean, you can still
miss missingness in the last 20% of the file — exactly the region you
care about.

## Create the test set (and do not snoop)

Split **before** exploration plots that will change your mind, and
before any transformer `.fit`. A random 20% is the default cartoon.
It is wrong when a **stratum** must stay balanced: income bands,
fraud vs not, plant id. In the housing example, median income strongly
drives house value. If 4% of districts are "high income" and your
test set accidentally gets 1%, every later number is noise.

**Stratified sampling** draws the test set so each stratum keeps its
share. You pick the stratum from a variable that must not swing the
metric by chance — often a coarse bin of a driver (income, severity),
not the target itself unless you have a good reason.

```python
from sklearn.model_selection import StratifiedShuffleSplit

splitter = StratifiedShuffleSplit(n_splits=1, test_size=0.2,
                                  random_state=42)
for train_idx, test_idx in splitter.split(frame, frame["income_band"]):
    train = frame.iloc[train_idx].copy()
    test = frame.iloc[test_idx].copy()
```

`StratifiedShuffleSplit` yields train/test index arrays that preserve
the class (or bin) proportions of `income_band`. Lock `random_state`.
Persist the ids. If a future row's id was in train last month, it does
not get to be test this month unless you are running a new experiment
with a new protocol.

**No snooping** means: do not look at test distributions "just to
sanity check" in a way that changes features, bins, or which model
you pick. Do not `fit` a scaler on `train+test`. Do not drop outliers
using a threshold you computed on all rows.

A "random" split that is actually ordered by time, so test is the
future — which can be *correct* for forecasting — mixed with i.i.d. CV
as if time did not exist, is two protocols smashed together. Name the
protocol. Time-based vs stratified-i.i.d. are different walls.

## Visualize, correlate, combine (train only)

Plots are cheap ways to find **nonlinearity, clusters, and bad
geocodes**. On a housing-shaped table you expect geography to matter:
nearby districts are not independent. That does not license copying
anyone's map. It licenses asking, on *your* train frame, whether
space, income, and density move together.

**Correlations** (Pearson on linear-ish columns, rank correlations
when the scatter is bent) tell you which raw columns move with the
target. They do not tell you causation and they miss interactions. In
the book, median income usually leads the pack for median house value;
total rooms is weaker until you normalize by households.

**Attribute combinations** are where domain sense still wins: rooms
per household beats rooms; bedrooms per room; population per household.
Invent them on train, put them in the **same pipeline** you will use
at score time, and let CV decide if they help. Engineering twenty
ratios by staring at the test scatter overfits the audit set with
extra steps.

## Prepare the data

Preparation is not a prelude. It **is** the model, because production
must apply the same steps.

### Cleaning

Missing numeric values: impute (median is the boring default for
skewed columns) or drop, but **fit the imputer on train folds**.
Missing categoricals: a dedicated "missing" level often beats silent
mode fill. Duplicates and impossible values (negative counts) are
bugs.

### Categoricals

Nominal fields (ocean proximity, plant, channel) need an encoding
that does not invent an order. One-hot is the chapter's workhorse;
it explodes when cardinality is huge. Ordinal encoding is for
*actual* orders (low < med < high). Label-encoding city names into
0..N and feeding them to a linear model teaches "Paris = 2 × Berlin."

### Custom transformers

Anything you would have done in an ad-hoc pandas cell (ratio columns,
clipping, rare-level grouping) belongs in a transformer with
`fit`/`transform` so CV can include it. If it only exists in a
notebook cell above `GridSearchCV`, it will drift from prod.

```python
from sklearn.base import BaseEstimator, TransformerMixin

class RatioAdder(BaseEstimator, TransformerMixin):
    def __init__(self, numer, denom, name):
        self.numer, self.denom, self.name = numer, denom, name

    def fit(self, X, y=None):
        return self

    def transform(self, X):
        out = X.copy()
        out[self.name] = out[self.numer] / out[self.denom]
        return out
```

`fit` returns `self` because the ratio needs no statistics from the
training set. `transform` adds the new column so every CV fold and
production score path applies the same formula. That is a shape; your
columns will differ.

### Feature scaling

Linear models, SVMs, neural nets, and regularized methods care about
scale. Trees mostly do not. **Standardization** (zero mean, unit
variance) and **min-max** (squash to a range) are the two defaults.
Fit on train. Transform val/test with those statistics. Never
recompute mean on the batch you are scoring in production unless you
*intend* a different protocol (you almost never do).

### `Pipeline`

The pipeline is how you make the above non-leaky and deployable:

```
  [num branch]  impute --> ratios --> scale
                                      \
  [cat branch]  impute --> one-hot ----+--> estimator
```

`ColumnTransformer` + `Pipeline` + an estimator is one object. `.fit`
on train (or a CV train fold). `.predict` on anything else.

```python
from sklearn.compose import ColumnTransformer
from sklearn.pipeline import Pipeline
from sklearn.impute import SimpleImputer
from sklearn.preprocessing import OneHotEncoder, StandardScaler
from sklearn.linear_model import LinearRegression

prep = ColumnTransformer([
    ("num", Pipeline([
        ("impute", SimpleImputer(strategy="median")),
        ("scale", StandardScaler()),
    ]), num_cols),
    ("cat", Pipeline([
        ("impute", SimpleImputer(strategy="most_frequent")),
        ("onehot", OneHotEncoder(handle_unknown="ignore")),
    ]), cat_cols),
])
model = Pipeline([("prep", prep), ("est", LinearRegression())])
model.fit(X_train, y_train)
```

`ColumnTransformer` routes numeric and categorical columns through
separate sub-pipelines, then concatenates the results. Wrapping that
prep with `LinearRegression` means one `.fit` learns imputer
statistics, scaler means, one-hot categories, *and* the regression
coefficients from train alone. Scaling in a cell, then
`cross_val_score(estimator, X_already_scaled, y)`, lets each fold's
scaler see the held-out fold. The CV number is optimistic.

## Train, select, and touch the test set once

Start with a **dumb baseline** (predict the median target) and a
**simple linear** model. Then a model that can bend (trees,
ensembles). You are asking whether extra capacity pays for itself on
CV.

**Evaluate on train** to know if you can even fit. A linear model with
huge train RMSE is underfitting or missing a leak/transform. A tree
with near-zero train RMSE and sad CV is overfitting.

**Cross-validation** (k-fold, usually shuffled; stratified if a class
or a bin must stay balanced) is the selector. Report mean **and**
spread. A 1% win with huge fold variance is not a win.

**Grid search** exhausts a small discrete grid of hyperparameters
(depth, regularization, imputer strategy). Fine when the grid is
tiny.

**Randomized search** draws from distributions when the space is
bigger. You get more coverage per CPU hour and a better chance of
finding that the interesting knob was not the one you discretized.

**Ensembles as a fine-tune move**: once a few *different* models are
decent, a blend or a stack can shave error. It is a last-mile tool,
not a substitute for a clean split and a sane target. Do not stack
ten copies of the same leak.

**Test set once.** After you pick *one* pipeline, score test. If you
hate the number, you may analyze **why** (drift, a stratum, a bug).
You do not get to run another grid and report the new test number as
if it were still unbiased. That second number is a rumor. "We did CV"
but the pipeline's scaler sat outside the CV loop, or twenty test-set
peeks renamed as "final-final-v8," are the same protocol failure.

## Launch, monitor, maintain

A fitted pipeline is a **snapshot of a distribution**. Shipping it
means:

- Serialize the **whole** pipeline, not just the estimator.
- Version the data snapshot and the code together.
- Monitor **inputs** (missingness, category mix, range) as well as
  **outputs** (prediction distribution) and a delayed **label metric**
  (RMSE/MAE vs eventual truth).
- Decide a **retrain trigger** (calendar, drift, or a metric floor).
  Retraining without a locked protocol is how you overfit last week's
  incident.

A weekly retrain on "all data we have" that slowly eats what used to
be the test period, with no holdout in time, will always look like you
are improving. You are watching **tabular model health**: input
distributions and delayed RMSE. That is a different job from tracing
LLM calls.

## What aged since 2019

- `Pipeline`, `ColumnTransformer`, `SimpleImputer`,
  `StratifiedShuffleSplit`, `GridSearchCV`, `RandomizedSearchCV` are
  still the spine. `OneHotEncoder` now handles unknown categories more
  cleanly; `set_output(transform="pandas")` makes debugging less
  painful. The ethic did not change.
- `HistGradientBoostingRegressor` is a strong default tabular
  estimator that this edition does not center. Use it *inside* the
  same pipeline-and-CV discipline.
- Fetching a public housing CSV is a teaching device. Your job is a
  warehouse snapshot with access control and a data contract.
- Experiment hygiene in sklearn is still "do not peek." Online A/B for
  generative apps is a different protocol for a different artifact.

## Check yourself

1. Frame a prediction you have seen at work. What is the grain, the
   action ŷ triggers, and the baseline you must beat? What goes wrong
   if grain and action disagree?
2. For that system, would you select on RMSE or MAE (or something
   weighted)? Give a miss that RMSE would obsess over and MAE would
   shrug at.
3. Name an assumption that bit you (a feature not available at score
   time, a category that appeared after launch). How would you have
   checked it in this chapter's "assumptions" pass?
4. You plotted the *full* frame, including test, "just to see the
   map." What kind of snooping is that, and what decision might it
   contaminate?
5. Why stratify, and on what? Give a slice of a real table that a
   naive shuffle would starve.
6. Walk a leak: a scaler or imputer fit on all rows. Where in *your*
   last notebook did that happen, or could it?
7. You want a ratio feature (rooms per household, spend per account).
   Why does it belong in a transformer inside the pipeline rather than
   a pandas cell above `fit`?
8. Grid vs randomized search: when does exhaustive search become a
   false sense of rigor? Steal an example from a hyperparameter you
   have actually swept.
9. The test RMSE is worse than CV. List three explanations (overfit
   of the val path, mismatch, a bug) and which plot or slice on a
   system you know would distinguish them.
10. After launch, which three monitors would you page on for a tabular
    housing-style pipeline, and what would each catch?
