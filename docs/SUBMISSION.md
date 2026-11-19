Ankur Chakraborty

[GitHub Repository Link]  https://github.com/<your-username>/ai-bi-agent
[Demo Video Link]         <your video link>
[Model / Provider]        Qwen2.5-7B-Instruct, run locally via Ollama (llama.cpp supported) — free, no API keys
[Database]                PostgreSQL (SQLite fallback via SQLAlchemy)
[Technologies]            Python, pandas, SQLAlchemy, PostgreSQL, FastAPI, Pydantic, Streamlit, Plotly, Ollama, pytest (ipytest), Kaggle GPU

[Brief description]
An AI business analyst for e-commerce data. Users ask questions in plain English through a Streamlit dashboard or a FastAPI endpoint. A locally hosted LLM chooses from a controlled set of validated tools (sales, product and customer metrics, charts, a multi-step "why did revenue change" investigation, and a sandboxed read-only SQL tool). All numbers are computed in code; the LLM only interprets the question and words the answer, and every number in its answer is checked against the tool results — if anything doesn't match, a deterministic, data-derived answer is shown instead. Code-level guards, fallbacks and a rule-based router handle LLM mistakes and outages, and unsupported questions are refused clearly. The project includes data cleaning/validation, KPIs and charts, AI-generated insights, and 71 tests, including AI questions verified against independently computed ground truth.
