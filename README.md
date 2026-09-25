# FraudGraph Agent

An agentic fraud investigator built for the TigerGraph x Hacker House Goa challenge.

Give it an alert — a risk score, a customer complaint, an analyst request — and it investigates the case on a TigerGraph knowledge graph, figures out what kind of fraud it might be, recommends the next action under the bank's fraud policy, asks for more evidence when it's unsure, explains its reasoning, and writes the finished case back into the graph as memory.

See [FEATURES.md](./FEATURES.md) for a full breakdown of what each part does, and a plain account of what's actually confirmed working versus what needs infrastructure not included here.

## How it works

```
alert → Investigate → Gather → GraphRAG → Assess → Recommend → (Evidence) → Explain → Remember
```

1. **Investigate** – pull the card's transaction history, device links, and prior cases from TigerGraph
2. **Gather** – rule-based detectors (card testing, structuring) plus a trained episode model figure out which transactions belong together
3. **GraphRAG** – vector search over 5,565 closed-case narratives and policy text for similar precedent
4. **Assess** – a calibrated ML model estimates fraud probability (the bank's own risk score is excluded)
5. **Recommend** – a deterministic policy engine (rules R1–R10) decides the action and approval route — the LLM never decides this
6. **Evidence** – if uncertain, request more evidence (simulated in this build) and re-run the recommendation
7. **Explain** – an LLM writes the analyst summary and, if warranted, a SAR — from structured facts only
8. **Remember** – the case is written back into the graph so future investigations can retrieve it

## Stack

- **Graph**: TigerGraph 4.2.5 Community Edition (Docker) — 590K transactions, 14.9K cards, 5.5K closed cases
- **Agent ↔ graph**: `tigergraph-mcp`, with a direct `pyTigerGraph` fallback
- **Models**: gradient-boosted classifiers over graph-derived features (`scikit-learn`)
- **LLM**: OmniRoute (OpenAI-compatible), Ollama Cloud as fallback
- **Backend**: FastAPI
- **Frontend**: React + Vite

## Running it

You'll need TigerGraph running locally, the dataset loaded, and API keys in `.env` — none of which ship in this repo. Once you have those:

```bash
docker run -d --name tg -p 14240:14240 -p 9000:9000 --ulimit nofile=1000000:1000000 tigergraph/community:latest

python scripts/prepare_data.py
# apply gsql/schema.gsql, gsql/vectors.gsql, gsql/load.gsql, gsql/queries.gsql, gsql/vector_queries.gsql
python scripts/load_graph.py
python scripts/embed_and_load.py

python scripts/build_training_set.py && python scripts/train_models.py && python scripts/train_episode.py

pip install tigergraph-mcp
python scripts/run_cases.py          # writes cases/HHG-001.json ... HHG-020.json

uvicorn api.app:app --port 8088      # dashboard at http://127.0.0.1:8088
```

`.env` needs `OMNIROUTE_BASE_URL`, `OMNIROUTE_API_KEY`, and the `OLLAMA_*` variables.

### Before you run this yourself

`/home/vinay/hackerhouse/...` is hardcoded as the data/output/.env location in 8 files — `agent/llm.py`, `agent/data_access.py`, and every script in `scripts/` (`prepare_data.py`, `build_training_set.py`, `train_episode.py`, `embed_and_load.py`, `run_cases.py`, `train_models.py`). Point these at wherever your own data and `.env` actually live before running anything.

You'll also need `data/HHGOA_IEEE/case_pack.csv` (the 20 benchmark alerts) already in place before `run_cases.py` — no script in this repo generates it.

### Just want to see the dashboard?

The 20 sample cases in `cases/*.json` are already investigated. You don't need a live TigerGraph connection to browse them — just the backend needs a `data/HHGOA_IEEE/case_pack.csv` file (`case_id`, `card_id`, `customer_id`, `risk_score`, `trigger_type`, `trigger_text`, `flagged_txn_id`) listing those 20 case IDs, since that file isn't shipped with the repo.

## Known limitations

- Customer and analyst replies are simulated; every assumption made is recorded under `evidence_requests`
- No labeled answer key exists, so live accuracy is unmeasured — the numbers below are cross-validated on closed cases only (fraud AUC 0.987, pattern accuracy 0.83, episode F1 0.80)
- The closed-case dataset has a quirk where cleared cases sit on light cards with widely shared devices; the model shrinks its probabilities accordingly and the agent runs a verification loop when signals are thin
- Account-takeover reconstruction is weakest on very heavy-usage cards
