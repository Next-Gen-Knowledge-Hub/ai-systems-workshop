# 15. Processing sequences using RNNs and CNNs

Companion notes for **Chapter 15** of *Hands-On Machine Learning with
Scikit-Learn, Keras, and TensorFlow* (2nd edition, Aurélien Géron;
O'Reilly, 2019).

Images had a 2D layout. This chapter is **1D layout over time** (or
over any ordered axis): a neuron that sees the past through a state,
how you train that loop, and what you do when the loop forgets or
explodes. Skip it and you will flatten a week's sensors into a bag of
means, or declare RNNs dead before you can draw a sequence-to-vector
shape. Small time series, streaming state, and the vocabulary (cell,
BPTT, teacher forcing, horizon) start here.

## The mental model

A recurrent layer is a cell applied at every timestep, with a
**hidden state** that carries a summary of the past into the
next call.

```
  x_0     x_1     x_2           x_t
   |       |       |             |
   v       v       v             v
  [h] --> [h] --> [h] --> ... --> [h]   hidden state (memory cell)
   |       |       |             |
   v       v       v             v
  y_0     y_1     y_2           y_t     (optional outputs)

  unroll in time, then backprop through the unroll (BPTT)
```

The one sentence to remember a year from now: **an RNN is
parameter sharing along an ordered axis plus a state that must
survive that axis** — and most of the chapter is how that state
dies (gradients, short memory) or how you avoid needing a long
state (1D conv, shorter windows, later: attention).

Two consequences fall out of that diagram. First, "sequence"
is a tensor rank and a time axis: you have to say whether each
step has an output. Second, a forecasting demo without a naive
baseline (persist last value, seasonal copy) is how a weak RNN
looks like a win.

## Recurrent neurons and layers

At one step, a simple cell is:

```
  h_t = act( W_x x_t + W_h h_{t-1} + b )
  y_t = act_out( W_y h_t + b_y )     # if you emit per step
```

`W_x` and `W_h` are **shared** across `t`. Depth in *time* is
the unroll; depth in *space* is stacking cells (a deep RNN).

A Dense layer on the last 24 hours treats hour 3 and hour 23 as
unrelated feature columns (unless you engineer that). Share the
step and keep a state, and the net can in principle look
arbitrarily far back. In practice, see LSTM/GRU below.

Keras: `SimpleRNN`, `LSTM`, `GRU` with `return_sequences`
controlling whether you get the last `h` or the full `y_t`
stack. `stateful=True` is a specialist mode: you keep `h`
across batches when those batches are consecutive chunks of
the same stream. Forget to `reset_states` and you leak one
series into the next.

An `input_shape` that swaps batch, time, and features
(`[B, T, F]` vs `[B, F, T]`) will still train. It will train
on nonsense.

## Memory cells, input and output shapes

"Memory cell" in this chapter means: the unit whose internal
state is the recurrence (plain `h`, or LSTM's `h` and `c`).
It is the recurrent state vector, not an external retrieval
store.

Four I/O patterns. Draw them before you code.

```
  seq2vec     [B, T, F] --> [B, D]         classify a series
  vec2seq     [B, D]     --> [B, T, D]     generate a length-T thing
  seq2seq     [B, T, F] --> [B, T, D]      aligned (same T)
  enc --> dec [B, T_in] --> [B, T_out]     lengths may differ
```

Seq2seq aligned: tag each frame, denoise each step, predict
the next step in a window (careful with leakage).
Encoder–decoder: the encoder eats the input sequence, the
decoder emits another; attention later puts a richer path on
that bottleneck. This chapter can still *name* the bottleneck.

`Dense` on `return_sequences=True` output vs `False` is where
shapes lie in the summary until you print them. One line per
model: batch, time, channels at each layer. If time vanished,
you have a vector-out model. If you needed per-step labels,
you just broke the loss.

Padding zeros to a common `T` and then a loss that treats
padding as real steps teaches the net to predict zero.
Masking (or a RaggedTensor) is part of the model.

## Training RNNs

Unroll, compute the loss on the outputs you meant, backprop
**through time** (BPTT). Truncated BPTT: backprop only `k`
steps even if the forward state ran longer — a bias/variance
trade with memory.

The graph for `T=1000` is a 1000-layer net that shares
weights. Truncate the unroll; window the data; or use cells
that carry state better (LSTM/GRU). Gradient clipping is
routine here.

Shuffling windows so that `stateful` RNNs see a random jump,
or not shuffling when windows are iid crops and you wanted
iid minibatches, are opposite mistakes. Stateful and
stateless are different datasets.

Teacher forcing (decoder gets the *true* previous token at
train) vs feeding its own prediction creates a train/serve
gap you will meet hard on language models. For numeric
forecasting, "feed the predicted value back" is the
multi-step story below.

## Forecasting time series

A series is `x_t` over time, possibly with extra channels
(temperature, price, counts). A **window** is a supervised
row: past `L` steps → future `H` steps (the horizon).

```
  ...  x_{t-L} ... x_{t-1} x_t  |  x_{t+1} ... x_{t+H}
           input window         |    labels (horizon)
```

**Baselines you run first:**

- Persist: `ŷ_{t+1} = x_t` (and seasonal persist:
  `ŷ_{t} = x_{t-season}`).
- Mean of the window.
- A linear model on the flattened window.

If your RNN cannot beat persist on a near-random walk, you
do not have a modeling win. You have a plot.

Reporting MSE on a series whose level walked up, and a model
that predicts the mean looks strong on one split and useless
on the next, is a metric trap. Stationarize or use a
scale-free metric when the level moves; split **in time** (no
future leak); keep a seasonal naive in the notebook forever.

StandardScaler fit on the whole series including the test
tail is the classic leak, easier to commit because the "rows"
are sequential.

### A simple RNN, then a deep one

Start with one `SimpleRNN` (or even a Dense on the window)
as a sanity check. Then stack cells (`return_sequences=True`
on every layer but the last if the last is vec-out).

Deep RNNs: each layer eats a sequence of hidden vectors.
You buy capacity; you buy vanishing/exploding *in depth and
in time*. Batch-norm is awkward; layer-norm shows up more.
Dropout on RNNs has "where to put it" variants (on inputs,
on recurrent connections); do not sprinkle Dense-dropout
wisdom blindly.

Three LSTM layers of 512 on 200 points because "deep is
better" will memorize the train window pattern.

### Multi-step forecasts

One-step: predict `t+1`. If you need `t+2` you have a
choice.

- **Recursive / autoregressive** — feed predictions back.
  Error compounds. Matches some serve paths.
- **Direct vector-out** — one head emits `H` steps
  (`[B, H]` or `[B, H, F]`). Needs more output capacity;
  does not reuse a one-step model.
- **Seq2seq** — encoder on the history, decoder over the
  horizon. Flexible; easy to overfit; natural when `H`
  varies.

One-step val MSE is excellent; a 24-step rollout is junk —
that is usually a train/serve mismatch. Measure the horizon
you will serve. If you roll out, train at least sometimes on
that rollout (or on scheduled sampling) so train and serve
match. Plotting only one-step dots on top of the series and
calling it a 24-hour forecast hides the gap.

Exogenous features (known future: holidays, planned
promos) belong in the decoder inputs when they are truly
known. Putting the future *target* in the input is leak.

## Long sequences

Two different diseases, often diagnosed as one.

### Unstable gradients

The unroll is a deep net. Products of Jacobians vanish or
explode. Non-saturating activations in the cell, clipping,
careful init, shorter truncation, and (mostly) **better
cells** are the toolkit.

Clipping a model whose real problem is that `T` is 5,000
and the cell is a tanh SimpleRNN caps NaNs. It does not buy
memory.

### Short-term memory → LSTM and GRU

Even with stable gradients, a simple cell overwrites `h`
every step. Information from `t=0` has to survive a
gauntlet of writes. Picture `h` as a whiteboard that gets
erased and rewritten at every timestep: a cue from eighty
steps ago has to be recopied perfectly every time or it
vanishes. That is short-term memory, even when gradients
are healthy.

**LSTM** adds a second highway: a **cell state** `c` that
gates can leave almost untouched. Three gates (forget,
input, output) decide what to erase from `c`, what new
content to write, and how much of `c` to expose as `h`.
The default can be "carry `c` unchanged," so a reset event
or an opening parenthesis can survive dozens of steps.

**GRU** is a cheaper cousin with a fused reset/update story
and one state vector instead of two.

```
  LSTM:  (h_t, c_t)   c is the highway
  GRU:   h_t          fewer weights, often similar quality
```

The net cannot use a cue from 80 steps ago if that cue was
never encoded. Gated cells help; also give the model a fair
window / features (calendar, lagged seasonal values). A gate
cannot invent a signal you never fed.

LSTM as a personality — "we use LSTM so we handle long
memory" — oversells. On many small seasonal series, a linear
model with lags plus a 1D conv beats a stacked LSTM.
Measure.

### 1D convolution and WaveNet-shaped ideas

A **1D conv** slides a kernel along time. No recurrent
state. Easy to parallelize. Receptive field = kernel size
and dilation and depth, not "theoretically infinite."

```
  WaveNet-shaped: dilated causal convs

  t:  0  1  2  3  4  5  6  7
  d1:   x--x  x--x  x--x  x--x     kernel 2, dilation 1
  d2:   x-----x     x-----x        dilation 2
  d4:   x-----------x              dilation 4
```

**Causal** = no peeking at the future (pad left, never
right). **Dilated** = holes in the kernel so the field
grows exponentially with layers.

You want local patterns (a spike shape, a phoneme, a
5-minute motif) and a GPU that is not waiting on a Python
time loop. 1D conv stacks (optionally dilated, optionally
followed by a small RNN or a Dense head) deliver that.
WaveNet-shaped nets are "all conv, causal, dilated" for
long audio-like signals.

A non-causal conv on a forecasting task so the "prediction"
of `t` saw `t+1` makes accuracy look magical. Serve will
not. Causal padding is a correctness issue.

You can mix: conv to downsample time, RNN on the shorter
grid. That hybrid is often the grown-up 2019 answer before
you reach Transformers.

## What aged since 2019

- **Transformers often beat RNNs** on language, many
  sequence-labeling jobs, and a lot of long-range tasks.
  The lesson that still belongs here is the sequence shape:
  state, horizon, and a causal window.
- **RNNs are still fine** for small tabular time series,
  online state, and teaching BPTT. A 2-layer GRU on a
  hundred-point sensor is not a research failure.
- **Temporal conv nets, N-BEATS / N-HiTS, PatchTST,
  and other TS-specific stacks** showed up as forecasting
  defaults in many shops. The *lesson* from this chapter
  remains: baseline, window, horizon, no leak, causal
  when you forecast.
- **Keras RNN APIs** grew `LSTMCell` vs layer, fused
  CUDA implementations, and masking details.
  `return_sequences` / `return_state` are still the shape
  knobs.
- **State-space models** and linear recurrences are a
  later fashion for long context. Same job as "a state
  that survives time"; different algebra. Stay on LSTM/
  GRU/1D-conv until you need attention.

## Check yourself

1. Draw seq2vec vs aligned seq2seq vs encoder–decoder.
   Which Keras flag is `return_sequences`, and which
   shape bug looks like "my loss compiled but T vanished"?
2. Why is a recurrent layer *weight sharing over time*
   rather than a different Dense per timestep? What
   would the parameter count do if it were not shared?
3. Stateful vs stateless RNNs: what must be true of the
   batch order, and what happens if you shuffle like a
   normal classifier?
4. Name two naive forecasting baselines. If your LSTM
   cannot beat them on a random-walk-ish series, what
   did you probably measure wrong?
5. One-step MSE is low; 12-step recursive forecast is
   not. Is that a cell bug or a train/serve mismatch?
   What are two ways to train for a horizon `H`?
6. Vanishing gradients vs short-term memory: which one
   is "the derivative died," which one is "the state was
   overwritten," and which tool (clip vs LSTM vs shorter
   window) maps to which?
7. LSTM vs GRU in one diagram each. When would you pick
   the cheaper one in a workshop?
8. A 1D conv beats your LSTM on a motif-heavy series.
   What did the conv get "for free" that the RNN had to
   learn in time? What can the RNN still do that a small
   receptive field cannot?
9. What does *causal* mean in a WaveNet-shaped stack,
   and how would a non-causal pad cheat a forecast
   metric?
10. Give one sequence job where skipping straight to
    Transformers is reasonable, and one (small TS,
    streaming cell) where you still want an RNN or a
    1D conv first.
