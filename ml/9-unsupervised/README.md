# 9. Unsupervised Learning

Companion notes for **Chapter 9** of *Hands-On Machine Learning with
Scikit-Learn, Keras, and TensorFlow* (2nd edition, Aurélien Géron;
O'Reilly, 2019).

Most of Part I assumed a **target column**. This chapter is the
toolkit for when you have rows and **no labels** (or almost none):
group similar rows, flag strange ones, sometimes steal a little
structure to help a tiny labeled set. Skip it and you will run
k-means with `k=8` because the API default felt like a choice, treat
cluster IDs as ground truth, or call every outlier method "anomaly
detection" without saying whether you had a clean training class.

See also: [ch. 8](../8-dimensionality-reduction/) is also unsupervised,
but its job is **axes and sheets**, not groups. Do not PCA-vs-k-means
as if they were two brands of clustering. Agent "memory clusters" and
platform indexes are other books.

## The mental model

Unsupervised work is not "ML without y." It is **imposing a shape**
on unlabeled rows and then using that shape as a product: segments,
a preprocessor, a novelty score, a way to spend a labeling budget.

```
  unlabeled rows X
       |
       |  you pick a SHAPE
       v
  +-----------+  +------------+  +------------------+
  | blobs     |  | density    |  | ellipsoids       |
  | (k-means) |  | islands    |  | (Gaussian mix)   |
  |           |  | (DBSCAN)   |  | + anomaly tail   |
  +-----+-----+  +------+-----+  +--------+---------+
        |               |                 |
        v               v                 v
   labels / distances / log-likelihood / -1 vs inlier
        |
        +-->  segment | preprocess | semi-supervise | alert
```

The one sentence to remember a year from now: **clustering and density
models are assumptions about shape** — spherical blobs, density
islands, Gaussian ellipsoids — and the "unsupervised" output is only
as real as that assumption plus a **job** you can evaluate without
pretending the cluster id is a label from God.

Two consequences fall straight out of that diagram. First, inertia
going down when *k* goes up is not evidence you found truth; it is
almost a tautology. Second, anomaly detection is a **different job**
from clustering even when you use the same GMM: you need a rule for
the tail (score, threshold, what "normal" training looked like), not
just a pretty blob plot.

## Clustering jobs

Before the algorithm list, freeze **why** you cluster. The same k-means
run is a success or a toy depending on the job.

Typical jobs in this chapter's sense:

- **Segmentation** — group customers, cells, behaviors so a human (or
  a policy) can treat segments differently. Success is usefulness,
  stability, and a story you can act on — not a silhouette record.
- **Preprocessing** — replace raw `X` with cluster ids, distances to
  centroids, or posterior responsibilities, then feed a supervised
  model. Success is **that** model's metric.
- **Semi-supervised leverage** — you can label 50 rows, not 50,000.
  Cluster first, label a **representative** per cluster (or propagate
  from labeled seeds). Success is accuracy per labeling hour.
- Also in the orbit: **image segmentation** (pixels as rows),
  **anomaly** (far from structure), **data analysis** (look at the
  groups). Do not mash them into one KPI.

**Problem** — A stakeholder asks for "the clusters" as if the table
contained a hidden categorical variable with a unique true *k*.

**Solution** — Answer with the **job**, the **shape assumption**, and
an evaluation that matches the job (downstream metric, silhouette as a
secondary check, human review of segment profiles). There may be many
acceptable partitions.

**Failure mode** — Optimizing silhouette until you have spherical
clusters that cut across the business action ("these users churn for
five different reasons but k-means put them together because they
spent the same").

## K-Means

**K-means** assumes *k* **spherical** (more honestly: equal-ish
isotropic) blobs and a Euclidean world. Algorithm, in the loop you
should be able to run on a napkin:

```
  1. place k centroids  (random, or k-means++)
  2. assign each row to the nearest centroid
  3. move each centroid to the mean of its rows
  4. repeat 2–3 until assignments freeze (or a budget)
```

sklearn: `KMeans(n_clusters=k)`. `k-means++` initialization (smart
spreading of starting centroids) is the default you want; plain random
init is how you get unlucky empty-ish clusters and a terrible local
minimum. `n_init` reruns the whole dance and keeps the lowest inertia.

### Inertia

**Inertia** is the sum of squared distances from each row to its
assigned centroid. Training **minimizes** inertia for a **fixed k**.
It always drops (or stays) when you add a cluster: a new centroid can
only help. Therefore:

- inertia is a **training loss**, not a truth score
- comparing inertia across **different k** without a penalty is how
  you "discover" that k = n (each row its own centroid, inertia 0)

**Failure mode** — A slide titled "optimal k" where the only curve is
inertia vs *k* with no elbow annotation and no second metric.

### Choosing k

Tools that are less circular than raw inertia:

- **Elbow** — plot inertia vs *k*, look for a bend where extra
  clusters buy little. Real plots are mushy. Use the elbow as a
  shortlist, not a verdict.
- **Silhouette** — for each row, (b − a) / max(a, b) with *a* = mean
  distance to its own cluster, *b* = mean distance to the nearest
  other cluster. Near +1: well placed. Near 0: on a boundary. Negative:
  probably in the wrong blob. Plot the **silhouette diagram** (sorted
  bars per cluster), not only the average. A high average can hide one
  garbage cluster.
- **Downstream job** — if clusters are a preprocessor or a labeling
  scheme, sweep *k* against **that** metric.

**Problem** — "The algorithm will tell us k."

**Solution** — You will tell it *k* (or a range). Domain + silhouette +
a human look at segment profiles. Mini-batch k-means (`MiniBatchKMeans`)
is the same assumption on a diet: faster, slightly noisier centroids,
appropriate when *n* is huge.

### Limits

K-means will **always** return k groups, including when:

- blobs have very **different sizes or densities**
- clusters are **ellipses, rings, or moons** (non-spherical)
- there are **outliers** (they drag centroids)
- *k* is wrong (it still partitions)

```
  truth: two moons          k-means: two cuts through both
     (  )                      \  /
    (    )                      \/
```

**Failure mode** — Scaling features incorrectly (or not at all) and
then declaring k-means "didn't find the segments." Euclidean k-means
on unscaled income vs age is a contest of units. Scaling is part of
the model. So is "this shape is wrong; use DBSCAN or a GMM."

### Image segmentation, preprocessing, semi-supervised

These are **uses**, not extra algorithms.

- **Image segmentation (color):** treat each pixel as a row of RGB
  (and maybe position). K-means with small *k* recolors the image to
  *k* palette centroids. That is compression / posterization, not
  semantic segmentation (a later CNN job). Good teaching demo of
  "rows can be pixels."
- **Preprocessing:** `fit` k-means on training `X`, then either use
  **distance to each centroid** as new features (`transform`) or a
  hard cluster id. A linear model on distance-to-blob features can
  beat the same model on raw `X` when the class boundary is blob-
  shaped. Pipeline it; do not fit k-means on the test set.
- **Semi-supervised:** cluster unlabeled (or mixed) data, then **label
  the centroids' nearest images/rows** instead of random rows. Those
  representatives buy more diverse labels per budget. Label propagation
  / spreading can then push those few labels through a similarity
  graph. Success is measured with the supervised metric on a hold-out,
  not with inertia.

**Problem** — Labeling 100 random images vs 100 centroid
representatives, then concluding "semi-supervised doesn't work"
because random was the protocol.

**Solution** — Spend the labeling budget on **structure** k-means (or
another clusterer) already found. That is the whole trick.

**Failure mode** — Using cluster ids as if they were human labels in a
compliance report. They are an artifact of *k* and a spherical
assumption. Downstream, maybe. As ground truth, no.

## DBSCAN

**DBSCAN** bets on **density**, not on a count of spheres.

- A **core** point has at least `min_samples` neighbors within `eps`
- Neighbors of cores (including non-cores) join the **same island**
- Points that are not near any core are **noise** (label −1)

```
  dense island   dense island      lonely points
    ****            ****               .
   ******          ******              .
    ****            ****
  <--eps-->
```

You do **not** set *k*. The number of clusters is an output. Shapes
can be moons and rings. Outliers are first-class.

Knobs: `eps` (too small → everyone noise or dust; too large → one
continent) and `min_samples`. Look at k-distance plots to shortlist
`eps`. sklearn: `DBSCAN`. There is no `predict` for new points in the
simple API the way k-means has; assigning a new row is a separate
policy (nearest core, or refit). That matters in production.

**Failure mode** — DBSCAN on data with **wildly varying densities**
(one tight blob, one sparse blob of the same "true" group). A single
`eps` cannot love both. Hierarchical density methods (aged section)
exist because of this.

### Other clustering algorithms

Names and **when**, not proofs:

| Algorithm | When you reach for it |
|---|---|
| **Agglomerative / hierarchical** | You want a **dendrogram**, a merge story, or clusters without picking *k* up front (you cut the tree later). |
| **BIRCH** | Large *n*, you can live with a CF-tree summary; a 2019-era sklearn option for scale. |
| **Mean-Shift** | Find blob modes; bandwidth replaces *k*; can be slow; cluster count is an output. |
| **Affinity Propagation** | Message-passing "exemplars"; you do not set *k*; can be slow and preference-sensitive. |
| **Spectral clustering** | Graph / embedding view of similarity; moons that k-means fails; you still often pass *k*. |

**Failure mode** — Collecting every clusterer in a bake-off scored
only by silhouette on spherical data. You will "discover" k-means.
Score the **job**, and include a shape that matches the method's bet.

## Gaussian mixtures

A **Gaussian mixture** (GMM) says: each row was drawn from one of *k*
Gaussians (different means and covariances), with some mixing weights.
**EM** (expectation–maximization) alternates:

```
  E:  responsibilities  —  soft P(component | row)
  M:  update means, covariances, weights from those soft assignments
  repeat
```

That **soft clustering** is the upgrade over k-means: a row can be 0.7
component A and 0.3 B. Covariance type (`full`, `tied`, `diag`,
`spherical`) is the shape assumption. `full` ellipsoids can swallow
moons they should not, and can overfit with too few rows per component.

sklearn: `GaussianMixture`. `predict` = hard argmax component.
`predict_proba` = responsibilities. `score_samples` = log-likelihood
per row — the hook for **anomaly detection**: rows in the low-density
tail of the mixture are candidates for "not like the training mass."

### Anomaly detection with a mixture

Train the GMM on **what you believe is normal** (or on all data if
anomalies are rare enough not to steal components). Score new rows.
Threshold the log-likelihood (or the pdf). Below threshold → alert.

This is **not** the same as clustering. You can cluster without ever
alerting, and you can alert with k-means distance-to-centroid if you
must, but GMM gives a density, which is the right object for a tail.

**Failure mode** — Fitting a GMM on a stream already full of the
outliers you wanted to catch, with enough components that a Gaussian
is dedicated to the bad island. The model *explains* the anomaly and
then scores it as normal. Train on clean, or use a method that does
not model the outliers as a component (Isolation Forest).

### Choosing components

*k* (here `n_components`) is again yours. **BIC** and **AIC** penalize
extra components; sklearn will compute them. Lower is better *as a
likelihood-plus-complexity cartoon*. Still confirm with the job:
anomaly PR curve, downstream classifier, visual ellipses on a 2-D
projection.

**Bayesian GMM** (`BayesianGaussianMixture`): you set a **generous
upper bound** on components and a prior that can drive unused weight
to ~0. The model can "turn off" extras. That does not free you from
thinking; it reduces the cost of overstating *k*. Watch whether
covariance type + small *n* invented phantom components.

**Failure mode** — BIC-selected *k* on unscaled mixed units, then a
heatmap in a board deck titled "we found 7 customer types." Scale,
then interpret, then validate with a human who owns the segment.

## Other anomaly and novelty detectors

Jobs, not leaderboards.

- **Isolation Forest** — random splits isolate points; **few splits**
  to isolate ⇒ likely anomaly (outliers are easier to fence off).
  Strong default on **tabular** mixed features. Unsupervised.
- **Local Outlier Factor (LOF)** — compares a row's local density to
  its neighbors'. Good when outliers are **local** (a sparse spot next
  to a dense blob). `novelty=True` vs outlier mode matters in sklearn:
  one is for scoring new points after fit, one is for in-sample flags.
- **One-class SVM** — learn a boundary around **inliers** in a kernel
  space. You need a notion of "training is (mostly) normal" and you
  will tune `nu` / kernel. Same kernel costs as [ch. 5](../5-svms/):
  medium *n*.

**Novelty vs outlier:** novelty usually means "fit on clean inliers,
score future rows." Outlier usually means "flag odd rows **inside**
the training set." APIs blur this. State which protocol you run.

**Problem** — One dashboard called "anomalies" mixing Isolation Forest
flags, GMM tails, and a one-class SVM, with no shared threshold
policy.

**Solution** — Pick the **shape of odd**: global isolation, local
density, ellipsoid tail, or kernel boundary. Then a threshold on a
**labeled slice** of incidents if you have any, or a budget ("we can
review 20 alerts a day").

**Failure mode** — One-class SVM on a million rows, or LOF with a
`n_neighbors` that spans two true regimes. These methods are not
k-means; they do not scale or tune the same way.

## What aged since 2019

- **k-means defaults.** `k-means++` stayed. `n_init` and algorithm
  (`lloyd` / `elkan`) shifted across sklearn 1.x; read the docstring
  on your version. Mini-batch k-means is still the scale valve.
- **HDBSCAN.** Density clustering with **variable density** (the
  DBSCAN `eps` pain) became a common default via `hdbscan` and, later,
  sklearn-adjacent tooling. Know DBSCAN first; reach for HDBSCAN when
  islands disagree on density.
- **Isolation Forest** became the boring, good **tabular anomaly**
  baseline. GMM remains the right *density* lecture and a solid tool
  when ellipsoids are honest.
- **Self-supervised representations.** Contrastive and masked models
  now often *create* the space you cluster in (embeddings, then
  k-means). That does not retire this chapter; it moves clustering
  one layer down.
- **Label efficiency.** Active learning and foundation-model
  embeddings changed how people spend a 50-label budget. The
  centroid-representative trick is still the right *idea*.

Keep jobs before algorithms, k-means as a spherical loop with inertia
and silhouette, DBSCAN as density islands plus noise, GMM as soft
ellipsoids plus a tail score, Bayesian GMM as "spare components can
die," and the three named detectors as different odd-shapes.

## Check yourself

1. Name three **jobs** for clustering. For each, what would you
   measure that is *not* inertia?
2. Why is a falling inertia-vs-*k* curve almost guaranteed, and what
   would a silhouette **diagram** show you that the average
   silhouette number can hide?
3. Walk one k-means iteration on a 1-D toy with 6 points and k=2.
   Then move one point far away. What happens to the centroid, and
   which assumption just broke?
4. You want to spend 40 human labels on 40,000 unlabeled images.
   Contrast **random 40** vs **40 nearest-to-centroid** after k-means.
   What extra leakage must you still avoid when you report accuracy?
5. K-means vs DBSCAN on two moons: who can return the moons, who must
   cut through them, and who can say "noise"? What knobs replace *k*
   in DBSCAN?
6. A production service must **score a new row** a month later.
   Compare k-means, DBSCAN, and a GMM on whether `predict` /
   `score_samples` is a natural socket.
7. GMM EM: what is a responsibility? How does `covariance_type='full'`
   fail with too many components on too few rows?
8. You fit a GMM on data that already contains the fraud cluster and
   set *k* high. Why might fraud look **normal** at scoring time?
   What protocol would you change?
9. Isolation Forest vs LOF vs one-class SVM: assign each to a
   one-line **shape of odd**. Which one is the usual tabular default
   after 2019, and which one inherits SVM scaling limits?
10. BIC picks 9 GMM components. Product wants "9 personas." What two
    extra checks would you run before those personas become a
    marketing taxonomy?

Continue to [ANNs with Keras](../10-anns-keras/).
