# 3. Classification

Companion notes for **Chapter 3** of *Hands-On Machine Learning with
Scikit-Learn, Keras, and TensorFlow* (2nd edition, Aurélien Géron;
O'Reilly, 2019).

This chapter is how you **measure a decision**, using a many-class
image set (MNIST: one handwritten digit per 28×28 row) as the running
picture, then the same math on any labeled table. Skip it and you will
ship on **accuracy**, celebrate a 97% that never caught the rare class,
and argue about "the model is bad" when you have not even picked an
**operating point**. A class is a decision, and decisions have
different costs.

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
opposite directions when you slide `t`. You pick a point and own the
misses.

## A binary classifier

MNIST work starts as **one digit vs the rest** (a "5-detector") because
binary metrics are easier to *see*. About one tenth of the digits are
5s, so the negative class dominates — the same shape as defect vs not,
churn vs not, "this login is stolen" vs not.

You need:

- a **scoring** model (decision function or probability),
- a **threshold**,
- a **split** that hides the test set and keeps transformers inside
  the CV loop.

```python
from sklearn.linear_model import SGDClassifier
from sklearn.model_selection import cross_val_predict

clf = SGDClassifier(random_state=0)
# CV *predictions* for honest matrices; not the same as .predict on train
y_pred = cross_val_predict(clf, X_train, y_binary, cv=3)
```

`SGDClassifier` fits a linear classifier with stochastic gradient
descent. `cross_val_predict` returns, for every training row, the
label (or score) from a fold that did not include that row. Training-set
accuracy of a model that saw those rows is a vanity metric.

The demo classifies ten digits, so teams jump to multiclass
dashboards. Get binary hygiene first: a matrix, a PR curve, a chosen
`t`. Multiclass is several of these stories glued together.

## The accuracy trap

**Accuracy** = fraction of labels you got right. It is a fine summary
when classes are balanced and costs are symmetric. It is a trap when
one class is rare or when FN and FP have different prices.

Walk the rare-positive intuition. Suppose 100 login attempts, and 2
are stolen. A dummy that always says "not fraud" gets 98 right and 2
wrong: **98% accuracy**, zero fraud caught. A useful detector that
flags 5 attempts, of which 2 are real fraud and 3 are false alarms,
has accuracy 95/100 = **95%** — *worse* on accuracy, and the only one
that did the job. Always print the **base rate** next to accuracy. If
they are close, the model may not be doing anything.

A launch review that only prints accuracy, plus a class balance of
1:50, has measured the prior.

## Confusion matrix, precision, recall, F1

The confusion matrix is four counts for a chosen threshold:

|  | pred + | pred − |
|---|---|---|
| **true +** | TP | FN |
| **true −** | FP | TN |

On the same 100-login toy: TP = 2, FP = 3, FN = 0, TN = 95.

- **Precision** asks: of the rows you flagged, how many deserved it?
  Here precision = 2 / (2+3) = **0.40**. High precision means you rarely
  cry wolf. Search "show me the 5s" wants this if a false 5 wastes an
  operator.
- **Recall** (sensitivity, true-positive rate) asks: of the rows that
  deserved a flag, how many you caught? Here recall = 2 / (2+0) =
  **1.0**. High recall means few leaks. Cancer screening, fraud holds,
  safety defects want this more than a pretty precision.
- **F1** is the harmonic mean of precision and recall:
  `2 × P × R / (P + R)`. It punishes the worse of the two. On the toy,
  F1 ≈ 0.57. Useful as a *single* number when you need to sort models
  and have no cost ratio yet. It is not a business license to ignore
  which error hurts.

```python
from sklearn.metrics import (
    confusion_matrix, precision_score, recall_score, f1_score,
)

cm = confusion_matrix(y_binary, y_pred)
p = precision_score(y_binary, y_pred)
r = recall_score(y_binary, y_pred)
f = f1_score(y_binary, y_pred)
```

`confusion_matrix` counts the four cells. The three score helpers
compute P, R, and F1 from those counts (or equivalent). Stakeholders
who hear "F1" and stop asking who pays for FN need **costs** on FP and
FN (even roughly: operator minutes vs missed fraud dollars). Optimizing
F1 while production thresholds on a different score (marketing wants
volume; risk wants precision) tunes a number the system never uses.

## The precision/recall trade-off

Most classifiers emit a **score**. You pick `t`. Raising `t` typically
**raises precision and lowers recall**. Lowering `t` does the
opposite. That is the geometry of ranking positives ahead of negatives,
then cutting the list.

```
  score high  ----------------  score low
  [....true positives....|..FP..]
                         t
  move t right --> more flags, more FP, recall up, precision down
```

On the 5-detector, a high threshold says "only shout when you are very
sure this is a 5": few false 5s (precision up), many real 5s missed
(recall down). A low threshold floods the queue with candidates.

The **PR curve** is precision as a function of recall (or vs
threshold). Use it when positives are rare: it stays honest about the
class you care about. A model can dominate another on PR in the
recall band you actually operate in, even if average precision looks
similar.

**Average precision** (area under PR, with care) is a threshold-free
summary of that curve. Still pick a `t` for production.

Quoting precision at the default `t=0.5` for a model whose scores are
not probabilities, or whose probabilities are uncalibrated, treats a
convention as a law. SGD-style decision functions are not "50% chance."

```python
from sklearn.metrics import precision_recall_curve

scores = cross_val_predict(clf, X_train, y_binary, cv=3,
                           method="decision_function")
prec, rec, thresh = precision_recall_curve(y_binary, scores)
# pick t from the curve for a recall floor, then freeze it
```

`method="decision_function"` asks each CV fold for the raw score
instead of a hard label. `precision_recall_curve` walks every
threshold and returns the precision/recall pairs so you can pick `t`
for a recall floor and freeze it.

## ROC and AUC

The **ROC** curve plots true-positive rate (recall) against
**false-positive rate** (FP / (FP+TN)) as `t` slides. **AUC** is the
area under that curve: probability that a random positive scores
higher than a random negative (for a well-behaved scorer).

ROC/AUC is handy when you care about **ranking** and classes are not
pathologically rare. When positives are rare, FPR can look tiny
because TN is huge: a "great AUC" with a useless PR curve in the
region you operate. Prefer PR for imbalanced detection. Prefer ROC
when both classes are real populations you care about (digit vs digit,
two medical conditions of similar prevalence).

"AUC 0.99" on 0.2% positives, no PR plot, threshold left at default,
means you ranked well on average and still flooded the queue — or never
filled it. Do not compare AUC across tasks with different base rates
as if it were a universal IQ.

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

Some estimators are **inherently multiclass** (softmax logistic
regression, trees, many boosting models). sklearn will wrap the
others. You still own the **metric**: macro vs micro vs weighted F1,
and a **confusion matrix that is N×N**.

```
  OvR:  [is 0?] [is 1?] ... [is 9?]   --> argmax score
  OvO:  [0 vs 1] [0 vs 2] ...         --> majority vote
```

Reporting "accuracy 94%" on digits and missing that 4 vs 9 is a
systematic tangle averages over easy classes.

### Error analysis

The N×N matrix, especially **row-normalized** (of the true 4s, where
did they go?), is the debugging tool. You are looking for:

- pairs that confuse (4/9, 3/5, "urgent" vs "billing"),
- a class that is never predicted (broken encoding, too-rare, bad
  thresholding in a wrapper),
- a class that eats everyone (a prior the model clung to).

Then you decide whether the fix is **data** (more messy 4s),
**features** (a stroke detector, a header field), or **the decision
rule** (don't force a single label; abstain). Collecting more data at
random instead of more data *on the confused pair* barely moves
average accuracy; the pair stays tangled.

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

Training one softmax over a cartesian product of tags
("refund+legal", "refund+not-legal", ...) until the label space
explodes fights multilabel reality. Independent labels that are
actually mutually exclusive (a digit is not both 3 and 5) need a
single multiclass head instead.

```python
from sklearn.multioutput import MultiOutputClassifier
from sklearn.ensemble import RandomForestClassifier

# Independent heads; not a substitute for a well-specified y.
multi = MultiOutputClassifier(RandomForestClassifier(n_estimators=50))
# multi.fit(X_train, Y_train)   # Y_train has several columns
```

`MultiOutputClassifier` fits one forest per output column. That is
independent heads sharing the same X; it does not invent a joint
label space for you.

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
  arithmetic.

## Check yourself

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
9. Why is `cross_val_predict` the matrix you want, rather than
   `predict` on the training frame? Point at a vanity number you have
   actually seen.
