# Idea: Log & Alert Triage Assistant (Classifier + Small Model + LLM Teacher)

> A three-tier system that sorts millions of log lines / alerts into **team-specific**
> categories. A cheap custom ML classifier handles almost everything, a local small
> language model handles the unclear cases privately, and a big LLM teaches on the
> rare hard ones. It gets more tailored to your team every week.

## Why this use case (and not, e.g., photo sorting)

A custom classifier + SLM + LLM setup wins where big products can't follow:

1. **Categories are unique to you/your team** — no generic product knows them.
2. **Data is private** — can't be shipped to a cloud provider at scale.
3. **Volume is huge** — running an LLM on everything is too expensive.

Log triage hits all three.

## The problem

- Production systems emit millions of log lines and hundreds of alerts per day → **alert fatigue**; the real outage gets buried.
- Vendor tools (Datadog, Splunk) do generic anomaly detection but don't know team knowledge:
  *"this 2am batch-job timeout is harmless"*, *"this exact error = payments vendor is down, call them."*
- Logs contain customer data and secrets → can't send it all to a cloud LLM.
- At millions of lines/day, LLM cost alone rules out LLM-only.

## Architecture

```
 Log line / alert  (millions per day)
        │
        ▼
 Tier 1: Custom ML classifier            ← ~95%, microseconds, free
   (text embedding + logistic regression / XGBoost)
   Team-specific categories, e.g.:
     ignore_noise | known_issue:db_failover | known_issue:vendor_outage |
     needs_ticket | PAGE_NOW
        │ unsure (low confidence)
        ▼
 Tier 2: Small language model (local)    ← ~4%, private
   - strips secrets / customer data
   - reads context (20 lines before, service, recent deploys)
   - picks a category or says "I don't know"
        │ still unsure
        ▼
 Tier 3: Big LLM — the teacher           ← ~1%, sees only cleaned text
   - diagnoses ("connection-pool exhaustion after deploy #4812")
   - assigns a category + one-line explanation
        │
        ▼
 Save (cleaned log, category) as a training example → retrain classifier daily
```

## Roles

| Part | Type | Role | Learns? |
|---|---|---|---|
| Classifier | Classic ML (embedding + logistic regression / XGBoost) | Fast first pass on everything | ✅ retrained daily |
| Small language model | Local, ~3B params | Privacy filter, context reader, middle tier | ✅ optionally LoRA fine-tuned later |
| Big LLM | Cloud | Teacher / diagnostician for new or weird cases | ❌ produces labels only |
| On-call engineer | Human | Final word ("actually noise") | — corrections are the best training data |

### How the classifier learns

- The embedding model (e.g. a sentence-embedding model) is **frozen** — it turns each log line into a vector.
- The classifier on top is ordinary supervised ML: `X` = embeddings, `y` = category from LLM / human, `model.fit(X, y)`.
- Retraining takes seconds; this is transfer learning ("linear probe").

## The automatic check (ground truth)

Every alert ends in an outcome: resolved as noise, ticket opened, or incident declared.
**That outcome is the label.** The system grades itself:

- *"I said `ignore_noise`, but it became an incident"* → high-priority training example.
- No separate labeling effort needed.

## Worked example

```
Day 1: "ERROR conn timeout payments-api upstream=stripe 30000ms"
       Classifier: unsure (40% vendor_outage, 35% needs_ticket)
       Small model: removes merchant IDs; context shows 50 similar errors in 2 min
       LLM: "Burst of upstream timeouts to payment vendor → known_issue:vendor_outage.
             Check their status page."
       On-call confirms ✅ → saved as a training example

Day 3: same pattern appears
       Classifier: 94% vendor_outage → auto-tagged, posted to Slack with explanation.
       No LLM call, no human woken up.
```

## Why it's hard for big players to copy

- Your categories and history ("2am batch timeouts are fine") exist only inside your team.
- Logs never leave your network — only cleaned, rare cases reach the LLM.
- It gets more specific to you every week; a generic vendor model can't.
- Near-zero cost at scale — the classifier handles ~95% of volume.

## Build plan (~4 weeks)

| Week | Build |
|---|---|
| 1 | Collect logs (own apps, or public **LogHub** datasets). Define 5–8 categories. LLM-label ~500 lines as seed data. |
| 2 | Tier 1: sentence embeddings + logistic regression, confidence threshold, metrics stored in DuckDB. |
| 3 | Tiers 2 & 3: local small model for redaction + context, LLM teacher for the rest. Log every escalation. |
| 4 | Learning loop: daily retrain, "wrong category" feedback button in `webapp/`, dashboard of escalation rate and missed incidents. |

## Metrics

- **Headline:** % of alerts reaching a human ↓, while **missed real incidents stays at zero**.
- Escalation rate to Tier 2 / Tier 3 over time.
- Classifier precision/recall per category (especially `PAGE_NOW` recall — must be ~100%).
- Cost per 1M log lines.

## Pitfalls

- **Missing a real incident is far worse than a false alarm** → bias thresholds so anything that might be `PAGE_NOW` escalates; never auto-suppress low-confidence lines.
- **New failure modes** look like nothing seen before → low confidence should route up, not default to `ignore_noise`.
- **Redaction leaks** → test the small model's scrubbing against planted secrets before anything reaches the cloud LLM.
- **Label drift** after deploys/system changes → weight recent examples more; keep a fixed eval set.

## Related ideas

- Same pattern as the SQL Flywheel: [slm-flywheel.md](slm-flywheel.md).
- Sibling use cases with the same shape: support-ticket routing, contract/invoice clause flagging,
  data-catalog PII tagging over `warehouse.duckdb`, personal research/news triage.
