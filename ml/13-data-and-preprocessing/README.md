# 13. Loading and preprocessing data with TensorFlow

Companion notes for **Chapter 13** of *Hands-On Machine Learning with
Scikit-Learn, Keras, and TensorFlow* (2nd edition, Aurélien Géron;
O'Reilly, 2019).

A fast training step is worthless if examples arrive through a sleepy
Python `for` loop. This chapter is how data *gets to* that step:
`tf.data` pipelines, TFRecord, categorical encodings, and
preprocessing that travels with the model. Skip it and you will
`np.load` the whole dataset, wonder why RAM dies at 10 GB, and then
"fix" features by pasting a frozen sentence-transformer into a
column named `embedding`.

Embeddings in this chapter are lookup tables trained with the task
loss. An integer category id becomes a small dense vector inside
*this* network. The vector moves when your classifier or regressor
moves. That is a capacity and generalization choice for this net.
It is a different job from retrieval vectors sitting in an index
for query-time search.

## The mental model

The training step should never wait on a disk syscall it could
have overlapped. `tf.data` is a pipeline of transformations that
the runtime can prefetch, parallelize, and fuse.

```
  storage (files, TFRecord, TFDS)
       |
       v
  Dataset  ->  shuffle  ->  map(preprocess)  ->  cache?
       |                         |
       |                    (CPU, parallel)
       v
     batch  ->  prefetch(to GPU)  ->  Keras fit / custom loop

  category id --one-hot-->  huge sparse 0/1
              --embed--->  small dense vector
                            (trained with THIS loss)
```

The one sentence to remember a year from now: **input pipelines
are part of the model** — shuffle, batch, parse, and categorical
encoding change both throughput and what the optimizer sees.

Two consequences fall out of that diagram. First, `model.fit(x,
y)` with giant NumPy arrays is a demo API; the real unit is a
`tf.data.Dataset` that yields batches. Second, choosing one-hot
versus an embedding table is a capacity and generalization choice
for this net.

## `tf.data`

A `Dataset` is a lazily composed graph of reads and transforms.
You chain methods; you do not materialize until iteration (or
`fit`).

Typical chain for training:

1. **List / read** — `from_tensor_slices` for what already fits,
   `list_files` + `interleave` / `TFRecordDataset` for what does
   not.
2. **Shuffle** — a buffer. Too small and you do not shuffle; too
   large and you are back to RAM. Shuffle *before* batch unless
   you have a reason.
3. **Map** — parse, decode, augment, cast. This is CPU work.
   `num_parallel_calls=AUTOTUNE` is the usual ask.
4. **Batch** — the unit the GPU wants. Drop remainder if the
   graph is picky about static shapes.
5. **Prefetch** — overlap the next batch's pipeline with the
   current step. `AUTOTUNE` again.
6. **Repeat** — epochs as `repeat(n)` or as `fit(..., epochs=n)`
   with a dataset that ends; pick one so you do not double-count.

```
  files --interleave(read)--> examples --shuffle-->
       --map(parse, parallel)--> --batch--> --prefetch-->
```

When step time is 90% `GetNext` and 10% matmul, the GPU is
starving. Profile one epoch. Then enlarge the shuffle buffer,
parallelize map, cache after expensive decode if RAM allows, and
prefetch at the end. Keras `fit` consumes `tf.data` directly; you
do not wrap it in a Python generator unless you like the GIL.

A `map` that uses a Python function with NumPy inside a
`tf.function` training step, or shuffle *after* batch (you are
permuting batches, not examples), wrecks the pipeline. Another
trap: `cache()` of an augmented dataset so every epoch sees the
same "random" flips.

Validation data usually **does not shuffle** (or shuffles once
for sanity) and often drops augmentation. Same parse, different
map. Two datasets, one model.

`from_tensor_slices` is fine for small tabular tensors. It is
not a data loader for ImageNet. If the argument to
`from_tensor_slices` is a giant array, you already paid RAM.

## TFRecord, compression, protobufs

When the source is many small files, the bottleneck is
open/close. **TFRecord** is a sequence of length-delimited byte
blobs, typically serialized protocol buffers, optionally
GZIP/ZLIB compressed.

A million JPEGs as individual files turns the training cluster
into a metadata server. Shard into a handful of TFRecord files,
interleave the shards, parse in `map`. Compression trades CPU for
disk and network. On a GPU box with weak CPUs, uncompressed can
win; measure.

### Example and SequenceExample

The usual protobuf payload is `tf.train.Example`: a map of
feature name → bytes / float list / int64 list. That is a
**single** instance (an image, a row).

`tf.train.SequenceExample` adds a *context* (per-example) plus
*feature lists* (per-timestep). That is a sequence: a sentence,
a user session, a time series of events.

You parse with `tf.io.parse_single_example` (or the sequence
variant) and a feature description dict. Wrong type in the
description is a runtime error at parse, which is preferable to
silent misaligned columns.

One giant TFRecord cannot shard and cannot parallel-read. A new
schema dumped into old files without a version field is an API
break dressed as a data dump. Treat the protobuf schema as an
API. Compressing already-compressed JPEGs inside GZIP TFRecords
for no reason wastes CPU twice.

You do not need to love protobuf to use this chapter. You need
to know **Example = one row**, **SequenceExample = one row plus
a time axis**, and that the file is a transport, not a training
algorithm.

## One-hot versus embeddings as categorical features

Categorical ids (words, user ids, product SKUs, pixels-as-ids)
have to become numbers a dense layer can multiply.

**One-hot** — a vector of length `vocab` with a single 1. Fine
when `vocab` is tiny (country codes, a dozen classes). A linear
layer on a one-hot is a per-category bias table.

**Embedding lookup** — a matrix of shape `[vocab, dim]`. Row
`id` is a `dim`-vector. That matrix is a `Variable`. It is
trained by the **same task loss** as the rest of the net
(cross-entropy, MSE, …). Similar ids can move toward each other
*if that helps the task*.

```
  id=42  -->  gather(E, 42)  -->  e_42  in R^{d}
                    ^
                    |  dL/dE  from the task
```

When `vocab` is 50,000, one-hot will not fit, and it does not
share statistical strength across ids. An embedding layer
(`keras.layers.Embedding` or a gather on a Variable) is the
usual answer. `dim` is a hyperparameter: too small and the table
cannot separate ids; too large and you overfit rare ids.

Calling the table "the embeddings" and then nearest-neighbor
searching it as if it were a retrieval index confuses two jobs.
Rows are optimized for *this head*, for *this* loss. A second
trap: one-hot of a huge vocab "for interpretability," then OOM.

Out-of-vocabulary ids need a slot (hashing trick, UNK token, or
a dedicated row). Hash collisions are a bias; UNK is a sink.
Pick one and log how often it fires.

This chapter's embeddings are **features inside the network** —
lookup tables whose rows move with the task. Word tables for
language tasks are the same idea: still features, still trained
with a loss on labels.

## Keras preprocessing layers

Putting parse-and-scale *in the model* means serve-time code
cannot forget to divide by 255.

Examples of the job (names moved across Keras versions; the
jobs did not):

- Rescaling, normalization (mean/variance adapted on train).
- Discretization, integer lookup, string lookup.
- Image resize / augmentation layers that honor `training=True`.
- Text vectorization (split, n-grams, output integers for an
  embedding).

Train Python did `X / 255`; the SavedModel did not — that is
the classic train/serve skew. A preprocessing layer as the first
layer(s), adapted on training data (`layer.adapt(dataset)`),
exported with the model, closes the gap. TF Transform (below) is
the production-scale version of the same idea: apply once,
consistently.

`adapt` on train+test, or augmentation layers left active at
serve so a mobile crop is random, reintroduces the skew. Augment
on the train dataset or behind a `training` flag you actually
pass.

Preprocessing in `tf.data.map` vs in-graph layers: map happens
on CPU in the pipeline (good for heavy JPEG decode); in-graph
layers travel with the model (good for per-example scale that
must match serve). Hybrid is normal: decode in `tf.data`,
normalize in the model.

## TF Transform

When the same statistics must be computed on a large cluster
and frozen for serving, a graph-only `adapt` on one machine
is not enough.

**TF Transform** (Apache Beam + TensorFlow) analyzes the train
corpus (min, max, vocab, mean) and emits a **transform graph**
that train and serve both run. That is how you avoid "we
recomputed the mean in Flask."

Vocabularies and scaling constants that drift between the
training job and the model server are a silent accuracy killer.
One analysis pass, one exported transform, two consumers
(trainer and server): that is the TFX-shaped answer.

Analyzing on a convenience sample that misses rare tokens, then
serving a flood of UNKs, looks like a serve bug. It is a vocab
computed on the wrong slice of data.

You can skip TF Transform in a workshop notebook. You cannot
skip the *idea* if you will ship.

## TensorFlow Datasets (TFDS)

TFDS is a catalog of ready `tf.data` pipelines (MNIST, ImageNet
downsamples, Wikipedia dumps, …) with splits, versioning, and
a builder API for your own sets.

Every tutorial redownloads MNIST into a different folder with a
different preprocess. `tfds.load(..., as_supervised=True)` and
then *your* `map`/`batch`/`prefetch` keeps the baseline honest.
Splits (`train`, `test`, `validation`) are part of the dataset
version; do not reshuffle a canned test split into train because
the val curve looked sad.

Treating TFDS as "the data team" oversells it. Catalog sets are
for learning and baselines. Production tabular dumps still need
the Example/TFRecord path and a schema you own.

## Putting it next to Keras `fit`

```
  train_ds = input_pipeline(shuffle=True, augment=True)
  val_ds   = input_pipeline(shuffle=False, augment=False)

  model.fit(train_ds, validation_data=val_ds, epochs=...)
```

If you wrote a custom loop, iterate the same datasets. Do not
have two parse functions. The pipeline is a contract: batch
structure, dtypes, padded shapes.

Train pipeline with `drop_remainder=True` and a val pipeline
that does not, then a model traced on static `[B, ...]`, is a
shape landmine. Prefetch missing on train only means you
benchmark the GPU on val and the CPU on train.

## What aged since 2019

- **Keras preprocessing layers moved house** —
  `tf.keras.layers.experimental.preprocessing` became stable
  `keras.layers` (and, under Keras 3, backend-agnostic layers).
  Read your installed docs for the import path. The jobs
  (lookup, rescale, text vectorize) are the same.
- **TFX still exists**; the product names around it have
  churned. The idea "analyze once, apply in train and serve"
  is what to keep. Some teams now do the same job in Spark,
  Beam, or a feature store.
- **`tf.data` service / snapshot / options** grew after 2.0.
  Prefetch, parallel map, shuffle buffer, cache are still the
  first four knobs.
- A frozen retrieval encoder feeding an index is a different
  architecture from `layers.Embedding` trained on your labels.
  Keep that distinction when someone says "we added embeddings."
- Mixed precision and large-batch input pipelines matter more
  on modern GPUs; they do not replace parse-and-prefetch.

## Check yourself

1. Sketch a `tf.data` chain for images on disk. Where do
   shuffle, map, batch, and prefetch sit, and what goes wrong
   if you swap shuffle and batch?
2. Your GPU util is 20% and CPU is hot. Which two `tf.data`
   knobs do you reach for first, and how would you tell
   augmentation-in-map from a tiny shuffle buffer?
3. When do you want TFRecord shards instead of
   `from_tensor_slices`? What does a single uncompressed
   200 GB TFRecord still get wrong?
4. `Example` vs `SequenceExample`: which one is a row, and
   which one has a time axis?
5. One-hot vs an embedding table for 12 countries vs 120,000
   SKUs. What is trained in the embedding case, and what loss
   moves those rows?
6. Someone says "we added embeddings" and points at an index
   of PDF chunk vectors from a frozen encoder. What would an
   embedding *in this chapter* look like instead?
7. Why put rescaling in a Keras layer that ships with the
   SavedModel rather than only in `dataset.map`? When would
   you still decode JPEGs in `map`?
8. What problem is TF Transform solving that `layer.adapt` on
   a laptop does not, and what failure mode looks like a
   "serve bug" but is a vocab computed on the wrong split?
9. TFDS gives you a `test` split. What is the honest use of
   that split, and why is "just this once, mix it into train"
   dishonest?
10. Name one thing you would change in the train pipeline but
    not the val pipeline, and one thing that must stay
    identical or the val metric is a lie.
