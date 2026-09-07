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
| Total capital ratio | (Tier 1 + admissible Tier 2) ÷ risk-weighted assets | 8% |
| Tier 1 ratio | equity ÷ risk-weighted assets | 6% |

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

Every rate you set is compared against a market policy rate (mean-reverting, 1%–9%) that is **re-set once
a year** by default, so a rate sheet stays good for a while and repricing is a scheduled decision rather
than a monthly chore. The interval is settable (yearly / half-yearly / quarterly / monthly) under
Settings; each move is scaled to the interval so the long-run distribution of the rate is the same
whichever you pick. Price a loan
above market and approved applicants decline your offer; price below market and you win volume but
thin the spread. Pay under market on deposits and accounts attrit to competitors. CD money is
locked until maturity — the most expensive funding, and the only funding that cannot run.

## Capital, and the levers you have over it

Capital is the constraint that bites a bank that is doing well. Equity grows only through retained
earnings, so a fast-growing book outruns it — every new loan adds risk-weighted assets faster than it
adds profit. The **Treasury** tab holds the five things a real bank does about that.

**Raise equity.** Outside investors will fund you, sized by reputation and your trailing twelve months
of earnings, for a placement fee. The size collapses to about a third and the fee doubles once you are
in breach or under a consent order — capital is cheapest when you do not need it. One raise per year.

**Issue subordinated notes.** A liability that counts as **Tier 2 capital**, admissible up to the size
of Tier 1. It lifts the total capital ratio without an equity raise, but you pay the coupon every month,
you repay the principal in ten years, and it does nothing for the Tier 1 ratio or for insolvency. The
coupon is priced off the policy rate and widens with distress, weak reputation, and non-performing
loans. Fail to repay a maturing tranche and noteholders put you into receivership — though depositors
are senior, so a liquidity failure usually gets there first.

**Pay a dividend.** The point of owning a bank, and the reason your capital ratio stops improving. Capped
by capital, by cash, and by trailing earnings, and blocked outright while you sit below your own limits.
Lifetime distributions show on the game-over screen.

**Cut credit lines.** Undrawn card limits convert into risk-weighted assets at 50% and earn nothing
until someone spends. Reducing unused lines to 125% of the current balance is the fastest capital relief
available — and cardholders resent it, so it costs reputation and some of them close the account.

**Sell a loan portfolio.** A buyer takes a random cross-section of one product's book for cash, at a
discount that widens with the credit risk of the paper and with a recession. You free the risk-weighted
assets immediately and hand over every dollar of future interest. Selling cards transfers their undrawn
commitments too.

**Call the notes early.** Subordinated notes can be redeemed before maturity at a premium: 2% at or after
the five-year call date, stepping up 1.5% for every year of call protection still to run. Calling retires
Tier 2 capital, so the ratio drops the moment you do it — and the call is refused if that would put you
under a regulatory minimum.

### Investment securities are a real position, not a parking space

Securities pay the yield they were **bought** at, carry a 20% risk weight, and are not cash for the
reserve ratio. Buy and sell them by hand on the Treasury tab, or leave the automatic sweep to do it.

The book is marked to market against a four-year duration: buy at 3%, watch rates go to 5%, and the book
is worth about 92 cents on the dollar. That loss is unrealised — right up until you need the cash and
have to sell, which is precisely when it becomes real. The Treasury card shows the mark and the
unrealised figure, and the overview raises an alert once the hole passes 10% of equity. It is entirely
possible to be solvent on the balance sheet and dead the moment there is a run.

A note on the sweep's **cash buffer**: it is the cash you *keep*, expressed as a share of deposits.
Anything above it is bought into securities at each month-end. A high buffer therefore sweeps *less*, not
more — set it to 89% and effectively nothing is ever swept. It also never sells what you already hold;
only a manual sale or a liquidity squeeze does that.

## Funding is a choice, not a given

Deposits are the cheap way to fund a bank, not the only way. Three levers make the liability side a real
decision.

**Close the door.** Each deposit product — checking, savings, CDs — can be switched off on *Rates &
Policy*. Applicants are turned away, existing accounts stay and keep earning, and the book stops
replacing what attrits. Refusing depositors costs reputation.

**Sell the book.** A buyer takes a deposit portfolio off your hands *and pays you for it*: you hand over
the balances in cash and keep a premium, because cheap sticky money is a franchise worth owning. Checking
fetches the most, CDs almost nothing, and every basis point you pay above market makes the book worth
less. It is the fastest way to shrink an expensive liability side and book a gain doing it.

**Borrow wholesale.** Money in size, instantly, with no branches and no marketing. It costs more than
retail deposits, it counts in the reserve requirement exactly like a deposit — the ratio is cash over
deposits *and* wholesale — and it rolls at maturity **only while you are well capitalised**. Breach a
capital minimum and it does not renew: the funding walks at precisely the moment you cannot replace it,
which is its own way to lose the bank. Tranches run 3, 6 or 12 months and can be repaid early for a
0.5% breakage fee, and total wholesale is capped at 40% of assets.

Together these make a deposit-free bank playable. Sell the whole deposit book, shut the products, and run
on equity and wholesale money as a finance company — profitable, faster to steer, and one capital breach
away from having no funding at all.

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
* **Regulator** — below 8% total capital *or* 6% Tier 1: month one a warning, month two a consent order
  barring new credit, month three seizure. Equity at or below zero is an immediate seizure with no cure
  period — subordinated debt is capital only while there is equity beneath it.
* **Subordinated debt default** — a maturing tranche you cannot repay ends the bank.
* **Wholesale funding withdrawn** — money that matures while you are below a capital minimum does not
  roll, and if the cash is not there to repay it, that is the end.

Either way you get a summary screen — days survived, peak assets, peak equity, customers served,
lifetime interest income, lifetime charge-offs, cause of failure — and a new charter.

## Difficulty

| | Paid-in capital | Opening deposits | Reputation |
|---|---|---|---|
| De novo (tight) | $2.0M | $1.0M | 50 |
| Comfortable | $5.0M | $10.0M | 58 |
| Undercapitalised (hard) | $1.2M | $0.5M | 38 |

## Playing on a phone

The repository is public and `index.html` sits at its root, so GitHub Pages will serve it as a website
with no build step: **Settings → Pages → Source: Deploy from a branch → `/ (root)` → Save**. A minute
later the game is at `https://<user>.github.io/Reserve/`.

On iOS, open that URL in Safari and use **Share → Add to Home Screen**; the page declares the standalone
meta tags and carries its own icon, so it launches full-screen without browser chrome. The layout adapts
below 760px: the metric strip becomes one swipeable row, tabs and tables scroll horizontally, touch
targets grow, and inputs render at 16px so iOS does not zoom in every time you tap a field.

Saves live in `localStorage`, which is per-origin and per-device — a bank started on a laptop will not
appear on a phone. Move one across with **Settings → Export save**, then **Import save** on the other
device.

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
* An equity raise has no share count behind it, so dilution is not modelled; the cost is the placement
  fee and the terms you get when you are desperate.
* Securities are one undifferentiated portfolio with a single blended book yield and a fixed four-year
  duration; there is no maturity ladder and no held-to-maturity versus available-for-sale distinction.

### Out of scope for v1

Competitor banks, commercial lending, securities trading beyond a passive sweep, and multiplayer.
Interest-rate hedging is the obvious next lever.
