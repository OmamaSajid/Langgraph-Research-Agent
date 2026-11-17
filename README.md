# Tool-Using Research Agent

A ReAct-style loop where Claude chooses between tools to answer a question with citations.

## Tools
| Tool | Purpose |
|---|---|
| `web_search` | DuckDuckGo search (no key needed) |
| `fetch_page` | Downloads a page, registers it as citable `[S#]` |
| `summarise` | Small-model summary of a fetched source |
| `final_answer` | Ends the loop with a cited answer |

## How it meets the requirements
- **2+ tools**: model decides the next call each step.
- **Hard step limit**: `MAX_STEPS` (default 10). On the last step `tool_choice` forces `final_answer`, so the loop can't run on.
- **Traceable claims**: only *fetched* pages get an ID. `check_citations` rejects answers with uncited sentences or unknown IDs (one retry, then a visible warning). Sources are listed with URLs.
- **Failure handling**: every tool returns an `ERROR:` message (flagged `is_error`) instead of crashing; the agent retries with another query/URL or admits insufficient evidence.

## Two versions
- `agent.py`: plain loop.
- `agent_graph.py`: LangGraph state machine (nodes `agent` -> `tools` -> `agent`, plus `verify`). Same tools; `python agent_graph.py --graph` prints the Mermaid diagram. Step budget = state counter + forced `final_answer` + `recursion_limit`.

## Run
```bash
pip install -r requirements.txt
export ANTHROPIC_API_KEY=sk-...
python agent.py "How does retrieval-augmented generation reduce hallucinations?"
```
Env: `MAX_STEPS`, `AGENT_MODEL`, `SUMMARY_MODEL`.

## Example runs
Paste real output here after running (save with `python agent.py "..." | tee runs/1.txt`):
1. Normal research question: `...`
2. Question with a failing/blocked page (shows error recovery): `...`
3. Obscure question (shows honest "insufficient evidence" / budget stop): `...`

## Free option: Groq
`agent_groq.py` is the same LangGraph agent running on Groq's free tier (Llama 3.3 70B).
```bash
pip install -r requirements-groq.txt
export GROQ_API_KEY=gsk_...      # free at console.groq.com
python agent_groq.py "your question"
```
Pages are truncated to 4k chars to stay inside free-tier token limits.
