# 18. Reinforcement Learning

Companion notes for **Chapter 18** of *Hands-On Machine Learning with
Scikit-Learn, Keras, and TensorFlow* (2nd edition, Aurélien Géron;
O'Reilly, 2019).

Unsupervised chapters learn from unlabeled *datasets*. This chapter
learns from **interaction**: an agent takes actions, the environment
answers with a next state and a number, and the agent updates a
**policy** so that the discounted sum of numbers goes up. Skip it and
you will call every loop that "does something" reinforcement learning,
ship a Q-table that cannot see pixels, or file a language-model tool
loop under Bellman and then wonder why there is no reward tensor.

## The mental model

```
                    +------------------+
                    |    POLICY π      |  (or Q, or both)
                    |  a ~ π(a | s)    |
                    +--------+---------+
                             | action a
                             v
  state s  --------->  ENVIRONMENT  --------->  next state s'
                             |
                             +---- reward r ----+
                                                |
                                                v
                          RETURN  G = r + γ r' + γ² r'' + ...
                          credit assignment: which a deserved G?
                          update π or Q so G goes up
```

The one sentence to remember a year from now: **an RL agent maximises a
scalar reward by changing a policy through value or policy updates.**

Supervised learning: `(x, y)` from a frozen dataset. RL: `y` is missing;
you only get `r`, often later, and your actions **change which data you
see**. That is why this chapter is longer than "fit a classifier on
rewards."

## Learning to optimize rewards

You can specify what you want as a **number per step** (or per
episode), not as a labeled action for every state. Design a reward.
The agent maximises expected return. This is a product decision
pretending to be an algorithm decision. Dense rewards ("+1 for
progress") are easy to learn and easy to game. Sparse rewards ("+1
only if the pole is still up at step 500") match the spec and starve
the learner.

**Reward hacking** is the agent maximising *what you wrote*, not what
you meant (infinite loops of a cheap +0.1, sitting still if motion is
penalised, dying on purpose if death resets a painful state). If you
cannot name how the policy could cheat, you have not finished the
spec. This is a misspecified scalar.

## Policy search

A **policy** maps states (or observations) to actions — deterministic
or a distribution. Policy search tweaks the policy's parameters to
raise expected return: hill-climbing, finite differences, evolutionary
strategies, later gradients.

You do not have to learn a value function. You can search directly in
policy space. That idea returns as policy gradients below.

Searching a high-dimensional neural policy with vanilla random
perturbations and no baseline drowns the signal in noise. This is why
the chapter bothers with credit assignment and gradients instead of
"just try bigger weights."

## OpenAI Gym (vintage)

The 2019 standard sandbox is **OpenAI Gym**: a small Python API so
algorithms can talk to many toys (CartPole, MountainCar, Atari
wrappers) without rewriting the environment each time.

```
  env.reset()  ->  obs
  env.step(action)  ->  obs, reward, done, info
  env.render()      ->  pixels or a window (optional)
```

That is the whole contract this chapter needs: reset, step, a flag
when the episode is over. The environment is a **simulator or a game
wrapper**, not a customer-support runtime.

Treating Gym as production infrastructure is a mistake. It is a
research interface. Seeds, wrappers, and "done" semantics were already
messy in 2019; they got a cleanup later (see **What aged**). Do not
build a company on `env.render()`.

## Neural network policies

The observation is a vector (or pixels). A table of actions per
discrete state will not fit. A net outputs action logits or means.
For CartPole-class problems a two-layer MLP that maps observation →
probability of "left" vs "right" is enough to *illustrate* a policy.
You sample an action, step the environment, collect the trajectory.

Pixels need convolutional layers. This chapter's first nets are small
on purpose so you can see the RL plumbing.

A huge conv net on CartPole makes the algorithm undebuggable because
the model is also a research project. A deterministic argmax policy
during **training** in a method that needs exploration stops
exploring; you start repeating one action.

## The credit assignment problem

Rewards arrive late. The action that doomed the pole happened thirty
steps before the fall. **Credit assignment** is the question: which
actions in the trajectory should we reinforce?

Monte Carlo style: wait until the episode ends, compute the return
from each step, push the policy toward actions that sat on high
returns. Unbiased, high variance, slow if episodes are long.

Rewarding every action in a winning episode equally credits the
random twitch before the good move the same as the good move.
Variance explodes; learning looks like luck.

## Policy gradients

You want to climb expected return with a neural policy and you cannot
differentiate through the environment. The REINFORCE-class identity:
increase the log-probability of actions that were followed by a high
return, decrease it when the return was poor. A **baseline** (often a
learned value `V(s)`) subtracts out "how good is this state anyway?"
so you credit the *advantage*, not the raw return.

This is on-policy: the data has to come from the policy you are
updating (or you need a correction this chapter does not make you
implement first).

No baseline, huge returns, and a single lucky episode can drag the
whole net. Or a learning rate that steps the policy so far that the
next batch of trajectories is from a different agent and the gradient
is fiction. Later algorithms (PPO, TRPO) exist *because* this failure
is the default.

## Markov decision processes

The math furniture:

- **State** `s` (or observation `o` if you do not see `s`).
- **Action** `a`.
- **Transition** `P(s' | s, a)`.
- **Reward** `R(s, a, s')` (notations vary).
- **Discount** `γ ∈ [0, 1)` so infinite horizons do not explode and so
  near rewards beat far ones (unless you set `γ ≈ 1` on purpose).
- **Policy** `π(a | s)`.
- **Value** `V^π(s) = expected return from s following π`.
- **Action-value** `Q^π(s, a)`: expected return if you take action
  `a` in `s`, then follow `π`.

**Markov** means the state is a sufficient statistic: the future does
not care about the path except through `s`. If your "state" is a single
frame of Pong, you have probably violated that and will bolt on frame
stacks.

Bellman equations relate `V` / `Q` at `s` to the same functions at
`s'`. They are the recurrence for "value." Dynamic programming solves
them when you **know** `P` and `R` and the state space is small.
The rest of the chapter is what you do when you do not.

Calling a non-Markov observation a state and then blaming Q-learning
is a common mistake. Also: `γ = 1` on a continuing task with positive
rewards makes values explode.

## Temporal difference learning and Q-learning

**Temporal difference (TD)** updates a value estimate from the next
step's estimate, without waiting for the episode to finish. Low
variance, some bias, you can learn online.

**Q-learning** (tabular, off-policy): bump `Q(s, a)` toward
`r + γ max_{a'} Q(s', a')`. Off-policy means the *behaviour* policy
can explore while the *target* is "what a greedy policy would do."
Exploration is usually ε-greedy (or a decaying variant).

When the tables fit, this is the algorithm you should be able to
implement on a whiteboard. It is also the last time RL will feel like
a spreadsheet.

### Approximate Q and DQN

Too many states (pixels). Tables do not fit. A neural net
`Q(s, a; θ)` approximates the table. Naive "TD on a net" diverges:
consecutive samples are correlated, the target moves every step, and
bootstrap plus function approximation plus off-policy is the deadly
triad.

**DQN** (the 2015-shaped recipe this chapter implements) adds the
minimum kit:

- a **replay buffer** of transitions, sampled off-policy as if they
  were a dataset,
- a **target network** copied from the online net on a slower clock
  so the bootstrap target does not chase itself every gradient step.

You still need exploration. You still need to stack frames if a single
image is not Markov.

A replay buffer of size 1000 on Atari, or size 10 million on CartPole,
mismatches the task: you either overfit yesterday's five transitions
or train on ancient policies forever. Updating the target net every
step (you did not have a target net) or never (the target is a random
init) both undo the recipe.

## DQN variants

The chapter's "rainbow ingredients" without requiring the full Rainbow
paper:

- **Fixed Q targets.** The target net already mentioned. Without it,
  DQN is often just "unstable Q-approx."
- **Double DQN.** The online net *selects* `argmax a'`; the target net
  *evaluates* that action. Cuts the systematic overestimate of
  `max Q`. Skipped: optimistic Q, policies that chase phantom high
  values.
- **Prioritized experience replay (PER).** Sample transitions with
  large TD error more often (plus importance weights so the bias is
  not silent). Skipped: the buffer is dominated by easy, useless
  transitions. Mis-tuned: you overfit a handful of noisy errors.
- **Dueling DQN.** Split the net into `V(s)` and advantage `A(s, a)`,
  recombine into Q. Helps when many actions share a state value and
  only some of them matter. Skipping is not always fatal; dueling is
  a capacity prior, not a new objective.

You can stack these. You should still be able to name the job of each
knob when the run looks cursed.

## TF-Agents as a 2019 library sketch

The book walks **TF-Agents** so the pieces have names in code, not so
you adopt a dead stack as production.

Sketch of the moving parts, not a how-to:

```
  ENVIRONMENT  (+ wrappers: frame skip, resize, reward clip, ...)
       |
       |  specs: observation spec, action spec  (shapes and dtypes)
       v
  POLICY / AGENT  (network + the update rule: DQN, DDPG, ...)
       |
       +--> DRIVER  runs the policy in the env, writes transitions
       |
       v
  REPLAY BUFFER  -->  dataset of batches  -->  training loop
```

- **Env / wrappers.** Same idea as Gym wrappers; Atari preprocessing
  is a pile of them (grayscale, stack, skip).
- **Specs.** The contract so networks and buffers refuse a wrong-shaped
  tensor *early*.
- **Replay.** Off-policy memory.
- **Driver.** The collector: "take N steps, dump them."
- **Training loop.** Sample, compute TD loss, apply gradients, sync
  target nets, decay ε, log.

Copy-pasting a 2019 TF-Agents notebook into a 2026 product is the wrong
move. The *architecture* (env, replay, collector, learner) is still how
every serious RL codebase looks. The library name on the import line is
not. See **What aged**.

## Survey of other algorithms (names and jobs)

You do not need implementations here. You need to know what problem
each name is paid to solve:

| Name | Job |
|---|---|
| REINFORCE / vanilla PG | On-policy gradient on full returns |
| Actor-critic (A2C / A3C) | Policy plus a learned `V` baseline; A3C adds async workers |
| TRPO / PPO | Trust-region / clipped updates so the policy cannot leap |
| DDPG / TD3 | Off-policy actor-critic for **continuous** actions |
| SAC | Maximum-entropy actor-critic; exploration via entropy |
| Rainbow | DQN plus the variant stack (double, PER, dueling, n-step, ...) |
| AlphaZero / MuZero-class | Plan in a model or a learned model; not a Gym DQN |

If the action space is continuous, vanilla DQN is the wrong first
call: Q-learning's `argmax_a` is easy when `a` is `{left, right}` and
a headache when `a` is a torque vector. That is why DDPG/TD3/SAC exist.

## What aged since 2019

- **Gym → Gymnasium.** The API migrated (the `done` flag split into
  terminated vs truncated; namespace changes). The reset/step idea
  did not. New code should use the maintained fork, not nostalgia.
- **TF-Agents went quiet.** Research and a lot of production moved to
  PyTorch ecosystems (CleanRL, Stable-Baselines3, RLlib, JAX stacks).
  Keep the driver / replay / learner *picture*; do not standardise
  your company on TF-Agents because it is in this edition.
- **PPO and SAC are the common defaults**, not vanilla DQN, especially
  for continuous control. DQN-class methods still matter for discrete
  control and for teaching. If you only remember one 2015 paper you
  will pick the wrong tool on MuJoCo-shaped tasks.
- **Simulators.** MuJoCo licensing eased; Isaac / Brax / other GPU
  sims showed up. Atari remains a benchmark, not a product.
- **RL on language models.** RLHF / RLAIF use RL *on* language models
  as a **training** method (reward model + PPO or a DPO-class
  substitute). That is still this chapter's family: a policy, a
  scalar, an update. Shipping a tool-calling chat loop is a different
  job.
- **Still teach Bellman and Q here.** Flashy algorithms are wrappers
  around `V`, `Q`, advantage, and a replay or a trajectory batch.
  If those four are fog, PPO will also be fog.

## Check yourself

1. Write one sentence that says what an RL agent maximises and what
   it updates. How does that differ from supervised learning on a
   fixed `(x, y)` dataset?
2. Give a reward that is easy to learn and a way an agent could hack
   it. Then give a sparser reward and say what dies in the learning
   signal.
3. Why is Gym (or Gymnasium) an interface and not a deployment
   platform? What are the two calls that define an episode?
4. Credit assignment: why is "the episode succeeded, reinforce every
   action" a bad update? What does a baseline change in the gradient?
5. On-policy vs off-policy in one line each. Which bucket is vanilla
   REINFORCE? Which bucket is DQN with a replay buffer?
6. Write the Q-learning target. Where does the `max` introduce
   overestimation, and which variant exists to cut it?
7. Name two reasons naive neural Q-learning diverges. Which DQN
   ingredient attacks each reason?
8. PER vs dueling: one changes *which transitions you see*, one
   changes *how Q is parameterized*. Which failure looks like
   "the net never revisits the rare crash"?
9. Why is vanilla DQN a poor default for continuous torques? Which
   surveyed names are the usual next call?
10. TF-Agents: list env, spec, wrapper, replay, driver, training
    loop. Which of those still exist if you throw the library away
    in 2026, and which name in **What aged** replaced Gym?
