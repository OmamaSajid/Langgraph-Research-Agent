# LangGraph Research Agent

A tool-using research agent built as a **LangGraph state machine**. It answers a research question by choosing between tools (web search, fetch page, summarise), cites only pages it actually fetched, and stops at a hard step budget


## How it works

```mermaid
flowchart TD
    S([START]) --> A[agent]
    A -- tool call --> T[tools]
    T --> A
    A -- final_answer --> V[verify]
    V -- ok, or one retry used --> E([END])
    V -- uncited claims, first time --> A
```

| Node | Job |
|---|---|
| `agent` | The LLM reads the shared state and **decides the next tool** (or decides to finish). |
| `tools` | Runs the chosen tool. A failure comes back as an `ERROR:` message the agent can read and react to. |
| `verify` | Rejects any answer with a sentence that has no `[S#]` citation, or that cites a source never fetched. It gets one retry, then the answer is shown with a warning. |

**Tools the agent chooses between**

| Tool | Purpose |
|---|---|
| `web_search` | DuckDuckGo search (no API key needed). Snippets are not citable. |
| `fetch_page` | Downloads a page and registers it as a citable source `[S1]`, `[S2]`, ... |
| `summarise` | Small-model summary of a fetched source. |
| `final_answer` | Ends the run with a cited answer. |

## Requirements checklist

| Requirement | How it is met | Seen in |
|---|---|---|
| At least two distinct tools the agent can choose between | `web_search`, `fetch_page` and `summarise`. The model picks one at every step. | Runs 1 and 2 use all three |
| A hard step limit | `MAX_STEPS` = 8. The last step offers no tools: the agent writes the answer from pages already fetched. LangGraph's `recursion_limit` is a second safety net. | Every run ends at `step 8/8` |
| Every claim traceable to a fetched source | Only fetched pages get an ID. `verify` checks that every sentence cites a real one, and the URLs are listed under **Sources**. | Runs 1 and 2 |
| Graceful handling when a tool fails or returns nothing | Tool errors become messages the agent reads, invalid model output is retried inside the step budget, and with no evidence the agent says so instead of inventing facts. | Runs 2 and 3 |

## Example runs

Each run below is a real, unedited terminal capture. Together they show the agent choosing tools, citing sources, handling failures, and stopping at the step limit.

| Run | Question type | Requirement it demonstrates |
|---|---|---|
| 1 | Normal research question | Tool choice, citations traceable to fetched pages, hard step limit |
| 2 | A page that doesn't exist | Graceful handling of a failed tool call |
| 3 | A topic with no information online | Graceful handling of empty results, no invented facts |

### Run 1: normal research question
```powershell
python agent_groq.py "How do black holes form?"
```

![Run 1 output](images/run1.png)

**What this shows**
- **Tool choice:** the agent decided its own order: two `web_search` calls, then `fetch_page` on Wikipedia, `summarise` on that page, and two more fetches. Nothing about the order was scripted.
- **A failed fetch handled:** the Britannica page (step 5) was never registered as a source, so there is no `[S#]` for it. The agent moved on to astronomy.com instead of crashing.
- **Error recovery:** at step 7 the model tried to call a tool that doesn't exist (`repo_browser.open_file`). The call was rejected, the agent recovered, and the run still finished.
- **Step limit:** the run ends at `step 8/8`. On the last step no tools are available, so the agent writes the answer from the pages it already fetched.
- **Traceable claims:** every sentence ends in `[S1]` or `[S2]`, and both map to pages that were really fetched. The Sources list gives their URLs.

### Run 2: a tool call fails
```powershell
python agent_groq.py "Fetch https://this-site-does-not-exist-12345.com/article and summarise what it says about black holes"
```

![Run 2 output](images/run2.png)

**What this shows**
- **Failed fetch:** step 1 tries the fake website, and the fetch fails. The agent does not crash.
- **Recovery:** it searches for an alternative and fetches a NASA page, which is registered as `[S1]`.
- **Retry inside the budget:** step 6 is another invalid model output (a parse error). The run retries and carries on, and the mistake counts against the 8 steps.
- **Honest answer:** the final answer says the requested page could not be retrieved, and cites only the NASA page it actually fetched.

### Run 3: search finds nothing useful
```powershell
python agent_groq.py "What did Zorblax Quentin publish about quantum turnips in 2031?"
```

![Run 3 output](images/run3.png)

**What this shows**
- **Persistence, then a stop:** the agent tried seven different search queries before the step budget ran out. It could not loop forever.
- **No invented facts:** it says the sources do not contain this information. The Sources list is empty because nothing was fetched.
- **The `Warning: unverified` line is the citation check working.** That sentence has no `[S#]` because there was nothing to cite, so the checker flagged it. For an honest "no evidence" answer, this is the intended behaviour.

## Run it yourself

Requires Python 3.10+ and a free Groq API key from [console.groq.com](https://console.groq.com).

**Windows (PowerShell)**
```powershell
python -m venv venv
venv\Scripts\Activate.ps1
pip install -r requirements-groq.txt
$env:GROQ_API_KEY="gsk_your_key_here"
python agent_groq.py "How do black holes form?"
```

**Mac / Linux**
```bash
python -m venv venv
source venv/bin/activate
pip install -r requirements-groq.txt
export GROQ_API_KEY=gsk_your_key_here
python agent_groq.py "How do black holes form?"
```

Print the graph as a Mermaid diagram: `python agent_groq.py --graph`

Each run is also saved to `runs/last_run.txt` (this file is not committed).

**Settings** (environment variables)

| Variable | Default | Meaning |
|---|---|---|
| `MAX_STEPS` | `8` | The hard step budget |
| `AGENT_MODEL` | `openai/gpt-oss-120b` | Model that decides which tool to call |
| `SUMMARY_MODEL` | `openai/gpt-oss-20b` | Model used by the `summarise` tool |

## Files

| File | What it is |
|---|---|
| `agent_groq.py` | The LangGraph agent (runs on Groq's free tier) |
| `requirements-groq.txt` | Python dependencies, including `langgraph` |
| `images/` | Screenshots of the three example runs |
| `WALKTHROUGH.md` | Script for the recorded walkthrough |

## Limitations

- `verify` confirms every sentence cites a page that was really fetched. It does **not** check that the page actually says it. Pages are cut to 4,000 characters, so a cited claim could come from the model's background knowledge.
- The small free models sometimes call tools that don't exist. The agent recovers, but each mistake uses up one of the 8 steps.
- Free-tier rate limits may mean waiting a minute between runs.
