# AI Data Analyst & Business Intelligence Agent

A business user asks questions about e-commerce data in plain English — *"What were the top 5 products by revenue?"*, *"Why did revenue decline last month?"*, *"Show monthly revenue as a chart"* — and gets answers **computed from the actual database**, explained by a **local LLM**, with every number verified.

Everything is **free and open-source**: no paid APIs, no API keys, and the data never leaves the machine.

| | |
|---|---|
| **LLM / provider** | Qwen2.5-7B-Instruct, run locally via **Ollama** (llama.cpp supported as an alternative) |
| **Database** | **PostgreSQL** (SQLite fallback, same code via SQLAlchemy) |
| **Backend** | FastAPI + Pydantic |
| **Dashboard** | Streamlit + Plotly |
| **Tests** | pytest (via ipytest) — 71 tests, including 6 AI questions verified against independent ground truth |
| **Format** | One notebook, `ai_bi_agent.ipynb`, run top to bottom |

---

## 1. Architecture

```
            Streamlit dashboard  ──HTTP──►  FastAPI  (validation, HTTP error codes)
                                               │
                                       ask(question)  ── hybrid agent ──────────────┐
                                               │                                    │
                         ┌─────────────────────┴───────────────┐          rule-based router
                         │  Local LLM (Ollama / llama.cpp)     │          (LLM down / LLM gives up)
                         │  understands question, picks tools, │                    │
                         │  words the answer                   │                    │
                         └─────────────────────┬───────────────┘                    │
                                               ▼                                    ▼
                     call_tool()  — whitelist, argument validation, code-level guards
                                               │
               ┌───────────────────────────────┼───────────────────────────┐
         11 fixed analytics tools     guarded query_data (SQL)      generate_chart
                                               │
                                     PostgreSQL / SQLite
```

**The LLM never calculates anything.** All numbers come from SQL/pandas functions. The LLM only (1) understands the question, (2) chooses a tool and its arguments, (3) words the result.

## 2. Setup & run

### Option A — Kaggle (recommended; free GPU)
1. Create a Kaggle notebook → **File → Import notebook** → `ai_bi_agent.ipynb`.
2. Settings: **Internet ON** (needs phone verification), **Accelerator: GPU T4 x2**.
3. **Run All.** First run takes ~5 min (installs PostgreSQL, Ollama, downloads the 4.7 GB model).
4. The dashboard cell near the end prints a public `https://….trycloudflare.com` link (valid while the notebook runs).
5. The LLM backend is set by `LLM_BACKEND` in the LLM cell: `"ollama"` (default), `"llamacpp"` or `"none"` (rule-based only).

### Option B — Local (Linux / macOS; Windows via WSL)
```bash
git clone <repo-url> && cd ai-bi-agent
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt

# LLM (optional — without it the app uses the rule-based fallback)
ollama serve &                 # install from https://ollama.com
ollama pull qwen2.5:7b         # or qwen2.5:3b on low-RAM machines

# Database (optional — without it SQLite is used automatically)
export DATABASE_URL=postgresql+psycopg2://user:pass@localhost:5432/bi_db

jupyter notebook ai_bi_agent.ipynb      # then Run All
```
Dashboard: http://localhost:8501 · API docs: http://localhost:8000/docs

## 3. Notebook layout

The notebook has no section headings; it runs top to bottom in this order:

1. Configuration, PostgreSQL startup, synthetic data generation, cleaning and validation, schema (PK/FK/CHECK/indexes) and load
2. Analytics layer: sales, product and customer metrics, cross-checked against independent pandas
3. Tool layer: registry, argument validation, guarded `query_data`, dynamic charts
4. Local LLM client, grounding check, rule-based router, hybrid agent
5. Multi-step "why did revenue change" investigation and grounding fixes
6. AI insights and the FastAPI backend (runs in a background thread)
7. Streamlit dashboard (plus a Cloudflare tunnel on Kaggle)
8. Test suite, plus the guards added after live tests failed

## 4. Data

Synthetic but realistic and **reproducible** (seed 42): 40 products in 5 categories, 600 customers who sign up gradually over 18 months (Jan 2024 – Jun 2025), ~3,300 orders, ~6,600 order lines. The raw CSVs are in `data/raw/`.

The raw data deliberately contains **duplicates, missing cities, unparseable dates, null and negative quantities** — the cleaning step removes/repairs them and reports what it did. It also removes **orphan orders** (orders whose every line was invalid) — a bug found by cross-checking two independent counts.

A **revenue dip is built into the last month** (fewer orders + an Electronics slump partly shifted to cheaper Beauty items), so the multi-step analysis has a real, known cause to find.

**Definitions:** revenue = sum of line totals of **completed** orders (cancelled excluded) · AOV = revenue ÷ completed orders · "last month" = latest month **in the data** (2025-06) · inactive = no completed order in N days (default 90) before the latest order date.

## 5. Features

- **KPIs:** revenue, orders, customers, AOV — overall or per month, with month-over-month change.
- **Charts:** monthly revenue, revenue by category, top products, customer trend, orders, AOV, top customers, revenue change by category.
- **Natural-language questions** through chat (dashboard) or `POST /ask`.
- **Dynamic charts:** "show me monthly revenue as a chart" builds the chart from live data.
- **Multi-step analysis** (`explain_revenue_change`) in six steps: headline KPIs → decomposition of the revenue change into order-volume and order-size effects (sums exactly) → category contribution → product contribution → customer activity (active / new / returning) → ranked drivers.
- **AI insights:** code computes headline facts, the LLM rewrites them as insights, every number is verified.
- **Unsupported questions** ("What's the weather?", "What's our profit margin?") are clearly refused.

## 6. AI / agent approach — how answers stay grounded

1. **Tool calling, not free text.** The LLM receives 11 tools (OpenAI function-calling format) plus `cannot_answer`.
2. **`call_tool()` is the only door** to the data: unknown tools/arguments, wrong types and out-of-range values are rejected; numeric strings are coerced.
3. **Code-level guards** on top of the LLM's choice (each added after a live test found a failure):
   - *month guard* — strips a `month` the user never asked for (the LLM once answered "top 5 products" for June only);
   - *chart guard* — `generate_chart` only for explicit chart requests;
   - *dimension guard* — breakdowns no fixed tool provides (city, order status, signup date) must use `query_data`.
4. **Guarded SQL (`query_data`)** — the hybrid fallback: single `SELECT`/`WITH` only, forbidden-keyword check, table whitelist, read-only connection, 200-row limit, 5 s timeout (PostgreSQL). The SQL is shown to the user.
5. **Verified facts.** Each tool result carries a `facts` field written by code (highest/lowest month, change %, totals). The LLM must base comparisons on these rather than reasoning over raw rows itself (it once named the wrong peak month).
6. **Grounding check.** Every number in the LLM's answer must match a number in the tool data or facts (tolerant to rounding, thousands, sign). If not, the answer is **replaced** by a deterministic summary and the user is told why.
7. **Source line.** Every answer ends with the tool and arguments used, e.g. `Source: get_inactive_customers(days=30, limit=50)`.
8. **Fallbacks.** LLM down/timeout → rule-based router. LLM refuses a question the router recognises → router answers. Off-topic questions (no router match) stay refused.

## 7. API

| Endpoint | Purpose |
|---|---|
| `GET /health` | database + LLM status, data range |
| `GET /kpis?month=YYYY-MM` | revenue, orders, AOV, units, customers |
| `GET /metrics/{name}` | `monthly_revenue`, `revenue_by_category`, `top_products`, `low_products`, `customers`, `inactive_customers`, `top_customers`, `revenue_change` |
| `GET /charts/{chart_type}` | Plotly chart JSON + data |
| `GET /tools` · `POST /tools/{name}` | list / call a tool directly |
| `POST /ask` | the agent: answer, mode, source, data, SQL, chart |
| `GET /insights` | AI-generated insights + underlying facts |

Status codes: **400** invalid input / no data · **404** unknown metric/tool · **422** malformed request · **503** database unavailable · **500** unexpected (no stack trace leaked).

## 8. Testing

Run the last cell of the notebook; it runs every test. **71 tests**, all passing on Kaggle with the live LLM:

- **Data processing:** duplicates, missing values, bad dates, invalid quantities, no orphans/invalid rows in the DB.
- **Business calculations** on a tiny dataset whose answers were worked out by hand (revenue ₹430 with a cancelled ₹500 order excluded, AOV ₹143.33, change +₹190 = +₹120 volume + ₹70 order size, repeat/new/inactive counts).
- **Tools & SQL guard:** 7 unsafe queries blocked, row limit enforced, database unchanged afterwards.
- **Error handling:** simulated **database outage** (→ 503), malformed LLM JSON, blank/too-long questions, invalid parameters.
- **API:** status codes for 13 requests.
- **Independently verifiable AI questions** — ground truth computed with plain pandas (not the app's SQL), each asked through both the deterministic path and the **live LLM**:

| Question | Verified answer |
|---|---|
| What is our total revenue? | ₹1,490,572.79 |
| How many orders did we get last month? | 137 |
| What is the top product by revenue? | Electronics Item 6 |
| Which category generated the most revenue last month? | Electronics |
| Why did revenue decline last month? | −42.4 % (May → June 2025) |
| Which city has the most customers? | Kolkata |

- **Agent behaviour** with a scripted fake LLM: hallucinated numbers replaced, unsupported questions refused, wrong tools blocked, the LLM giving up handled.

## 9. Problems found during development (and fixed)

| Symptom | Root cause | Fix |
|---|---|---|
| Two counts of completed orders disagreed (2,487 vs 2,485) | orders whose every line item was invalid stayed in the DB | cleaning removes orphan orders |
| "Top 5 products" answered for June only | LLM added a month nobody asked for | month guard in code |
| "Which city…" answered with a customer-trend chart | LLM picked an unsuitable tool | chart guard + dimension guard |
| Wrong "highest month" in a correct-looking answer | LLM compared 18 rows itself | code-computed `facts` field |
| Valid answers replaced as "unverified" (`'9102,850.16'`) | `json.dumps` sent ₹ as `\u20b9`; the model copied the escape | `ensure_ascii=False` + un-escaping |
| "revenue **dec**line" parsed as December | month names matched inside words | whole-word month parsing |
| pytest collected 0 tests | Streamlit's AppTest replaced `sys.modules["__main__"]` | tests passed to pytest as an explicit module |

## 10. Known limitations

- **Value-based grounding can't catch meaning errors** (a correct number attached to the wrong claim). Mitigated by guards, `facts`, and the source line — not eliminated.
- **Numbers 0–10 are always allowed** by the grounding check (so "top 5" passes); a hallucinated small value like ₹1.00 would not be caught (documented by a test).
- **A 7B local model is not fully deterministic**; tool choice can vary between runs. The guards and fallbacks make failures end in a correct answer or an honest refusal, but wording varies.
- **SQL guard is regex-based**, not a full SQL parser; the real protection is the read-only connection + table whitelist + row limit. SQLite has no statement timeout.
- **Synthetic data**; currency shown as ₹ by assumption.
- **Single-notebook architecture** (a project requirement here): fine for a take-home, but production code would be split into modules/packages with a separate API process.
- The Kaggle dashboard link is temporary (Cloudflare quick tunnel); latency is ~3–13 s per LLM answer on a T4, slower on CPU.
