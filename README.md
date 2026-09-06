# Reserve

A browser-based bank operator simulation. You own the charter: you set the rate sheet, write the
underwriting policy, decide who gets credit, and answer for the balance sheet. There is no win
condition — only how long you last and how large you get before the regulator, the depositors, or
your own loan book takes the bank away from you.

Open `index.html` in any modern browser. No build step, no server, no dependencies.

## The premise

You earn on the spread between what you pay depositors and what you charge borrowers, plus fees.
You can genuinely lose the bank three ways:

* **Liquidity failure** — depositors demand cash you do not have.
* **Regulatory seizure** — the capital ratio stays below the minimum through a warning and a
  consent order, or equity hits zero.
* **Slow death** — persistent losses erode equity until one of the above happens.

## What is actually modelled

### The balance sheet is a real ledger

Every movement of money goes through a single double-entry function, `post()`, which refuses to let
`Assets − Liabilities − Equity` drift from zero and records any violation. **Settings › Developer**
re-derives the aggregates from the individual accounts and shows the residuals; anything other than
zero is a bug, not a game mechanic. Across 90+ headless runs of 10–15 simulated years each, the
worst observed residual is float noise (~1e-5 on nine-figure balance sheets) with zero unbalanced
journal entries.

```
Assets       cash + investment securities + gross loans − allowance for loan losses
Liabilities  customer deposits (checking / savings / 12-month CDs)
Equity       paid-in capital + retained earnings
```

### Lending capacity is a real constraint

| Ratio | Definition | Regulatory floor |
|---|---|---|
| Reserve ratio | cash ÷ deposits | 10% |
| Capital ratio | equity ÷ risk-weighted assets | 8% |

Risk weights: cash 0%, securities 20%, secured lending (auto, mortgage) 50%, unsecured (personal,
card) 100%. Undrawn credit-card lines convert into risk-weighted assets at 50%, so handing out
limits is not free even before anyone spends.

Growth is funded by gathering deposits and retaining earnings. If booking a specific application
would breach a ratio, it is declined with the actual reason and the actual post-booking numbers.

### Loan-loss provisioning

Each month the bank books a provision to hold the **allowance for loan losses** at one year of
expected loss across the portfolio (`balance × PD × LGD`, by tier, product and macro state). When a
loan charges off, the loss draws the allowance down first; only once the allowance is exhausted does
a loss hit equity directly. Provision releases are shown as releases, not negative expenses.

### Credit risk is per-customer

Annualised default probability by score band, multiplied by product (mortgage 0.55× … card 1.30×),
by debt-to-income, by loan seasoning, and by the macro cycle (recession 2.5×, normal 1.0×, boom 0.6×):

| Band | Score | Annual PD |
|---|---|---|
| A | 800+ | 0.3% |
| B | 740–799 | 0.8% |
| C | 670–739 | 2.0% |
| D | 580–669 | 6.0% |
| E | below 580 | 15.0% |

Payment behaviour is not a single dice roll. A performing account can enter distress; a distressed
account either cures (26% per month unsecured, 34% secured) or misses again, walking 30 → 60 → 90
DPD and charging off at 120. Distress entry is calibrated so the *eventual* charge-off rate matches
the table above, which means delinquency runs several times higher than the default rate — as it
does in reality. Interest stops accruing at 90 days past due.

Cardholders are transactors or revolvers depending on their score: transactors pay in full and earn
you interchange and the annual fee but no interest; revolvers pay near the minimum and carry a
balance. Term borrowers amortise, and prepay early when your rate has drifted above the market.

### The market pushes back

Every rate you set is compared against a drifting market rate (mean-reverting, 1%–9%). Price a loan
above market and approved applicants decline your offer; price below market and you win volume but
thin the spread. Pay under market on deposits and accounts attrit to competitors. CD money is
locked until maturity — the most expensive funding, and the only funding that cannot run.

## The loop

1. Applicants arrive daily — deposit accounts most often, then cards, personal loans, auto loans,
   and mortgages least often but largest.
2. Your policy auto-approves, auto-declines, or routes to the **manual review queue**, where you see
   the full applicant file: income, existing debt, DTI with the new payment, expected loss, spread
   over your cost of funds, take-up odds, and what the booking does to your ratios. Approve, deny,
   or counter with a different rate, amount, or term.
3. Unreviewed files walk away after 14 days and cost reputation. The queue caps at 60.
4. Each month closes: interest accrues, payments and defaults resolve, the allowance is trued up,
   operating costs and marketing are charged, tax is paid on positive pre-tax income, and net income
   flows into equity. The regulator then reviews the ratios.
5. Reputation and marketing spend drive the applicant pipeline and deposit inflows. That is the
   growth loop.

## Failure

* **Bank run** — 15 consecutive days below the reserve requirement, or a reputation collapse,
  triggers a wave of withdrawals lasting 10–20 days. Your lever is the **emergency APY premium**:
  it slows the outflow immediately and costs real interest expense on every deposit. Securities are
  liquidated automatically at a haircut before the bank fails. If cash still cannot cover the
  demand, that is a liquidity failure.
* **Regulator** — below 8% capital: month one a warning, month two a consent order barring new
  credit, month three seizure. Equity at or below zero is an immediate seizure with no cure period.

Either way you get a summary screen — days survived, peak assets, peak equity, customers served,
lifetime interest income, lifetime charge-offs, cause of failure — and a new charter.

## Difficulty

| | Paid-in capital | Opening deposits | Reputation |
|---|---|---|---|
| De novo (tight) | $2.0M | $1.0M | 50 |
| Comfortable | $5.0M | $10.0M | 58 |
| Undercapitalised (hard) | $1.2M | $0.5M | 38 |

## Controls

Speed is paused / 1× / 4× / 15×, where 1× is 2.5 real seconds per simulated day, so a 30-day month
closes every 75 seconds (5 seconds at 15×). Space toggles pause; `1`, `2`, `3` select the speeds.
The theme button cycles auto / light / dark.

## Notes on the implementation

Single file, no build step. Simulation and rendering are decoupled: the sim marks dirty flags and a
throttled loop repaints only the visible tab, so a session with tens of thousands of accounts does
not re-render the DOM every tick. The portfolio table is filtered, sorted and paged rather than
dumped.

State autosaves to `localStorage` every 20 simulated days and on every decision, and always resumes
paused. Because a long session can reach tens of thousands of accounts — far more than
`localStorage` will hold as plain JSON — accounts are serialised as positional arrays with money
rounded to cents, and the rounding is reconciled back through the ledger on load so the balance
sheet still balances after a reload. If the save would still be too large, closed accounts are shed
first. Saves can be exported and imported as JSON from **Settings**.

### Deliberate simplifications

* 30-day months and 360-day years, so a monthly rate is exactly APR ÷ 12.
* Recoveries are booked in full at charge-off from the loss-given-default assumption, rather than
  collected over the following years.
* Income tax is a flat 21% of positive pre-tax income with no loss carry-forward.
* One representative applicant per file; no household or joint underwriting.

### Out of scope for v1

Competitor banks, commercial lending, securities trading beyond a passive sweep, and multiplayer.
