# Model selection

Data: `data/churn_v1.csv` (1,200 customers, 25% churn), `data_sha256` = `76a6447e764b`.
Test set: 20% split with `random_state=42`, so all three candidates are judged on the same 240 customers.

## The question
The model predicts which customers are about to leave so the retention team can call them first.
A missed churner (false negative) is a lost customer; a false alarm only costs one phone call, so I care most about F1 on the churn class, not accuracy.

## Candidates
Only 25% of customers churn, so every candidate uses `class_weight="balanced"`. Without it, logistic regression scored F1 0.495; with it, 0.589.

| Candidate | Why I tried it | AUC | F1 | F1 per slice (lowest) | p95 latency (ms) | Size (KB) | MLflow run ID |
|---|---|---|---|---|---|---|---|
| `logreg_balanced` | Linear baseline: fast, tiny, easy to explain | 0.797 | **0.589** | 0.333 (two-year) | 5.2 | 4 | `3329e3c798194b008bb7a0b28052d9c6` |
| `random_forest_balanced` (200 trees, depth 6) | Trees can learn interactions such as short tenure AND many support calls | 0.780 | 0.582 | 0.400 (two-year) | 20.7 | 1,335 | `8755da83cdc64a16bd916ed4bbe6cc12` |
| `hist_gb_balanced` (200 rounds, lr 0.05, depth 3) | Boosting usually wins on tabular data | 0.760 | 0.561 | 0.400 (two-year) | 8.0 | 190 | `e6e6a3f4d426475f9c0727e6ea39c188` |

F1 per contract type:

| Candidate | month-to-month | one-year | two-year |
|---|---|---|---|
| `logreg_balanced` | 0.626 | 0.571 | 0.333 |
| `random_forest_balanced` | 0.602 | 0.615 | 0.400 |
| `hist_gb_balanced` | 0.582 | 0.571 | 0.400 |

## The champion
I picked **`logreg_balanced`**. It has the best overall F1 (0.589) and AUC (0.797), and the best F1 on month-to-month contracts (0.626), which is where most churn happens (35% of those customers leave).
It is also about 4x faster than the forest (5.2 vs 20.7 ms) and over 300x smaller (4 KB vs 1,335 KB), so it is cheap to serve and quick to load after a restart or a scale-to-zero.

What I accepted losing:
- On **two-year** contracts it has the lowest F1 (0.333 vs 0.400). This slice has only about 49 test customers and 6 churners, so the difference is about one customer: noise, not a real weakness. I will watch it rather than pick a bigger model for it.
- The random forest is better on **one-year** contracts (0.615 vs 0.571), but it is 0.007 F1 worse overall, 4x slower and over 300x larger. That is not worth it.

## My promotion rules
- Latency budget (p95): **50 ms**, because one prediction must fit easily inside a ~100 ms API response. My slowest candidate took 20.7 ms on my laptop, and CI runners are roughly 2x slower (~41 ms), so 50 ms leaves a safety margin and the gate does not fail in CI for the wrong reason.
- Minimum gain to replace the champion (F1 points): **0.01**, because the test set has only about 60 churners. Catching one more churner moves F1 by about 0.01, so a smaller gain is noise. Swapping the champion for noise would mean a new deployment, and a new risk, with no real improvement.

## What I would check next
The two-year slice is too small to trust (6 churners in the test set). With more data, or with `churn_v2.csv`, I would retrain and check whether the weak two-year score is real. If it is, I would add a minimum per-slice F1 rule to the gate so a model that fails one customer group cannot be promoted.