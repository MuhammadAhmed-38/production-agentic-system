# Production Agentic System — Multi-Tool AI Agent

A multi-tool agent that decomposes user queries into an explicit JSON plan, executes tools in parallel where possible, and synthesises grounded final answers. A personal project exploring explicit-planning agent architectures, parallel tool execution, and rigorous LLM evaluation.

## TL;DR results

- **9/10 queries passed** on the evaluation set (v2 structured prompts)
- **Mean cost $0.014/query**, mean latency ~13s
- **Ablation delta:** v2 scored 10pp higher than v1 (minimal prompts)
- 5 production tools: calculator, web_search, knowledge_base, document_qa, code_executor
- Hard budget caps, parallel tool execution, JSON repair, graceful failure recovery

## Documents

| Document | What it covers |
|---|---|
| [`REPORT.md`](./REPORT.md) | 2-page report: results, failures, learnings, ablation |
| [`ARCHITECTURE.md`](./ARCHITECTURE.md) | System design, trade-offs, scaling analysis (100 concurrent users) |
| [`ROADMAP.md`](./ROADMAP.md) | 5 features to ship next week, grounded in observed limitations |
| [`FAILURE_LOG.md`](./FAILURE_LOG.md) | Raw debugging log from development |
| [`REPORT_NOTES.md`](./REPORT_NOTES.md) | Development notes, eval findings |

## Quick start

```bash
# 1. Clone and enter
git clone https://github.com/MuhammadAhmed-38/production-agentic-system.git
cd production-agentic-system

# 2. Install
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt

# 3. Configure API keys
cp .env.example .env
# Edit .env: add ANTHROPIC_API_KEY and TAVILY_API_KEY

# 4. (Optional) Ingest PDFs for document_qa tool
python -m tools.document_qa

# 5. Run a query
python main.py "What is the population of France multiplied by 2?"
```

## CLI usage

```bash
# Human-readable output (default)
python main.py "your query"

# JSON output (for eval harnesses or programmatic use)
python main.py "your query" --json

# Use baseline (minimal) prompts instead of structured
python main.py "your query" --prompt-version v1

# Verbose logging
python main.py "your query" --verbose
```

### Example output

```bash
QUERY: What is the population of France multiplied by 2?
PLAN (2 step(s), 2 group(s)):
Reasoning: Need Frances population from KB, then multiply by 2...
[1] knowledge_base_lookup({'action': 'lookup', 'path': 'countries.france.population_millions'})
[2] calculator({'expression': '{{step_1.output}} * 2'}) depends_on=[1]
Parallel groups: [[1], [2]]
EXECUTION:
[1] knowledge_base_lookup -> OK (1ms)
[2] calculator -> OK (0ms)
FINAL ANSWER:
The population of France is 67.97 million. Multiplied by 2, this equals 135.94 million.
======================================================================
METRICS: 10.50s wall time | 2 LLM calls | $0.012457 query spend | 1 iteration(s)
```

## Tools

| Tool | Purpose | Backend |
|---|---|---|
| `calculator` | Safe arithmetic evaluation | Python AST whitelist |
| `web_search` | Current/real-time information | Tavily API |
| `knowledge_base_lookup` | Structured facts lookup | Local JSON (`data/kb/facts.json`) |
| `document_qa` | Q&A over ingested PDFs | ChromaDB + `all-MiniLM-L6-v2` |
| `code_executor` | Python code execution | Subprocess-isolated sandbox |


## Running the evaluation

```bash
# Full baseline (v2 prompts, all 10 queries)
python -m eval.runner --prompt-version v2

# Ablation (v1 minimal prompts)
python -m eval.runner --prompt-version v1

# Specific queries only
python -m eval.runner --prompt-version v2 --queries Q1 Q4 Q7
```

## Evaluation summary

Ran on 10 queries covering all 5 tools across sequential, parallel, and mixed-source scenarios. v2 passes 9/10 (90%); v1 passes 8/10 (80%). The single v2 failure (Q8, multi-hop KB lookup) reveals a structural planning limitation documented in [`REPORT.md`](./REPORT.md) and addressed in the 

roadmap. Full per-query breakdown: [`runs/eval_v2_baseline.json`](./runs/eval_v2_baseline.json).


Results are saved as JSON in `runs/`. Baseline + ablation artifacts are
committed as `runs/eval_v2_baseline.json` and `runs/eval_v1_ablation.json`.

## Project structure

```bash
agent/
config.py         Env-var loading, model pricing
budget.py         Token + cost tracking with hard caps (thread-safe)
prompts.py        v1 (baseline) + v2 (structured) prompts
planner.py        Query → Plan JSON (Sonnet) with JSON repair + retry
executor.py       Plan → observations (parallel tools) + synthesis (Haiku)
orchestrator.py   Top-level: plan → execute → AgentResult
tools/
base.py           Tool ABC + ToolRegistry + ToolResult
calculator.py     AST-safe arithmetic
web_search.py     Tavily wrapper (async via thread pool)
knowledge_base.py JSON lookup with case-insensitive dot-paths
code_executor.py  Subprocess sandbox with network blocks + timeout
document_qa.py    Incremental hash-based PDF ingestion + vector search
eval/
queries.py        10 evaluation queries with per-query rubrics
judge.py          LLM-as-judge (Haiku) with structured Judgment output
runner.py         Eval harness with per-capability aggregation
data/
kb/facts.json     Structured facts (countries, companies, constants)
pdfs/             Sample PDFs for document_qa
runs/               Evaluation results (JSON)
main.py             CLI entry point
test_setup.py       Pre-flight check (API keys, dependencies)
```

## Model usage & costs

- **Planner + synthesis:** `claude-sonnet-4-5` — plan quality dominates outcomes
- **Judge (eval only):** `claude-haiku-4-5` — classification task, cheap
- **Budget enforcement:** `BUDGET_CAP_PER_QUERY=0.10` (hard reject on over-spend)
- **Observed per-query cost:** ~$0.010–$0.019

## Known limitations (honest)

See `REPORT.md` for detailed analysis. Short version:

- **Multi-hop KB queries** — planner commits to a single lookup path and
  doesn't iterate. Q8 (Apple CEO + iPhone release year) fails on this.
  Mitigation in `ROADMAP.md` item #1 (reflection loop).
- **Stochastic failure distribution** — Q10 gave different outputs across
  runs. Production eval needs multi-run variance measurement.
- **Single-process architecture** — not designed for concurrent load.
  Scaling analysis in `ARCHITECTURE.md`.

## License & attribution

A personal portfolio project demonstrating production agentic system design.