# Appendix B. Machine Learning project checklist

Companion notes for **Appendix B** of *Hands-On Machine Learning with
Scikit-Learn, Keras, and TensorFlow* (2nd edition, Aurélien Géron;
O'Reilly, 2019).

The end-to-end project chapter walks this spine on California housing:
frame, split, explore, pipeline, shortlist, fine-tune, present,
launch. This appendix is the same spine **without the dataset**, so you
can copy it onto the next problem. Skip it and you will remember the
housing plot and still ship a model with a leaked test set, no
rollback, and a metric nobody asked for.

The one sentence to remember a year from now: **every item below
exists to prevent a specific lie** — to yourself, to a stakeholder,
or to the production log.

This is not a dump of the book's appendix and not solutions to
Appendix A. If a line here does not tell you *why* and *what breaks*,
it does not belong on a workshop wall.

Use this list when you are **fitting and shipping weights from a
table, pixels, or a similar dataset**.

## The mental model

```
  1 frame          what decision, what metric, what constraints
        |
        v
  2 get data       legal, stored, a test set you will not fondle
        |
        v
  3 explore        on train (or a copy). Hypotheses, not conclusions
        |
        v
  4 prepare        functions / pipelines. Fit on train only
        |
        v
  5 shortlist      many dumb models, honest CV, diverse errors
        |
        v
  6 fine-tune      more data, search, ensembles; *then* one test peek
        |
        v
  7 present        the decision, not the leaderboard
        |
        v
  8 launch         serve, monitor inputs *and* outputs, retrain
```

Walk the eight phases below in order on your next project. Write
answers in your own words for that problem. If a heading does not
bite, you are probably looking at a different kind of product, or you
skipped framing.

## 1. Frame the problem

When you frame the problem, write down the **decision** someone will
take with a number: approve, rank, alert, price, route. "Build a
model" is not a problem. Framing writes that decision down before you
touch CSV files, so the metric and the data match the job.

Ask, in writing:

- What is the business outcome, and how will this score be *used*?
  (Online API? Batch report? A human still decides?)
- What is the current workaround (rules, a vendor, a human)? That
  is your baseline, not "random."
- Supervised or not? Classification, regression, ranking, clustering,
  or a reward-driven interaction problem?
- Batch or online learning? Does the world move weekly?
- What performance measure matches the decision (MSE vs MAE vs
  precision at a recall floor)?
- What is the minimum number that would still be *worth shipping*?
- What would a competent human do with the same inputs?
- List assumptions (IID rows, missingness at random, labels are
  ground truth). Then try to falsify them.

If you skip framing, you optimize RMSE because a tutorial did, while
the business cares about under-pricing a tail of houses. Or you build
a classifier for a problem that needed a ranking. Or you discover in
month three that labels are the previous model's scores. Framing is
cheap. Rework is not.

## 2. Get the data

When you get the data, treat logistics as first-class work. Most ML
failures are data-logistics failures with a neural costume: you
cannot replay the extract, you were not allowed to use that column,
the test set was the last file you downloaded and you already plotted
it.

Do the unglamorous work:

- List sources, volumes, and **who owns** them.
- Check legal / contractual use, retention, and PII. Get the yes in
  writing. Delete or hash what you must not keep.
- Size on disk; format; whether this is a snapshot, a stream, or
  geography / time (those leak if you shuffle like housing IDs).
- Create a workspace that is not `Downloads/`.
- Convert once to a format you can version (tables, TFRecords for
  deep nets).
- **Carve a test set now** and put it behind a door. Stratify if the
  target is imbalanced or if a category must appear in both sides.
  Do not "just peek."

If you skip this phase, every chart you draw on the full file is a
tiny hyperparameter search (snooping). Legal failure: a model you
cannot ship. Replay failure: you cannot rebuild the training set when
someone asks "what was v3 trained on?"

## 3. Explore

When you explore, form hypotheses **without** fitting twenty models
first. Pipelines encode guesses ("this skew wants a log," "this
missingness is a signal"). Exploration is also how you catch that the
label is a function of a future column.

Rules of the room:

- Explore a **copy**, preferably of *train* (or a sample of train).
  The test set is still behind the door.
- For each attribute: type, missingness, obvious noise, usefulness,
  rough distribution.
- For supervised tasks, stare at the **target** until its units are
  boring.
- Plots: univariate, then the ones that stress your assumptions
  (geo, time, target vs a suspect leak).
- Correlations and simple attribute combos — rooms-per-household
  instincts, ratios, composites that a domain person would invent.
- Write down how you would solve it **manually**. If you cannot, you
  do not understand the features yet.
- Note extra data that would help, and transformations worth trying.

If you skip exploration, you discover the leak after the press
release. Or you spend GPU hours on a column that is 99% missing. Or
you skip the manual story and cannot explain the model because you
never had a story.

Exploration is not a license to tune. If you change the task because
of a plot, that is framing again — go back to section 1 on purpose.

## 4. Prepare

When you prepare the data, turn ad-hoc notebook cells into a dataset
you can rerun. The same clean/encode/scale steps must run in
training, CV, and production. Use `Pipeline`, `tf.data`, or Keras
preprocessing — the checklist item is **reproducible transforms, fit
on train only**.

Typical work, always as functions:

- Clean: outliers (fix, clip, or drop with a rule), missing values
  (impute or drop *with a recorded policy*).
- Select: drop attributes that are IDs, leaks, or empty.
- Engineer: decompose (date → month), combine (ratios), discretize
  only if you can say why.
- Scale / encode: fit the scaler and the one-hot vocabulary on
  **train folds**, not on the universe.

If you skip prepare, leakage walks in through the scaler (min/max of
the test set in the train transform). A production client that "does
preprocessing" differently creates serving skew. A teammate who
cannot rerun your notebook because cell 17 edited the dataframe in
place and cell 4 no longer runs is a process failure you chose.

## 5. Shortlist models

When you shortlist, train breadth with cheap models so you learn the
shape of the errors. The first algorithm you remember is rarely the
one you should ship, and a single CV number without error analysis
will crown a lucky overfit.

- Train several reasonable defaults (linear, tree, forest / boost,
  maybe a small net if the data is pixels or sequences).
- Compare with **N-fold CV** on train. Same folds, same metric as
  framing.
- Look at **which features** each model clung to and **which errors**
  it made (confusion matrices for classification; residual plots for
  regression).
- Prefer a shortlist of 3–5 that fail **differently** — that is
  what ensembles later buy.
- One quick round of extra features if error analysis screams for
  them; do not disappear into a competition rabbit hole yet.

If you skip shortlisting, you grid-search a neural net for a week on
a problem a ridge model already solved. Or you ensemble three clones
of the same tree and call it diversity. Or you pick a winner on a
single train/val split that happened to be kind.

## 6. Fine-tune

When you fine-tune, defaults are over. Shipping wants the search you
can defend, on as much training data as you can honestly use, with the
test set still unused.

- Automate: grid, random, or a Bayesian-class search. Know which
  knobs actually move the metric. If the search is a fleet of GPU
  jobs, treat each run as a logged trial with a clear objective.
- Try ensembles when the shortlist disagrees usefully.
- Keep a log: params, fold scores, code version, data version.
- When you are ready to **stop**, measure **once** on the test set.
  That number is for the report, not for another tuning loop.

If you skip discipline here, you peek at test, tweak, peek again —
the number is now a training number in costume. Or you tune 40
hyperparameters with 10 trials and ship noise. Or you fine-tune so
hard on CV that the test drop is a surprise you had no monitoring
plan for (see launch).

## 7. Present

When you present, put the score back into the framing from section 1:
what to do, when not to trust it, what it costs. A model that cannot
change a decision was a hobby.

- Write the story: question, data, metric, baseline, lift, limits.
- Visuals that carry the argument (error by segment, not a wall of
  ROC curves unless the operating point is the product).
- Call out assumptions that survived and ones that did not.
- Name the failure modes (slice X is bad, labels drift, human
  override still required).
- Recommend a ship / no-ship / ship-with-guardrail call.

If you skip presentation, engineering dumps an F1 in Slack. The
stakeholder hears "it works." Six weeks later nobody can say why the
false negatives cluster in one region — you saw it in explore and
never put it on a slide.

## 8. Launch, monitor, maintain

When you launch, treat production as a moving input distribution, a
moving label process, and a serving path that can skew. Offline CV is
a snapshot. Launch is **plumbing + alarms + a retrain loop**, not a
ceremony.

- Plug into production inputs through the **same** prepare path you
  fitted (SavedModel signatures, sklearn `Pipeline`, not a rewrite).
- Version the artifact; know how to roll back.
- Monitor **outputs** (metric proxies, prediction distributions,
  latency) *and* **inputs** (missingness, schema, drift of features
  you depend on). Alerts need owners.
- Retrain on a schedule or on a drift trigger, with the same
  checklist from "get data" downward — including a fresh honest
  split when time is the axis.
- Record what vN was trained on. You will be asked.

If you skip launch hygiene, silent skew appears: training used
log-price and serving uses price. Silent drift: a sensor dies at zero
and the model is confidently wrong. Silent process death: the person
who knew the notebook left. Monitoring only the HTTP 200 rate is how
classifiers rot in place.

## How you know you are ready to launch

You are ready to launch from *this* checklist when you can answer yes
to every phase in writing for *this* problem:

1. The decision, metric, baseline, and ship-worthiness threshold are
   written down, and the assumptions you cannot defend are named.
2. Sources, ownership, and legal use are clear; the test set has been
   carved and left alone; you can rebuild the training extract.
3. Exploration ran on train (or a train sample); you have a manual
   story and a short list of transforms worth trying; leaks you
   spotted were fixed or documented.
4. Prep is functions / a pipeline fitted on train only; the same path
   will run in CV and in production.
5. Several cheap models have honest CV scores and *different* error
   patterns; you shortlisted 3–5 for a reason.
6. Search and ensembles are logged with data and code versions; the
   test set has been measured **once** for the report.
7. Stakeholders have a ship / no-ship / guardrail recommendation,
   plus known failure slices and surviving assumptions.
8. The serving path uses the same prepare contract, versions roll
   back, inputs *and* outputs have owners on alerts, and a retrain
   loop (schedule or drift) is defined with a fresh honest split when
   time matters.

If any of those eight is still a shrug, stay in that phase. Launching
with a shrug is how the lies this appendix exists to prevent reach the
production log.
