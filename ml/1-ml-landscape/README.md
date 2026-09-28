# 1. The Machine Learning landscape

Companion notes for **Chapter 1** of *Hands-On Machine Learning with
Scikit-Learn, Keras, and TensorFlow* (2nd edition, Aurélien Géron;
O'Reilly, 2019).

This chapter is the map for the whole ML track. Machine learning is
software whose behavior is set by examples, rather than by a growing
pile of hand-written branches. Skip it and later folders look like a
zoo of estimators, while you still cannot say whether you should train
at all, which learning mode you are in, or why a nice hold-out number
died the week the product met real traffic.

## The mental model

```
  observations (rows, pixels, events)
              |
              v
       [  DATASET  ]
              |
      +-------+--------+
      |                |
   labels /            no labels
   a reward            (or the reward is
      |                 "find structure")
      v                      |
  fit a predictor            v
  class / regress /     cluster, reduce,
  rank / policy         density, associate
      |                      |
      +----------+-----------+
                 v
          [ GENERALIZE ]
          useful on *new* rows,
          not on the training sheet
```

The one sentence to remember a year from now: **the machine is the
thing that changes when the data changes** — and the only honest test
is data it has not trained on.

Two consequences fall out of that diagram. First, a clever regex you
wrote is not ML, even if it is "smart"; the program does not change when
the examples change. Second, calling a hosted model is not ML *you*
did. The provider already trained. Your job in this track starts when
*your* distribution has to move the weights.

## What machine learning is

"We added AI" can mean a rule engine, a vendor API, a spreadsheet
formula, or a fitted estimator. Design reviews stall because those
four jobs share a slide title. Use a working definition you can test: a
program **learns** if its performance on a task improves after seeing
examples, without you rewriting the decision logic by hand. The
artifact is a **model** (parameters plus the code that applies them).
The fuel is a **training set**. The claim you are allowed to make is a
number on a **held-out** set.

A dashboard that "learns" because an analyst edits thresholds every
Friday is operations, not a learner. The thresholds did not come from a
training procedure you can re-run. Treat "the model" as a function
`f(x) → ŷ` with a loss you chose, rather than as a personality.

### Why write a learner instead of rules

Rules win when the logic is short, stable, and auditable: VAT by
country, "order status is one of these four strings." Learners win when
the mapping is messy, high-dimensional, or drifting faster than you can
maintain branches: which login is fraud, which ticket is urgent, which
sku will stock out.

You also reach for ML when the same *kind* of problem keeps coming
back with new data (a new market, a new sensor) and you would rather
re-fit than re-interview domain experts for every clause. Training
because the roadmap said "AI," then discovering the labels are the rule
you already had ("overdue if days > 30"), wastes a quarter to relearn
an if-statement with worse debuggability.

```python
# Sketch: a learner is fit(), not an if-tree you keep editing.
from sklearn.linear_model import LogisticRegression

clf = LogisticRegression(max_iter=200)
clf.fit(X_train, y_train)          # parameters move
y_hat = clf.predict(X_new)         # same code, new rows
```

`fit` estimates the coefficients from the labeled training rows.
`predict` applies those fixed coefficients to new rows with the same
feature shape. That `fit` call is the whole point of the track. Later
chapters change *how* `fit` finds parameters. They do not change the
job.

## Application types (a survey, not a catalog)

Do not memorize product names. Name the **job**:

- **Classify** a row (spam / not, defect / not, intent A/B/C).
- **Regress** a number (demand next week, time-to-resolve, price).
- **Rank** (which document, which candidate, which ad).
- **Group** (which users behave alike; which machines share a fault).
- **Reduce** (too many sensors; keep the axes that move).
- **Detect the unusual** (the row that does not belong).
- **Forecast sequences** (the next reading, the next token — the
  *training* version, not a chat runtime).
- **Control** (pick an action, see a reward — reinforcement learning).

If you cannot put the ticket on that list, you do not yet know whether
this book is the right tool. "Make the assistant nicer" is not on the
list; that is product copy and interaction design.

## Supervised, unsupervised, semisupervised, reinforcement

Four families. Mixing their names is how you pick the wrong metric and
the wrong split.

```
  SUPERVISED          each row has a target y
                      class or number (or ranks)

  UNSUPERVISED        only x; you want structure
                      clusters, components, densities

  SEMISUPERVISED      a little y, a lot of x
                      use the unlabeled mass carefully

  REINFORCEMENT       an agent acts; a scalar reward
                      arrives, often delayed
```

**Supervised** is the default in Part I of HOML: you have a column you
wish you could type for every future row. Classification vs regression
is "discrete label" vs "number." Neither is automatically harder.

**Unsupervised** is a different question: *what is the structure?*
Clustering, PCA, density models. Using k-means as if it were a
classifier without a label policy is how "the cluster looks like fraud"
becomes an un-auditable production rule.

**Semisupervised** is the honest state of many companies: 2% of
tickets labeled, 98% sitting in the warehouse. The unlabeled rows can
help the geometry of the problem. They can also leak the test set if
you are sloppy. Do not treat "we fine-tuned on everything we had" as
semi-supervised science.

**Reinforcement learning** is a learner that **acts**, sees a **reward**,
and updates a **policy**. That requires a reward signal and a policy
update. A sense–plan–act loop around a frozen language model is a
control loop; it does not maximise a Bellman backup unless you add
those pieces.

## Batch versus online

**Batch** (offline) learning: you train on a snapshot, freeze the
parameters, ship the snapshot. Simple. Reproducible. Stale the morning
the distribution moves.

**Online** (incremental) learning: you update parameters as examples
arrive, one row or a mini-batch at a time. Useful for unbounded
streams and slow drift. You now own **forgetting**, **poisoning**, and
an evaluation story that is not "we shuffled a parquet file."

```
  BATCH                         ONLINE / INCREMENTAL
  -----                         --------------------
  snapshot --> fit --> freeze   example arrives
                                      |
                                      v
                                 partial_fit / update
                                      |
                                      v
                                 model is already live
```

Ops sometimes hears "online learning" and means "the homepage updates
when the user talks." In Géron, online means **the weights move**. A
Redis transcript is session memory. Feature pipelines that *score* a
new row with a frozen model are just **inference**. Inference can be
real-time without any learning at all.

A nightly batch job labeled "online" because it runs every night is
still batch if you retrain from scratch on a new snapshot. Cadence is
not the axis. The axis is whether `fit` saw the new rows as a stream
with an incremental API.

## Instance-based versus model-based

**Instance-based**: remember the training rows (or a subset); predict
by comparing the new point to them. k-NN is the cartoon. Fast to
"train" (often just store). Slow or RAM-heavy at predict time. The
generalization lives in the distance and the stored set.

**Model-based**: squeeze the training set into a **fixed-size** object
(weights, a tree, a net). Predict by running that object. Training is
the expensive part. Prediction is a function of the parameters, not of
every historical row.

Most of Part I is model-based, with instance-based methods as a useful
contrast. k-NN on a million customers is a product decision. Shipping
"the model" as a pickle of the entire training frame and calling it a
neural net ships a lookup. That can be valid. It is a different ops
envelope from a 12 KB linear model.

## Challenges

These are not a mood board. Each one is a bug you will hit, with a
name.

### Insufficient data

The estimator is hungry and you have 80 labeled rows. More labels beat
a fancier family *until* you have shown that the family is the
bottleneck. Transfer, data collection, and a simpler model are the
grown-up responses. Deep nets make data hunger worse, not better. A
leaderboard win on 80 rows with a 200-tree ensemble memorized the
sheet.

### Nonrepresentative data

The training distribution is not the production distribution: only
weekday traffic in train, weekends in prod; only one plant; only users
who accepted the old UI. A beautiful metric on last year's cohort, then
a silent collapse after a market launch, is sampling bias. You trained
on the people who already converted.

### Poor quality

Missingness that is not random, duplicated ids, clocks in the wrong
timezone, labels that mean three things, sensors that stick. Cleaning
is model work. It is not a prelude you skip so you can get to XGBoost.
Imputing with the global mean *including the test fold*, then bragging
about RMSE, is a leak dressed as hygiene.

### Irrelevant features

A learner given 200 columns of ids, leaked timestamps, and one useful
signal will often prefer the leak. Feature selection and domain sense
come before you celebrate a score. "We have 400 features" is not a
boast. If `user_id` or `request_time` is a top feature, you learned
which customers you already knew, or that night-shift volume is
different.

### Overfitting

The model explains the training sheet, including the noise. Hold-out
hurts. Regularization, more data, simpler hypotheses, and early
stopping are the toolkit. Cross-validation that you keep peeking at is
a slow-motion overfit of the *validation* set. Train accuracy 99%, val
91%, prod 70%, starting the same week you added 20 polynomial
features, is capacity you did not pay for with data — often blamed on
"concept drift" when the real story is memorization.

### Underfitting

The hypothesis cannot express the pattern even on train. Linear on a
curve, two trees on a XOR-ish rule, no recency feature when the world
is seasonal. Blaming the data when train and val are *both* bad, while
a scatter plot already shows a bend you refused to model, is
underfitting with a story attached.

```
  HIGH TRAIN ERROR          LOW TRAIN, HIGH VAL
  ----------------          -------------------
  underfit                  overfit
  (too small a hypothesis   (too much capacity /
   or too little signal)     too little data)
```

You will diagnose this with **learning curves** later in the book.
This chapter only needs you to have the two names.

## Testing, validating, and the split you will be tempted to cheat

You tune until the test number looks like the slide. Split first. Hide
the test set. Use the training mass for fitting, and a **validation**
path (hold-out slice or, better, cross-validation) for model
*selection* and hyperparameters. Touch the test set **once**, as a
confirmation.

```
  ALL LABELED ROWS
        |
        +-- TEST  (lock; one shot at the end)
        |
        +-- TRAIN
              |
              +-- fit folds  (CV)  --> pick family + hyps
              +-- optional tiny val for early stopping
```

**Hyperparameter tuning and model selection** are the same temptation
with two names. A hyperparameter is a knob *you* set (regularization
strength, tree depth, k in k-NN). Model selection is choosing among
families (linear vs tree vs SVM). Both consume validation signal. If
you run a hundred grid searches and pick the winner on the test set,
the test set is no longer a test.

**Data mismatch** (sometimes "train/dev mismatch"): the validation
rows are not drawn like production, even if they never appeared in
`fit`. Classic pattern: easy in-house photos in train and val, blurry
phone photos in prod. You then overfit the *mismatch*, not the task —
you tune until the lab set is perfect and the field still fails.

A "test" set that was rebuilt after each disappointing number is a
second training set with extra steps. Sampling val uniformly when the
metric that matters is the rare class leaves you wondering why
production recall collapsed.

Generalization is a protocol. Later chapters make the split mechanical
(including **stratified** sampling). This chapter only needs the ethic.

## What aged since 2019

- The sklearn *estimator* contract (`fit` / `predict` / `transform`,
  `Pipeline`, CV splitters) is still the right mental API. Names of a
  few estimators moved; the job did not.
- Histogram-based gradient boosting
  (`HistGradientBoostingClassifier` / `Regressor`) became a first-class
  sklearn answer for tabular data after this edition. It does not
  replace this chapter. It replaces some later "reach for XGBoost on
  day one" reflexes.
- The taxonomy here (supervised / batch / model-based / hold-out) is
  still the one you need when a 2026 design doc says "we will just
  use AI." The confusion with session memory, provider APIs, and
  agent loops got *worse*.
- You do not need new math to start. You need the habit of naming the
  learning mode and locking a test set.

## Check yourself

1. A stakeholder says the new rules engine "is ML because it is
   smart." What is missing from the definition in this chapter, and
   what would go wrong if you evaluated it as if it had a `fit`?
2. Pick a task from a system you have worked on. Is it classify,
   regress, rank, group, or detect-unusual? What disaster happens if
   you pick the wrong job name and therefore the wrong metric?
3. Where does *your* last "AI feature" sit on train-weights vs
   call-a-provider vs wrap-a-loop? What would break if you merged
   those three into one ops playbook?
4. Someone wants "online learning" because the chatbot must remember
   the user. Which axis are they on (weights moving vs session
   memory), and what would an incremental `partial_fit` look like
   instead?
5. Instance-based vs model-based: for a fraud table you know, which
   one were you implicitly shipping, and what fails at predict-time
   or at train-time if you guessed wrong?
6. Name a time you had too little labeled data. Did you add capacity
   (overfit) or simplify (maybe underfit)? What would a learning curve
   have shown?
7. Nonrepresentative data: describe a cohort your training set
   quietly dropped (a region, a device, a weekend). How did the metric
   stay pretty?
8. You peek at the test set every afternoon "just to check." What
   quantity have you actually overfit, and what production surprise
   does that protocol hide?
9. Train looks great, val looks great, prod dies. Is that overfit,
   data mismatch, or label leakage? Give one example of each from a
   system you know, and the check you would run tomorrow.
10. Why is an LLM agent loop not reinforcement learning as this
    chapter uses the word? What would have to exist (reward, policy
    update) before you would agree?
