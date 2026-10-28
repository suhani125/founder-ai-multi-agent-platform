# Founder AI

A small multi-agent project for startup founders. There are three agents (hiring, feedback, support) and an orchestrator that sits on top so you can talk to all of them from one chat box.

I built the orchestrator and the hiring agent. The support and feedback agents are my friend Sakshi's work (edit this line if that's not right).

## What's in here

- **Hiring agent**: you upload resumes (PDF) and it scores each candidate on skills, experience, education, projects and culture fit, with a short explanation.
- **Feedback agent**: you upload customer feedback and it works out sentiment, common complaints, feature requests and topics. There's a dashboard and a chat where you can ask questions about your data.
- **Support agent**: you upload documents (policies, FAQs) and it answers questions from them. If the documents don't have the answer it can search the web or create a ticket.
- **Orchestrator**: takes one message in plain English, works out which agent(s) should handle it, calls them, and gives you back one answer.

Each agent is its own app with its own frontend, backend and database. They don't know about each other. Only the orchestrator talks to all three.

## How it fits together

```
          orchestrator console (browser)
                     |
                     v
              orchestrator :8000
            (FastAPI + LangGraph)
           /         |          \
          v          v           v
   hiring :8002  feedback :8001  support :8003
        |            |             |
    postgres     postgres+redis  postgres+redis
```

Each agent also has its own frontend on 3002 (hiring), 3001 (feedback) and 3003 (support), so you can use them without the orchestrator too.

## The orchestrator

This is the part I spent the most time on, so it gets the longest section.

It's a FastAPI app with no database. When a message comes in at `POST /api/orchestrate`, a LangGraph agent reads it. That agent has a set of tools, and each tool is really just an HTTP call to one of the other agents (for example "list resumes" or "search the knowledge base"). The model decides which tools to call, sometimes more than one in a row, then writes the final answer.

The response looks like this:

```json
{
  "answer": "...",
  "tools_used": ["hiring_list_resumes", "hiring_analyze"],
  "thread_id": "..."
}
```

Send the same `thread_id` back on your next message and it remembers the conversation.

**Auth.** The three agents each have their own users, so the orchestrator can't just log in for you. Instead you log in to each agent and pass that agent's token in a header:

- `X-Hiring-Token`
- `X-Feedback-Token`
- `X-Support-Token`

The orchestrator forwards the right one when it calls that agent. If you don't send a token for an agent, that agent just isn't available for the request.

**Config** lives in `orchestrator/.env`:

```env
FEEDBACK_AGENT_URL=http://feedback-backend:8000
HIRING_AGENT_URL=http://hiring-backend:8000
SUPPORT_AGENT_URL=http://support-backend:8000

LLM_PROVIDER=groq
GROQ_API_KEY=your_key_here
GROQ_MODEL=some-model-your-key-can-use
SUPERVISOR_MODEL=some-model-your-key-can-use

ALLOWED_ORIGINS=["http://localhost:3000", "http://127.0.0.1:3000"]
```

Model names get retired by the provider every so often. If you suddenly get `model_not_found`, ask Groq which models your key can use and change the two model lines:

```powershell
Invoke-RestMethod -Uri "https://api.groq.com/openai/v1/models" -Headers @{ Authorization = "Bearer $env:GROQ_API_KEY" }
```

There's also a small test page, `orchestrator-console.html`. Log in to the agents in the sidebar and chat. Serve it from an address that's in `ALLOWED_ORIGINS`:

```
python -m http.server 3000
```

then open http://127.0.0.1:3000/orchestrator-console.html.

## The hiring agent

You upload PDF resumes and the backend pulls the text out with PyMuPDF. Then a small LangGraph pipeline goes through each resume: one step pulls out skills and experience, a second step combines everything into scores and a recommendation. It uses a ChromaDB collection of hiring rubrics for context.

Endpoints: `POST /auth/signup`, `POST /auth/login`, `POST /hiring/upload-resumes`, `GET /hiring/resumes`, `POST /hiring/analyze`.

Stack: FastAPI, SQLAlchemy, PostgreSQL, PyMuPDF, LangChain/LangGraph, ChromaDB, Next.js.

## Tech used

Python, FastAPI, SQLAlchemy, PostgreSQL, Redis, LangChain, LangGraph, Groq, ChromaDB, FAISS, Next.js, Docker Compose.

## Folder layout

```
founder-ai-feedback-agent/   feedback agent
founder-ai-final/            hiring agent
support_agent/               support agent
orchestrator/                orchestrator
orchestrator-console.html    test page for the orchestrator
docker-compose.yml
```

## Running it

You need Docker Desktop, Git and a Groq API key.

1. Clone the repo.

2. Every service has its own `.env.example`. Copy each one to `.env` and put your key in:

```
founder-ai-feedback-agent/backend/.env.example
founder-ai-final/backend/.env.example
support_agent/.env.example
orchestrator/.env.example
```

3. From the folder with `docker-compose.yml`:

```
docker compose up --build
```

The first build takes a while because of the ML libraries.

Ports:

| Service | Port |
|---|---|
| Orchestrator | 8000 |
| Feedback backend / frontend | 8001 / 3001 |
| Hiring backend / frontend | 8002 / 3002 |
| Support backend / frontend | 8003 / 3003 |

The login route is different for each agent, which tripped me up for a while:

- hiring: `/auth/login`
- feedback: `/api/v1/auth/login`
- support: `/api/auth/login`

## Things that went wrong (and might go wrong for you)

- **"Failed to fetch" in the browser.** Usually the backend's CORS list doesn't include the page you're on. But a backend that crashes with a 500 shows up as the same error in the browser, so check `docker compose logs <service>` before changing CORS.
- **Changed a `.env` and nothing happened.** `docker compose restart` doesn't reload `.env`. Use `docker compose up -d <service>`.
- **`model_not_found`.** See the note about model names above.
- **Resumes show "no such file" after a rebuild.** The uploads folder isn't a persistent volume yet, so rebuilding the container wipes the files.
- **`ERR_CONNECTION_REFUSED`.** Docker Desktop or the container isn't running.

## Things I'd still like to do

- One login for everything instead of signing in to each agent
- Persistent volumes for uploads
- Tests
- Better logging
- More agents later (meetings, finance, email)

## Credits

- Orchestrator and hiring agent: Suhani ([@suhani125](https://github.com/suhani125))
- Support and feedback agents: Sakshi Agrawal ([@sakshiagra1903](https://github.com/sakshiagra1903))

Built for learning and demo purposes.
