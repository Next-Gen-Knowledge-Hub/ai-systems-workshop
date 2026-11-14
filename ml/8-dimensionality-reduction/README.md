### **Chapter 8 Summary: Dimensionality Reduction**

Chapter 8 of *Hands-On Machine Learning with Scikit-Learn, Keras, and TensorFlow* (2nd edition, Aurélien Géron) addresses the curse of dimensionality and techniques to compress or visualize high-dimensional data. The chapter contrasts projection and manifold learning, then develops PCA and extensions, kernel PCA, LLE, and a survey of related methods in Scikit-Learn.

---

#### **1. The Curse of Dimensionality**
* **Sparse high-D space**: Points cluster near **hypercube borders**; random pairs are **far apart**, so neighbors-based learning extrapolates unreliably.
* **Overfitting risk**: More features raise the chance of fitting **noise** unless the training set grows exponentially with dimension.
* **When to reduce**: Try strong baselines on **full data** first; compression trades **information** for speed and simpler pipelines.
* **Visualization motive**: Mapping to **two or three dimensions** reveals clusters and communicates structure to non-specialists.

---

#### **2. Projection and Manifold Learning**
* **Projection**: Many datasets lie near a **low-dimensional subspace**; orthogonal projection onto that subspace drops dimensions with modest loss.
* **Swiss roll caveat**: A naive projection **squashes** curved manifolds; unfolding requires **manifold learning**, not a single linear drop.
* **Manifold hypothesis**: Real data often concentrates on a **lower-dimensional manifold** embedded in feature space.
* **Task dependence**: A simpler boundary in 3D may become **complex** after unrolling, so reduction is not always beneficial for accuracy.

---

#### **3. Principal Component Analysis**
* **Variance objective**: PCA finds orthogonal directions that **preserve maximum variance**; the first axes minimize reconstruction error.
* **SVD link**: Centered data decomposed via **SVD** yields principal component directions in **`components_`**.
* **`PCA` in Scikit-Learn**: **`fit_transform`** centers data and projects to **`n_components`** dimensions.
* **Explained variance**: **`explained_variance_ratio_`** shows how much variability each PC retains; cumulative sums guide **`n_components=0.95`** selection.
* **Compression**: MNIST at 95% variance shrinks features dramatically with small **reconstruction error** via **`inverse_transform`**.
* **Randomized and incremental PCA**: **`svd_solver="randomized"`** speeds large sparse problems; **`IncrementalPCA`** streams **mini-batches** or memory-mapped arrays.

---

#### **4. Kernel PCA**
* **Nonlinear embeddings**: The **kernel trick** maps data implicitly to a high-D space where **linear PCA** captures nonlinear structure.
* **`KernelPCA`**: Choices such as **RBF** can separate clusters or unroll rolls where linear PCA fails.
* **Hyperparameter search**: Supervised **pipelines with `GridSearchCV`** can tune kernel and gamma for downstream accuracy.
* **Reconstruction**: Feature-space pre-images are approximate; **`fit_inverse_transform=True`** learns a mapping to score **reconstruction error** in input space.

---

#### **5. Locally Linear Embedding**
* **Manifold unfolding**: LLE preserves **local linear combinations** of neighbors when embedding in fewer dimensions.
* **`LocallyLinearEmbedding`**: **`n_neighbors`** sets locality; good for **unrolling** swiss rolls but can distort **global distances**.
* **Two-step procedure**: First solve for **neighbor weights**, then find low-D coordinates that reproduce those weights.
* **Scalability**: Cost includes **O(m²)** terms, limiting use on very large sets without approximations.

---

#### **6. Other Dimensionality Reduction Techniques**
* **Random projections**: **Johnson–Lindenstrauss** guarantees preserve distances with surprisingly few random dimensions.
* **MDS and Isomap**: Preserve **pairwise distances** or **geodesics** on neighborhood graphs.
* **t-SNE**: Favors **local neighborhood** structure for visualization, often separating digit classes in 2D plots.
* **LDA**: Supervised method projecting onto axes that **maximize class separation**, useful before classifiers such as SVMs.
