### **Chapter 9 Summary: Unsupervised Learning Techniques**

Chapter 9 of *Hands-On Machine Learning with Scikit-Learn, Keras, and TensorFlow* (2nd edition, Aurélien Géron) extends unsupervised learning beyond dimensionality reduction into clustering, density modeling, and anomaly detection. The chapter centers on K-Means and DBSCAN, Gaussian mixture models with EM and Bayesian variants, and related outlier algorithms.

---

#### **1. Clustering Foundations and K-Means**
* **Unsupervised grouping**: Clustering assigns instances to groups **without labels**, useful for exploration, segmentation, and preprocessing.
* **Applications**: Customer segments, **image color quantization**, search by similarity, semi-supervised **label propagation**, and outlier screening.
* **`KMeans`**: Iterates **assign to nearest centroid** and **recompute means** until convergence; **`n_clusters=k`** must be supplied.
* **Voronoi regions**: Decision boundaries are **perpendicular bisectors** between centroids; `transform` yields distances to all centers.
* **Initialization**: **`n_init`** runs multiple seeds; **K-Means++** spreads initial centroids; **inertia** scores the best run.
* **Variants**: **Elkan** accelerates standard K-Means; **`MiniBatchKMeans`** updates with subsets for **large-scale** data.
* **Choosing k**: **Elbow** on inertia or **silhouette score** balances cohesion versus separation; diagrams inspect cluster quality.

---

#### **2. K-Means Limits and Clustering Applications**
* **Assumptions**: K-Means favors **roughly spherical, similar-sized** blobs; elongated or uneven clusters break the method.
* **Scaling**: Features should be **standardized** so one axis does not dominate distance.
* **Image segmentation**: Pixels clustered in **RGB space** and replaced by **centroid colors** simplify images.
* **Preprocessing pipeline**: K-Means distances as features before **logistic regression** on digits can **cut error** versus raw pixels.
* **Semi-supervised learning**: Label **centroid-near exemplars**, propagate to clusters, optionally limit to **closest fraction** per cluster for cleaner pseudo-labels.

---

#### **3. DBSCAN and Other Clustering Algorithms**
* **Density-based clusters**: **DBSCAN** marks **core points** with enough neighbors within **ε**, expands clusters, and labels sparse points as **noise (-1)**.
* **Arbitrary shapes**: Works when clusters are **dense islands** separated by low density; **`eps`** and **`min_samples`** are critical.
* **Prediction workaround**: No `predict`; train **`KNeighborsClassifier`** on core samples to assign **new points** and optional distance-based anomalies.
* **Alternatives surveyed**: **Agglomerative**, **BIRCH**, **mean-shift**, **affinity propagation**, and **spectral clustering** differ in scalability, shape assumptions, and need for connectivity graphs.

---

#### **4. Gaussian Mixture Models**
* **Generative mixtures**: Data assumed drawn from **k Gaussians** with weights **φ**, means **μ**, and covariances **Σ**; cluster identity is latent.
* **`GaussianMixture`**: **EM** alternates **responsibility** (soft assignments) and **parameter updates**; supports `predict` and `predict_proba`.
* **`covariance_type`**: **`full`, `tied`, `diag`, `spherical`** trade flexibility against parameters and stability in high dimensions.
* **Sampling and density**: **`sample`** draws new points; **`score_samples`** gives **log probability density** for each observation.
* **Model selection**: **BIC** and **AIC** penalize complexity when choosing **k**; **`BayesianGaussianMixture`** can **shrink** unused components toward zero weight.

---

#### **5. Anomaly and Novelty Detection**
* **Low-density outliers**: Points with **small mixture density** or poor fit are flagged as anomalies; thresholds set **contamination** rates (e.g., 4% in manufacturing).
* **Novelty vs anomaly**: **Novelty detection** trains on **clean** data; anomaly detection tolerates **mixed** training sets.
* **Robust fitting**: Refit after removing extreme points or use **EllipticEnvelope** for Gaussian inliers amid contamination.
* **Other detectors**: **Isolation Forest** isolates points quickly in random trees; **LOF** compares local density to neighbors; **one-class SVM** separates training data from the origin in kernel space; **PCA reconstruction error** flags hard-to-compress instances.
