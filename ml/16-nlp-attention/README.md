# 16. NLP with RNNs and Attention

Companion notes for **Chapter 16** of *Hands-On Machine Learning with
Scikit-Learn, Keras, and TensorFlow* (2nd edition, Aurélien Géron;
O'Reilly, 2019).

Earlier sequence work used recurrence on numbers: forecast a series,
keep a hidden state, reach for LSTM when the gradient dies. This
chapter points the same machinery at **language** — characters,
sentiment labels, translations — and then at the idea that made
recurrence optional: **attention**, and the Transformer built from
it. Skip it and you will treat "the model" as a chat box, paste a
Transformer diagram you cannot explain, and call an embedding table
a search index. You will also be unable to say what you are training
when you later sample a hosted language model.

## The mental model

Four ways to turn a string into a prediction. Mixing their names is
how a design review becomes "we should use BERT" with no training
set.

```
  characters / tokens
       |
       v
  [ EMBED ]          lookup table. Task loss trains (or loads) vectors.
                     One vector per vocabulary id for this net.
       |
       +-- Char-RNN / LSTM ---- hidden state walks left -> right
       |
       +-- Encoder-decoder ---- last encoder state is a bottleneck
       |         |
       |         +-- Attention  decoder *looks at* every encoder step
       |
       +-- Transformer -------- no recurrence. Q, K, V. Positions added.
                                encoder stack + decoder stack + FFN
       |
       v
  next character  |  class (sentiment)  |  translated token
```

The one sentence to remember a year from now: **this chapter trains a
sequence model from data you own.** Calling an already-trained
language model through an API is a different job.

If you cannot point to which box you are training, you are reading
the wrong notes. If there is no training job, you are outside this
chapter.

## Generating Shakespeare-shaped text with a character RNN

You want a model that continues a string in a style, and you only
have a long document, not labeled pairs. Treat the document as a
stream of **characters**. Slide a window of length `n_steps` across
it. The target is the next character. A recurrent net (often
LSTM/GRU) reads the window and predicts a distribution over the
vocabulary. At generation time you sample from that distribution,
append, and repeat.

This is a toy on purpose. Character-level Shakespeare is small enough
to fit in a notebook and large enough to show every sequential-data
trap that word-level models still have.

### Splitting sequential data

Rows of housing prices are (approximately) exchangeable. Characters
in a play are not. If you shuffle windows into train and test, the
test set contains phrases the train set already saw one character to
the left.

A "99% next-char accuracy" that is just memorizing overlapping
windows is a lie. Split **by position in the stream**: earlier text
for training, a later slice for validation/test. The cut is a time
cut, even when the "time" is page number.

### Windows

One long string is not a batch. You chop it into sequences of fixed
length so gradient steps have a shape. Windows may overlap; more
overlap means more training examples and more correlation between
them. The input is `n_steps` characters; the label is the following
character (or, in some setups, the shifted sequence for seq-to-seq
training).

If `n_steps` is too short, the net never sees a sentence. If it is
too long, you wait on vanishing gradients and GPU RAM before you
have learned anything. Start short; the Transformer later in the
chapter is how the field stopped needing one huge recurrent unroll.

### Stateful RNNs

A **stateless** RNN resets the hidden state at the start of every
window. The model can only condition on the `n_steps` you fed it.

A **stateful** RNN keeps the hidden state across consecutive batches
so the memory can, in principle, span the whole document. That only
works if batch *i+1* is the true continuation of batch *i*. You must:

- turn shuffling off,
- arrange batches as parallel consecutive streams,
- reset the state at the start of each epoch (the document is not a
  circle unless you decide it is).

Stateful training plus shuffled batches makes the hidden state a lie
from another play. Accuracy looks noisy; generation is gibberish with
confidence. If you cannot draw the batch layout, use stateless.

Sampling from *your* char-RNN is generation from this softmax while
you decode this net. Temperature here is a knob on that distribution.

## Sentiment analysis

Variable-length reviews, a binary (or few-class) label, vocabulary
in the tens of thousands. Map each token to a vector, run a sequence
encoder (RNN, 1D CNN, later a Transformer encoder), pool, classify.
Padding makes a rectangular batch; **masking** tells the loss and the
recurrent cell to ignore pad steps.

Masking is not a convenience. Without it the model learns that a lot
of zeros mean "this review is short," or worse, treats pad as a real
token and lets it vote on the class. Padding changes the tensor shape.
Masking changes which steps contribute to the gradient.

### Pretrained embeddings (still model features)

Embedding **tables** map integers to dense vectors and train with the
rest of the net. This chapter adds the 2019 habit of **loading**
vectors that someone else trained on a large corpus (word2vec / GloVe
/ a TF Hub sentence encoder of that era) and either freezing them or
fine-tuning.

That is **transfer of features into this classifier**. The vectors
sit in the first layer. There is no retrieve-then-generate loop. There
is no chunker. There is no tenant index. An `Embedding` layer stores
one vector per vocabulary id for a net you are fitting.

Domain mismatch is a second trap. Wikipedia GloVe will not know your
ticket codes. Then you either fine-tune the table on *your* labeled
reviews or you accept that the pretrained geometry is a prior, not a
product.

## Encoder–decoder neural machine translation

Input is a sentence in language A; output is a sentence in language B
of a **different length**. A fixed window classifier cannot emit a
variable-length translation.

An **encoder** RNN reads the source and compresses it into a thought
vector (usually the last hidden state). A **decoder** RNN is
initialized from that vector and emits target tokens one by one,
trained with teacher forcing (the previous *true* token is fed in
during training; at inference you feed the model's own previous
guess).

This is sequence-to-sequence. It is the same shape as "summarize,"
"transcribe," "parse into JSON" — whenever the output is a sequence
whose length you do not know in advance.

The thought vector is a bottleneck. Long sources get squashed into
one vector; the decoder forgets the beginning by the time it
translates the end. Attention exists because this failure was
measured.

### Bidirectional RNNs

A forward RNN sees the past. A backward RNN sees the future of the
*source* (you have the whole source at encode time). Concatenate the
two hidden states per position and the encoder representation of a
word can use both sides of its sentence.

You cannot run the same trick uncritically on the **decoder** at
generation time: there is no future target token yet.
Bidirectionality is an encoder gift. On a stream that must be causal
(online speech, token-by-token generation), a bidirectional encoder
would cheat with future information that production will not have.

### Beam search

Greedy decoding picks the most likely next token every time. Errors
compound: a slightly-wrong first word can make the rest of the
sentence the "best" continuation of a mistake.

**Beam search** keeps *k* partial translations (the beam), expands
each with the vocabulary, and retains the *k* highest-scoring
sequences. Beam 1 is greedy. Exact search over all strings is
impossible.

A tiny beam buys little. A huge beam costs latency and tends toward
bland, high-probability mush (especially without a length penalty,
short translations win because probabilities multiply). For production
NMT you also need an end-of-sequence token and a cap, or the decoder
never stops.

## Attention mechanisms

The decoder needs different parts of the source at different times
("the" does not need the same encoder step as the main verb).

At each decoder step, compute a **score** between the decoder state
(the query) and every encoder output (the keys). Softmax those scores
into weights. Take a weighted sum of encoder values. That sum is the
context vector for this step. The decoder no longer depends on a
single thought vector.

Intuition: attention is a **soft lookup**. The query asks "what in
the source matters now?"; the keys are addresses; the values are what
you actually mix in.

You can visualize the weight matrix as an alignment table: target
position vs source position. When it is diagonal-ish, you are looking
at a reasonably aligned translation. When it is a blob, the model is
confused or the languages do not align monotonically.

### Visual attention (mention)

The same idea was used in image captioning: the "encoder" is a conv
net's spatial grid, the "decoder" is a language RNN, and the weights
highlight **which region** the caption is talking about. This chapter
mentions it so you do not file attention under "NLP-only."

A pretty heatmap is a useful debug plot. It is not a proof of what
the model "used," and it is not a substitute for a held-out BLEU /
exact-match / human eval.

## The Transformer (architecture intuition)

The 2017 sequence paper this chapter walks is a bet: **if you have
attention and positions, you do not need recurrence or convolution**
to translate. Hold this stack in your head:

```
  source tokens + positional encoding
       |
       v
  +---- Encoder block x N ------------------+
  |  multi-head self-attention (Q,K,V from   |
  |  the same sequence)                      |
  |  residual + layer norm                   |
  |  position-wise feed-forward (two linear  |
  |  layers, same MLP at every position)     |
  |  residual + layer norm                   |
  +------------------------------------------+
       |
       |  encoder outputs as K, V for the decoder
       v
  target tokens (shifted) + positional encoding
       |
       v
  +---- Decoder block x N ------------------+
  |  masked self-attention (cannot peek at   |
  |  future target tokens)                   |
  |  encoder-decoder attention (Q from the   |
  |  decoder, K/V from the encoder)          |
  |  feed-forward, residuals, layer norms    |
  +------------------------------------------+
       |
       v
  softmax over target vocabulary
```

### Q, K, V and multi-head

For each attention head you project the incoming vectors into
**queries, keys, and values**. Compatibility of query *i* with key
*j* becomes the weight on value *j*. Multi-head means several of
these projections in parallel, concatenated, then mixed. Different
heads can specialize (syntax vs longer links) without you assigning
the job by hand.

Self-attention: Q, K, V all from the same sequence. Encoder-decoder
attention: Q from the decoder, K and V from the encoder — that is the
"look at the source" hop.

### Positional encoding

Attention has **no built-in order**. Shuffle the encoder inputs and,
without positions, the set is the same. The architecture adds a
position signal (sinusoids in the original design; learned position
embeddings in many later nets) so "dog bit man" is distinct from
"man bit dog."

### Feed-forward, depth, residuals

The FFN is applied independently at each position: attention mixes
*across* the sequence; the FFN mixes *within* a position's channels.
Residuals and layer normalization are why you can stack N blocks
without the signal dying — the same deep-net hygiene as earlier
training chapters, new wiring.

Implementing "a Transformer" as only self-attention, then wondering
why generation copies the future, means you skipped the **mask** on
decoder self-attention. That mask is the causal contract. Skipping
positions and assuming recurrence will sneak back in will also fail:
it will not.

This is enough architecture to read a block diagram and to know what
you would be fitting if you trained one. It is not enough to
reproduce the paper, and it is not a product LLM.

## "Recent innovations" as of 2019

The book closes the chapter on two shapes that were news then:

- **GPT-shaped (decoder-only).** Unsupervised next-token training on
  a large corpus, then (in that era) supervised fine-tune on a task.
  Generation is the native mode: you already trained a language model.
- **BERT-shaped (encoder, bidirectional).** Mask tokens, predict them
  from both sides; optionally a next-sentence objective. Natural fit
  for classification and span tasks. The encoder does not emit a full
  generative decoder by itself.

Those two bets — **pretrain a Transformer on unlabeled text, then
adapt** — are the hinge between this chapter and everything that
followed. In 2019 you were still expected to *run* that adaptation.
The architecture did not change its name. The default *job* did.

## What aged since 2019

The architecture intuition did not expire. The default workflow did.

- **You usually do not train this from scratch.** From ~2022 onward
  the product move is an **instruction-tuned, preference-tuned API**
  (chat models, system prompts, tools). You still need this chapter
  to know what that API is made of. You do not need to start from a
  char-RNN to ship a support bot.
- **Decoder-only Transformers ate most generation.** BERT-shaped
  encoders remain in retrieval and classification; the public "LLM"
  people mean is almost always a causal decoder stack plus
  post-training (instruction data, RLHF / DPO-class methods — those
  algorithms live closer to reinforcement learning than to this file).
- **Tokenization moved.** Character RNNs are a teaching device.
  Production vocabularies are subword (BPE, Unigram, SentencePiece).
  The windowing and split discipline still apply at the token level.
- **Attention implementations changed under the hood** (sparse,
  flash-style kernels, longer context). The QKV picture did not.
- **Embedding tables and pretrained word vectors** still belong with
  the net you train or fine-tune. Hugging Face hubs made loading a
  pretrained encoder a one-liner; that still trains or fine-tunes **a
  net**.
- **Seq2seq RNNs left the default stack.** Attention + Transformer is
  the backbone. Encoder-decoder RNNs remain the right *story* for why
  attention was invented.

If your task is "call a chat API and retrieve our wiki," you are
outside this chapter. If your task is "explain multi-head attention"
or "fine-tune a classifier on our labeled tickets," you are in the
right one.

## Check yourself

1. A teammate says they "trained an LLM" because they called a chat
   API with a system prompt. Which box in this chapter's diagram did
   they skip?
2. Why is a random shuffle of character windows a leak, and what
   split would you use instead?
3. Draw batch layout for a **stateful** RNN on one document. Where
   does shuffling break the state?
4. Masking vs padding: which one changes the tensor shape, and which
   one changes the *gradient*? What does the classifier learn if you
   skip masking?
5. You load GloVe into an `Embedding` layer and freeze it. Where do
   the vectors live, and what job are they doing in the classifier?
6. Encoder-decoder NMT without attention fails on long sentences.
   Name the bottleneck in one sentence, then say what the decoder
   queries in the attention variant.
7. Why can the **source** encoder be bidirectional while a generating
   decoder cannot peek at future *target* tokens?
8. Beam 1 vs beam 8 vs "search the whole vocabulary tree": what do
   you gain and what goes bland or slow?
9. In one Transformer decoder block, which attention is masked, which
   one reads the encoder, and why do you add a positional encoding at
   all?
10. BERT-shaped vs GPT-shaped in 2019 terms: which one is a native
    generator, and what aged about "you will fine-tune these
    yourself"?
