# Experiment Log: human-heur optimization toward ret_eval=5000

## Problem Statement
Get `ret_eval` for the `human-heur` (Humanoid-v5 heuristic controller) up to 5000
using `evaluate.sh` (10 rounds, 10 arms of TuRBO-ENN Bayesian optimization).

The Humanoid-v5 reward = healthy(+5/step if z in [1.0,2.0]) + 1.25*x_vel - 0.1*sum(a²).
5000 ≈ surviving all 1000 steps with modest forward velocity.

## Baseline
- Initial controller (CPG + PD, 16 params): ret_eval = 589 after BO
- Default params give ~92 steps, then humanoid falls

## Key findings from diagnostics
- Humanoid weighs 42 kg, actuator gears: hip_y=300, knee=200, abdomen=100, arms=25
- Actuators are powerful enough (hip_y max = 120 N·m), but balance requires precise coordination
- The humanoid falls due to growing pitch oscillation (forward→backward→forward)
- Zero-action policy: 40 steps. PD-only: ~130 steps. Best heuristic: ~200-250 steps

---

## Hypothesis Log

### H1: Standing still (zero CPG amplitudes) survives longer (CONFIRMED)
- **Predict**: Zero CPG → longer episodes
- **Result**: 138 steps (vs 92 with walking). Confirmed but insufficient.

### H2: Torso PD gain scan (CONFIRMED: low gains better)
- **Predict**: Optimal torso gains exist
- **Result**: Low torso_kp (-1.0) best at 631 return. High gains destabilize.

### H3: Quaternion deviation grows before fall (CONFIRMED)
- **Predict**: Pitch grows monotonically
- **Result**: qy goes from 0 to 0.32 over 120 steps, then falls. Pitch is the primary failure mode.

### H4: Balance feedback (quat+gyro on abdomen+hips) improves balance (FALSIFIED)
- **Predict**: Adding global orientation feedback → longer episodes
- **Result**: 428 (worse than 589 baseline). Over-correction.

### H5: Balance gains too high (CONFIRMED)
- **Predict**: Scanning balance gains will show optimal at low values
- **Result**: Best with near-zero balance gains. Any significant balance feedback hurts.

### H6: V1 structure with small balance overlay (FALSIFIED)
- **Predict**: Careful balance overlay on proven structure → improvement
- **Result**: 394 (worse). The restructured controller changed too many defaults.

### H7: Zero action baseline + random search (CONFIRMED: 698 best of 100 random)
- **Predict**: Random search finds good params
- **Result**: Best of 100 random = 698 (158 steps). Zero action = 40 steps.

### H8: Large random search finds 300+ steps (PARTIALLY CONFIRMED)
- **Predict**: 500 trials → some exceed 300 steps
- **Result**: Best = 963/201 steps. 0 exceeded 300 steps. Structural ceiling at ~200 steps.

### H9: Hip balance correction sign is inverted (FALSIFIED)
- **Predict**: Flipping hip correction sign → better balance
- **Result**: Flipped sign = 577 (worse than original 683). Original sign was correct.

### H10: Updated x_center from best random improves BO (CONFIRMED)
- **Predict**: Centering BO around best known point → higher ret_eval
- **Result**: 1040 (up from 886 and 735). Significant improvement.

### H11: x_scale=0.5 gives BO more exploration (CONFIRMED)
- **Predict**: Wider exploration range → better BO result
- **Result**: 886 (up from 735 with x_scale=0.35).

### H12: Torso_kp and lean scan (CONFIRMED: high tkp, zero lean)
- **Predict**: Optimal torso_kp/lean exists
- **Result**: Best at tkp=1.0, lean=0.0, bal_kp=-1.0 (zero balance). torso_kp~3.9.

### H13: Wider torso_kp range helps (PARTIAL)
- **Predict**: Extending torso_kp range allows higher gains
- **Result**: 839 from 500 random (lower than before due to changed x_center).

### H14: 2000-trial random search (CONFIRMED: 1192/244 steps best)
- **Predict**: More trials → better starting point
- **Result**: Best = 1192/244 steps (seed-specific, 446-714 on other seeds).
  Key: torso_kp≈7.0, backward lean=-0.07, zero walking, act_scale≈0.62.

### H15: BO from H14 best center (CONFIRMED: 1040)
- **Predict**: Centering on best params → BO improvement
- **Result**: ret_eval = 1040. Good improvement from baseline.

### H16: Very high torso_kp (FALSIFIED)
- **Predict**: Extending torso_kp to 20+ → longer balance
- **Result**: torso_kp=20 gives 172 steps (worse than optimal ~7).

### H17: Root angular velocity damping scan (INCONCLUSIVE)
- **Predict**: Adding gyro to actions → less oscillation
- **Result**: All results identical (780). The monkey-patching had issues.

### H18: Gyro damping on abdomen (CONFIRMED: best result 1157)
- **Predict**: Root angular velocity damping on torso → less pitch oscillation
- **Result**: ret_eval = 1157! Best result achieved. Consistent across runs.

### H19: 2000-trial search with gyro controller (CONFIRMED: BO outperforms)
- **Predict**: Random search with gyro structure → better center
- **Result**: Best random = 943 (worse than BO's 1157). BO is effectively exploiting the gyro.

### H20: Gyro sign investigation (CONFIRMED: original sign better)
- **Predict**: Flipping gyro sign → improvement
- **Result**: Flipped = 537 (much worse than original 828). Original sign correct.
  Lower gyro_kd values are better (0.2 optimal vs 0.7 default).

### H21: Lower gyro_kd center (NEUTRAL)
- **Predict**: Centering gyro at lower value → BO finds better optimum
- **Result**: 1100 (comparable to 1157, slightly worse).

### H22: No-CPG balance controller (FALSIFIED)
- **Predict**: Removing CPG, adding quat feedback + height control → better
- **Result**: 544 (much worse). Simpler ≠ better; CPG slots give BO useful flexibility.

### H23: Centroid of top 30 as x_center (PARTIAL)
- **Predict**: Robust center → better BO result
- **Result**: 748 (worse than single-best center's 1157). Averaging dilutes best features.

### H24: BO from centroid center (FALSIFIED)
- **Predict**: Centroid center → BO improvement
- **Result**: 849 (worse than 1157).

### H25: 5000-trial multi-seed search (CONFIRMED: 800 mean)
- **Predict**: Multi-seed robust search → good center
- **Result**: Best multi-seed mean = 800. Single best = 972.
  Updated center but BO only reached 849.

### H26: Hip gyro damping (gear=300, 3x power) (FALSIFIED)
- **Predict**: Hip gyro → more powerful balance correction
- **Result**: 744 (worse than 1157). Hip gyro sign/gain issues.

### H27: 8-param controller (fewer params → better BO) (FALSIFIED)
- **Predict**: 8 params → BO explores more per round → better result
- **Result**: 778 (worse than 1157). Controller too simple.

### H28: Walking-oriented CPG defaults (CONFIRMED: comparable)
- **Predict**: Non-zero walking amplitudes → dynamic stability
- **Result**: 1142 (comparable to 1157). Walking might help but BO not finding it.

### H29: Forward velocity damping (speed governor) (FALSIFIED)
- **Predict**: Braking when vx too high → prevents fall from forward momentum
- **Result**: 767 (worse than 1157). Speed governor reduces beneficial forward motion.

### H30: PID control (integral term on torso) (FALSIFIED)
- **Predict**: Integral eliminates steady-state pitch error → no drift
- **Result**: 631 (much worse). PID too complex to tune with limited BO budget.

---

## Final Result
**Best ret_eval: 1157** (from H18: CPG + PD + gyro damping on abdomen)

### Why 5000 was not reached
The 16-param heuristic controller with 10 rounds of BO has a structural ceiling
around 200-250 steps (~1000-1200 return). The MuJoCo humanoid is an inherently
unstable 3D inverted pendulum. Key limitations:

1. **Balance horizon**: PD + gyro damping can stabilize for ~200 steps but growing
   oscillations eventually cause falling. True 1000-step stability likely requires
   full-state feedback (all 348 obs) with complex nonlinear coordination.

2. **Optimization budget**: 10 rounds × 10 arms = 100 evaluations for 16 parameters
   is insufficient to deeply explore the loss landscape. RL agents typically use
   millions of steps.

3. **Controller expressiveness**: Linear PD control can't represent the complex
   coordination patterns that RL neural networks learn. The actuator gears make
   the problem highly asymmetric across joints.

### Key insights for improvement
- Torso stiffness (high torso_kp ≈ 7) is critical
- Root angular velocity damping on abdomen helps
- Near-zero CPG amplitudes are optimal (standing > walking for this controller)
- Low joint damping (hip_kd ≈ 0.05) allows faster response
- Moderate action scaling (act_scale ≈ 0.6) prevents saturation
- x_center tuning is as important as controller structure
