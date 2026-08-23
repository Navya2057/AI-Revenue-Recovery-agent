# The problem
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
