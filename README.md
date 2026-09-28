# Multi-Agent Orchestration: Research Agent

A FastAPI service that turns a topic into a researched article using a **supervisor-led multi-agent workflow** built with [LangGraph](https://langchain-ai.github.io/langgraph/). A supervisor LLM routes work between a **Researcher** agent (web search via Tavily) and a **Writer** agent (article generation via Groq), and decides when the job is finished.

---
## 🚀 Live Demo
https://multi-agent-orchestration-research-agent.onrender.com/
> Note: The app is hosted on a free tier, so the first request may take 30-60 seconds while the server wakes up.

## Table of Contents

- [Features](#features)
- [Architecture](#architecture)
- [Tech Stack](#tech-stack)
- [Project Structure](#project-structure)
- [Getting Started](#getting-started)
- [Configuration](#configuration)
- [API Reference](#api-reference)
- [Docker](#docker)
- [How It Works](#how-it-works)
- [Known Limitations](#known-limitations)
- [Roadmap](#roadmap)

---

## Features

- **Supervisor pattern**: a central agent decides which worker acts next (`researcher`, `writer`, or `FINISH`).
- **Live web research**: the Researcher agent queries the web through the Tavily search API.
- **Article generation**: the Writer agent produces a clear, structured article from the gathered research.
- **Prompt-injection guardrails**: the supervisor prompt ignores attempts to override its role and falls back to `FINISH` on suspicious input.
- **Safe routing fallback**: any invalid supervisor output is treated as `FINISH`.
- **Isolated runs**: each request gets its own `thread_id`, so runs do not share state.
- **Built-in web UI**: `index.html` is served at `/` for a simple browser front end.
- **Container-ready**: includes a Dockerfile that respects the `PORT` environment variable.

---

## Architecture

```
                  ┌──────────────┐
   user topic --> │  Supervisor  │ <-----------------┐
                  └──────┬───────┘                   │
             ┌───────────┼─────────────┐             │
             v           v             v             │
        ┌──────────┐ ┌────────┐   ┌────────┐         │
        │Researcher│ │ Writer │   │ FINISH │         │
        │ (Tavily) │ │ (Groq) │   │  (END) │         │
        └────┬─────┘ └───┬────┘   └────────┘         │
             └───────────┴───────────────────────────┘
               each worker returns control to the supervisor
```

**Flow**

1. The supervisor reads the conversation and outputs one word: `researcher`, `writer`, or `FINISH`.
2. If no research exists yet, it routes to the **Researcher**, which searches for the user's topic and stores the findings in shared state (`research_data`).
3. Control returns to the supervisor, which routes to the **Writer**.
4. The Writer drafts the final article from the research.
5. The supervisor sees the finished article and returns `FINISH`. The API returns the last message as the result.

**Shared graph state**

| Field | Purpose |
|---|---|
| `messages` | Conversation history (LangGraph `add_messages` reducer) |
| `next` | The supervisor's routing decision |
| `research_data` | Scratchpad holding the Researcher's findings |

---

## Tech Stack

| Layer | Technology |
|---|---|
| API server | FastAPI + Uvicorn |
| Orchestration | LangGraph (`StateGraph`, `MemorySaver`) |
| LLM | Groq (`openai/gpt-oss-120b`) via `langchain-groq` |
| Search tool | Tavily via `langchain-community` |
| Config | `python-dotenv` |
| Frontend | Single-page `index.html` |
| Deployment | Docker (`python:3.12-slim`) |

---

## Project Structure

```
.
├── main.py             # FastAPI app, LangGraph agents, routing, /research endpoint
├── index.html          # Frontend served at "/"
├── requirements.txt    # Python dependencies
├── Dockerfile          # Container build
├── .dockerignore
├── .gitignore
└── README.md
```

---

## Getting Started

### Prerequisites

- Python 3.12 (matches the Dockerfile)
- A [Groq API key](https://console.groq.com/)
- A [Tavily API key](https://tavily.com/)

### Installation

```bash
# 1. Clone the repository
git clone https://github.com/MonikaK28/Multi-agent-orchestration-Research-agent.git
cd Multi-agent-orchestration-Research-agent

# 2. Create and activate a virtual environment
python -m venv venv
source venv/bin/activate        # Windows: venv\Scripts\activate

# 3. Install dependencies
pip install -r requirements.txt
```

### Configuration

Create a `.env` file in the project root:

```env
GROQ_API_KEY=your_groq_api_key
TAVILY_API_KEY=your_tavily_api_key
```

> Never commit your `.env` file. It is already covered by `.gitignore`.

### Run the app

```bash
uvicorn main:app --reload
```

Open <http://localhost:8000> for the web UI, or interactive API docs at <http://localhost:8000/docs>.

---

## API Reference

### `GET /`

Serves the web front end (`index.html`).

### `POST /research`

Runs the full supervisor, researcher and writer workflow for a topic.

**Request body**

```json
{
  "topic": "Impact of AI on healthcare diagnostics"
}
```

**Example**

```bash
curl -X POST http://localhost:8000/research \
  -H "Content-Type: application/json" \
  -d '{"topic": "Impact of AI on healthcare diagnostics"}'
```

**Response**

```json
{
  "topic": "Impact of AI on healthcare diagnostics",
  "article": "...the generated article..."
}
```

---

## Docker

```bash
# Build
docker build -t research-agent .

# Run (pass keys at runtime, do not bake them into the image)
docker run -p 8000:8000 \
  -e GROQ_API_KEY=your_groq_api_key \
  -e TAVILY_API_KEY=your_tavily_api_key \
  research-agent
```

The container starts Uvicorn on `0.0.0.0` and reads the port from the `PORT` environment variable (default `8000`), which makes it compatible with platforms such as Render and Railway.

---

## How It Works

**Supervisor.** The supervisor prompt lists the available workers and the routing rules. It must answer with exactly one word. Anything outside `researcher`, `writer`, `FINISH` is coerced to `FINISH`. The prompt also tells the model to treat conversation content as data, never as new instructions.

**Researcher.** Takes the user's original request, runs a Tavily search, formats the results into text, and stores them both as a message and in `research_data`.

**Writer.** Combines the user's request with `research_data` and asks the LLM for a well-structured article.

**Graph wiring.** The supervisor is the entry point. Conditional edges route to a worker or to `END`, and both workers have an edge back to the supervisor. The graph is compiled with an in-memory checkpointer (`MemorySaver`).

---

## Known Limitations

- Only **one** search result is retrieved per query, so articles rest on limited sources.
- Sources and URLs are not included in the output, so there are no citations.
- The endpoint has **no authentication or rate limiting**, and CORS is open to all origins (`*`). Restrict both before any public deployment.
- Errors from Groq or Tavily are not handled gracefully yet.
- The supervisor relies on free-text output rather than a structured schema, and there is no explicit step limit beyond LangGraph's recursion limit.
- Memory is in-process only and is not persisted between restarts.

---

## Roadmap

- [ ] Retrieve multiple search results and return source URLs (citations)
- [ ] Structured supervisor output (`with_structured_output`) and a max-steps guard
- [ ] Add a **Critic/Reviewer** agent with a revision loop
- [ ] Add a **Planner** agent that splits topics into sub-questions and searches in parallel
- [ ] Stream agent progress to the UI
- [ ] Error handling, API-key auth and rate limiting
- [ ] Unit tests with mocked LLM and search calls
- [ ] Migrate to the `langchain-tavily` package (the community tool is deprecated)

---

## Contributing

Issues and pull requests are welcome. Please open an issue first to discuss significant changes.

## License

No license has been specified yet. Consider adding one (for example MIT) before sharing the project publicly.
