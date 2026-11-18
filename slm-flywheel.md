# Idea: The Flywheel — a Self-Shrinking AI

> Start with a big LLM doing most of the work, and continuously teach a small local
> model until it can do almost everything itself. The system gets cheaper, faster,
> and more private the more it's used.

**North-star metric:** *escalation rate* — % of requests the small model can't handle.
If that curve goes down week over week, the idea works.

---

## 1. The problem

Goal: ask questions about our data in plain English (e.g. *"Which customers spent the
most last month?"*) and get a SQL query that answers it, against `warehouse.duckdb`.

- **Big LLM** (Claude, GPT…): very good, but costs money per call, is slower, and sends data off-machine.
- **Small model** (1–4B params, runs free on a Mac via Ollama / MLX): fast and private, but doesn't know our tables and gets many questions wrong.

## 2. The key fact: models only learn from examples

A model is a pile of numbers (**weights**). It learns only when shown examples of the
correct answer and its weights are nudged toward producing them — **fine-tuning**.

**The model does NOT learn while you use it.** Learning happens only in a separate
training run. So the real question is: *where do we get thousands of correct examples cheaply?*

## 3. The trick: the big LLM + DuckDB generate examples for free

For every question:

```
Question: "Top 5 customers by revenue last month?"

A. Small model writes SQL
   → SELECT name, SUM(amt) FROM ordrs ...          (typo)
B. Verify in DuckDB
   → ERROR: table "ordrs" does not exist             ❌
C. Escalate to big LLM
   → SELECT c.name, SUM(o.amount) AS revenue FROM orders o JOIN customers c ...
D. Verify in DuckDB
   → runs, returns 5 rows                            ✅
E. SAVE the pair:
   { "question": "Top 5 customers by revenue last month?",
     "sql": "SELECT c.name, SUM(o.amount) ..." }
```

- User gets the right answer immediately.
- We gain one **verified training example**.
- **DuckDB is the judge** — no human grading needed.
- If the small model passes at step B, we use its answer and skip the LLM (free).

## 4. The learning step (nightly / weekly)

After a few days we have e.g. `training.jsonl` with ~2,000 verified question→SQL pairs.
Fine-tune the small model on it:

```bash
mlx_lm.lora --model <small-model> --train --data ./training_data --iters 1000
```

What happens: the model sees each question, compares its SQL to the saved correct
SQL, and its weights are adjusted slightly — thousands of times.

- **LoRA** trains only a small add-on layer (a few MB), not the whole model → runs on a laptop.
- This is **distillation**: big model = teacher, small model = student.
- Each week the small model trains on exactly the questions it *failed* — learning from its own mistakes, using the teacher's corrections.

## 5. Expected curve (illustrative — measuring the real one is the experiment)

| Week | Small model handles | Escalated to LLM | Cost |
|---|---|---|---|
| 1 | 40% | 60% | $$$ |
| 2 | 65% | 35% | $$ |
| 4 | 85% | 15% | $ |
| 8 | 93% | 7% | ¢ |

## 6. "Port to LLM" is built in

Put the model behind one interface:

```python
def generate_sql(question, schema, model) -> tuple[str, float]:  # sql, confidence
    ...
```

- Develop and iterate on the **small model** (free, fast).
- Plug the **big LLM** into the same interface as teacher/fallback.
- Run LLM-only as a baseline to see how close the small model gets.

## 7. Whole loop

```
   Question
      │
      ▼
 Small model tries ──► DuckDB verifies ──✅──► Answer (free)
                            │
                            ❌ / low confidence
                            ▼
                   Big LLM answers ──► DuckDB verifies ──✅──► Answer
                                                  │
                                    Save (question, correct SQL)
                                                  │
                                    Nightly: LoRA fine-tune small model
                                                  │
      ◄──────────── next day: fewer escalations ◄─┘
```

## 8. Confidence routing (when should the small model give up?)

- Sample 5 SQL candidates; agreement between them = confidence.
- Token log-probabilities.
- Verifier signals: parses? executes? sane result shape (non-empty, plausible row count)?

## 9. Pitfalls

- **Verifier blind spots** — SQL that runs but answers the wrong question. Mitigate: LLM spot-checks a sample of passes; result-shape sanity checks.
- **Catastrophic forgetting** — keep a frozen eval set; only accept a new adapter if it doesn't regress.
- **Schema drift** — retrieve schema context at query time rather than baking schema into weights.

## 10. Six-week plan

| Week | Build |
|---|---|
| 1 | Small-model text-to-SQL on `warehouse.duckdb` + DuckDB verifier; baseline on 50 hand-written questions |
| 2 | Confidence scoring + router that escalates to the LLM |
| 3 | Logging + dataset builder (verified pairs only, deduped) |
| 4 | LoRA fine-tune loop (MLX-LM on Apple Silicon); re-measure |
| 5 | Dashboard: escalation rate, cost/query, latency, accuracy over time |
| 6 | Generalize to a second verifiable task (e.g. JSON extraction validated by schema) |

## 11. Why it matters

The pattern generalizes to any task with a cheap verifier — regex, JSON extraction,
config generation, unit-tested code, API calls. A chart showing *"escalations fell from
60% → 8% in a month, cost per query fell 10×"* is a strong talk / blog post, and
potentially a product: a drop-in proxy that makes any LLM workflow cheaper over time.
