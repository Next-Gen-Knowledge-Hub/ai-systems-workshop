### **Chapter 16 Summary: Natural Language Processing with RNNs and Attention**

Chapter 16 of *Hands-On Machine Learning with Scikit-Learn, Keras, and TensorFlow* (2nd edition, Aurélien Géron) ties recurrent models to language: generating text character by character, classifying sentiment, translating with encoder–decoder RNNs, then improving and ultimately replacing that stack with attention and Transformers, closing with large pretrained language models from the late 2010s.

---

#### **1. Character RNNs and text generation**
* **Char-RNN**: A model trained to predict the **next character** can compose novel text; the chapter walks from tokenizing Shakespeare to building `tf.data` **sliding windows** with non-overlapping splits for train/validation/test to avoid leakage in sequential data.
* **Model and sampling**: Stacked **GRU** layers with a time-distributed softmax predict the next symbol; at generation time **temperature-scaled** categorical sampling trades repetition for diversity instead of always picking the argmax.
* **Stateful RNNs**: Carrying **hidden state across batches** (with aligned, non-shuffled windows) helps learn patterns longer than the truncated BPTT window, at the cost of awkward batching and inference constraints unless weights are copied to a stateless twin.

---

#### **2. Sentiment analysis and variable-length text**
* **IMDb pipeline**: Word-level RNNs on movie reviews use **embeddings**, stacked GRUs, and binary classification; the chapter contrasts Keras’s pre-indexed IMDb set with **TensorFlow Datasets**, vocabulary tables, and **out-of-vocabulary buckets**.
* **Masking**: `mask_zero=True` on `Embedding` (or explicit masks) tells downstream layers to **ignore padding** so the model does not waste capacity on filler tokens.
* **Transfer**: **TensorFlow Hub** sentence encoders supply **pretrained word statistics**, so a small classifier on top can train quickly with little labeled data.

---

#### **3. Encoder–decoder neural machine translation**
* **Seq2seq**: An encoder compresses a source sentence; a decoder emits the translation **one target word at a time**, fed shifted targets while training and its own previous outputs at inference.
* **Practical constraints**: **Length bucketing**, masking past **EOS** in the loss, **sampled softmax** for huge vocabularies, **bidirectional encoders**, and **beam search** at decode time improve quality without retraining.
* **Tooling**: **TensorFlow Addons** supplies training samplers (including scheduled sampling), decoders, and beam-search wrappers for production-style NMT.

---

#### **4. Attention mechanisms**
* **Motivation**: Long encoder–decoder paths blur information; **attention** lets each decoder step **re-weight all encoder outputs**, shortening the effective path for aligning source and target tokens (e.g., “milk” → “lait”).
* **Variants**: **Bahdanau (concatenative)** and **Luong (dot-product)** attention differ in how alignment scores are computed; attention also supports **visual captioning** and **explainability** by showing what the model focused on.
* **Beyond translation**: The same idea applies wherever a model must **pick relevant context** from a long input.

---

#### **5. The Transformer architecture**
* **Attention-only stack**: The 2017 Transformer removes recurrence, using **encoder and decoder stacks** of multi-head self-attention, feed-forward blocks, residuals, and layer normalization.
* **Positional information**: **Sine/cosine positional encodings** (or learned positions) inject order because self-attention is **permutation-invariant** without them.
* **Multi-head attention**: Parallel **scaled dot-product** heads attend in different representation subspaces; masked decoder attention prevents peeking at **future** tokens during training.

---

#### **6. Large language models (2018–2019)**
* **Contextual embeddings and pretraining**: **ELMo** supplies context-dependent vectors; **ULMFiT** and **GPT** show **unsupervised pretrain + fine-tune** (and GPT-2 **zero-shot** use) on Transformer-like stacks.
* **BERT**: **Masked language modeling** and **next-sentence prediction** yield **bidirectional** representations that fine-tune strongly on classification, entailment, and question answering.
* **Trend**: Subword tokenization plus **self-supervised Transformers** became the default NLP recipe, though the chapter notes architecture fashion can shift.
