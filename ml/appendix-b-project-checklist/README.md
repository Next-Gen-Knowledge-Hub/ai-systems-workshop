# Appendix B. Machine Learning project checklist

Companion notes for **Appendix B** of *Hands-On Machine Learning with
Scikit-Learn, Keras, and TensorFlow* (2nd edition, Aurélien Géron;
O'Reilly, 2019), meant to be used next to
[chapter 2](../2-end-to-end-project/).

Chapter 2 is the checklist **walked** on California housing: frame,
split, explore, pipeline, shortlist, fine-tune, present, launch.
This folder is the same spine **without the dataset**, so you can
copy it onto the next problem. Skip it and you will remember the
housing plot and still ship a model with a leaked test set, no
rollback, and a metric nobody asked for.

The one sentence to remember a year from now: **every item below
exists to prevent a specific lie** — to yourself, to a stakeholder,
or to the production log.

This is not a dump of the book's appendix and not solutions to
Appendix A. If a line here does not tell you *why* and *what breaks*,
it does not belong on a workshop wall.

If the product is an **LLM app** (chat, tools, RAG, traces, judges),
that is a different checklist — [agents ch. 7](../../agents/7-evaluation-and-feedback/)
and [platform ch. 7](../../platform/7-observability/). Do not merge
those folders into this one. Use this list when you are **fitting
and shipping weights from a table, pixels, or a similar dataset**.

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

[Chapter 2](../2-end-to-end-project/) is the worked example. [Chapter
19](../19-scale-and-deploy/) is how the launch box looks when the
artifact is a TensorFlow graph. Neither replaces the questions in
frame.

## 1. Frame the problem

**Why it exists.** "Build a model" is not a problem. A problem is a
**decision** someone will take with a number: approve, rank, alert,
price, route. Framing writes that decision down before you touch
CSV files, so the metric and the data match the job.

Ask, in writing:

- What is the business outcome, and how will this score be *used*?
  (Online API? Batch report? A human still decides?)
- What is the current workaround (rules, a vendor, a human)? That
  is your baseline, not "random."
- Supervised or not? Classification, regression, ranking, clustering,
  something [ch. 18](../18-reinforcement-learning/) would own?
- Batch or online learning ([ch. 1](../1-ml-landscape/))? Does the
  world move weekly?
- What performance measure matches the decision (MSE vs MAE vs
  precision at a recall floor — [ch. 3](../3-classification/))?
- What is the minimum number that would still be *worth shipping*?
- What would a competent human do with the same inputs?
- List assumptions (IID rows, missingness at random, labels are
  ground truth). Then try to falsify them.

**Failure if skipped.** You optimize RMSE because the book did, while
the business cares about under-pricing a tail of houses. Or you build
a classifier for a problem that needed a ranking. Or you discover in
month three that labels are the previous model's scores. Framing is
cheap. Rework is not.

## 2. Get the data

**Why it exists.** Most ML failures are data-logistics failures with
a neural costume: you cannot replay the extract, you were not allowed
to use that column, the test set was the last file you downloaded and
you already plotted it.

Do the unglamorous work:

- List sources, volumes, and **who owns** them.
- Check legal / contractual use, retention, and PII. Get the yes in
  writing. Delete or hash what you must not keep.
- Size on disk; format; whether this is a snapshot, a stream, or
  geography / time (those leak if you shuffle like housing IDs).
- Create a workspace that is not `Downloads/`.
- Convert once to a format you can version (tables, TFRecords for
  deep nets — [ch. 13](../13-data-and-preprocessing/)).
- **Carve a test set now** and put it behind a door. Stratify if the
  target is imbalanced or if a category must appear in both sides.
  Do not "just peek."

**Failure if skipped.** Snooping: every chart you draw on the full
file is a tiny hyperparameter search. Legal failure: a model you
cannot ship. Replay failure: you cannot rebuild the training set
when someone asks "what was v3 trained on?"

## 3. Explore

**Why it exists.** Pipelines encode hypotheses ("this skew wants a
log," "this missingness is a signal"). Exploration is how you form
them **without** fitting twenty models first. It is also how you
catch that the label is a function of a future column.

Rules of the room:

- Explore a **copy**, preferably of *train* (or a sample of train).
  The test set is still behind the door.
- For each attribute: type, missingness, obvious noise, usefulness,
  rough distribution.
- For supervised tasks, stare at the **target** until its units are
  boring.
- Plots: univariate, then the ones that stress your assumptions
  (geo, time, target vs a suspect leak).
- Correlations and simple attribute combos — the housing chapter's
  "rooms per household" instinct.
- Write down how you would solve it **manually**. If you cannot, you
  do not understand the features yet.
- Note extra data that would help, and transformations worth trying.

**Failure if skipped.** You discover the leak after the press release.
Or you spend GPU hours on a column that is 99% missing. Or you skip
the manual story and cannot explain the model because you never had
a story.

Exploration is not a license to tune. If you change the task because
of a plot, that is framing again — go back to section 1 on purpose.

## 4. Prepare

**Why it exists.** Ad-hoc notebook cells are not a dataset. The
same clean/encode/scale steps must run in training, CV, and
production. [Chapter 2](../2-end-to-end-project/) uses `Pipeline`
for this; deep chapters use `tf.data` / Keras preprocessing. The
checklist item is **reproducible transforms, fit on train only**.

Typical work, always as functions:

- Clean: outliers (fix, clip, or drop with a rule), missing values
  (impute or drop *with a recorded policy*).
- Select: drop attributes that are IDs, leaks, or empty.
- Engineer: decompose (date → month), combine (ratios), discretize
  only if you can say why.
- Scale / encode: fit the scaler and the one-hot vocabulary on
  **train folds**, not on the universe.

**Failure if skipped.** Leakage through the scaler (min/max of the
test set in the train transform). A production client that "does
preprocessing" differently ([ch. 19](../19-scale-and-deploy/) serving
skew). A teammate who cannot rerun your notebook because cell 17
edited the dataframe in place and cell 4 no longer runs.

## 5. Shortlist models

**Why it exists.** The first algorithm you remember is rarely the
one you should ship, and a single CV number without error analysis
will crown a lucky overfit. Shortlisting is **breadth with cheap
models** so you learn the shape of the errors.

- Train several reasonable defaults (linear, tree, forest / boost,
  maybe a small net if the data is pixels or sequences).
- Compare with **N-fold CV** on train. Same folds, same metric as
  framing.
- Look at **which features** each model clung to and **which errors**
  it made (the [ch. 3](../3-classification/) confusion-matrix habit
  applies to regression residuals too).
- Prefer a shortlist of 3–5 that fail **differently** — that is
  what ensembles later buy.
- One quick round of extra features if error analysis screams for
  them; do not disappear into a Kaggle rabbit hole yet.

**Failure if skipped.** You grid-search a neural net for a week on
a problem a ridge model already solved. Or you ensemble three clones
of the same tree and call it diversity. Or you pick a winner on a
single train/val split that happened to be kind.

## 6. Fine-tune

**Why it exists.** Defaults are for shortlisting. Shipping wants
the search you can defend, on as much training data as you can
honestly use, with the test set still unused.

- Automate: grid, random, or a Bayesian-class search. Know which
  knobs actually move the metric ([ch. 2](../2-end-to-end-project/),
  [ch. 10](../10-anns-keras/) for nets, [ch. 19](../19-scale-and-deploy/)
  if the search is a fleet of GPU jobs).
- Try ensembles when the shortlist disagrees usefully
  ([ch. 7](../7-ensembles/)).
- Keep a log: params, fold scores, code version, data version.
- When you are ready to **stop**, measure **once** on the test set.
  That number is for the report, not for another tuning loop.

**Failure if skipped.** You peek at test, tweak, peek again — the
number is now a training number in costume. Or you tune 40
hyperparameters with 10 trials and ship noise. Or you fine-tune
so hard on CV that the test drop is a surprise you had no
monitoring plan for (see launch).

## 7. Present

**Why it exists.** A model that cannot change a decision was a
hobby. Presentation is how you put the score back into the framing
from section 1: what to do, when not to trust it, what it costs.

- Write the story: question, data, metric, baseline, lift, limits.
- Visuals that carry the argument (error by segment, not a wall of
  ROC porn unless the operating point is the product).
- Call out assumptions that survived and ones that did not.
- Name the failure modes (slice X is bad, labels drift, human
  override still required).
- Recommend a ship / no-ship / ship-with-guardrail call.

**Failure if skipped.** Engineering dumps an F1 in Slack. The
stakeholder hears "it works." Six weeks later nobody can say why
the false negatives cluster in one region — you saw it in explore
and never put it on a slide.

## 8. Launch, monitor, maintain

**Why it exists.** Offline CV is a snapshot. Production is a moving
input distribution, a moving label process, and a serving path that
can skew. Launch is **plumbing + alarms + a retrain loop**, not a
ceremony.

- Plug into production inputs through the **same** prepare path you
  fitted (SavedModel signatures, sklearn `Pipeline`, not a rewrite).
- Version the artifact; know how to roll back
  ([ch. 19](../19-scale-and-deploy/) if this is a TF graph).
- Monitor **outputs** (metric proxies, prediction distributions,
  latency) *and* **inputs** (missingness, schema, drift of features
  you depend on). Alerts need owners.
- Retrain on a schedule or on a drift trigger, with the same
  checklist from "get data" downward — including a fresh honest
  split when time is the axis.
- Record what vN was trained on. You will be asked.

**Failure if skipped.** Silent skew: training used log-price and
serving uses price. Silent drift: a sensor dies at zero and the
model is confidently wrong. Silent process death: the person who
knew the notebook left. Monitoring only the HTTP 200 rate is how
classifiers rot in place.

LLM products still need eval, traces, and A/B — that checklist is
[A7](../../agents/7-evaluation-and-feedback/) / [P7](../../platform/7-observability/).
Do not paste those items here because the word "monitor" appeared.

## How to use this with chapter 2

Walk [chapter 2](../2-end-to-end-project/) once with housing so the
spine has muscle memory. Then print this folder's eight headings
for the next dataset and fill *why* / *failure* in your own words
for that problem. If a heading does not bite, you are probably not
looking at an ML-from-data project, or you skipped framing.

There is no next ML chapter after this appendix. Part II's last
stop was [scale and deploy](../19-scale-and-deploy/). For the rest
of the workshop, use the [topic index](../../INDEX.md): train here,
build the agent in `agents/`, serve the organization in `platform/`.
Do not merge the three because a checklist sounded generic enough
to cover chatbots.
