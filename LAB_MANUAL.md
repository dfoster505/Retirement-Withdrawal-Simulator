# Retirement Withdrawal Simulator — Lab Manual

**A self-paced build.** You write all the code. This document gives you the concepts, the tasks, and a checkpoint after each lab so you can tell whether what you built is correct.

---

## What you are building

A simulator that answers one question:

> Given a retiree who withdraws a fixed inflation-adjusted amount each year, what fraction of possible 30-year futures does their portfolio survive — and how much does that depend on *when* the money comes out and *whether* they hold a cash buffer?

**Primary metric:** success rate over a fixed 30-year horizon. **Success:** balance \> 0 in real terms at the end of month 360, measured after the final withdrawal. **Failure:** any month where the required withdrawal cannot be fully funded. Record the shortfall amount; keep the path running.

### Locked design decisions

| Decision | Setting |
| :---- | :---- |
| Time step | Monthly (360 months) |
| Horizon | Fixed 30 years, no longevity distribution |
| Units | Real (inflation-adjusted) throughout |
| Withdrawal cadences | Annual (month 0 of each year) and monthly (1/12 each month) |
| Ordering | Rebalance first, then withdraw |
| Rebalancing interval | Annual (plus a monthly-rebalancing control arm) |
| Sell-from rule | Hybrid: cash buffer first (when triggered), then pro-rata from the invested sleeve |
| Buffer location | Separate account, **outside** the rebalanced portfolio |

---

## Prerequisites

You need working Python — pandas, numpy, matplotlib. You do **not** need prior finance knowledge; each lab introduces what it needs.

Budget roughly 20–30 hours across all ten labs.

---

## Lab 0 — Environment and repo

### Objective

Get a repo you can edit and a Colab notebook that imports from it.

### Background

Colab wipes its filesystem between sessions. If you write code in notebook cells, you will lose the structure that makes this a portfolio piece. So: **code lives in a GitHub repo; the notebook is only a driver.**

`%autoreload 2` re-imports your modules whenever a file changes, so you can edit a `.py` file and re-run a cell without restarting the runtime.

### Tasks

1. Create a public GitHub repo with this skeleton:

retirement-sim/

├── src/

│   ├── data.py         \# loading and cleaning raw series

│   ├── engines.py      \# generating return paths

│   ├── strategies.py   \# withdrawal \+ buffer rules

│   ├── scenarios.py    \# named economic scenarios

│   ├── metrics.py      \# success rate, shortfall stats

│   └── viz.py

├── tests/

├── notebooks/

├── config/             \# scenarios as YAML

└── README.md

2. In Colab, before any other import:

%load\_ext autoreload

%autoreload 2

3. Clone the repo, add `src/` to the path, import a trivial function, confirm it works.  
     
4. Mount Google Drive for a cache directory. This is where downloaded data and simulation results go so you are not re-downloading every session.

### Checkpoint

Edit a function in `src/data.py` on GitHub, `git pull` in Colab, re-run the calling cell, and see the new behavior without restarting the runtime.

### Common failure modes

- Running `%load_ext autoreload` *after* your first import — it won't take effect. It must come first.  
- `autoreload` does not reliably pick up **new files** or changed **dataclass definitions**. When something stops making sense, restart the runtime rather than debugging a stale module.  
- If you make the repo private, store a fine-grained personal access token in Colab Secrets and read it with `google.colab.userdata.get()`. Never paste a token into a cell.

---

## Lab 1 — Data

### Objective

Build a clean monthly panel of real returns from 1871 to present.

### Background

**Real vs. nominal.** A nominal return includes inflation; a real return strips it out. A retiree who needs $40,000/year needs $40,000 of *purchasing power*, which grows in nominal dollars. Doing the whole simulation in real terms means the withdrawal is a constant number and inflation is handled once, at the data layer.

Mixing real and nominal is the single most common bug in this class of model. Convert at load time and never think about it again.

**Why not use a bond index?** You want to model rate changes as a *mechanism*, not a label. Bond total return decomposes as:

return ≈ starting\_yield − duration × Δyield \+ convexity\_term

So when rates rise, you get a capital loss proportional to duration, offset over time by higher reinvestment income. Building bond returns this way lets you *shock rates* in Lab 6 and see what happens. An index would just be a black box.

### Data sources

| Series | Source | Notes |
| :---- | :---- | :---- |
| S\&P price, dividends, CPI, long rate (monthly, 1871–) | `shillerdata.com` → `ie_data.xls` | Primary driver. Shiller's data moved here from the old Yale URL. |
| Annual stock/bond/bill returns (1928–) | Damodaran, NYU Stern (`histretSP.html`) | Calibration and sanity-checking only — it's annual, so it can't drive a monthly engine. |
| CPI, 10y yield, 3m bill, recession flag | FRED: `CPIAUCSL`, `DGS10`, `TB3MS`, `USREC` | For scenario definitions and cross-checks. |

### Tasks

1. Download `ie_data.xls`, parse it, cache to Drive as parquet. It has header junk and footnote rows at the bottom — strip them.  
2. Build monthly **real total return** for equities. Total return means price change *plus* reinvested dividends. Shiller gives you monthly price and an annualized dividend figure — you need to convert the dividend to a monthly amount before adding it.  
3. Build monthly **real bond return** from the long rate using the duration formula above. Assume a constant-maturity 10-year bond; pick a duration around 8 and document the choice.  
4. Build monthly **real cash return** from the 3-month bill rate.  
5. Write a data dictionary in your README: every column, its units, its source, its date range.

### Checkpoint

Compare your equity series' annualized real return and volatility over 1928–present against Damodaran's published figures. You should land close to a long-run real return in the high single digits with volatility near 20% annualized. If you're materially off, you probably forgot dividends (too low) or double-counted inflation (too low or negative).

### Common failure modes

- Shiller's dividend column is an **annual rate**, not a monthly payment.  
- Shiller's date format is `1871.01`, `1871.02`, … `1871.10` — where `.1` means October, not January. Parse carefully.  
- Forgetting that the last few rows may be provisional or incomplete.

---

## Lab 2 — One path, by hand

### Objective

Build the hero chart before you build the engine.

### Background

**Sequence-of-returns risk** is the core idea of the whole project. Two portfolios can experience the *exact same set of returns* in a different order, ending with identical average returns — and one survives 30 years while the other dies in 15\.

This happens because withdrawals interact with returns. When you sell into a decline, you liquidate more shares to raise the same dollars, and those shares aren't there for the recovery. Losses early in retirement are therefore far more damaging than identical losses late in retirement. Without withdrawals, order doesn't matter at all. With withdrawals, it dominates.

### Tasks

1. Take a real historical 30-year monthly return series.  
2. Run a single portfolio: $1,000,000 start, 4% real withdrawal, 60/40 allocation, annual withdrawal at month 0, annual rebalancing.  
3. Now **reverse the return series** and run it again.  
4. Plot both balance paths on one chart.

### Checkpoint

Both runs must have identical arithmetic mean monthly return (verify this numerically — it's a good assertion). The balance paths should diverge dramatically. If they don't diverge much, pick a starting period with a worse early drawdown; 1966 and 2000 are good candidates.

Also run both with **zero withdrawal**. The terminal balances should now be nearly identical. That contrast — order matters with withdrawals, doesn't without — is the cleanest way to explain sequence risk, and it's your best chart.

---

## Lab 3 — Historical cohorts

### Objective

Reproduce the classic result and learn why it's less certain than it looks.

### Background

The **4% rule** comes from Bengen (1994), who tested fixed inflation-adjusted withdrawals against every historical 30-year window and found \~4% survived all of them. The **Trinity Study** (Cooley, Hubbard & Walz, 1998\) extended this across allocations and horizons. Both are worth reading — they're short and non-technical.

A **rolling cohort** analysis runs one simulation per possible start date. With monthly data from 1871, you get roughly 1,500 overlapping 30-year windows.

Here's the catch: those windows are **not independent**. A cohort starting January 1970 shares 359 of its 360 months with one starting February 1970\. You have about *five* non-overlapping 30-year periods in the entire dataset. So "96% of historical cohorts succeeded" is not a 96% probability estimate — it's five observations wearing a trenchcoat. Report it as a small number of dependent observations and never compute naive confidence intervals from it.

### Tasks

1. Run every rolling 30-year cohort at 4%, 60/40.  
2. Report success rate, and separately report how many non-overlapping periods you actually have.  
3. Build a heatmap: start year (rows) × withdrawal rate 3–6% (columns) → success or failure.  
4. Identify the worst-performing start years.

### Checkpoint

Your worst cohorts should cluster around 1929, 1937, 1966, and 1973\. If 1966 doesn't show up as one of the worst, check your inflation handling — 1966 is bad specifically because of inflation eroding real withdrawals through the 1970s, and a nominal-terms bug will hide it entirely.

Compare your 4% success rate against Bengen's published figure. Being within a couple of percentage points is a successful replication; document any gap and explain it (different bond proxy, different data vintage, different rebalancing assumption).

---

## Lab 4 — The vectorized engine

### Objective

Build the period state machine that everything else runs on.

### Background

**Vectorize across paths, loop over months.** Represent state as `(n_paths,)` arrays and run a single Python loop over 360 months. The buffer logic is path-dependent and conditional, so you cannot vectorize the time axis — but you don't need to. 360 iterations of numpy operations on 50,000-wide arrays runs in seconds. A per-path Python loop is 18 million iterations and will time out your runtime.

### The period state machine

State per path: `cash`, `equity`, `bonds`, `target_spend`, `cum_shortfall`, `failed`, `first_failure_month`, `high_water_mark`, `trailing_returns`.

For month `t` in 0..359:

1. **Required withdrawal.** Annual: `W = target_spend if t % 12 == 0 else 0` Monthly: `W = target_spend / 12` every month  
     
2. **Rebalance** — if `t % rebalance_interval == 0`, and *before* any withdrawal. Rebalance the invested sleeve only. The buffer is untouched.  
     
3. **Evaluate buffer trigger** using information available at the *start* of month `t` — returns through `t−1` only.  
     
4. **Fund the withdrawal.**  
     
   - If triggered: `from_cash = min(W, cash)`, `remaining = W − from_cash`  
   - If not triggered: `remaining = W`  
   - `from_invested = min(remaining, equity + bonds)`, taken pro-rata across equity and bonds  
   - If the invested sleeve is exhausted, fall back to cash even when untriggered — a real retiree would not starve with money in the buffer  
   - `shortfall = remaining − from_invested − fallback_cash`

   

5. **Record failure.** If `shortfall > 0`: add to `cum_shortfall`; if not already failed, set `failed = True` and `first_failure_month = t`. Keep running the path at zero balance.  
     
6. **Apply month `t` returns** to equity, bonds, and cash. All real. Applying returns *after* the withdrawal is what makes month 0 a genuine start-of-period withdrawal.  
     
7. **Update high-water mark, then evaluate refill** — annually, not monthly.

### Why the buffer sits outside the portfolio

If cash is one of your target allocation weights, then "rebalance first" silently tops the buffer back up every single period. The refill rule you're trying to test never runs, and every buffer configuration produces the same answer. Keep the buffer as a separate account with its own explicit refill logic.

Note that step 4 is self-consistent: rebalancing to target and then withdrawing pro-rata leaves you *at* target, because pro-rata withdrawal preserves weights.

### Checkpoint

Run the engine on the Lab 3 historical cohorts. You must reproduce your Lab 3 numbers exactly. If they differ, the engine has a bug — find it now, before anything is built on top.

---

## Lab 5 — Test suite

### Objective

Write the tests that catch the bug classes that matter. Do this *before* the stochastic engines, not after.

### Tasks

Each of these catches something real:

| Test | Catches |
| :---- | :---- |
| **Conservation:** total assets after step 4 \= total before − amount funded | Double-counting between buffer and invested sleeve |
| **Zero returns, zero withdrawal** → all balances constant for 360 months | Drift, indexing errors, accidental fees |
| **100% cash, 4% annual, zero returns** → exactly 25 successful withdrawals, first shortfall at month 300 | Off-by-one in the withdrawal schedule |
| **No lookahead:** shift the return series forward one month; trigger decisions must shift with it | Using month `t`'s return to make month `t`'s decision |
| **Cadence parity:** with zero returns, monthly and annual cadences produce identical failure months | Indexing bugs between the two arms |

That last one is the highest-value test in the project. Your entire intra-year result depends on the two cadences differing *only* where they genuinely should.

### On lookahead bias

Using information that wouldn't have been available at decision time is the most common way this class of model produces results that are too good. Your buffer trigger fires based on what has already happened, never on what is about to. The shift test above is how you prove it.

---

## Lab 6 — Stochastic engines

### Objective

Generate synthetic futures, and understand why the obvious method is wrong.

### Background

Three approaches:

1. **IID Monte Carlo** — draw each month independently from a fitted distribution.  
2. **Block bootstrap** — draw multi-year *blocks* from history, preserving what happens inside each block.  
3. **Regime-switching** — model a Markov chain over states (expansion, recession, stagflation) with state-dependent return distributions.

**IID Monte Carlo systematically understates sequence risk.** Shuffling returns independently destroys autocorrelation — and autocorrelation is exactly what creates bad sequences. Real markets have clustered drawdowns: 2000, 2001, and 2002 were all negative. Independent draws almost never produce three consecutive bad years, so they under-sample precisely the scenarios that kill retirees.

Quantifying that gap — running the same strategy through engine 1 and engine 2 and reporting the difference in success rate — is one of the most valuable results in the project, and it's a genuine model-validation finding.

**Block bootstrap** fixes this by sampling contiguous chunks (say 24–60 months) and stitching them together. The *stationary bootstrap* (Politis & Romano) uses random block lengths, which avoids artifacts from a fixed block size.

### Tasks

1. Implement IID Monte Carlo.  
2. Implement a block bootstrap with configurable block length.  
3. Run the same strategy through both. Report the success-rate gap.  
4. Sweep block length (12, 24, 36, 60 months) and show how the result moves.

### Checkpoint

Block bootstrap should give a *lower* success rate than IID at the same withdrawal rate. If it doesn't, your blocks probably aren't contiguous — check that you're sampling a start index and taking a slice, not sampling indices independently.

---

## Lab 7 — Scenarios

### Objective

Define named economic environments as correlated shock bundles.

### Background

A recession is not "equities go down." It is equities down **and** rates cut **and** inflation falling **and** credit spreads widening — arriving together. If you model these as independent knobs, you'll never generate a realistic bad scenario, because the correlation is the whole point.

**Starting conditions matter enormously.** Forward returns are conditional on the valuation and yield level at retirement date. A retiree starting at CAPE 40 and a 2% 10-year yield faces a materially different distribution than one starting at CAPE 12 and 8%. Make starting CAPE and starting yield explicit parameters, not implicit averages.

### Scenarios to encode

| Scenario | Analogue | Mechanism |
| :---- | :---- | :---- |
| Early crash | 1929–32, 2008–09 | Deep equity drawdown in years 1–3 |
| Rising rates \+ inflation | 1966–82 | Bond capital losses *and* real withdrawal erosion |
| Stagflation | 1973–74 | Negative real returns across both sleeves |
| Lost decade | 2000–2013 | Flat nominal equities, slow grind |
| Low-yield start | 2010s | Low bond income, high starting valuations |
| Benign | 1982–2000 | Control case |

Store these as YAML in `config/` so scenarios are declarative and reviewable, not buried in code.

### Checkpoint

1966–82 should be your worst case for a retiree — worse than 1929\. If it isn't, your inflation handling is wrong. This is the single best diagnostic in the whole project, because it only comes out right if real-terms accounting is correct end to end.

---

## Lab 8 — The buffer

### Objective

Test the safety-net fund honestly.

### Background

The intuition is that a cash bucket lets you avoid selling equities into a downturn. The complication is that **a cash buffer is partly just a lower equity allocation in disguise.** Holding three years of spending in cash means holding less in equities, and that costs you return in every good scenario.

So the honest test is not "buffer vs. no buffer." It is **buffer vs. a static allocation holding the same average cash weight.** Under that comparison, the buffer's advantage usually shrinks substantially. The typical finding is that buffers improve the worst percentiles while lowering median terminal wealth — a real but narrower benefit than the marketing suggests.

Build the project so it *can* produce that answer. A project that concludes "buffers good" is a brochure. A project that quantifies exactly how much of the benefit is allocation effect versus genuine sequencing benefit is analysis.

### Parameters to sweep

- **Size:** 0, 1, 2, 3, 5 years of spending  
- **Trigger:** trailing 12-month invested return \< 0; or drawdown from high-water mark \> 10%/20%  
- **Refill:** on any up year / only above high-water mark / fixed schedule / never

The refill rule matters more than the size. Confirm or refute that.

### Tasks

1. Full grid over size × trigger × refill.  
2. For each buffer configuration, construct the matched static-allocation control (same time-averaged cash weight) and run it on identical paths.  
3. Report success rate, median terminal wealth, and 5th-percentile outcome for both.

### Checkpoint

Your matched controls should close much of the gap. If a buffer strategy massively outperforms its matched control, check whether the rebalancing logic is quietly refilling the buffer — that's the Lab 4 trap.

---

## Lab 9 — Intra-year timing and the control arm

### Objective

Measure an effect smaller than your noise floor.

### Background

**Success rate is a proportion, and proportions have sampling error.** The standard error is `sqrt(p(1−p)/n)`. At 90% success with 10,000 paths, that's about 0.3 percentage points — so any difference under roughly 1pp between two independent runs is noise.

Intra-year timing effects are genuinely small, likely in the 0.2–0.5pp range. Run this naively and you will report noise.

**The fix is common random numbers.** Generate one set of return paths. Run every strategy variant on the *same* paths. Compare pairwise, per path — not variant A's overall rate against variant B's overall rate as independent estimates. Most of the sampling variance cancels because both arms face identical markets. For paired binary outcomes, McNemar's test is the correct significance test.

### The confound, and the control arm

With annual rebalancing at month 0 and annual withdrawal also at month 0, the two cadences aren't symmetric. Annual withdrawals always come out of a freshly-rebalanced portfolio. Monthly withdrawals come out of a rebalanced portfolio in month 0 and increasingly drifted weights in months 1–11.

So the raw monthly-vs-annual difference bundles **two** effects: the cash-out timing you want to measure, and an interaction with rebalance drift.

The fix is a **control arm with monthly rebalancing**, where the only difference between the two cadences is when money leaves:

| Arm | Rebalance | Withdraw |
| :---- | :---- | :---- |
| A | Annual | Annual, month 0 |
| B | Annual | Monthly |
| C (control) | Monthly | Annual, month 0 |
| D (control) | Monthly | Monthly |

- **B − A** \= the realistic combined effect  
- **D − C** \= pure timing, drift held constant  
- **(B − A) − (D − C)** \= the drift interaction

That decomposition turns an ambiguous single number into a result you can actually explain.

### Tasks

1. Build the common-random-numbers harness: one path set, all arms, paired comparison.  
2. Run all four arms.  
3. Report the decomposition with paired significance tests.  
4. Sample sizing: 10,000 paths for exploration, 50,000 for anything you publish where the effect is under 2pp.

### Checkpoint

Run two arms that are *identical* through the paired harness. The difference must be exactly zero, not merely small. If it isn't, your arms aren't actually sharing the same random draws and the entire lab is measuring noise.

---

## Lab 10 — Results and writeup

### Objective

Turn simulation output into findings.

### Tasks

1. **Success surfaces**, not point estimates. Success rate across withdrawal rates 3–6%, across equity allocations, across starting CAPE. Sequence risk shows up in the *slope* of these surfaces, not the level.  
2. **Failure severity.** You recorded shortfall amounts and first-failure months — use them. "Failed" is much less informative than "failed at year 22 with $310k cumulative real shortfall."  
3. **Limitations section.** Taxes and account location are out of scope. US 1871–present is the luckiest capital-market sample in the world; run a haircut case (e.g. equity real returns reduced 1.5%) and report how much the conclusions move. Fixed 30-year horizon ignores longevity uncertainty.  
4. **Reproducibility.** Every figure regenerable from a seeded config file.

### Suggested findings to look for

- How much does the IID-vs-block-bootstrap engine choice change the "safe" withdrawal rate?  
- Does the buffer survive its matched-allocation control?  
- Which matters more, buffer size or refill rule?  
- Is the intra-year timing effect distinguishable from zero once drift is controlled?

---

## Reading list

Work through these as you hit the relevant lab — not all at once.

- **Bengen (1994), "Determining Withdrawal Rates Using Historical Data"** — origin of the 4% rule. Read before Lab 3\.  
- **Cooley, Hubbard & Walz (1998), the Trinity Study** — success rates across allocations and horizons. Read before Lab 3\.  
- **Guyton & Klinger (2006), decision rules** — dynamic withdrawal strategies, if you extend past fixed withdrawals.  
- **Politis & Romano on the stationary bootstrap** — the statistical basis for Lab 6\.  
- **Kitces' writing on cash reserve strategies** — the argument that buffers are largely an allocation effect. Read before Lab 8; it's the counterargument your project needs to engage with.

---

## Why this is a portfolio piece

Anyone can run a Monte Carlo. What distinguishes this build is the validation layer: the replication of published results, the no-lookahead test, the matched control for the buffer, the paired harness for small effects, and an explicit limitations section. That's the difference between a simulation and a validated model — and it's exactly the work a model validation function does.  
