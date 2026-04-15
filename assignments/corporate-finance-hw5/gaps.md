# Knowledge Gaps: Corporate Finance HW5

*Gaps identified during Socratic dialogue sessions.*

## Efficient Market Hypothesis (Remember)
**Gap:** Conflated "prices reflect all available information" with "prices are always correct/equal fundamental value"
**Teaching:** EMH says prices incorporate all available information, making the market hard to beat — but this doesn't mean prices equal true fundamental value. The information itself can be incomplete or models can be wrong. EMH = hard to beat, not always right.
**Verified:** yes

## Ambiguity Aversion (Remember)
**Gap:** Initially said investors require a higher "price" — confused price and return. Higher required return means lower price, not higher.
**Teaching:** Ambiguity aversion = demanding higher returns for securities with hard-to-estimate probability distributions. Higher required return implies a lower price today. Distinct from traditional risk (known variance) — ambiguity is about not knowing the odds at all.
**Verified:** yes

## Financial Distress Costs (Remember)
**Gap:** Initially guessed firms with lots of debt or firms that handle money. Needed to understand the key factor is ongoing service obligations vs one-and-done products.
**Teaching:** Firms with ongoing obligations (software, insurance, warranties) lose more customers in distress because customers depend on the firm's continued existence. One-and-done sellers (soup, shoes) are less affected. Correctly applied to Allstate vs Adidas.
**Verified:** yes — correctly identified Allstate (insurance = ongoing obligation)

## Asset Beta / Unlevered Beta (Remember)
**Gap:** Had equity beta and asset beta reversed (thought asset beta = equity+debt, unlevered = just equity). Did not know how to account for cash holdings when unlevering beta.
**Teaching:** Equity beta is just the stock's observed beta (amplified by leverage). Asset/unlevered beta strips out leverage to show underlying business risk. Formula: β_assets = (E/(D+E)) x β_equity (when β_debt ≈ 0). When company holds significant cash, use β_operating = β_equity x E / EV to strip out the zero-beta cash that dilutes observed equity beta.
**Verified:** yes — correctly computed Qualcomm's asset beta as 1.671

## Enterprise Value (Remember)
**Gap:** Did not know the concept or formula.
**Teaching:** Enterprise Value = Equity + Debt - Cash. It's the price tag on the company's operating business. Cash is subtracted because it's not part of the operating business — you'd get it back after buying the company.
**Verified:** yes — correctly computed Qualcomm's EV as $52.53B

## Unlevered Cost of Capital (Remember)
**Gap:** Initially said r_U = WACC (they're only equal without taxes). Then said r_U < WACC (reversed the direction). Needed to understand that WACC < r_U because the tax shield subsidizes debt, pulling the blended cost down.
**Teaching:** r_U = the return the firm's assets generate regardless of financing (as if 100% equity). WACC = the blended cost of capital including the tax benefit on debt. WACC < r_U because the government subsidizes debt via tax deductions, making the effective cost lower.
**Verified:** yes

## WACC Formula (Remember)
**Gap:** Did not know the WACC formula. Needed to understand why r_E is a "cost" from the company's perspective (it's the return the company must generate to keep equity investors happy).
**Teaching:** WACC = (E/(D+E)) x r_E + (D/(D+E)) x r_D x (1 - Tax Rate). Nearly identical to the MM weighted average equation but with (1 - Tax Rate) on debt to capture the tax shield. WACC is the company's blended cost of financing. Without taxes, WACC = r_U. With taxes, WACC < r_U because the tax shield makes debt cheaper.
**Verified:** yes — correctly computed Goodyear's WACC as 5.22%

## MM Proposition II (Remember)
**Gap:** Did not know what MM stood for or what the proposition says. Initially guessed that more leverage means less return to equity holders (opposite of the truth). Did not know the formula.
**Teaching:** MM = Modigliani-Miller. Proposition II: r_E = r_U + (D/E) x (r_U - r_D). More leverage amplifies equity risk because debt payments are fixed — equity absorbs all volatility on a smaller base. Higher risk → higher required return. User correctly connected this to ambiguity aversion logic (higher required return = lower price).
**Verified:** yes — correctly computed Q12 (16%) and Q13 (21%)

## Interest Tax Shield (Remember)
**Gap:** Did not know what a tax shield was. Guessed it might mean interest isn't taxed.
**Teaching:** Interest payments are tax-deductible, reducing the company's tax bill. Tax Shield = Tax Rate x Interest Payment. Interest Payment = Debt x Interest Rate. User derived the formula algebraically from first principles once taught the concept.
**Verified:** yes — correctly calculated $11M x 7% x 20% = $0.154M

## Arbitrage (Remember)
**Gap:** Knew the buy-low-sell-high mechanic but explained "risk-free" in terms of avoiding limits to arbitrage rather than the core reason: simultaneous execution with guaranteed profit.
**Teaching:** Risk-free arbitrage means buying and selling simultaneously — no time gap where prices can move against you. The profit is locked in at the moment of execution. This is distinct from convergence arbitrage where you wait for prices to converge.
**Verified:** yes

## Limits to Arbitrage (Remember)
**Gap:** Knew arbitrage conceptually but didn't know what limits it. Didn't know bid-ask spread mechanics (bid vs ask, who buys/sells at which). Didn't understand margin buying or margin calls. Didn't distinguish pure arbitrage from convergence arbitrage.
**Teaching:** Two main limits: (1) Financing/margin constraints — in convergence arbitrage, you must hold a position while waiting for price to reach fundamental value, and margin calls can force you out before you're proven right. (2) Liquidity/transaction costs — wide bid-ask spreads on thinly traded stocks eat profits from pure arbitrage. Also taught: bid = buyer's price (you sell here), ask = seller's price (you buy here). Margin = buying with borrowed money, broker forces sale if equity drops below threshold.
**Verified:** yes

## Overreaction vs Underreaction (Remember)
**Gap:** Initially described both as "investors price differently than true value" without distinguishing the patterns. Also initially said underreaction drift happens because of new information rather than the original information being slowly absorbed.
**Teaching:** Overreaction = price overshoots, then reverses. Underreaction = price doesn't move enough, then drifts in the same direction. The drift is NOT from new information — it's the original news being gradually absorbed. Key pattern: reversal vs drift.
**Verified:** yes
