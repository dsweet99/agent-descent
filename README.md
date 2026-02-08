# Agent Descent

Like grad-student descent, but with agents.

## 3-Minute MNIST

I asked Opus 4.6, inside Cursor's CLI [agent](https://cursor.com/blog/cli) to write a `torch` `nn.Module`, i.e., a neural network along with
an evaluation script that fit for only 3 minutes and reported the test-set accuracy.

Then I asked it to follow the Popperian approach (KPop) to the scientific method
  1. Hypothesize: Write a falsifiable statement about the problem under study.
  2. Falsify: Run a test that could falsify the hypothesis.


### NOTES
- After each hypothesis, the agent has a little more information about the problem.
- "Hallucination" is mostly irrelevant in this process. All hypotheses are welcome, as long as they can
be falsified. The system grounds itself in reality by running the evaluation. The agent doesn't even need to be
all that smart! It just needs to be disciplined and tenacious.
- Restricting the fitting time speeds up evaluation *and* makes it clear that fitting is done. Compare this to "fit until converged" (which is indeterminately long and hard to even define). Also, if you can get the loss, in general wouldn't you rather get done in less time?


```bash
export PYTHONPATH=${PWD}
python experiments/mnist/fit_mnist.py
```

You should see something like:
```
epoch 1/4  train_loss=0.4003  test_acc=0.9785
epoch 2/4  train_loss=0.0643  test_acc=0.9867
epoch 3/4  train_loss=0.0280  test_acc=0.9907
timeout after 180s (during epoch 4)
final test_acc=0.9917
```
That's on my old Intel(R) Xeon(R) CPU E5-2695 v2 @ 2.40GHz, so YMMV.

In fact, I get better results on my Macbook Air (still CPU, not GPU):
```
epoch 1/4  train_loss=0.4049  test_acc=0.9634
epoch 2/4  train_loss=0.0640  test_acc=0.9859
epoch 3/4  train_loss=0.0286  test_acc=0.9910
epoch 4/4  train_loss=0.0124  test_acc=0.9914
final test_acc=0.9914
```

See [exp_log.md](experiments/mnist/exp_log.md) and [exp_log_2.md](experiments/mnist/exp_log_2.md) for all hypotheses and falsification results.  The agent reports:

## Key Takeaways

1. **Architecture matters most**: MLP -> CNN was the biggest single jump (0.981 -> 0.989).
2. **Speed is accuracy on a budget**: On CPU with a time limit, anything that slows epochs (deeper nets, torch.compile overhead, data augmentation) hurts even if it would help given unlimited time.
3. **Diminishing returns on hyperparams**: Once architecture and LR schedule are right, tuning dropout/label smoothing/weight decay yielded <0.1% changes.
4. **OneCycleLR schedule mismatch helps**: Configuring OneCycleLR for more epochs than actually complete keeps the LR from decaying too fast.


## Low-Budget Humaniod-v5

The idea here is to construct a controller in simulation that could be tuned with only a few rounds of experimental Bayesian optimization. We don't expect our simulations to match reality. We just hope that they're close enough to (i) put us in the ballpark of reality, and (ii) let us know whether the controller is tunable under a reasonable variety of conditions.

I gave Opus [TuRBO-ENN](https://github.com/yubo-research/enn) as a CLI tool to optimize a heuristic controller of its design. It followed the "KPop" approach (described above) and found several ways to improve its baseline controller while developing some understanding of what mattered and what didn't. Read [more](experiments/humanoid/README.md).
