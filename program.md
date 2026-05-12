# hopsworks-autoresearch

Agent does its own ML research on Hopsworks data. Loop: edit a single
training script, run it, log result to a Hopsworks feature group, keep
wins, repeat. Same shape as Karpathy's autoresearch but the result store
is a feature group instead of a tsv, and "kept" experiments register
their model in the Hopsworks model registry.

## Inputs from the user

The wizard (or a human) hands you these up-front. If any are missing,
ask before doing anything else.

- **features**: a comma-separated list `fg_v.feature` (e.g. `sales_v3.qty,
  weather_v1.temp`). May be empty if the user hasn't picked from the
  catalog — in that case ask which feature view / feature groups to use,
  or whether to start from a fresh upload / connector / public dataset.
- **intent**: the optimization target in plain English. Example:
  "minimize MAE on next-hour demand", "maximize AUC on churn within 30
  days". This pins the metric for the whole run.
- **budget**: total experiment budget. Example: "5 minutes per experiment,
  hard cap of 30 experiments (~2.5h total)".
- **compute**: `cpu` or `gpu`. CPU = stay in sklearn / XGBoost /
  LightGBM territory, no CUDA deps. GPU = PyTorch / TF / CUDA libraries
  OK, pick model classes that actually benefit from the GPU.
- **tag**: a short run tag. If none given, propose one based on today's
  date (e.g. `dec5`). Branch `autoresearch/<tag>` must not already exist.

## Setup

1. `git checkout -b autoresearch/<tag>` from current main/master.
2. If features were given, build (or reuse) a feature view that selects
   them. If features were empty, work with the user on a feature view
   based on the source they chose.
3. Create a Hopsworks feature group named
   `autoresearch_experiments_<tag>` with this schema if it doesn't
   exist. Online enabled = false. Primary key = `commit`.

   | column          | type      | notes                          |
   | --------------- | --------- | ------------------------------ |
   | commit          | string    | short git hash, primary key    |
   | val_metric      | double    | the optimization target value  |
   | peak_memory_gb  | double    | peak RSS / VRAM in GB          |
   | status          | string    | keep / discard / crash         |
   | description     | string    | one-line summary of the change |
   | ts              | timestamp | event time                     |

4. Write `train.py` as a baseline. Single file, single process. It must:
   - Log into Hopsworks: `import hopsworks; project = hopsworks.login()`.
   - Pull the feature view's training data.
   - Train a sensible baseline (XGBoost / LightGBM / sklearn linear
     model — pick by the problem type implied by `intent`).
   - Print exactly these three lines at the end (one per line, exact
     format — the loop greps them):
     ```
     val_metric: <float>
     peak_memory_gb: <float>
     training_seconds: <float>
     ```
   - Honour a per-experiment wall-clock budget (default 5 min). Stop
     training and still print the three lines if time runs out.

5. Run the baseline once: `python train.py > run.log 2>&1`.
6. Insert the baseline row into `autoresearch_experiments_<tag>`,
   `status=keep`, description="baseline".
7. Register the baseline as `autoresearch_<tag>` v1 in the Hopsworks
   model registry, with `metrics={"val_metric": <value>}` and a
   description that includes `status=keep` and `metric_direction`.
   Every subsequent experiment (keep or discard) registers as the
   next version of the same model name.

Confirm the baseline number with the user once, then enter the loop
without further confirmations.

## What the agent CAN change

- `train.py` is the only file you edit. Architecture, hyperparameters,
  loss, optimiser, feature selection, transformations, model class — all
  fair game.

## What the agent CANNOT change

- The optimization target inside `intent`. Same metric throughout the
  run, same direction (min vs max). If you redefine the metric mid-run
  the leaderboard becomes meaningless.
- The schema of `autoresearch_experiments_<tag>`.
- The 3-line stdout output format above.
- The user's feature view definition without asking.

## The experiment loop

```
while not budget_exhausted:
    1. tune train.py with ONE experimental idea (commit message says what)
    2. git commit
    3. python train.py > run.log 2>&1
       - kill if it exceeds 2x the per-experiment budget
    4. parse: grep "^val_metric:\|^peak_memory_gb:" run.log
       - if empty, run crashed; tail -n 50 run.log, attempt fix
       - few fix attempts max; if still broken, status=crash, move on
    5. insert row into autoresearch_experiments_<tag>
    6. register the model as a new version of autoresearch_<tag>
       (keep, discard, and crash all get a version so the registry
       UI shows the full run; status and a one-line summary go on
       the model description)
    7. if val_metric improved per the metric direction:
         - advance the branch (keep the commit)
         - status=keep
       else:
         - git reset --hard HEAD~1
         - status=discard
```

Budget enforcement: stop when total budget is hit OR the user
interrupts you. Otherwise never stop. Never ask "should I keep going?".
The user might be asleep.

## Crash policy

- Dumb error (typo, missing import): fix and re-run.
- Fundamentally broken idea (OOM at any reasonable size, divergent
  loss, schema mismatch): log `status=crash`, move on.

## Hopsworks specifics

Two surfaces, one run:

- The experiments FG `autoresearch_experiments_<tag>` is the
  leaderboard. Every experiment writes a row (keep, discard, or
  crash) so the run is fully queryable from a notebook or `hops fg
  preview`. Don't enable online for this FG; it's a log, not a serving
  surface.
- The model registry is the visualisation. Every experiment registers
  as a **new version** of a single model named `autoresearch_<tag>`,
  regardless of keep/discard. The UI then renders the run as one
  model with N versions sorted by version number, with metrics charted
  per version. Set `metrics={"val_metric": <value>}` so the chart has
  a y-axis; put `status` and a one-line `description` of the change
  on the model so a human scanning the registry sees the keep/discard
  decision and what was tried without opening the FG.
- Pick the registry namespace that matches what you trained:
  `mr.python.create_model(...)`, `mr.sklearn.create_model(...)`,
  `mr.tensorflow.create_model(...)`, `mr.torch.create_model(...)`, or
  `mr.llm.create_model(...)`. All versions of a given run must use
  the same namespace so version numbers are consecutive.
- If you create the feature view, name it `autoresearch_<tag>_fv`. One
  FV per run, no shared state with other autoresearch branches.

## Simplicity rule

A 0.001 improvement that adds 20 lines of hacky code: discard. A 0.001
improvement that *removes* code: keep. Equal results with simpler code:
keep. Complexity demon is the enemy.
