# Sales_RAG_SQL_agent

A conversational SQL agent built on top of the Northwind sales database. Ask questions in plain English, get answers back — or flip to SQL mode and see the exact query that ran. Built with LangChain, ChromaDB, Groq, and Streamlit.

Currently deployed and being used by sales, ops, and analytics teams to self-serve data questions without touching the database directly.

---

## why this exists

The same five questions were being asked every week. Top customers by revenue. Category performance. Rep leaderboards. Pending orders. Someone always had to stop what they were doing, write a query, and send back a screenshot.

The other option was giving everyone DB access, which meant expensive queries, schema confusion, and no guardrails on what people were running.

This sits in the middle. It understands the schema, knows what business terms like "top customer" or "pending order" mean in SQL, and generates accurate queries without anyone needing to know what a JOIN is.

---

## how it works

Two modes in the UI:

**Insight mode** — ask a question, get a plain English answer with an auto-generated chart if the result is tabular.

**SQL mode** — same question, but the generated SQL is returned alongside the answer. Useful for analysts who want to verify the query or adapt it.

Under the hood, before the LLM touches the user's question, a retrieval step pulls relevant schema docs and business glossary entries from a ChromaDB vector store. That grounded context gets injected into the prompt before SQL is generated — which is what makes it accurate on complex multi-table queries rather than guessing column names.

The full pipeline per query:

```
user question
    → ChromaDB retrieval (schema docs + business glossary)
    → enriched prompt sent to LangChain SQL agent
    → agent inspects schema, generates SQL, executes it
    → result parsed and formatted
    → answer + optional chart returned to UI
```

---

## stack

- **LangChain** — SQL agent orchestration, ReAct loop, tool use
- **ChromaDB** — vector store for schema embeddings and business glossary
- **Sentence Transformers** — all-MiniLM-L6-v2 for embedding schema docs locally (no API cost)
- **Groq** — LLM inference, llama-3.3-70b-versatile
- **SQLite** — database layer for the demo (Northwind dataset)
- **Streamlit** — chat UI, sidebar schema explorer, mode toggle
- **Plotly** — auto-generated charts in insight mode
- **Docker** — containerized for deployment

---

## project structure

```
sales-rag-agent/
├── app.py                   # Streamlit UI entry point
├── northwind.db             # SQLite database (Northwind dataset)
├── glossary.json            # Business term → SQL pattern mappings
├── schema_docs/             # Per-table markdown docs embedded into ChromaDB
│   ├── Customer.md
│   ├── Order.md
│   ├── OrderDetail.md
│   ├── Product.md
│   ├── Employee.md
│   ├── Category.md
│   ├── Supplier.md
│   └── Shipper.md
├── requirements.txt
├── Dockerfile
├── docker-compose.yml
└── .streamlit/
    └── secrets.toml         # API keys — never committed
```

---

## the RAG layer

This is the part that separates it from a basic SQL chatbot.

Each table has a markdown doc in `schema_docs/` that describes it in business terms — what the table represents, what each column means, how to join it, and what the common use cases are. The `glossary.json` maps business terms to their SQL equivalents: "top customer" maps to an ORDER BY revenue DESC pattern, "pending orders" maps to WHERE ShippedDate IS NULL, and so on.

All of this gets embedded into ChromaDB on startup using a local sentence transformer model. When a query comes in, the most semantically relevant docs are retrieved and injected into the prompt before the LLM writes any SQL. The LLM is working from grounded documentation rather than guessing.

---

## running locally

**Prerequisites:** Python 3.10+, a Groq API key (free at console.groq.com)

```bash
git clone https://github.com/yourusername/sales-rag-agent.git
cd sales-rag-agent

python -m venv venv
source venv/bin/activate

pip install -r requirements.txt

export GROQ_API_KEY=gsk_your_key_here

streamlit run app.py
```

Open `http://localhost:8501`

---

## running with Docker

```bash
export GROQ_API_KEY=gsk_your_key_here
docker-compose up --build
```

---

## example queries

```
Who are the top 5 customers by total revenue?
Which product categories generate the most revenue?
Which sales rep closed the most deals?
Show me all pending orders with customer names
Which products are low on stock?
What is the average discount rate by sales rep?
How many orders were placed in 2016?
Which customers have placed more than 10 orders?
```

Switch to SQL mode on any of these and you'll see the exact query that ran.

---

## deployment

**Streamlit Community Cloud** — push to GitHub, connect the repo at share.streamlit.io, add GROQ_API_KEY under Settings → Secrets. Takes two minutes.

**Docker on a VM** — run docker-compose up -d on any Linux box. Point it at your internal database by swapping the SQLite connection string for a Postgres URI in app.py. Use a read-only service account — the agent only needs SELECT.

**Google Cloud Run** — stateless container, autoscales to zero, works well for burst usage patterns. Build the image, push to Artifact Registry, deploy with gcloud run deploy.

---

## environment variables

```
GROQ_API_KEY=          # required — get free key at console.groq.com
```

---

## known limitations

- Complex analytical queries with multiple nested aggregations can sometimes trip the agent. Specific, direct questions work better than vague ones.
- The DB connection uses a read-only pattern by design. If you point this at a production database, make sure the user account has SELECT permissions only.
- Chart generation is best-effort. Works well for rankings and aggregations, falls back to a plain text answer for anything it cannot confidently visualize.
- Response time depends on Groq API latency — typically 3 to 6 seconds per query.

---

## things to add in v2

- Query result caching for repeated questions
- User feedback loop to flag bad answers
- Fine-tuned retrieval using logged Q&A pairs as few-shot examples
- Support for uploading schema docs through the UI
- Per-user query history

---

## license

MIT
