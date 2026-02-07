# Agent Descent

Like grad-student descent, but with agents.

## MNIST

I asked Opus 4.6, inside Cursor's CLI [agent](https://cursor.com/blog/cli) to write a `torch` `nn.Module`, i.e., a neural network along with
an evaluation script that fit for only 5 minutes and reported the test-set accuracy.

Then I asked it to follow the Popperian approach to the scientific method
  1. Hypothesize: Write a falsifiable statement about the problem under study.
  2. Falsify: Run a test that could falsify the hypothesis.

After each hypothesis, the agent has a little more information about the problem.

Notice this: "Hallucination" is mostly irrelevant in this process. All hypotheses are welcome, as long as they can
be falsified. The system grounds itself in reality by running the evaluation. The agent doesn't even need to be
all that smart! It just needs to be disciplined and tenacious.


