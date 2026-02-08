# Experiment Log 4: Transparent control law changes

## Problem
Get `ret_eval` to 5000 for `human-heur` at original `healthy_reward=5`.
Current ceiling: 1157 (evaluate.sh/TuRBO), 1275 (CMA-ES).
Humanoid falls at ~207-230 steps. Need ~800+ steps for 5000.

## Strategy
Make changes to the control LAW that don't alter any parameter's meaning.
All 16 params keep their exact interpretation. The x_center stays valid.
The BO searches the same landscape but the underlying dynamics are improved.

## Baseline
- ret_eval = 1157 (10 arms × 10 rounds, turbo-enn-f-p)

---

## H1: Velocity feedforward in joint PD
- **Hypothesis**: PD tracks position but ignores desired velocity. Adding cos(phi) feedforward reduces phase lag and improves tracking. No param meaning changes.
- **Prediction**: ret_eval > 1157.
- **Result**: FALSIFIED. ret_eval=739. The feedforward term effectively reduces the damping (kd now damps velocity ERROR, not raw velocity), shifting the optimal gain balance. Even "transparent" changes shift the landscape.

---

## H2: Action smoothing (EMA on outputs)
- **Hypothesis**: Jerky torque changes excite body resonances. Smoothing via `a = alpha*a_raw + (1-alpha)*a_prev` reduces high-frequency oscillations.
- **Prediction**: ret_eval > 1157.
- **Result**: FALSIFIED. alpha=0.3: 826. alpha=0.1: 875. Even mild smoothing adds lag that harms responsiveness more than it helps stability.

---

## H3: Nonlinear PD — sqrt(|error|)
- **Hypothesis**: Linear PD under-corrects small errors (drift) and over-corrects large errors (overshoot). sqrt(|error|) inverts this.
- **Prediction**: ret_eval > 1157.
- **Result**: FALSIFIED. Full nonlinear: 1048 (10 rounds), 1074 (30 rounds). Legs-only: 875. Torso-only: 751. Best modification yet (1048) but still below baseline. The nonlinear PD has a LOWER ceiling than linear PD — the controller was co-optimized with linear dynamics.

---

## H4: tanh instead of clip for action saturation
- **Hypothesis**: Hard clip creates gradient discontinuity at ±1. tanh is smooth. No param changes.
- **Prediction**: ret_eval > 1157.
- **Result**: FALSIFIED. ret_eval=721. tanh(a) compresses the output more than clip(a) for moderate values (tanh(1)=0.76 vs clip(1)=1). The controller needs the full ±1 range.

---

## H5: Quaternion-based absolute orientation correction
- **Hypothesis**: Current torso PD only corrects relative torso-pelvis angle. If the whole body tilts, no correction fires. Add fixed quaternion-based pitch/roll correction for absolute orientation.
- **Prediction**: ret_eval > 1157.
- **Result**: FALSIFIED. gain=2*tkp: 901. gain=0.5: 889. The abdomen joints already reflect most of the body tilt. Adding quaternion correction creates over-correction.

---

## H6: Fixed yaw damping through abdomen_z
- **Hypothesis**: Yaw divergence is the root cause of falling. Add a fixed (no param) yaw correction to abdomen_z: `a[1] += -K * wz`.
- **Prediction**: ret_eval > 1157.
- **Result**: FALSIFIED. K=1.0: 832. K=0.3: 778. Abdomen_z (gear=40) has too little torque authority for yaw correction. The correction also interferes with the existing abdomen_z PD.

---

## H7: Subspace optimization — freeze boundary params
- **Hypothesis**: 5 of 16 params are pushed to boundaries by BO (cpg_freq, hip_y_amp, knee_amp, knee_offset, knee_kp). Freeze them at optimal values and let BO focus on 11 remaining params. Fewer effective dimensions = better coverage.
- **Prediction**: ret_eval > 1157.
- **Result**: FALSIFIED. ret_eval=853. Freezing boundary params breaks co-adaptation — the frozen params' optimal values depend on the free params' values.

---

## H8: Optimize x_scale (search radius)
- **Hypothesis**: x_scale=0.5 may be too wide (wastes BO budget on bad regions) or too narrow (misses the optimum). There's a sweet spot.
- **Prediction**: Some x_scale gives ret_eval > 1203.
- **Result**: PARTIALLY CONFIRMED. x_scale=0.3: 1203 (best). x_scale=0.2: 1165. x_scale=0.25: 850. x_scale=0.35: 824. x_scale=0.5: 1157 (original baseline). x_scale=0.3 is the sweet spot but doesn't break the ceiling — only +4% improvement.

---

## H9: Bootstrapped x_center + tight x_scale
- **Hypothesis**: Set x_center to the BO-optimal absolute values from x_scale=0.5 run, then refine with tight x_scale=0.15. Searches a small region around a known-good point.
- **Prediction**: ret_eval > 1203.
- **Result**: PARTIALLY CONFIRMED, but only with more budget. 10 rounds: 1058 (worse than baseline!). 30 rounds: **1579** (new record!). 100 rounds: 1579 (ceiling). The tight search needs more evals to converge. At 10 rounds (evaluate.sh default), it's worse.

---

## H10: Per-parameter x_scale (anisotropic)
- **Hypothesis**: Give narrow range (0.15) to boundary params, wide range (0.5) to middle params. Focuses BO on uncertain dimensions.
- **Prediction**: ret_eval > 1203.
- **Result**: FALSIFIED. ret_eval=749. TuRBO assumes isotropic trust region. Anisotropic scaling wastes dimensions on narrow-range params without informing the optimizer.

---

## H11: Force walking by constraining hip_y_amp > 0
- **Hypothesis**: The BO chooses STANDING (hip_y_amp=0, ~5/step × 230 steps ≈ 1150). If forced to walk, forward reward adds ~1.25/step, potentially exceeding standing.
- **Prediction**: ret_eval > 1203 with walking.
- **Result**: FALSIFIED. hip_y_amp∈[0.1,0.5] + cpg∈[0.05,0.15]: 862. hip_y_amp∈[0.05,0.3] + original cpg: 965. Walking destabilizes the humanoid — falls sooner, losing more health reward than it gains in forward reward. STANDING IS OPTIMAL for this controller.
- **Key Insight**: The controller CAN'T walk stably. Extending standing time (currently ~230 steps) is the only path forward.

---

## H12: Yaw correction through hip_z joints
- **Hypothesis**: Use hip_z (transverse) joints to resist yaw drift. Both legs rotate against yaw angular velocity.
- **Prediction**: ret_eval > 1203 by extending standing time.
- **Result**: FALSIFIED. gain=−0.5: 982. gain=+0.2: 1062. gain=+0.1: 1059. Hip_z correction interferes with the standing posture's lateral stability.

---

## H13: Z-height maintenance reflex
- **Hypothesis**: When z-height drops below 1.25, extend both knees to push back up. Fixed reflex, no params.
- **Prediction**: ret_eval > 1203.
- **Result**: FALSIFIED. ret_eval=1024. The knee extension is too aggressive and disrupts the existing PD balance.

---

## H14: Arms for balance
- **Hypothesis**: Arms can serve as angular momentum reservoirs. Swing arms opposite to body tilt to counterbalance.
- **Prediction**: ret_eval > 1203.
- **Result**: FALSIFIED. gain=0.3: 921. gain=0.1: 882. Arms are too light (~5% body mass) and arm movement creates perturbations that outweigh stabilization benefits.

---

## H15: Online lean adaptation
- **Hypothesis**: z-height drops gradually. Integral controller on z-height adjusts torso lean angle during episode.
- **Prediction**: ret_eval > 1203.
- **Result**: FALSIFIED. ret_eval=847. Slow lean drift confuses the BO-optimized balance.

---

## H16: Arm PD gain tuning
- **Hypothesis**: The hardcoded arm PD gains (kp=0.4, kd=0.04) may not be optimal for standing stability.
- **Prediction**: Different gains improve ret_eval.
- **Result**: FALSIFIED. kp=1.0/kd=0.1: 762 (too stiff). kp=0.1/kd=0.01: 1082 (slightly worse). Zero arms: 882 (floppy). Original gains are near-optimal.

---

## H17: 17th parameter for yaw damping
- **Hypothesis**: Give yaw its own independent gain (separate from pitch/roll gyro_kd). Applied to abdomen_z and hip_z joints.
- **Prediction**: ret_eval > 1203.
- **Result**: FALSIFIED. ret_eval=796. Adding a 17th dimension hurts BO efficiency more than yaw damping helps. The x_center for yaw_kd starts at 0 (off), and the BO can't find the right gain in 100 evals.

---

## H18: Iterative bootstrapping of x_center
- **Hypothesis**: Iteratively update x_center by finding best params via random search, then shrink x_scale. This walks the center toward the global optimum.
- **Prediction**: ret_eval > 1203.
- **Result**: MIXED. Manual random search (500 trials × 5 iters): final center scored 934 (10 rounds). BO-optimal bootstrapped center from exp_log_3: x_scale=0.15 → 1058 (10 rounds), x_scale=0.2 → 980, x_scale=0.3 → 996. The bootstrapped center only shines with 30+ rounds (1579).

---

## H19: Measure variance across runs
- **Hypothesis**: The BO might be stochastic, so multiple runs could find different optima.
- **Prediction**: Results vary across runs.
- **Result**: FALSIFIED. All 5 runs produce EXACTLY 1203. The BO is fully deterministic (fixed seed=18, noise_seed_0=180). Zero variance. This means 1203 is the exact answer, not a stochastic estimate.

---

## H20: CMA-ES optimizer
- **Hypothesis**: CMA-ES handles correlated parameters better than TuRBO.
- **Prediction**: ret_eval > 1203 at 10 rounds.
- **Result**: FALSIFIED. CMA-ES at x_scale=0.3 (10 rounds): 1046. x_scale=0.5 (10 rounds): 927. x_scale=0.5 (100 rounds): 1196. Bootstrapped (30 rounds): 1315. CMA-ES is WORSE than TuRBO for this problem, especially with bootstrapping.

---

## H21-22: Fine-tuning bootstrapped center configuration
- **Hypothesis**: The bootstrapped center (1579) can be improved by tuning x_scale and arms/rounds.
- **Prediction**: Some config beats 1579.
- **Result**: FALSIFIED. x_scale=0.1 (30 rounds): 1442. x_scale=0.2 (30 rounds): 1138. 50 arms × 6 rounds: 981. 5 arms × 60 rounds: 1140. x_scale=0.15 + 10×30 remains optimal at **1579**.

---

## H23: Wider torso_kp range [0.5, 15.0]
- **Hypothesis**: The BO pushes torso_kp high. Maybe it wants even more stiffness.
- **Prediction**: ret_eval > 1203.
- **Result**: FALSIFIED. ret_eval=694. Widening the range changes the MAPPING at x_center (kp jumps from 7.0 to 13.0), breaking co-adaptation.

---

## H24: Wider act_scale range [0.1, 1.5]
- **Hypothesis**: The controller may need more torque authority.
- **Prediction**: ret_eval > 1203.
- **Result**: FALSIFIED. ret_eval=795. Same issue — wider range shifts the value at x_center.

---

## H25: Asymmetric duty cycle CPG
- **Hypothesis**: Longer stance phase (foot on ground) improves stability. Use `max(0, sin) + 0.5*min(0, sin)` for half-amplitude swing, full-amplitude stance.
- **Prediction**: ret_eval > 1203.
- **Result**: FALSIFIED. ret_eval=797. The asymmetric CPG creates different effective amplitudes that break the co-optimized balance.

---

## H26: Knee gear ratio scaling
- **Hypothesis**: Knee (gear=60) and hip (gear=40) have different torque ratios. Scaling knee PD by 40/60=0.67 equalizes effective torque.
- **Prediction**: ret_eval > 1203.
- **Result**: FALSIFIED. ret_eval=772. Knees need MORE torque than hips, not less. The BO already accounts for gear differences through knee_kp/knee_kd.

---

## H27: Phase-locked CPG
- **Hypothesis**: Make CPG phase advance proportional to actual hip velocity. The oscillator syncs with the body's natural frequency instead of running open-loop.
- **Prediction**: ret_eval > 1203.
- **Result**: FALSIFIED. ret_eval=880. Phase-locked CPG creates irregular oscillation that disrupts the co-optimized balance.

---

## H28: Gyro damping on all 3 axes (shared gain)
- **Hypothesis**: Add yaw damping to abdomen_z and hip_z using the existing gyro_kd gain. The BO can find a compromise gain for pitch/roll/yaw.
- **Prediction**: ret_eval > 1203.
- **Result**: FALSIFIED. ret_eval=900. Shared gain forces suboptimal compromise — optimal pitch/roll damping ≠ optimal yaw damping.

---

## H29: 2-stage iterative bootstrapping via random search
- **Hypothesis**: Use dense random search (3000 trials each) to iteratively refine x_center, then TuRBO from the refined center.
- **Prediction**: ret_eval > 1579.
- **Result**: FALSIFIED. Stage 1: 1009. Stage 2: 1145. TuRBO from stage 3 center: 1035. Random search in 16D is far less efficient than TuRBO — finds worse regions even with 10× more evaluations.

---

## H30: Fine-tune bootstrapped center x_scale
- **Hypothesis**: x_scale=0.15 might not be the optimal radius for the bootstrapped center. Try 0.10, 0.18, 0.20.
- **Prediction**: Some x_scale beats 1579.
- **Result**: FALSIFIED. x_scale=0.10: 1442. x_scale=0.18: 1089. x_scale=0.20: 1138. x_scale=0.15 confirmed as optimal. **1579 is the ceiling.**

---

## Scoreboard

| Hypothesis | Description | ret_eval | Rounds | Config |
|---|---|---|---|---|
| Baseline | Original center, x_scale=0.5 | 1157 | 10 | evaluate.sh |
| **H8** | **x_scale=0.3** | **1203** | **10** | **Best at 10 rounds** |
| H3 | Nonlinear PD (full) | 1048 | 10 | |
| H1 | Velocity feedforward | 739 | 10 | |
| H2 | Action smoothing | 875 | 10 | |
| H4 | tanh saturation | 721 | 10 | |
| H5 | Quaternion correction | 901 | 10 | |
| H6 | Yaw via abdomen_z | 832 | 10 | |
| H7 | Subspace optimization | 853 | 10 | |
| **H9** | **Bootstrapped center + x_scale=0.15** | **1579** | **30** | **All-time best** |
| H10 | Per-param x_scale | 749 | 10 | |
| H11 | Force walking | 965 | 10 | |
| H12 | Yaw via hip_z | 1062 | 10 | |
| H13 | Z-height reflex | 1024 | 10 | |
| H14 | Arm balance | 921 | 10 | |
| H15 | Online lean adaptation | 847 | 10 | |
| H16 | Arm PD tuning | 1082 | 10 | |
| H17 | 17th param yaw_kd | 796 | 10 | |
| H18 | Iterative random bootstrap | 934 | 10 | |
| H19 | Variance test | 1203 | 10 | Deterministic |
| H20 | CMA-ES | 1046 | 10 | |
| H22 | Arms/rounds tuning | 1140 | 60 | |
| H23 | Wider torso_kp | 694 | 10 | |
| H24 | Wider act_scale | 795 | 10 | |
| H25 | Asymmetric CPG | 797 | 10 | |
| H26 | Knee gear scaling | 772 | 10 | |
| H27 | Phase-locked CPG | 880 | 10 | |
| H28 | 3-axis gyro damping | 900 | 10 | |
| H29 | 2-stage random bootstrap | 1035 | 30 | |
| H30 | x_scale tuning (bootstrap) | 1442 | 30 | |

---

## Conclusions

1. **Best result**: ret_eval = **1579** (bootstrapped center + x_scale=0.15 + TuRBO + 10×30).
2. **Best at 10 rounds (evaluate.sh)**: ret_eval = **1203** (original center + x_scale=0.3).
3. **The BO is fully deterministic** (problem_seed=18, noise_seed_0=180). Zero variance across runs.
4. **Every control law modification degrades performance** because the x_center was co-optimized with the exact original CPG+PD law. ANY change—even "transparent" ones—shifts the landscape.
5. **The humanoid stands, not walks.** BO-optimal hip_y_amp=0, cpg_freq=0.02 (minimum). The reward ceiling is ~5/step × survival steps. Walking adds forward reward but reduces survival.
6. **The survival ceiling is ~300 steps** (1579/5.3 ≈ 298 steps). The humanoid falls due to accumulated yaw drift → lateral instability.
7. **ret_eval=5000 is not achievable** with this 16-param CPG+PD controller at healthy_reward=5. It would require ~800+ steps of survival, which demands a fundamentally different controller architecture (e.g., RL-trained neural network).

