# Experiment Log 6: Diagnostic Exploration

## Problem
Improve `ret_eval` of `human-heur` controller (Humanoid-v5) to 5000 at `healthy_reward=5`.
Current best: 1900 (with `evaluate.sh`: turbo-enn-f-p, 10 arms, 10 rounds, seed 18).

## Key Diagnostic Findings

### 1. Budget is NOT the bottleneck
- 50 rounds TuRBO at x_scale=0.01: still 1900 (no improvement from 5x more budget)
- 100 rounds TuRBO at x_scale=0.05-0.50: 1131-1552 (worse)
- 100 rounds CMA-ES at x_scale=0.05-0.50: 1131-1635 (worse)

### 2. Other optimizers perform worse
- CMA at x_scale=0.01: 1373
- turbo-enn-f: 1441
- Optuna: 1500
- Sobol: 1511
- turbo-enn-f-p remains the best optimizer for this problem

### 3. The 1900 is EXTREMELY seed-specific
- 5 replicates (seeds 18-22): returns = 1900, 579, 524, 704, 585
- Only seed 18 produces 1900; other seeds yield 524-704

### 4. The 1900 is a sharp spike at x_scale=0.01
- x_scale sweep: 0.001:1580, 0.003:1655, 0.005:1414, 0.008:1535, **0.01:1900**, 0.012:1230, 0.015:1562, 0.02:1243
- The 1900 depends precisely on x_scale=0.01 + the specific TuRBO seed (45)

### 5. The optimal controller is NOT walking
Physical parameter values at center:
- hip_y_amp = 0.0 (AT LOWER BOUND) -- no sagittal leg swing!
- hip_kd = 0.0 (AT LOWER BOUND) -- no hip damping
- torso_kp = 7.94 (AT UPPER BOUND) -- maximum torso stiffness
- torso_kd = 0.0 (AT LOWER BOUND) -- no torso damping
- yaw_kd = 0.0 (AT LOWER BOUND) -- no yaw damping
- z_reflex_gain = 0.087 (near lower bound)

The humanoid quasi-stands with tiny lateral/knee wobbles, surviving ~400 steps before falling.

### 6. Forced walking is WORSE
- Forced hip_y_amp >= 0.15, CMA-ES 200 rounds: best = 1547 (vs 1900 quasi-standing)

### 7. Failure mechanism: yaw drift + forward pitch
The 1900 trajectory:
- Steps 0-350: humanoid rotates CCW through ~490deg, z stays at ~1.27
- Steps 350-399: pitch increases, vx accelerates negatively, z drops 1.27 -> 0.99 -> fall
- Controller uses global-frame velocity feedback, which is misaligned after rotation

### 8. Body-frame velocity transform doesn't improve results
- Body-frame center (CMA from scratch, 300 rounds): 1160
- Body-frame + TuRBO sweep: max 1333 at x_scale=0.02
- Body-frame with old center + CMA 300 rounds: 1423
- All significantly below global-frame 1900

### 9. The center is a very strong local max
- 1000 random perturbations across 5 scales: ZERO improvements over center (1603)
- The 1900 comes from TuRBO finding a specific 22-way parameter interaction

### 10. Sensitivity analysis
- Individually, 20/22 perturbations HURT performance
- Only knee_amp (-0.05 -> 1769) and arm_kp (-0.10 -> 1679) help individually
- Combined knee_amp + arm_kp: 1771
- But TuRBO finds 1900 through complex nonlinear interaction of ALL params

### 11. Action saturation
- abd_x (roll) saturates 46% of the time -- controller needs more roll authority
- abd_y (sagittal) saturates 21%
- Removing act_scale from torso: 1241 (worse, breaks TuRBO landscape)

### 12. CMA-ES from scratch
- Global-frame, x_center=0, x_scale=1, CMA 100 rounds: 1130
- Global-frame, x_center=0, x_scale=1, TuRBO 100 rounds: 924
- Both far below bootstrapped center (1603 direct, 1900 TuRBO)

### 13. Iterative centering doesn't help
- Updated center to TuRBO-found best params: 1231 (WORSE)
- The 1900 depends on the specific center-scale-seed interaction

## Summary
The 1900 result is a fragile coincidence of:
1. The CMA-ES bootstrapped center
2. x_scale=0.01 exactly
3. TuRBO seed 45 (from problem_seed=18+27)

Achieving 5000 likely requires a controller architecture that can maintain balance beyond 400 steps. The fundamental limitation is: the humanoid has no ankle joints, rotates continuously due to asymmetric dynamics, and the controller can't compensate because velocity feedback is in the wrong frame. Body-frame correction is theoretically correct but hasn't yet found a competitive operating point.

## Status
Controller restored to global-frame, CMA-ES bootstrapped center, x_scale=0.01. All quality checks pass (ruff, kiss, pytest 964/964).
