# Human-Heur Progress Report

## Best Stable Result

**`ret_eval = 1900`** via `evaluate.sh` with the 22-parameter CPG+PD heuristic controller at `x_scale=0.01`.

## How It Was Built

1. Designed a 22-parameter controller: CPG oscillator + PD joint control + torso stabilization + velocity feedback + arm PD
2. Bootstrapped `x_center` via multi-stage CMA-ES (4 stages, 150-200 rounds each)
3. Swept `x_scale` — 0.01 was the sweet spot for TuRBO

## What the Controller Actually Does

Quasi-stands (hip_y_amp converged to 0). Survives ~400 steps on seed 18 before yaw drift (~490 deg) causes a forward pitch and fall.

## What Was Tried and Didn't Beat 1900

- 9+ KPOP hypotheses: yaw PD control, yaw bias, body-frame velocity transform, forced walking, z-reflex, multi-restart CMA, 28-param expansion
- Different optimizers (CMA, optuna, sobol, turbo-1, etc.)
- More budget (50-100 rounds)
- Wider/narrower x_scale sweeps
- Random center search (1000 perturbations)
- Iterative re-centering on TuRBO-found best
- CMA-ES from scratch (full parameter space)

## Key Fragility

The 1900 is specific to x_scale=0.01 + seed 18 + the exact CMA-bootstrapped center. Other seeds give 524-704. Adjacent x_scale values give 1230-1650. The improvement from 1603 (center) to 1900 (TuRBO) comes from a nonlinear 22-way parameter interaction.

## Most Promising Unexplored Direction

Body-frame velocity correction is theoretically the right fix for the yaw-drift failure mode, but needs more CMA-ES bootstrapping budget to find a competitive center.
