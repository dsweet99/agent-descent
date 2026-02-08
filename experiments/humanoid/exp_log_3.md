# Experiment Log 3: Architecture changes for human-heur

## Problem
Get `ret_eval` to 5000 for `human-heur` at the **original** `healthy_reward=5`.
The 16-parameter CPG+PD controller has a proven hard ceiling of ~1215 (tested with
100 rounds = 1000 BO evaluations in exp_log_2 H22). The architecture must change.

## Baseline
- ret_eval = 1157 (10 rounds), 1215 (100 rounds)
- Controller: 16 params — CPG frequency, leg amplitudes/offsets/phases, PD gains, torso PD, gyro damping, action scale
- Survival: ~207 steps max

---

## H1: Second harmonic in CPG for non-sinusoidal gait trajectories
- **Hypothesis**: Real gaits have asymmetric swing/stance timing. Adding sin(2*phi) terms to hip_y and knee targets (4 new params → 20 total) allows the BO to shape richer joint trajectories.
- **Prediction**: ret_eval > 1157 because the optimizer can discover asymmetric gaits.
- **Test**: Add hip_y_amp2, hip_y_phase2, knee_amp2, knee_phase2. Set x_center=-1.0 for amps (default=0, preserving baseline behavior).
- **Result**: FALSIFIED. ret_eval=889. The 4 extra dimensions diluted the BO's 100-eval budget. The search space grew 25% but the budget didn't. The 2nd harmonic was a good idea, but BO can't find it efficiently.

---

## H2: Pure standing controller (no CPG, just PD balance)
- **Hypothesis**: healthy_reward=5 × 1000 steps = 5000. If the humanoid can stand still for all 1000 steps, we hit the target. The CPG forces walking, which creates instability. Remove it.
- **Prediction**: ret_eval > 1157 because standing is more stable than walking.
- **Test**: 8-param controller (hip/knee/torso PD gains + gyro_kd + act_scale, no CPG, no offsets). 10 rounds: 696. 30 rounds: 696.
- **Result**: FALSIFIED. Standing is HARDER than walking. The humanoid needs dynamic stability (momentum from walking) to stay upright. Static balance with this body morphology is intrinsically unstable. Key finding: the solution MUST involve locomotion.

---

## H3: Torso roll setpoint replaces hip_x_amp
- **Hypothesis**: The humanoid yaws and tips sideways. A tunable roll angle for abdomen_x (instead of zero) could counteract the natural lateral drift. Replace hip_x_amp (which BO uses weakly) with torso_roll.
- **Prediction**: ret_eval > 1157.
- **Test**: Replace hip_x_amp with torso_roll ∈ [-0.2, 0.2] in abdomen_x PD setpoint.
- **Result**: FALSIFIED. ret_eval=880. Removing hip_x_amp lateral sway was worse than adding a roll setpoint was helpful. The lateral hip oscillation provides dynamic lateral stability during walking.

---

## H4: Forward velocity feedback replaces hip_y_amp
- **Hypothesis**: The humanoid accelerates uncontrollably and trips. Replacing hip_y_amp (near zero) with vel_kp creates a negative feedback loop: `hip_y_target = offset - vel_kp * vx`. Faster → less lean → slow down.
- **Prediction**: ret_eval > 1157 through velocity regulation.
- **Test**: vel_kp ∈ [-1, 1], modulates hip_y_offset by forward velocity.
- **Result**: FALSIFIED. ret_eval=835. The velocity feedback changes x_center's meaning for param 1, invalidating the tuned center. Also, the humanoid doesn't fail due to excessive speed — it fails due to yaw.

---

## H5: Yaw-dependent CPG phase offset for heading correction
- **Hypothesis**: Yaw divergence is the primary failure mode. Asymmetric stepping (phase offset proportional to wz) creates corrective turning, like humans adjusting heading mid-stride. Replace hip_y_amp with yaw_kp.
- **Prediction**: ret_eval > 1157 through heading correction.
- **Test**: `rp = phi + yaw_kp*wz`, `lp = phi + pi - yaw_kp*wz`. yaw_kp ∈ [-1, 1].
- **Result**: FALSIFIED. ret_eval=854. Modulating CPG phase with a noisy signal (wz) disrupts the gait timing. Also invalidates x_center for param 1.

---

## H6: PID torso control (integral term eliminates steady-state drift)
- **Hypothesis**: PD control has steady-state error under constant disturbance (the CPG IS a constant disturbance). An integral term accumulates pitch/roll error and corrects drift. This adds persistent state — a qualitative architectural change.
- **Prediction**: ret_eval > 1157.
- **Test**: (a) Replace hip_y_amp with torso_ki, 16 params: ret_eval=967. (b) Add torso_ki as param 17: ret_eval=834.
- **Result**: FALSIFIED. Both variants worse. (a) shows PID is promising (967 > other modifications' ~850) but losing hip_y_amp costs more. (b) shows 17D BO is less efficient than 16D. Key observation: PID gave the BEST score among all architecture modifications (967) suggesting integral control is directionally correct, but the x_center invalidation penalty outweighs the benefit.

---

## H7: Bootstrapped x_center (iterative BO refinement)
- **Hypothesis**: The BO is stuck in a local basin. Extract the best params from round 1, use as new x_center, run BO again. The second BO explores a DIFFERENT neighborhood centered on the first optimum.
- **Prediction**: ret_eval > 1157 because the second BO refines the first optimum.
- **Test**: Extracted BO-optimal x_center. x_scale=0.5: ret_eval=1049. x_scale=0.2: ret_eval=1004.
- **Result**: FALSIFIED. Both worse. The original hand-tuned x_center was BETTER as a starting point than the BO's own optimum. Some BO-optimal params hit the clip boundary (cpg_freq→-1.0, hip_kd→-1.0), meaning the BO wanted values BEYOND the boundary. Re-centering on these clipped values restricts access to good regions.

---

## H8: Widen parameter ranges for boundary-clipped params
- **Hypothesis**: The BO pushes cpg_freq to the boundary (lowest value). Widening the range allows even lower frequencies that the BO wants but can't reach.
- **Prediction**: ret_eval > 1157 with wider freq range.
- **Test**: cpg_freq range [0.005, 0.12] instead of [0.02, 0.12].
- **Result**: FALSIFIED. ret_eval=1030. Changing the range shifts the mapping for the SAME x_center value, invalidating the tuned center. The x_center=-0.70 now maps to freq=0.022 instead of 0.035. Every parameter range change is effectively an x_center change.

---

## H9: Contact-force feedback for stance/swing gain modulation
- **Hypothesis**: The CPG is blind — it doesn't know which foot is on the ground. Using cfrc_ext (obs[318:324] right foot, obs[336:342] left foot) to stiffen stance leg gains (×1.5) and soften swing leg gains (×0.7) would improve stability. Fixed modulation, no new params.
- **Prediction**: ret_eval > 1157 through biomechanically meaningful feedback.
- **Test**: Modulate hip/knee kp by 1.5 (stance) or 0.7 (swing) based on foot contact force norm.
- **Result**: FALSIFIED. ret_eval=627. The gain modulation shifted the effective PD gains far from their BO-optimized values. The BO tuned gains for CONSTANT values; multiplying by 0.7-1.5 breaks that optimization.
- **Key insight**: ANY fixed behavioral change that alters the effective gain landscape makes the existing x_center suboptimal. The BO's 100 evals can't re-adapt to the changed landscape.

---

## H10: CMA-ES optimizer instead of TuRBO
- **Hypothesis**: The "hard ceiling of 1215" was demonstrated with TuRBO. CMA-ES is a population-based optimizer that explores differently and may escape TuRBO's local basin.
- **Prediction**: ret_eval > 1215 with CMA-ES.
- **Test**:
  - CMA-ES 10×10 (100 evals): ret_eval=1026
  - CMA-ES 10×30 (300 evals): ret_eval=1256 ✓ **EXCEEDS TuRBO's 1215!**
  - CMA-ES 10×100 (1000 evals): ret_eval=1273
  - CMA-ES 30×30 (900 evals): ret_eval=1166
- **Result**: PARTIALLY CONFIRMED. CMA-ES reaches 1273, exceeding TuRBO's ceiling of 1215. But the improvement is only 5% — CMA-ES hits its own ceiling at roughly the same level. The true controller architecture ceiling is ~1275, not 1215. The gap to 5000 remains enormous (4x).

---

## Summary

### Scoreboard

| # | Hypothesis | ret_eval | Status |
|---|-----------|----------|--------|
| H1 | 2nd harmonic CPG (20 params) | 889 | FALSIFIED |
| H2 | Pure standing (no CPG, 8 params) | 696 | FALSIFIED |
| H3 | Torso roll setpoint replaces hip_x_amp | 880 | FALSIFIED |
| H4 | Forward velocity feedback replaces hip_y_amp | 835 | FALSIFIED |
| H5 | Yaw-dependent CPG phase offset | 854 | FALSIFIED |
| H6 | PID torso control (integral term) | 967 / 834 | FALSIFIED |
| H7 | Bootstrapped x_center (iterative BO) | 1049 / 1004 | FALSIFIED |
| H8 | Wider parameter ranges for clipped params | 1030 | FALSIFIED |
| H9 | Contact-force stance/swing gain modulation | 627 | FALSIFIED |
| H10 | CMA-ES optimizer | **1273** | PARTIAL |

### Key findings

1. **The architecture ceiling is real: ~1275.** Both TuRBO (1215 with 1000 evals) and CMA-ES (1273 with 1000 evals) converge to essentially the same ceiling. The 16-param CPG+PD controller physically cannot make this humanoid survive more than ~230 steps.

2. **The x_center is load-bearing.** Every architectural modification that changes a parameter's meaning degrades performance because the carefully tuned x_center becomes invalid. The penalty for an invalid x_center exceeds the benefit of any structural improvement.

3. **Standing is harder than walking.** The humanoid requires dynamic stability (momentum from stepping) to stay upright. Pure PD balance (no CPG) gives only 696.

4. **Adding params hurts; removing params hurts.** With 100 BO evaluations, 16D is the sweet spot. Both 17D and 8D performed worse.

5. **To reach 5000 at healthy_reward=5: need ~800 steps survival.** The current controller maxes out at ~230 steps. Closing this 3.5x gap requires either:
   - A fundamentally different controller architecture (e.g., neural network with RL training, not BO)
   - A different problem formulation (e.g., different humanoid morphology, different reward structure)

### Final configuration
- Original 16-param CPG+PD controller (no modifications)
- x_center and x_scale unchanged
- Best optimizer: CMA-ES with 10 arms × 30 rounds → ret_eval=1256
- Standard TuRBO (turbo-enn-f-p) with 10×10 → ret_eval=1157

