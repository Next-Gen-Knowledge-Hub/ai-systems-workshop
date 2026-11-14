### **Chapter 5 Summary: Support Vector Machines**

Chapter 5 of *Hands-On Machine Learning with Scikit-Learn, Keras, and TensorFlow* (2nd edition, Aurélien Géron) introduces support vector machines for classification and regression. The chapter emphasizes **large-margin decisions**, kernel-induced nonlinearity, hyperparameters *C* and *γ*, and—at a conceptual level—how dual formulations and kernel evaluations relate to predictions without walking through full derivations.

---

#### **1. Linear SVM Classification**
* **Large margin**: A linear SVM seeks a **widest feasible street** between classes, relying on points at the margin edge—the **support vectors**.
* **Soft margin**: Hyperparameter **C** trades **margin width against slack violations**, softening hard separability and reducing outlier sensitivity.
* **Scaling**: Feature scale mismatch skews margins; **`StandardScaler` in a `Pipeline`** is standard before `LinearSVC` or related linear trainers.
* **API variants**: `LinearSVC`, `SVC(kernel="linear")`, and hinge-loss `SGDClassifier` can all implement **linear SVM-style classifiers** with different speed profiles.

---

#### **2. Nonlinear SVM Classification**
* **Feature expansion**: Polynomial features or **similarity landmarks** can make nonlinear boundaries in the original space linearly separable in a lifted space.
* **Kernel trick**: Kernelized `SVC` computes decisions as if features were mapped, using **kernel functions** (polynomial, RBF, etc.) without building huge explicit matrices.
* **RBF intuition**: Gaussian kernels measure **closeness to training points**; larger **γ** yields tighter, more complex boundaries; smaller γ smooths the decision surface.
* **Model choice**: Try **linear first for scale**, then **RBF** on smaller problems; grid search with cross-validation narrows *C*, *γ*, and degree settings.

---

#### **3. SVM Regression**
* **ε-insensitive tube**: `LinearSVR` and `SVR` fit a band around predictions where **errors inside width ε do not change the solution**.
* **Regularization via C**: As in classification, **C controls tolerance of points outside the tube**, balancing fit tightness and generalization.
* **Kernelized regression**: `SVR` with polynomial or RBF kernels captures **nonlinear trends** analogous to kernel classifiers, at higher training cost on large *m*.

---

#### **4. Under the Hood**
* **Decision function**: Classifiers score **wᵀx + b**; sign determines the predicted class without native probability outputs in basic SVM trainers.
* **Primal vs. dual**: Optimization can minimize **weight norm subject to margin constraints** or solve an equivalent **dual problem in α coefficients** over training pairs.
* **Kernelized predictions**: At inference, scores depend on **kernels between the new point and support vectors**, so only stored support patterns matter—not the full weight vector in feature space.
* **Complexity awareness**: Liblinear-style linear solvers scale roughly **O(m×n)**; kernel SVC training often grows between **O(m²n) and O(m³n)**, favoring smaller but rich datasets.
