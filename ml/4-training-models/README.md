### **Chapter 4 Summary: Training Models**

Chapter 4 of *Hands-On Machine Learning with Scikit-Learn, Keras, and TensorFlow* (2nd edition, Aurélien Géron) opens the hood on how common models are trained. It contrasts closed-form linear regression with gradient-based optimization, shows how polynomial features and learning curves reveal bias–variance issues, surveys regularized linear models, and connects logistic and softmax regression to classification probabilities.

---

#### **1. Linear Regression**
* **Model form**: Predictions are a **weighted sum of features plus bias**; vector notation bundles parameters in θ for compact math and code.
* **Training objective**: Minimizing **MSE** (often reported as RMSE) finds θ that best fits labeled training points.
* **Normal equation**: For moderate feature counts, **a closed-form least-squares solution** (via pseudoinverse/SVD) solves for θ directly.
* **Complexity trade-offs**: Exact methods scale poorly with **very large feature dimensionality**, while prediction after training is cheap and linear in instance size.

---

#### **2. Gradient Descent**
* **Idea**: Iteratively move θ along the **negative gradient of the cost** with step size controlled by the learning rate η.
* **Batch GD**: Each update uses **the full training set’s gradient**, stable for convex linear least squares but slow on huge data.
* **Stochastic and mini-batch**: **Single-instance or small-batch** updates add noise but enable speed, online learning, and escape from some local roughness in non-convex tasks.
* **Practical needs**: **Feature scaling** reduces ill-conditioned valleys; learning schedules and tolerances govern convergence; `SGDRegressor` mirrors these ideas in Scikit-Learn.

---

#### **3. Polynomial Regression**
* **Linear in parameters, nonlinear in inputs**: Adding powers (and cross terms) lets a **linear solver fit curved relationships**.
* **Combinatorial cost**: High polynomial degree explodes **feature count**, risking slow training and severe overfitting.
* **Model selection angle**: Degree choice balances **flexibility vs. stability** on the same underlying algorithmic family.

---

#### **4. Learning Curves**
* **Diagnostic plots**: Training and validation error vs. **training set size** separate underfitting from overfitting without guessing from a single score.
* **Underfitting signal**: Both curves **plateau high and close together**—more data alone rarely fixes a too-simple hypothesis.
* **Overfitting signal**: Low training error with **higher validation error** and a gap suggests variance; more data or simpler models may help.
* **Bias–variance view**: Error decomposes conceptually into **wrong assumptions, sensitivity to sampling noise, and irreducible label noise**.

---

#### **5. Regularized Linear Models**
* **Ridge (ℓ2)**: Shrinks weights toward zero with **α-weighted squared penalties**, stabilizing coefficients when features correlate or noise is high.
* **Lasso (ℓ1)**: Encourages **sparse solutions** by driving some weights exactly to zero, performing implicit feature selection.
* **Elastic Net**: Mixes ℓ1 and ℓ2 penalties to **keep sparsity while handling correlated groups** better than pure lasso in some regimes.
* **Early stopping**: Halting iterative training when **validation error bottoms out** regularizes without an explicit penalty term.

---

#### **6. Logistic Regression and Softmax Regression**
* **Binary probabilities**: A linear score passed through the **logistic sigmoid** yields class probabilities; a **0.5 threshold** on that probability sets the class decision boundary.
* **Training loss**: **Log loss** penalizes confident wrong predictions; convexity enables reliable gradient-based fitting (no normal equation).
* **Multiclass softmax**: One weight vector per class produces scores normalized by **softmax into a probability simplex**; prediction picks the argmax class for mutually exclusive labels.
* **Regularization note**: Scikit-Learn’s `LogisticRegression` uses **inverse-strength C** for ℓ2 by default; `multi_class="multinomial"` engages softmax training with compatible solvers.
