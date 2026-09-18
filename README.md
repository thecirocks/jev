# Jev vs LLM Router: An Anti-Churn Agent Benchmark

**Replacing the Claude Sonnet router with Jev in this LangGraph agent cut routing latency by 53% and routing cost by 97%, with 46 of 48 routes matching Claude's choice (N=48, p < 0.001).**

Independent experiment. Not affiliated with TypeSafe AI, Anthropic or Vercel.

---

## What is this?

A Jupyter Notebook (jev.ipynb) that benchmarks one architectural change in a LangGraph agent: replacing the LLM router node with Jev (https://typesafe.ai), TypeSafe AI's System One model.

Jev generates no text. It takes a state (text or JSON) and typed questions (choice, score, or yes/no probability) and returns decisions with probabilities and a confidence value. It was in early access in September 2026, priced at $0.042 per 1M input tokens (output free).

---

## Hypothesis

In a LangGraph agent, a routing step only picks one label out of a fixed list. Using a generative LLM for that costs seconds of latency and output tokens. A model built for typed decisions (Jev) should route faster and cheaper without losing routing quality.

---

## Architecture

Two identical LangGraph pipelines. Only the first node changes.

Step 1: Customer message arrives.
Step 2: Router decides the area. Graph A uses Claude Sonnet as router. Graph B uses Jev as router.
Step 3: Conditional edge sends to 1 of 8 specialist areas.
Step 4: Specialist responds using Claude Sonnet in BOTH graphs.
Step 5: Response and log are returned (latency, cost, route, confidence).

The specialist node uses the same Claude model and the same area-specific system prompt in both graphs, so the measured difference is isolated to the router.

### Eight simulated knowledge bases

No vector store, no RAG. Each area is a system prompt.

| Area | What it handles |
|---|---|
| billing | Duplicate charges, refunds, invoice issues |
| technical_support | Bugs, errors, outages, crashes |
| logistics | Late deliveries, missing orders, returns |
| product | Removed features, competitor comparisons, dissatisfaction |
| onboarding | Setup problems, documentation gaps, training |
| cancellation | Direct cancellation requests |
| plans | Price complaints, upgrade/downgrade questions |
| security | Hacked accounts, unauthorized access, data concerns |

### How the routers work

**Claude router (Graph A):** a zero-shot prompt listing the 8 areas and their descriptions, max_tokens=10. Returns the area key as text. Falls back to cancellation if the output is not a valid key.

**Jev router (Graph B):** one choice question whose criteria are the same 8 area descriptions. Returns the chosen key, probabilities for all options, and a confidence score. On error, retries once after 5 s, then falls back to cancellation.

---

## Results (N=48)

### Router comparison

| Metric | Claude Sonnet | Jev | Difference |
|---|---|---|---|
| Latency (mean) | 1,179 ms | 555 ms | 52.9% faster |
| Cost per call | $0.000714 | $0.000021 | 97.0% cheaper |
| Route agreement | n/a | 46 of 48 | 95.8% match |
| Confidence score | not available | mean 0.94, min 0.37 | native |

### Statistical significance (latency)

| Item | Value |
|---|---|
| Mean paired difference | 624 ms |
| SD of differences | 214 ms |
| 95% confidence interval | 562 ms to 686 ms |
| Test | Paired t-test (df = 47) |
| t-statistic | 20.22 |
| p-value | 7.9e-25 |

### Scale projection (router node only)

| Scenario | Claude | Jev | Saving |
|---|---|---|---|
| 1M messages/month | $714 | $21 | about $693/month |

### Disagreements

Two of 48 routes did not match. Both were ambiguous messages where either area is defensible:

| Test | Message | Claude | Jev | Jev confidence |
|---|---|---|---|---|
| 26 | The search function is useless | product | technical_support | 0.72 |
| 28 | Your documentation is outdated | onboarding | technical_support | 0.37 |

Test 28 had the lowest confidence of the entire run, a signal the Claude router does not provide.

---

## Setup

### Requirements

- Python 3.10+
- An Anthropic API key
- A Vercel AI Gateway API key (Vercel requires a card on file to unlock requests)

### Install
```
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
cp .env.example .env
```


Then fill in your keys in the .env file.

### Environment variables
```
AI_GATEWAY_API_KEY=your_vercel_ai_gateway_key
ANTHROPIC_API_KEY=your_anthropic_key
LLM_MODEL=claude-sonnet-4-5
LLM_MAX_TOKENS=2000
```


### Run

Open jev.ipynb in Jupyter or VS Code and run the cells in order. The 50-message benchmark takes roughly 10 to 15 minutes (50 messages x 2 graphs x 2 nodes = 200 API calls).

### How Jev is called

Jev is called over HTTP through the Vercel AI Gateway evaluation endpoint. The endpoint is POST https://ai-gateway.vercel.sh/v4/ai/evaluation-model with headers for authorization, protocol version (0.0.1), evaluation model specification version (4), and model id (typesafe-ai/jev). The body contains state and questions.

Vercel documents this feature through the AI SDK (TypeScript). The raw HTTP format used here was taken from community projects and worked in this run, but it is not an official Python interface and may change.

---

## Repository structure

```
.
|-- jev.ipynb # the benchmark notebook
|-- benchmark_50_antichurn.csv # per-message results
|-- requirements.txt
|-- .env.example
|-- .gitignore
|-- README.md
```


---

## Notebook steps

| Step | Content |
|---|---|
| 0 | Highlights, notes and limitations |
| 1 | Setup: Claude and Jev clients |
| 2 | State, 8 areas, test messages |
| 3 | Router functions (Claude vs Jev) |
| 4 | Specialist node (shared) |
| 5 | Build Graph A and Graph B |
| 6 | Run the 5-message demo |
| 7 | Summary of the demo |
| 8 | Statistical benchmark (50 planned, 48 completed) |
| 9 | Statistical analysis (paired t-test, 95% CI) |

---

## Test design

**Demo (Steps 6-7):** 5 messages, illustrative only. Not statistically meaningful.

**Benchmark (Step 8):** 50 planned messages, 6 to 7 per area, covering different tones and churn signals. 48 completed (test 49 timed out on the Jev call, test 50 was not run).

**Messages:** synthetic, in English, written for this demo. Not real customer data.

**Execution:** each message runs through Graph A then Graph B, sequentially from one machine, with a 1 s pause between messages. Order was not randomized.

**Claude cost:** exact, from the token counts in the Anthropic API response. Prices are hardcoded in the notebook ($3 input / $15 output per 1M tokens). Check them against current pricing before reusing.

**Jev cost:** the exact figure reported by the Vercel AI Gateway (providerMetadata.gateway.cost), measured in a separate pass over the same messages. 45 of 48 calls returned a value (3 timed out). It is not paired call by call with the latency measurements. An earlier version of this notebook estimated Jev cost from character counts and understated it (98.9% saving vs the real 97.0%).

---

## Limitations

1. Synthetic, English-only messages, about 6 per area.
2. Sequential calls from one machine. Graph A always ran before Graph B (order not randomized), and network noise affects latency.
3. Jev was reached through the Vercel AI Gateway, not TypeSafe's API directly. Results may differ on the direct API.
4. The baseline is a zero-shot Claude Sonnet router. A smaller model or prompt caching would likely narrow the gap. This was not tested.
5. The specialist step (about 5 to 14 s) dominates end-to-end time in both graphs, so the router saving is a small share of the full pipeline. End-to-end time was not analyzed in the N=48 run.
6. Reliability: Jev is in early access. Across the runs there was 1 error response (test 6, first attempt, cause not confirmed) and several 30 s read timeouts (test 49 in the main benchmark, tests 15, 18 and 19 in the cost pass). In production you would need timeouts, retries and a fallback route.
7. Results are from a single run in September 2026. Model versions (typesafe-ai/jev alias, LLM_MODEL) and prices can change, so numbers will vary.

---

## Statistical notes

**Latency test:** paired t-test on router latency per message (scipy.stats.ttest_rel), plus a 95% confidence interval on the mean paired difference (t distribution). The result is far below the usual 0.05 threshold.

**Assumptions and gaps:** the t-test assumes roughly normal differences, and latency data is often right-skewed. A non-parametric check (Wilcoxon signed-rank) would be a sensible addition and was not run.

**Agreement uncertainty:** 46 of 48 has a wide interval at this sample size. The 95% Wilson interval is roughly 86% to 99%.

**Cost:** the difference comes from pricing and token counts, so no hypothesis test was applied. The relevant caveat is the per-call variation in Jev cost, which depends on message length.

**Agreement is not accuracy:** there are no human-labeled ground-truth routes. A match means Jev chose the same area as Claude.

---

## Reproducibility

The runs are not deterministic. Latency depends on network and provider load, and Claude specialist outputs vary between runs. Expect the same direction and similar magnitude, not identical numbers.
