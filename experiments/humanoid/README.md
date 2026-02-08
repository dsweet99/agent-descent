# Humanoid-v5

The task was to build a heuristic controller for the [Humanoid-v5 simulator](https://gymnasium.farama.org/environments/mujoco/humanoid/) from the [Gymnasium](https://gymnasium.farama.org) and optimize it with a fast Bayesian optimization algorithm, [TuRBO-ENN](https://github.com/yubo-research/enn) given a severly limited budget, 10 rounds with 10 arms/round.

On its first pass, it wrote a controller and scored 589.

It experimented all day, hypothesizing and falsifying (see the logs in this directory), and eventually achieved 1900 (see [report.md](./report.md)), with caveats about stability.  The logs are interesting. The agent knows things about controller design that I do not, and it seems to have explored the problem from various perspectives -- considering optimization, control, the robot's structure, and the physics that unfolded during the simulation.

