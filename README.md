# 51% Attack Simulator

I built this interactive simulation of a **51% attack** to understand the concept and make it visual so that I can actually _see_ how the attack plays out. Sharing it for anyone else curious to go deeper into how the attack works.

You can set the attacker's hashpower, the number of confirmations the exchange waits, and how patient the attacker is, then watch the honest chain and the attacker's secret fork race block by block and see the estimated odds of success update live. The notes below explain the mechanics, the math behind the odds (the Nakamoto double-spend probability), and the questions that tend to come up while playing with it.

> Built with the help of Claude (Anthropic); I directed the design, decided what to model, and verified the logic and the math. The simulations are deliberately simplified teaching tools, not production-grade models. See the notes for exactly which parts are rigorous and which are illustrative.

[![51% attack simulator demo](assets/51_attack.gif)](https://aleksandra1104.github.io/crypto-attack-visualized/)


## How the attack odds work (Nakamoto double-spend probability)

The 51% simulator's "estimated attack success" number is grounded in the **Nakamoto double-spend probability** — the formula from the original 2008 Bitcoin whitepaper that answers one question:

> If an attacker controls fraction `q` of the mining power, and the merchant waits for `z` **confirmations** before trusting a payment, what is the probability the attacker can secretly build a longer chain and reverse that payment?

A **confirmation** is just one block stacked on top of the block that contains your transaction. 1 confirmation = your transaction is in the latest block; 6 confirmations = five more blocks have been mined on top of it. The deeper it's buried, the more work an attacker must redo to rewrite it, so more confirmations means more safety.

#### The race it models

The attacker pays the merchant, then secretly mines a **private fork** that omits the payment, trying to out-grow the public chain. By the time the merchant has counted `z` confirmations, the honest chain is `z` blocks ahead, so the attacker, mining with only their share of the power, has to catch up and overtake from behind.

The formula combines two pieces:

1. **A random head start.** By the time the honest chain reaches `z`, the attacker has already secretly mined some random number of blocks. How many is a lottery, so it follows a Poisson distribution.
2. **Gambler's ruin.** From whatever gap of `k` blocks remains, the chance of ever closing it is `(q/p)^k`, where `p = 1 − q` is the honest share. Note this is an **exponent** — `q/p` multiplied by itself `k` times, not `(q/p) × k`.

The closed form (for a minority attacker, `q < p`):

```
λ = z · (q / p)

P(success) = 1 − Σ (k = 0 … z)  [ e^(−λ) · λ^k / k! ] · [ 1 − (q/p)^(z−k) ]

P(success) = 1            when q ≥ p   (a majority attacker eventually always wins)
```
#### Reading the formula

- **`λ = z · (q/p)`** is the attacker's _expected_ number of secret blocks by the time the honest chain reaches `z`. (The honest side did `z` blocks; the attacker mines at relative rate `q/p`, so on average it gets `z·(q/p)`.) 
    
- **`e^(−λ) · λ^k / k!`** is the **Poisson probability** of the attacker having mined _exactly_ `k` secret blocks, given the average is `λ`. Poisson is the standard formula for "how many independent, rare events happened at a known average rate" — exactly the situation for blocks found by mining. `λ^k` lifts the likely count toward the average, `k!` (factorial) crushes counts far above it, and `e^(−λ)` (e ≈ 2.718) is a fixed scaling factor that makes all the probabilities sum to 1. The result is a hump centred on `λ`.
    
- **`Σ (k = 0 … z)`** means "sum over every possible head start." We don't know how many secret blocks `k` the attacker actually got, so we walk through each possibility, weight it by its Poisson likelihood, multiply by the chance it _still_ fails to catch up — `[ 1 − (q/p)^(z−k) ]` — and add them all up. That total is the probability the attacker **loses**.
    
- **`P(success) = 1 − (that sum)`** because success is simply "not failure."
    

In one sentence: _for every secret head start the attacker might randomly have (Poisson), multiply by the chance it can't close the remaining gap (gambler's ruin), sum over all of them, and subtract from 1._

> Why Poisson? It's the limiting case of repeated coin-flips (the binomial distribution) when there are very many opportunities, each individually unlikely, with a fixed average — which is exactly how blocks are found. The `e^(−λ)` is the mathematical fingerprint of that limit: the chance that "no block happened" across countless tiny moments.
#### Why each confirmation matters so much

Because `(q/p)^k` is an exponent, it collapses fast. For a 10% attacker (`q/p = 1/9`), the chance of closing a gap of `k` blocks:

|gap `k`|`(1/9)^k`|≈|
|---|---|---|
|1|1/9|0.111|
|2|1/81|0.0123|
|3|1/729|0.00137|
|6|1/531,441|0.0000019|
|10|1/3,486,784,401|0.0000000003|

Each extra block of depth multiplies the difficulty — independent unlikely events compound by _multiplying_, not adding. That exponential decay is the whole reason confirmations buy so much safety.

Two key takeaways the formula reveals:

- **Below 50% hashpower:** success shrinks exponentially as `z` grows. At `z = 1` it's a real, non-trivial number; by `z = 6` it's small; by `z = 30` it's vanishing — but never _exactly_ zero for finite `z`. This is why a 35% attacker can occasionally win against a low confirmation count (see the FAQ), and why exchanges wait for many confirmations.
- **At or above 50%:** the probability is `1`. Given enough time the attacker always catches up, no matter how long you wait. This is the real meaning of "you need 51%."

#### How the simulation uses it

The simulator doesn't plug numbers into the closed form — it **runs the race directly**. Each block is a single biased coin-flip: with probability `q` (the hashpower slider) the attacker mines the next block, otherwise the honest network does. The merchant "credits" the payment once the public chain reaches `z` confirmations (the confirmations slider). The attacker wins if its private fork ever gets longer than the public chain after that, and gives up if it falls `FAIL_GAP` blocks behind.

The "estimated attack success" figure is a **Monte-Carlo estimate**: it replays that exact race thousands of times and reports the fraction the attacker won. Because it uses the same rules, the number always matches the visible behavior — and it closely tracks the Nakamoto closed form above. (It runs slightly _below_ the theoretical curve, because the closed form assumes an infinitely patient attacker, while the simulation's attacker gives up once it's `FAIL_GAP` blocks behind.)

```js
const FAIL_GAP = 8;

// Monte-Carlo estimate using the exact simulation rules.
// q = attacker hashpower (0–1), conf = confirmations the exchange waits.
function estimateOdds(q, conf, trials) {
  let wins = 0;
  for (let n = 0; n < trials; n++) {
    let pub = 0, att = 0, credited = false;
    for (let i = 0; i < 6000; i++) {
      if (Math.random() < q) att++;        // attacker mines a block
      else                   pub++;        // honest network mines a block
      if (!credited && pub >= conf) credited = true;   // exchange releases funds
      if (credited && att > pub) { wins++; break; }     // attacker's fork overtakes → success
      if (credited && (pub - att) >= FAIL_GAP) break;   // hopelessly behind → attacker gives up
    }
  }
  return wins / trials;                     // fraction of runs the attacker won
}
```

So: `q` is the hashpower slider, `conf` is the confirmations slider, and the percentage on screen is `estimateOdds(q, conf, …)` — a live, honest measurement of the Nakamoto double-spend probability for the current settings.

---

### Formula vs. Monte Carlo: do they give the same result?

There are two ways to compute the same number, and they could in principle disagree, so it's worth being precise:

- **The Nakamoto formula** — the closed-form equation above. You feed in `q` and `z`, evaluate once, and get the exact theoretical probability. Same answer every time.
- **Monte Carlo** (`estimateOdds`) — _play the attack out_ thousands of times with real randomness and count how often the attacker wins. No equation; an empirical estimate.

Here is the closed form written as code:

```js
// Nakamoto double-spend probability (Bitcoin whitepaper).
// q = attacker hashpower fraction, z = confirmations.
function nakamotoProb(q, z) {
  if (q >= 0.5) return 1;                 // a majority attacker eventually always wins
  const p = 1 - q, r = q / p, lambda = z * r;
  let pois = Math.exp(-lambda), sum = 0;
  for (let k = 0; k <= z; k++) {
    if (k > 0) pois *= lambda / k;        // Poisson term  e^-λ · λ^k / k!
    sum += pois * (1 - Math.pow(r, z - k));
  }
  return 1 - sum;
}
```

#### They track closely, but they are not identical

Two reasons they differ:

1. **Sampling wobble (harmless).** Monte Carlo is a finite sample, so it jitters a little run to run (e.g. 20.1% then 19.7%). This noise is centered on the true value and shrinks as you add trials.
    
2. **A systematic gap (the real one).** Even with infinite trials the Monte-Carlo value settles slightly _below_ the formula, because the two measure slightly different games:
    
    - The **formula** assumes an **infinitely patient** attacker who never quits — it still counts the rare runs where the attacker falls far behind and claws all the way back.
    - The **simulation** has the attacker **give up once it is `FAIL_GAP` (8) blocks behind** — so those far-behind comebacks are written off as losses.
    
    The formula's scenario therefore includes extra winning paths the simulation excludes, so the formula always reads a bit higher.
    

#### Side-by-side (z = 6 confirmations)

|Attacker hashpower|Nakamoto formula|Simulation (Monte Carlo)|
|---|---|---|
|25%|~5%|~3%|
|35%|~28%|~20%|
|45%|~77%|~58%|

They agree on the **shape** (minority attackers fade fast as confirmations rise) and the **big conclusions** (a majority attacker is certain), but the numbers aren't equal — and the gap widens as the attacker nears 50%, because that's exactly where "infinite patience" buys the most extra comeback wins (a near-even random walk wanders far and long before it resolves, so cutting it off at 8 blocks removes more winning runs).

The divergence is a **modeling choice, not an error**: raising `FAIL_GAP` makes the simulated attacker more patient and pushes the Monte-Carlo estimate up toward the formula. The simulation deliberately models a realistic attacker who abandons a hopeless fork, so its number sits just under the idealized theoretical ceiling.

**Two things worth knowing about why this simplification is fair, and where it cuts corners:**

It captures the part that matters — that block production is probabilistic and proportional to hashpower, which is the entire basis of the 51% threshold. With q below 0.5 the coin favors the honest side, so the attacker drifts behind and usually loses; at q above 0.5 the coin favors the attacker, so they eventually pull ahead. That threshold behavior falls straight out of the weighted coin.

What it abstracts away: real mining doesn't take discrete turns — both sides mine simultaneously and continuously, block times are random intervals (averaging ~10 minutes on Bitcoin) rather than neat ticks, and there's network latency and the occasional natural tie. The coin-flip model collapses all of that into "one block, one weighted flip, next." For understanding the double-spend race and reproducing the Nakamoto odds, that's a faithful simplification; it just isn't a cycle-accurate mining emulator, which is the kind of honest caveat worth keeping in the README.

### What does the "confirmations" parameter do?

It's **the number of blocks the exchange waits before releasing the funds** — the exchange's safety threshold. In the simulation it gates exactly one thing: _when the exchange pays out._

The exchange credits the attacker only once the **public (honest) chain** reaches the confirmation count, and the attacker can only **win after that payout has happened**:

```js
if (!credited && publicChain.length >= conf) credited = true;          // exchange releases funds
if (credited && attackerChain.length > publicChain.length) success;     // win requires BOTH
```

So a win needs **both**: the exchange must have already paid out (public chain reached `conf`) **and** the attacker's fork must be strictly longer than the public chain.

#### Worked example: confirmations = 6, public chain = 3, attacker = 4

Is the attacker ahead? `4 > 3` → yes. So has it won? **No** — because the exchange hasn't paid out yet: the public chain is only at 3 and the exchange waits for 6, so `credited` is still false and the win condition isn't even checked.

And this is _correct_, not a technicality: **there's nothing to steal yet.** The whole point of a double-spend is to let the exchange hand over real value and _then_ reverse the deposit so the attacker keeps both. If the exchange hasn't paid out, reversing the deposit gains nothing. So being ahead 4-to-3 this early is real but useless.

A smart attacker therefore **waits** — it holds its longer fork private, lets the public chain grow to 6 so the exchange pays out, and only then reveals, needing to be longer than whatever the public chain has reached _by that moment_ (now at least 6, not 3). The simulation models exactly this: success is checked only after `credited`, against the public chain's _current_ length.

#### Why higher confirmations make attacks harder

- **Low (e.g. 1):** the exchange pays out almost immediately, so the attacker only has to nudge ahead very early, when the race is short and variance can easily hand it the lead. Easy.
- **High (e.g. 6–12):** the attacker must stay ahead of a public chain that has already grown much longer before the payout even unlocks — a longer race, giving a sub-majority attacker's negative drift more time to drag it behind. Hard.

This is the safety dial: the confirmation count sets how deep the deposit must be buried before the exchange trusts it, which sets how much sustained out-mining the attacker must achieve to overturn it. It's the same `z` as in the Nakamoto formula — which is why turning the confirmations slider up makes the "estimated attack success" figure fall.
### Why does a 51% attacker sometimes fail?

Because **51% guarantees success only with unlimited patience and time — not victory in any single run.** A run that ends with the honest chain ahead (e.g. honest 16, attacker 8, lead +8) is a normal unlucky outcome, not a bug.

Two things are going on:

**1. The run hit the give-up line.** Each block is awarded by a weighted coin flip. When the honest chain pulls ahead by your **Attacker patience** setting (the lead reached +8 in the example), the attacker abandons the fork. So the run didn't fail because 51% is "too little" — it failed because _this particular race_ drifted against the attacker far enough to trip that cutoff.

**2. 51% is almost a fair coin.** At 51% the attacker wins each block only 51% of the time vs the honest network's 49% — a razor-thin 1% edge. With an edge that small the _drift_ toward the attacker is tiny but the _variance_ is huge: the lead between the two chains is a random walk that wanders widely and, over a short stretch, easily wanders the wrong way. In the example the coin simply came up honest more often early (8 vs 16 over 24 blocks), the lead ballooned to +8, and the attacker gave up before it could pull ahead.

**The casino analogy.** "51% always wins" is a statement about averages over _infinite_ time, like a casino's house edge. The house wins in the long run on a tiny per-bet advantage, but loses individual hands constantly. A 51% attacker is the house: it wins _most_ attempts and loses plenty of individual ones — especially short ones cut off early.

This is the mirror image of "why does a 35% attacker sometimes _win_?" — a minority attacker has a small-but-real chance, and a majority attacker has a large-but-not-certain one. Neither is guaranteed in a single run.

**See it yourself:**

- Raise the **Attacker patience** slider (to ~25–30) and re-run at 51%. The attacker no longer quits at −8, so it usually claws back and overtakes given enough blocks. Low patience kills runs a more patient attacker would eventually win.
- Run several times at 51% without changing anything: it wins most runs and loses some. The win _rate_ reflects the 1% edge; no single outcome does. (This is also why the "estimated attack success" for 51% is below 100% — a finite, impatient attacker on a near-fair coin sometimes loses.)
