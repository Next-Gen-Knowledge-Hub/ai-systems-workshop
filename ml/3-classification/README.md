### **Chapter 3 Summary: Classification**

Chapter 3 of *Hands-On Machine Learning with Scikit-Learn, Keras, and TensorFlow* (2nd edition, Aurélien Géron) switches from regression to classification using the MNIST digit dataset. It stresses that **accuracy alone is misleading on skewed classes**, introduces precision–recall and ROC tools, and extends to multiclass, multilabel, and multioutput setups.

---

#### **1. MNIST**
* **Dataset role**: MNIST’s 70k **28×28 grayscale images** (784 features per row) is the customary benchmark for new classifiers.
* **Labels and splits**: Targets are cast to integers; the canonical split keeps **60k train / 10k test**, with the test set reserved until final evaluation.
* **Exploration caveat**: Visualizing digits is fine on the training portion, but **peeking at the test set** biases later choices.
* **Complexity**: Wide stylistic variation among writers shows why **linear models need careful preprocessing** to compete.

---

#### **2. Training a Binary Classifier**
* **5-detector task**: A boolean target (`y == 5`) turns multiclass data into **binary classification** for simpler metric study.
* **SGD classifier**: `SGDClassifier` scales to large sparse data and supports **online-style updates** with reproducibility via `random_state`.
* **Prediction flow**: After `fit`, `predict` outputs whether the model believes the instance is the positive class.
* **Evaluation need**: High-level accuracy on cross-validation still requires **deeper metrics** before trusting deployment.

---

#### **3. Performance Measures**
* **Accuracy pitfall**: A majority-class “never 5” classifier scores **~90% accuracy** while learning nothing about fives—skewed priors inflate accuracy.
* **Confusion matrix**: Counts of **true/false positives and negatives** show where errors concentrate.
* **Precision, recall, F1**: Precision is **trust in positive calls**; recall is **coverage of actual positives**; F1 harmonizes both when a single scalar helps comparison.
* **Threshold trade-off**: `decision_function` scores versus a cutoff move along a **precision–recall frontier**; `precision_recall_curve` and ROC analysis guide cut selection.
* **ROC and AUC**: Plotting **TPR vs. FPR** summarizes ranking quality; **PR curves are preferred when positives are rare**.

---

#### **4. Multiclass Classification**
* **Native vs. meta strategies**: Some algorithms handle **all ten digits directly**; strict binary models use **one-vs-rest (OvR)** or **one-vs-one (OvO)** decompositions.
* **SVM behavior**: `SVC` on MNIST may train **many binary machines** and pick the class with the strongest vote or score.
* **Scaling lift**: **`StandardScaler` on pixel features** often improves linear multiclass SGD accuracy noticeably.
* **Wrappers**: `OneVsRestClassifier` forces OvR when the base estimator is inherently binary.

---

#### **5. Error Analysis**
* **Confusion matrix plots**: Normalizing rows highlights **which true digits get mistaken for which labels** (e.g., 3 vs. 5, 8 confusion).
* **Remediation ideas**: Targeted **data collection, feature engineering, or preprocessing** (centering, deskewing) address systematic confusions.
* **Qualitative review**: Side-by-side misclassified images reveal **ambiguous glyphs** versus **model limitations** (linear decision surfaces).

---

#### **6. Multilabel Classification**
* **Multiple binary tags**: Each instance may activate **several independent labels** (e.g., large digit and odd digit simultaneously).
* **KNeighbors example**: Some estimators accept **two-dimensional binary targets** and predict multiple outputs per row.
* **Metrics**: Per-label F1 scores aggregate with **`average="macro"` or `"weighted"`** depending on whether rare labels should dominate.

---

#### **7. Multioutput Classification**
* **Per-pixel targets**: Denoising MNIST treats **each intensity as a label**, blending classification-shaped outputs with regression-like values.
* **Setup**: Noisy inputs map to **clean training images**; a neighbor classifier can predict entire flattened vectors.
* **Scope**: Multioutput extends multilabel by allowing **multiclass or continuous targets per output slot**.
