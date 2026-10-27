# CFR+ equilibrium solver for Kuhn poker

After 50,000 iterations the solver reaches a game value of -0.055629 for the first player, against the
exact value of -1/18 = -0.055556 that Kuhn proved in 1950. Its remaining exploitability is 0.000956,
meaning a perfect opponent who knew the strategy in advance could win less than a hundredth of a bet
per hand from it. Three more of its decisions match the closed-form answer, which is the point of
using a game this small: you can check the solver rather than trust it.

Kuhn poker is poker with three cards and one round of betting. Each player antes one chip, is dealt
one card from a deck of jack, queen and king, and then either checks or bets a single chip. It is the
smallest game that still has bluffing in it, and it is small enough that the full set of equilibria
is known on paper.

## What the solver produces

| Situation | Check or fold | Bet or call | What theory says |
|---|---|---|---|
| Jack, first to act | 0.749 | 0.251 | bluff with probability a, any a from 0 to 1/3 |
| Queen, first to act | 0.999 | 0.001 | never bet |
| King, first to act | 0.249 | 0.751 | bet with probability 3a |
| Jack, facing a bet | 1.000 | 0.000 | always fold |
| Queen, facing a bet | 0.662 | 0.338 | call one time in three |
| King, facing a bet | 0.000 | 1.000 | always call |
| Queen, checked then faced a bet | 0.414 | 0.586 | call with probability a + 1/3 |

The solver settled on a = 0.251. Three of those rows then check themselves: the king bet rate should
be three times the bluff rate, and 3 x 0.251 = 0.753 against a measured 0.751. The queen facing a bet
should call a third of the time, and it calls 0.338. The queen that checked and then faced a bet
should call with probability a + 1/3 = 0.584, and it calls 0.586. None of those three numbers was put
in by hand; they come out of self-play.

![Exploitability falling against iteration count on a logarithmic scale](results/convergence.png)

## How it works

**The game.** `kuhn.py` holds the rules, the payouts at the end of each line of play, and the keys
that identify what a player knows at each point. There are twelve of those: three cards times four
betting situations a player can face.

**The solver.** `cfr.py` walks the whole game tree once per iteration, for all six ways the cards can
be dealt. At each decision it works out how much it regrets not having played each action, then plays
the next iteration in proportion to the regret it has accumulated. CFR+ adds one change to the
original method: regrets are never allowed to stay negative, which makes them recover faster when a
line that looked bad becomes good. Tammelin showed this converges considerably faster, and it is the
version used here because the plain method needs roughly an order of magnitude more iterations to
reach the same exploitability on this game.

The strategy that converges is the average over all iterations, not the strategy from the last one.
That distinction matters and it catches people out: the final iteration's strategy keeps oscillating,
while the running average settles.

**Checking the answer.** Exploitability is computed by working out the best possible counter-strategy
against the solver's average strategy, for each player in turn, and seeing how much it wins. At a true
equilibrium that number is zero, so whatever is left measures how far from equilibrium the solver
stopped. Reporting it is the whole reason to prefer a tiny game: on a game the size of real poker,
this quantity cannot be computed at all, and you are left taking convergence on faith.

![Strategy at each of the twelve information sets](results/strategy_heatmap.png)

## Running it

```bash
pip install -r requirements.txt
python solve.py
```

Takes about a minute and writes four figures to `results/`. Every number in this README came from that
command.

## What this is not

Kuhn poker has three cards. Real poker has 1,326 starting hands and four betting rounds, and solving
it needs card abstraction, sampling instead of full tree walks, and far more compute. Nothing here
scales to that without being rewritten. What it does show is the method working where the answer is
known, which is the right place to start and the only place you can check yourself.

Equilibrium play is also not the same as winning play. An equilibrium strategy cannot be beaten in the
long run, and it does not punish a bad opponent either: it plays the same way against someone who
folds too often as against someone who never folds. Exploiting a specific opponent is a different
problem that needs a model of that opponent.

## References

1. Kuhn, H. W. (1950). Simplified two-person poker. *Contributions to the Theory of Games*, 1, 97-103.
2. Zinkevich, M., Johanson, M., Bowling, M. & Piccione, C. (2007). Regret minimization in games with
   incomplete information. *Advances in Neural Information Processing Systems*.
3. Tammelin, O. (2014). Solving large imperfect information games using CFR+. *arXiv:1407.5042*.
4. Neller, T. W. & Lanctot, M. (2013). An introduction to counterfactual regret minimization.

## License

MIT License. See [LICENSE](LICENSE).
