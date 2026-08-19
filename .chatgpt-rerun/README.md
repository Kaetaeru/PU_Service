# ChatGPT Rerun Protocol

This directory is the durable coordination surface between ChatGPT Rerun and the GitHub project.

## Mandatory read order

Every resume or dispatch cycle must read in this order:

1. `.chatgpt-rerun/README.md`
2. `.chatgpt-rerun/control.json`
3. `.chatgpt-rerun/STATE.md`
4. `.chatgpt-rerun/PLAN.md`

`STATUS.md` is a human-facing projection only. It is not a reconciliation source of truth.

## Preflight reconciliation

Before doing implementation work, reconcile README, `control.json`, `STATE.md`, and `PLAN.md`. Preserve an existing active run's `run_id`, `sequence`, current task, and validation history. Repair only missing or incompatible protocol details; do not reset active state.

The authoritative state-write order is:

1. `PLAN.md`
2. `STATE.md`
3. `control.json`

`control.json` must be the final authoritative write for a state transition.

## Runtime limits and checkpoints

A single active execution has a 20-minute hard stop. At about 18 minutes, write a checkpoint before stopping so that a later resume can continue safely.

For long active executions, keep `STATUS.md` reasonably fresh, targeting roughly 5-minute freshness. Also update it immediately on meaningful state changes.

## Watcher and control semantics

Chrome Side Panel **Start/Stop controls only the tab watcher**. It is independent from GitHub control status.

`continue` is the authorization to start or resume work. `complete`, `needs_user`, and `blocked` are dispatch waiting states; they do not turn off the watcher, and polling must continue.

If a terminal/waiting state is later changed back to `continue` with the same sequence, that new `continue` is fresh work authorization and the watcher must automatically resume dispatch. Do not use a `working` status.
