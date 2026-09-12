# 3. Classification

Companion notes for **Chapter 3** of *Hands-On Machine Learning with
Scikit-Learn, Keras, and TensorFlow* (2nd edition, Aurélien Géron;
O'Reilly, 2019).

This chapter is how you **measure a decision**, using a
many-class image set (MNIST-shaped: one digit per row) as the running
picture, then the same math on any labeled table. Skip it and you
will ship on **accuracy**, celebrate a 97% that never caught the rare
class, and argue about "the model is bad" when you have not even
picked an **operating point**. Regression RMSE from
[ch. 2](../2-end-to-end-project/) does not save you here. A class is
a decision, and decisions have different costs.

**See also (do not merge).** Agent evaluation is traces, rubrics,
grounding, and LLM-as-judge —
[agents ch. 7](../../agents/7-evaluation-and-feedback/). Platform
scores, datasets, and A/B are
[platform ch. 7](../../platform/7-observability/). A confusion matrix
does not tell you whether an agent refunded the wrong order. A
Phoenix trace does not tell you whether a digit classifier is
calibrated. If you have both jobs, read both folders; do not paste
judges into `sklearn.metrics`. Rows:
[`TRADEOFFS.md`](../../TRADEOFFS.md) ("Hold-out metrics vs judges vs
platform scores").

## The mental model

```
  scores / probabilities from a classifier
                 |
                 v
         [ THRESHOLD t ]
           t high: few positives, precise
           t low:  many positives, noisy
                 |
                 v
            predicted label
                 |
                 v
         [ CONFUSION MATRIX ]
           TP FP
           FN TN
                 |
      +----------+-----------+
      |                      |
   PRECISION              RECALL
   TP / (TP+FP)           TP / (TP+FN)
      |                      |
      +------ F1, PR, ROC ---+
              pick t for the *cost*
```

The one sentence to remember a year from now: **accuracy is a
summary that can hide the only class you care about; the confusion
matrix is the object; the threshold is a product choice.**

Two consequences. First, "we got 99%" on a 1% fraud rate can mean
"we predicted nobody is fraud." Second, precision and recall move in
opposite directions when you slide `t`. You do not "fix both" with a
prettier color bar. You pick a point and own the misses.

## A binary classifier

MNIST-shaped work starts as **one digit vs the rest** (a "5-detector"
is the usual cartoon) because binary metrics are easier to *see*.
The same pattern is everywhere: defect vs not, churn vs not,
"this login is stolen" vs not.

You need:

- a **scoring** model (decision function or probability),
- a **threshold**,
- a **split** that respects [ch. 1](../1-ml-landscape/) (no peeking)
  and [ch. 2](../2-end-to-end-project/) (pipeline-safe CV).

```python
from sklearn.linear_model import SGDClassifier
from sklearn.model_selection import cross_val_predict

clf = SGDClassifier(random_state=0)
# CV *predictions* for honest matrices; not the same as .predict on train
y_pred = cross_val_predict(clf, X_train, y_binary, cv=3)
```

`cross_val_predict` is the habit: every row's predicted label (or
score) comes from a model that did not train on that row. Training-set
accuracy of a model that saw those rows is a vanity metric.

**Problem** — The demo classifies ten digits, so the team jumps to
multiclass dashboards.

**Solution** — Get binary hygiene first: a matrix, a PR curve, a
chosen `t`. Multiclass is several of these stories glued together.

**Failure mode to recognise** — Reporting train accuracy of a
nonlinear model on digits (or on tickets) as if it were a test.

## The accuracy trap

**Accuracy** = fraction of labels you got right. It is a fine summary
when classes are balanced and costs are symmetric. It is a trap when
one class is rare or when FN and FP have different prices.

A dummy that always says "not-5" (or "not fraud") is extremely
accurate and completely useless.

**Failure mode to recognise** — A launch review that only prints
accuracy, plus a class balance of 1:50. You have not measured the
product. You have measured the prior.

Always print the **base rate** next to accuracy. If they are close,
the model may not be doing anything.

## Confusion matrix, precision, recall, F1

The confusion matrix is four counts for a chosen threshold:

|  | pred + | pred − |
|---|---|---|
| **true +** | TP | FN |
| **true −** | FP | TN |

- **Precision** — of the rows you flagged, how many deserved it.
  High precision: you rarely cry wolf. Search "show me the 5s" wants
  this if a false 5 wastes an operator.
- **Recall** (sensitivity, true-positive rate) — of the rows that
  deserved a flag, how many you caught. High recall: few leaks. Cancer
  screening, fraud holds, safety defects want this more than a pretty
  precision.
- **F1** — harmonic mean of precision and recall. It punishes the
  worse of the two. Useful as a *single* number when you need to sort
  models and have no cost ratio yet. It is not a business license to
  ignore which error hurts.

```python
from sklearn.metrics import (
    confusion_matrix, precision_score, recall_score, f1_score,
)

cm = confusion_matrix(y_binary, y_pred)
p = precision_score(y_binary, y_pred)
r = recall_score(y_binary, y_pred)
f = f1_score(y_binary, y_pred)
```

**Problem** — Stakeholders hear "F1" and stop asking who pays for FN.

**Solution** — Put **costs** on FP and FN (even roughly: operator
minutes vs missed fraud dollars). F1 is a convenience, not a utility.

**Failure mode to recognise** — Optimizing F1 while production
thresholds on a different score (marketing wants volume; risk wants
precision). The number you tuned is not the number the system uses.

## The precision/recall trade-off

Most classifiers emit a **score**. You pick `t`. Raising `t` typically
**raises precision and lowers recall**. Lowering `t` does the
opposite. That is not a bug. It is the geometry of ranking positives
ahead of negatives, then cutting the list.

```
  score high  ----------------  score low
  [....true positives....|..FP..]
                         t
  move t right --> more flags, more FP, recall up, precision down
```

The **PR curve** is precision as a function of recall (or vs
threshold). Use it when positives are rare: it stays honest about the
class you care about. A model can dominate another on PR in the
recall band you actually operate in, even if average precision looks
similar.

**Average precision** (area under PR, with care) is a threshold-free
summary of that curve. Still pick a `t` for production.

**Failure mode to recognise** — Quoting precision at the default
`t=0.5` for a model whose scores are not probabilities, or whose
probabilities are uncalibrated. `0.5` is a convention, not a law.
SGD-style decision functions are not "50% chance."

```python
from sklearn.metrics import precision_recall_curve

scores = cross_val_predict(clf, X_train, y_binary, cv=3,
                           method="decision_function")
prec, rec, thresh = precision_recall_curve(y_binary, scores)
# pick t from the curve for a recall floor, then freeze it
```

## ROC and AUC

The **ROC** curve plots true-positive rate (recall) against
**false-positive rate** (FP / (FP+TN)) as `t` slides. **AUC** is the
area under that curve: probability that a random positive scores
higher than a random negative (for a well-behaved scorer).

ROC/AUC is handy when you care about **ranking** and classes are not
pathologically rare. When positives are rare, FPR can look tiny
because TN is huge: a "great AUC" with a useless PR curve in the
region you operate. **Prefer PR for imbalanced detection.** Prefer
ROC when both classes are real populations you care about (digit vs
digit, two medical conditions of similar prevalence).

**Failure mode to recognise** — "AUC 0.99" on 0.2% positives, no PR
plot, threshold left at default. You ranked well on average and still
flooded the queue — or never filled it.

Do not compare AUC across tasks with different base rates as if it
were a universal IQ.

## Multiclass: OvR and OvO

Ten digits is **multiclass**: exactly one label per row, more than
two labels. Strategies:

- **One-versus-rest (OvR, OvA)** — fit one binary model per class
  ("is this a 7?"). At predict time, pick the class with the strongest
  score. Scales as *N* classifiers.
- **One-versus-one (OvO)** — fit one model per pair of classes. At
  predict time, vote. Scales as *N(N−1)/2* classifiers, but each is
  trained on a smaller slice. Historically attractive for learners
  that hate big sets (classic SVMs).

Some estimators are **inherently multiclass** (softmax in
[ch. 4](../4-training-models/), trees, many boosting models). sklearn
will wrap the others. You still own the **metric**: macro vs micro vs
weighted F1, and a **confusion matrix that is N×N**.

```
  OvR:  [is 0?] [is 1?] ... [is 9?]   --> argmax score
  OvO:  [0 vs 1] [0 vs 2] ...         --> majority vote
```

**Failure mode to recognise** — Reporting "accuracy 94%" on digits
and missing that 4 vs 9 is a systematic tangle. The number is an
average over easy classes.

### Error analysis

The N×N matrix, especially **row-normalized** (of the true 4s, where
did they go?), is the debugging tool. You are looking for:

- pairs that confuse (4/9, 3/5, "urgent" vs "billing"),
- a class that is never predicted (broken encoding, too-rare, bad
  thresholding in a wrapper),
- a class that eats everyone (a prior the model clung to).

Then you decide whether the fix is **data** (more messy 4s),
**features** (a stroke detector, a header field), or **the decision
rule** (don't force a single label; abstain).

**Failure mode to recognise** — Collecting more data at random
instead of more data *on the confused pair*. Average accuracy will
barely move; the pair will.

## Multilabel and multioutput

**Multilabel**: several binary questions on one row that can be true
together. A digit image might be "odd" *and* "large"; a ticket might
be `needs_refund` *and* `needs_legal`. Metrics must decide whether
you average per label, per row, or with a sample-average F1. Accuracy
as "all labels must match" is brutal and often the wrong product
(missing one tag is not the same as tagging the wrong customer).

**Multioutput** (here): each output can be more than binary — e.g.
predict a cleaned pixel, or several ordinal fields. The chapter's
picture is "denoise / fill in" as a supervised problem with a
structured y. The lesson for production: **y is allowed to be a
vector**. Your split, leakage, and per-output metrics still apply.
Do not collapse to a single accuracy unless the product is
all-or-nothing.

**Failure mode to recognise** — Training one softmax over a cartesian
product of tags ("refund+legal", "refund+not-legal", ...) until the
label space explodes, instead of a multilabel head. Or the reverse:
independent labels that are actually mutually exclusive (a digit is
not both 3 and 5).

```python
from sklearn.multioutput import MultiOutputClassifier
from sklearn.ensemble import RandomForestClassifier

# Independent heads; not a substitute for a well-specified y.
multi = MultiOutputClassifier(RandomForestClassifier(n_estimators=50))
# multi.fit(X_train, Y_train)   # Y_train has several columns
```

## What aged since 2019

- `sklearn.metrics` for matrices, PR, ROC, and `cross_val_predict`
  is still the right lab. `classification_report` remains a decent
  first printout, not a launch criterion.
- Calibration tools (`CalibratedClassifierCV`) and proper scoring
  rules got more attention after this edition. If you treat scores as
  probabilities in a policy, check calibration; the chapter's
  thresholds still work on raw decision functions.
- Histogram boosting classifiers are a strong tabular default now.
  They do not change PR vs ROC.
- Digit recognition as a *research* problem is saturated. As a
  *teaching* problem it is still the cleanest way to see a matrix.
  Your real matrix is tickets, transactions, or defects — same
  arithmetic. Judges and trace scores for generative apps live in
  the See-also folders, not in `sklearn.metrics`.

## Check yourself

Good answers need the takeaway, the failure mode, and a labeled
system you have actually touched.

1. Quote an accuracy you have seen in a review. What was the base
   rate, and what dummy policy would have matched it?
2. Draw (counts or a sketch) a confusion matrix for a detector you
   know (fraud, spam, defect, intent). Which cell is the expensive
   one, and does the current threshold honor that?
3. Precision vs recall for that detector: who pays for FP, who pays
   for FN? What operating point would you defend, and what F1-only
   story would hide it?
4. You raise the threshold to "improve quality." What happens to
   recall, and what queue or miss-rate in a real system moves?
5. When would you trust ROC-AUC, and when would you insist on a PR
   curve? Give an imbalanced example from your work.
6. OvR vs OvO: for a 30-class intent model, which pain do you buy
   (N models vs N² models / smaller pairwise sets), and what error
   pattern would you look for in the N×N matrix?
7. Error analysis: name a confused pair from a system you know (two
   intents, two defect types). Would you collect more random data or
   more data on that pair — and what happens if you choose wrong?
8. A ticket can carry several tags. Is that multiclass, multilabel,
   or a cartesian monster? What metric would lie if you used
   all-or-nothing accuracy?
9. Why is `cross_val_predict` the matrix you want, not
   `predict` on the training frame? Point at a vanity number you have
   actually seen.
10. An agent team wants to "reuse chapter-3 metrics" on a chatbot.
    What object is missing (a gold label per row vs a judge on a
    trace), and which folder owns that job instead?

Continue to [Training models](../4-training-models/).
