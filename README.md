# AI‑Threat Detective (MVP)

Neural/ML system for anomaly detection in smart contracts and blockchain transactions. This repository is an **MVP skeleton**: feature extraction → detectors → stacking → explainability → real-time scoring API.

## What is included in the MVP

- **Feature engineering**: transaction window features, simple graph metrics, sequence features.
- **Models**:
  - tabular detector (RandomForest + IsolationForest),
  - Transformer (transaction sequences),
  - GNN (address graph from→to),
  - fusion: mean of all available modules,
  - basic explainability via SHAP (if available) or permutation importance.
- **FastAPI service**:
  - `POST /score` — score a transaction window,
  - `POST /explain` — technical explanation (top_features, method),
  - `POST /explain_rich` — XAI for DeFi: summary, risk_factors, recommendations (CFO/CISO),
  - `WS /ws` — real-time streaming: buffered window + score,
  - `GET /health` — healthcheck,
  - `GET /metrics` — Prometheus metrics.
- **Scripts**:
  - synthetic data generation for MVP,
  - training and saving artifacts into `artifacts/`,
  - streaming Ethereum mainnet blocks and logging anomalous blocks.

## Who is this for and how to use it

- **Exchanges and DeFi protocols**: integrate as a Python SDK or internal microservice.
- **Analytics / IR / security teams**: run as a separate service with API keys for internal clients.
- **White‑label / resale**: deploy under your own brand and expose HTTP API to downstream clients.

## Quick start (Windows / PowerShell)

Create a virtual environment and install dependencies:

```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install --upgrade pip
python -m pip install -r requirements.txt
```

Train all models on synthetic data:

```powershell
# All models with a single command
python .\scripts\retrain_all.py

# Or individually (with parameters -n, --anomaly-rate):
python .\scripts\train_mvp.py
python .\scripts\train_transformer_mvp.py
python .\scripts\train_gnn_mvp.py
```

Each script saves an artifact into `artifacts/`. The API uses all available artifacts for fusion scoring. If GNN/Transformer artifacts are missing, it falls back to tabular only.

Run the API:

```powershell
python -m uvicorn app.main:app --host 0.0.0.0 --port 8000
```

**Web UI**: open http://127.0.0.1:8000/ in your browser — simple form with Score / Explain / Explain Rich buttons.

**Integration**: see [docs/INTEGRATION.md](docs/INTEGRATION.md) and the `/examples` folder.

Example request:

```powershell
$body = @{
  chain_id = 1
  txs = @(
    @{ from="0xaaa"; to="0xdefi"; value=0.2; gas=21000; gas_price=30e9; function="swapExactTokensForTokens"; timestamp=1700000000 },
    @{ from="0xaaa"; to="0xdefi"; value=25.0; gas=180000; gas_price=90e9; function="swapExactTokensForTokens"; timestamp=1700000030 }
  )
} | ConvertTo-Json -Depth 6

Invoke-RestMethod -Method Post -Uri http://localhost:8000/score -ContentType application/json -Body $body
```

## Project structure

- `ai_threat_detective/` — core library (features, models, pipeline).
- `app/` — FastAPI application.
- `scripts/` — training / data generation / utilities.
- `artifacts/` — saved models (created automatically).

## GNN module

For the GNN module you need `torch_geometric`. It is included in `requirements.txt`; on Windows you may need to install it manually:

```powershell
pip install torch_geometric
```

or, if you need optional wheels for PyTorch 2.4:

```powershell
pip install pyg_lib torch_scatter torch_sparse torch_cluster torch_spline_conv -f https://data.pyg.org/whl/torch-2.4.0+cpu.html
pip install torch_geometric
```

See: [PyG Installation](https://pytorch-geometric.readthedocs.io/en/latest/install/installation.html).

## XAI for DeFi

Endpoint `POST /explain_rich` returns:
- **summary** — short verdict for CFO/CISO,
- **risk_factors** — risk factors (factor, description, severity, evidence),
- **recommendations** — recommended actions,
- **top_features** — technical features (impact, value).

There are templates for common DeFi attack patterns: value spike, sink nodes, risky functions (delegatecall, flashLoan, selfdestruct, etc.), rapid sequences.

## Real-time

WebSocket `ws://127.0.0.1:8000/ws`: the client sends JSON `{"tx": {...}}`, `{"txs": [...]}` or `{"flush": true}`. The server buffers transactions and, once the window is ready (20 tx or 10 seconds), returns `{"event": "score", "result": {...}}`. Module `ai_threat_detective.realtime.TxBuffer` is designed for integration with a mempool stream.

## Configuration (ATD_*)

- `ATD_ANOMALY_THRESHOLD` — anomaly threshold (default 0.5).
- `ATD_WEBHOOK_URL` — URL to POST to when `is_anomaly=true`.
- `ATD_REQUIRE_API_KEY` — require `X-API-Key` for protected endpoints.

You can override the threshold per-request in `/score`: `{"threshold": 0.6, "txs": [...]}`.

## Future work

- Support multiple networks (L2, EVM‑compatible chains),
- Improve models on real customer datasets,
- TimescaleDB/ClickHouse for historical time-series storage,
- Additional metrics (latency, throughput, per‑client SLA).

## FAQ

**Q: Is this production‑ready?**  
**A:** It is an MVP engine: models, features, fusion, XAI, and API are ready and work on real Ethereum mainnet data. For internet‑facing SaaS you should still harden deployment (HTTPS reverse proxy, secrets management, logging, monitoring).

**Q: Does it only work with Ethereum mainnet?**  
**A:** No. The core works with abstract `Tx` objects and `chain_id`. We provide an Ethereum mainnet streamer as an example; for other EVM chains or your own systems you just need an adapter that maps your transactions/events into `Tx`.

**Q: Which database do you use? Can I switch to PostgreSQL?**  
**A:** By default we use SQLite (`data/atd.db`) only for API keys and quotas. You can easily replace this with PostgreSQL or any other DB by re‑implementing the small layer in `app/db.py`.

**Q: How secure is this?**  
**A:** The engine does not execute user code, uses Pydantic validation for inputs, and loads only local model artifacts. For a secure deployment you should run it inside your trusted infrastructure, behind HTTPS and with proper access control / monitoring.

**Q: Can I resell this as my own service?**  
**A:** Yes. The design supports white‑label usage: you can deploy the API under your brand, issue API keys to your customers, and build your own UI or billing around the stable HTTP and WebSocket interfaces.

