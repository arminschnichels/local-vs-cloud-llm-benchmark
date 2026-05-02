## 🔍 Analytical Approach

### Ground Truth Labelling
- **3 human annotators**, each independently scoring ~65 reviews
- Scale: -1 (negative) to +1 (positive) across 5 aspects
- Known limitation: no annotator overlap, so inter-annotator agreement
  (Cohen's Kappa) cannot be calculated — individual scoring bias cannot
  be ruled out, particularly for the highly subjective ambiance category

### Models Tested

| Model | Type | Infrastructure |
|---|---|---|
| LLaMA 3.2 (3B) | Local | Ollama / MacBook Air 16GB |
| Gemma 4 (9B) | Local | Ollama / MacBook Air 16GB |
| GPT-OSS-120B | Cloud API | Groq |

### Evaluation Framework
- **Reliability:** Mean Absolute Error (MAE) + Standard Deviation per aspect
- **Coverage:** Recall — what % of human-labelled reviews did the AI rate?
- **Speed:** Processing time extrapolated to 50,000-review production workload
- **Cost:** API token costs (input + output) projected at scale



## 📊 How the Decision Was Made

### Step 1 — LLaMA is eliminated on coverage alone

LLaMA 3.2 3B was the first model tested. Despite being competitive on service
MAE (0.18), its coverage collapsed on the aspects that matter most operationally:

- Price recall: **41.67%** — ignores 6 out of 10 price mentions
- Ambiance recall: **40.48%** — ignores 6 out of 10 ambiance mentions

A model that misses 60% of feedback in key categories cannot support
operational decisions, regardless of how accurately it rates what it does process.
LLaMA was eliminated without further analysis.

### Step 2 — Gemma vs. GPT: accuracy is close, coverage is not

With LLaMA removed, the comparison narrows to Gemma 4 9B vs. GPT-OSS-120B.

On accuracy, the models are competitive — GPT edges Gemma on price and ambiance
MAE, Gemma wins on overall, food, and service. Neither has a decisive advantage.

On coverage, Gemma pulls clearly ahead:

| Aspect | Gemma | GPT | Gemma advantage |
|---|---|---|---|
| Service | 91.89% | 89.19% | +2.7pp |
| Price | 87.50% | 79.17% | +8.3pp |
| **Ambiance** | **80.95%** | **52.38%** | **+28.6pp** |

The ambiance gap is the deciding factor. GPT misses nearly half of all
ambiance feedback — the most subjective and experience-driven category.
If Velox Foods wants to improve dining atmosphere across locations, a
model that ignores half the signal is not fit for purpose.

### Step 3 — Cloud vs. local deployment

Gemma 4 9B locally takes ~25 days per 50,000 reviews on a MacBook Air 16GB.
Even a Mac Studio with 96GB RAM is estimated at 3–5 days — still impractical,
and the hardware investment dwarfs the $24 API cost per run.
Cloud deployment is the only viable production path.



## 🔬 Error Autopsy — Where Models Fail

*Analysis focused on ambiance — the primary decision factor between Gemma and GPT.
LLaMA error autopsy was not conducted as it had already been eliminated on recall.*

### Failure Mode 1 — Context Collapse (both Gemma and GPT)

> *"It also gives you the time to take in the 'New Orleans charm.'
> It was very old fashioned style on the inside."*

| | Score |
|---|---|
| Human | -0.5 |
| GPT | +0.5 |
| Gemma | +0.5 |
| Error | 1.00 (maximum possible) |

Both models anchored on positive surface phrases and ignored the broader
negative experience described across the full review — crowding, long waits,
poor conditions, overpriced food. The "New Orleans charm" framing is
sarcastic, not a genuine compliment. Neither model resolved net sentiment.

### Failure Mode 2 — Implicit Cultural Signal (Gemma)

> *"This restaurant is a cafeteria style setting."*

| | Score |
|---|---|
| Human | -0.5 |
| Gemma | +0.5 |
| Error | 1.00 (maximum possible) |

Syntactically neutral, but carries a clear negative connotation in a
restaurant context — impersonal, noisy, low-quality experience. Gemma
treated it as a factual description rather than a sentiment signal. This
type of culturally-loaded neutral language is one of the hardest problems
in sentiment analysis and cannot be fully solved with prompt engineering alone.

### What This Means for Deployment

Both failure modes are concentrated in the ambiance category — consistent
with why ambiance shows the highest MAE across all models. Recommended
mitigations for production deployment:

- Prompt engineering should explicitly instruct the model to resolve
  net sentiment in mixed-tone reviews
- A domain-specific calibration corpus for restaurant ambiance language
  is advisable
- Human spot-checks on ambiance ratings should be maintained for the
  first 90 days post-deployment



## 🚧 Technical Challenges Solved

**Challenge 1 — Pydantic Validation Errors on Sparse Reviews**
Problem: Groq returned `null` for `price.quotes` and `ambiance.quotes`
when a review didn't mention those aspects. Pydantic `list[str]` rejects null.
Solution: Changed field type to `list[str] | None = None` to accept null values.
Learning: Schema design must anticipate sparse data — not all reviews
mention all aspects.

**Challenge 2 — Recall Exceeding 100%**
Problem: Initial recall calculation returned 173% for Overall.
Root Cause: AI DataFrames had more rows than ground truth due to index
mismatch after merge — predictions were counted against a smaller denominator.
Solution: Recalculated using `count()` on matched, non-null values only.
Learning: Always validate merge outputs before computing ratios.

**Challenge 3 — Column Name Conflicts After Merge**
Problem: After merging three model DataFrames, columns appeared as
`food_score`, `food_score_x`, `food_score_y`.
Root Cause: Rename was applied before inspecting actual column names —
columns were `food_score` not `food`.
Solution: Inspected `df.columns` first, then renamed correctly before merging.
Learning: Always inspect column names before renaming. Assumptions about
DataFrame structure cause silent bugs.

**Challenge 4 — Silent Model Mismatch**
Problem: Initial Gemma results were identical to LLaMA — suspicious.
Root Cause: The wrong model was selected in Ollama while running the
Gemma notebook. Results were LLaMA run twice.
Solution: Verified active model in Ollama before rerunning. Corrected
Gemma results showed meaningful improvement over LLaMA.
Learning: Always confirm which model is actually loaded before benchmarking.
Silent configuration errors produce plausible-looking but wrong results.



## 💼 Skills Demonstrated

**Technical:**
- Python (Pandas, Scikit-learn, Matplotlib, Seaborn)
- LLM API integration (Groq) and local model deployment (Ollama)
- Pydantic schema design for structured LLM outputs
- Statistical evaluation: MAE, standard deviation, recall per aspect
- Data pipeline debugging and validation

**Business:**
- Translated statistical metrics into operational impact for a non-technical
  Operations Director audience
- Identified coverage — not accuracy — as the critical decision variable
- Risk-balanced recommendation: cloud API over local or full rollout

**Analytical:**
- Three-model comparison with systematic elimination framework
- Manual error autopsy to understand model failure modes qualitatively
- Extrapolated 200-review sample results to production-scale cost and time



## 📈 Business Impact

This analysis directly informed a build-vs-buy infrastructure decision for
processing Velox Foods' 50,000-review backlog. The recommendation to use
Gemma 4 31B via Cloud API — rather than the tested GPT alternative or
either local model — was driven by a 28-percentage-point coverage advantage
in ambiance: the category most directly linked to dining experience and
location-level performance differences.



## 🎓 Key Learnings

1. **Coverage gaps are more dangerous than accuracy gaps.** A model that
   silently ignores 60% of feedback is not fit for purpose, regardless of
   how accurately it rates what it does process.
2. **The real cost of "free" is analyst time and delayed decisions.** 25 days
   of locked hardware per run is not a viable production pipeline.
3. **Implicit sentiment is the hardest problem.** Culturally-loaded neutral
   phrases fool every model tested. Prompt engineering helps but does not
   eliminate this failure mode.
4. **Silent configuration errors produce plausible-looking wrong results.**
   Always verify which model is loaded before benchmarking.
5. **Ground truth quality sets the ceiling.** Without annotator overlap,
   individual scoring bias — especially on ambiance — cannot be detected
   or corrected for.



## 📞 Contact

Armin Schnichels
arminschnichels@gmail.com



## 📝 Project Background

This project was completed as part of a data analytics Weiterbildung at WBS Coding School.
The original brief was to compare a local LLM (LLaMA 3.2 3B via Ollama) against a
cloud API model (GPT-OSS-120B via Groq) and deliver a cost-benefit recommendation.

Gemma 4 9B was added independently, outside the scope of the assignment, after noticing
that the initial two-model comparison didn't answer a question that seemed relevant:
*does a larger local model close the gap with the cloud API, or is the bottleneck
something else entirely?*

Adding Gemma answered that question — and unexpectedly produced the strongest performer
on coverage, which became the basis for the final recommendation.



## 📜 Project Context

This analysis was completed as part of a data analytics bootcamp case study.
The dataset contains anonymized customer reviews used for educational purposes.
My scope covered the full pipeline: data ingestion, local and cloud model
evaluation, statistical analysis, and executive recommendation.
