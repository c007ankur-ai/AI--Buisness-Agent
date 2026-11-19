# Demo video script (~5–6 minutes)

Follows §16 of the brief: startup → dashboard → AI question → tool execution → insight → chart → unsupported/invalid.
**Before recording:** Kaggle notebook with GPU on, **Run All** finished, tests green, dashboard link open. Ask each question once beforehand so the model is warm.

| # | Time | Show | Say (roughly) |
|---|---|---|---|
| 1 | 0:00–0:30 | README top table | "An AI business analyst over e-commerce data. Everything is free and local: Qwen2.5-7B on Ollama, PostgreSQL, FastAPI, Streamlit." |
| 2 | 0:30–1:15 | **Startup:** notebook outputs: `Database: postgresql`, the cleaning report, `Ollama ready`, the API health JSON, the dashboard link | "The notebook starts Postgres, generates and cleans the data — here's what cleaning removed — loads it, starts the local LLM, the API and the dashboard." |
| 3 | 1:15–2:00 | **Dashboard tab**: KPI cards; switch sidebar to 2025-06 | "Revenue, orders, customers and AOV. Picking June shows the drop versus May: revenue −42.4%. The four charts come live from the API." |
| 4 | 2:00–2:40 | **Ask:** "What were the top 5 products by revenue?" → open *Tool calls & data* | "The LLM chose the get_top_products tool; the numbers come from SQL, and the source line shows exactly what was run." |
| 5 | 2:40–3:40 | **Multi-step:** "Why did revenue decline last month?" | "This runs a six-step investigation in code: half the drop is fewer orders, half is smaller orders; Electronics fell 59%, Beauty grew; returning customers dropped from 78 to 50. The LLM only explains verified results." |
| 6 | 3:40–4:10 | **Chart:** "Show me monthly revenue as a chart." | "The chart is generated on request from the live data." |
| 7 | 4:10–4:35 | **SQL fallback:** "Which city has the most customers?" → open the SQL expander | "No fixed tool covers cities, so the LLM wrote SQL — validated, read-only, row-limited." |
| 8 | 4:35–5:00 | **Insights tab** → Generate | "Facts are computed in code; the LLM turns them into insights and every number is checked." |
| 9 | 5:00–5:30 | **Unsupported/invalid:** "What's the weather in Kolkata?", then "revenue for 2030-01" | "Off-topic questions are refused clearly; invalid months return a clear error, not a made-up answer." |
| 10 | 5:30–6:00 | Last notebook cell: `All tests passed` | "71 tests, including AI questions checked against independently computed answers, a simulated database outage, and replays of real LLM failures I found and fixed." |

Tips: record at 1080p; zoom the browser to ~110%; if an LLM answer is slow, keep talking over the spinner; don't hide the "mode" and "source" captions — they are the proof of grounding.
