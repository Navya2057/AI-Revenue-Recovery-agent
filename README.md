I Revenue Recovery
**Razorpay AI Buildathon** | Direction: *Payment degradation → root cause → recovery action*

Built by Navya Vashistha.

## The problem
When a payment fails, most systems either do nothing (silent revenue loss) or blast
every customer with the same retry/SMS regardless of whether they were ever likely
to come back. Neither is honest about what actually drives recovery.

## What this does
1. **Detects** revenue at risk from real failed-payment events (a Kaggle credit-card
   transactions log: `errors` column = genuine gateway decline reasons like
   *Insufficient Balance*, *Bad CVV*, *Bad Expiration*).
2. **Diagnoses** each failure into a `SOFT` / `HARD` / `HUMAN` bucket via a
   transparent, documented rule (`recovery_config.py`) — no black box here, the
   decline reason is already known data, not something to predict.
3. **Scores risk** with a real ML model (`recovery_model.py`, gradient-boosted
   trees) trained on ~209K historical failures, predicting the probability each
   failure recovers **on its own**, calibrated to the true ~95% base rate.
4. **Decides a bounded action** (`policy_engine.py`) per event — auto-retry, SMS/
   WhatsApp nudge, agentic call, human escalation, or hold — by expected
   *incremental* value, respecting hard stopping rules: max automated attempts,
   DND hours, and a minimum-value-to-act floor.
5. **Measures money recovered** (`simulate_batch.py`) with a Monte Carlo
   simulation comparing three policies on the same batch: do-nothing, a naive
   "retry everyone the same way" baseline, and this agent's policy.
6. **Logs every decision** with a plain-English reason — the audit trail.

## Honest limitations (stated up front, not hidden)
- The raw transaction log has no record of which recovery *action* was ever
  tried on a failure — it's a POS log, not a merchant's dunning-ops log. So the
  risk model is trained only to predict *organic* recovery, not action effects.
- Action **uplift** numbers (`ACTION_UPLIFT_PP` in `recovery_config.py`) are a
  documented industry-range assumption, not learned from this data. A real
  deployment would replace these within weeks via A/B-testing the actions.
- Because ~95% of failures self-recover anyway, plain accuracy is a useless
  metric here — we report ROC-AUC and precision/recall on the riskiest slice
  instead, and calibrate `predict_proba` explicitly (see comments in
  `recovery_model.py`) so the Rupee math downstream is trustworthy.

## Results (on the last 30 days of real data, ~1,836 failed transactions, Rs 95L at risk)
| Policy | Recovered | Recovery % | Cost | Net value |
|---|---|---|---|---|
| Do nothing | ~Rs 90.2L | ~95.5% | Rs 0 | ~Rs 90.2L |
| Blind retry everyone | ~Rs 91.6L | ~96.0% | Rs 7.7K | ~Rs 91.5L |
| **This agent** | **~Rs 92.9L** | **~97.3%** | **Rs 3.1K** | **~Rs 92.9L** |

The agent beats the naive baseline while spending *less*, because it doesn't
waste contact budget on transactions already near-certain to self-recover.
(Numbers shift slightly run to run — Monte Carlo — and are larger on the full
209K-row historical set vs. the 20K sample shipped in this repo; regenerate
with the full Kaggle download for the exact numbers in the pitch video.)

## Data source
[Credit Card Transactions Dataset](https://www.kaggle.com/) (Kaggle) —
`transactions_data.csv`, `cards_data.csv`, `users_data.csv`. Not committed to
this repo (too large for git); download it yourself and run
`build_dataset_from_kaggle.py` to regenerate the full dataset. A 20K-row
sample of the derived historical log ships in `data/` so everything else runs
out of the box.

## Run it
```bash
pip install -r requirements.txt
python3 recovery_model.py      # trains + evaluates the risk scorer
python3 policy_engine.py       # produces data/recovery_decisions_audit_trail.csv
python3 simulate_batch.py      # produces data/simulation_results.csv
```
Or open `Track3_Revenue_Recovery_Agent.ipynb` in Google Colab and run all cells.

## What's next (extending in Antigravity)
- **Hinglish voice recovery**: wire `agentic_call_offer_plan` to a real Gemini
  API call that drafts a Hinglish payment-plan offer message per customer.
- Replace the documented `ACTION_UPLIFT_PP` assumptions with real A/B-tested
  numbers once actions are live.
- A small dashboard (Streamlit/Next.js) over `recovery_decisions_audit_trail.csv`
  for the "show the audit trail" pitch moment.
- Extend beyond card-present POS failures to the other Track 3 directions this
  pipeline is already structured for: subscription mandate failures, B2B
  receivables (see `event_type` field, currently all `payment_failure` since
  that's what this Kaggle set contains).

## Files
```
recovery_config.py               shared config: buckets, actions, costs, uplift assumptions, stopping rules
build_dataset_from_kaggle.py     ETL: raw Kaggle CSVs -> revenue_at_risk_events.csv / historical_recovery_outcomes.csv
recovery_model.py                the ML risk scorer + honest evaluation
policy_engine.py                 chooses one bounded action per event + audit trail
simulate_batch.py                Monte Carlo: do-nothing vs blind-retry vs agent, money recovered
Track3_Revenue_Recovery_Agent.ipynb   Colab-ready walkthrough of the whole pipeline
data/                            customers, live batch, historical sample, audit trail, sim results
```

