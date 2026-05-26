# AlphaAnalyst

**An autonomous equity-research agent that reads SEC filings + investor-relations releases, builds a navigable _LLM-Wiki_ in Obsidian, answers analyst questions with citations, and learns from how its own past calls played out against real stock prices.**

You give it a ticker. It pulls the company's SEC filings (10-K / 10-Q / 8-K) and IR press releases, assembles them into a cross-linked markdown wiki, generates and answers buy-side research questions (every answer grounded in a verbatim quote), scores an investment checklist, and writes a decisive **BUY / HOLD / AVOID** memo in **English and Korean**. Then it records the stock price at that moment — and on every future run it back-tests those past verdicts against live prices and feeds the lessons back into its own prompts.

> ⚠️ **Not investment advice.** Output is automated research for informational and educational purposes only. See the [Disclaimer](#disclaimer).

---

## Features

- 🧭 **LLM-Wiki, not vector RAG** — sources become a human- and LLM-readable Obsidian vault with backlinks; the agent *navigates and reads* it instead of chunk→embed→retrieve. Inspired by Andrej Karpathy's LLM-Wiki idea (see below).
- 🔁 **Self-improving** — snapshots an entry price per report, back-tests every past call against live prices, and injects distilled "lessons" into future prompts.
- 📎 **Citation-grounded answers** — each answer must quote a verbatim span from a real wiki page, or it's marked insufficient.
- 🗒️ **Bilingual memos** — `REPORT.md` (English) + `REPORT.ko.md` (Korean).
- 🔬 **Per-stage observability** — every pipeline stage is checkpointed, logged, self-checking, and independently re-runnable (`--from-step`, `--only-step`, `--list-checkpoints`).
- 🕸️ **Related-company traversal** — follows companies mentioned in IR releases one hop out and pulls their filings too.

---

## How it works

`aa analyze <TICKER>` runs a 14-stage [LangGraph](https://github.com/langchain-ai/langgraph) pipeline:

```
resolve_ticker
   ├─► fetch_edgar
   └─► fetch_ir_releases ─► extract_mentions
          ─► resolve_mentions ─► fetch_related_edgar

[ fetch_edgar + fetch_related_edgar join ] ─► normalize
   ─► build_llm_wiki ─► generate_questions ─► answer_questions
   ─► run_checklist ─► verdict ─► report ─► snapshot_prediction
```

*Two branches run in parallel after `resolve_ticker` — the EDGAR fetch and the IR-scrape → mention-chain — and rejoin at `normalize` before the analysis stages run in sequence.*

| Stage | What it does |
|-------|--------------|
| `resolve_ticker` | Map your input (ticker or company name) to a canonical SEC ticker + CIK |
| `fetch_edgar` | Download latest 10-K, 4× 10-Q, 20× 8-K from SEC EDGAR |
| `fetch_ir_releases` | Scrape the company's IR newsroom (via Playwright) |
| `extract_mentions` → `resolve_mentions` → `fetch_related_edgar` | Find other companies named in IR releases and pull their filings |
| `normalize` | Extract text and split filings into SEC Item sections |
| `build_llm_wiki` | Assemble the Obsidian LLM-Wiki (one page per filing section / release / entity) |
| `generate_questions` | Produce analyst questions, each pointing at likely wiki entry pages |
| `answer_questions` | Navigate the wiki and answer each question with verbatim citations |
| `run_checklist` | Score a 15-item fixed rubric + LLM-derived sector-specific items |
| `verdict` | Combine score + red flags into BUY/HOLD/AVOID (+ catalyst timing if BUY) |
| `report` | Write the English memo, then the Korean translation |
| `snapshot_prediction` | Record the verdict + a live entry price for future back-testing |

---

## The LLM-Wiki approach (inspired by Andrej Karpathy)

Most "chat with your documents" systems use **vector RAG**: chop documents into chunks, embed them into a vector database, and retrieve the top-k nearest chunks at query time. That's fast, but it's opaque (why was this chunk retrieved?), it shreds document structure, and it hides the source from the model.

### Why not vector RAG?

Two limitations make vector similarity a poor fit for equity research specifically:

- **It can't do multi-hop reasoning.** Vector RAG retrieves chunks by one-shot similarity to *your query* and has no notion of links between documents, so it can't *follow a relationship*. Take "NVIDIA names TSMC as a key foundry partner → so how exposed is the thesis to TSMC's capacity?" — the second hop (TSMC's filings) isn't textually similar to the original NVIDIA question, so a similarity search never surfaces it. The LLM-Wiki encodes these relationships as explicit `[[backlinks]]` and dedicated mentioned-company pages, and the pipeline literally walks them (`extract_mentions → resolve_mentions → fetch_related_edgar`). Multi-hop is native here, not an afterthought.
- **Embeddings blur numbers and abbreviations — the tokens that matter most in finance.** Filings are dense with exact figures (revenue, %, bps, dates) and finance shorthand/tickers (FCF, EBITDA, ROIC, YoY, 10-K, NVDA). Embedding models smear precisely these: *"revenue grew 3%"* and *"revenue grew 30%"* map to almost the same vector, and rare acronyms or tickers get washed out — so cosine-similarity retrieval is least reliable exactly where precision is most important. Reading the actual page text preserves every figure verbatim and lets each answer cite a **verbatim span** that is then checked against the page body, so a wrong or hallucinated number can't slip through.

The trade-off is reading more text per question (more tokens) in exchange for transparency, faithful figures, and real multi-hop traversal — acceptable because each ticker's wiki is bounded and navigable.

AlphaAnalyst instead follows **Andrej Karpathy's LLM-Wiki notion**: give the model a curated, human-readable, **cross-linked wiki** that it *reads directly*, leaning on long context windows and explicit links rather than embedding similarity. Knowledge lives as plain markdown a human can open, audit, and edit — and the model traverses it the way a person browses a wiki.

Concretely, for each ticker `build_llm_wiki` produces:

```
vaults/{TICKER}/
├── raw/                       # immutable source documents (the originals)
│   ├── edgar/{accession}/...
│   └── ir/{date}-{slug}.html
└── wiki/
    ├── index.md               # table of contents
    ├── {filing}-item-1a-risk-factors.md   # one page per SEC Item
    ├── {ir-release-slug}.md               # one page per press release
    └── {company-slug}.md                  # backlink pages for mentioned peers
```

Every page carries YAML frontmatter, a one-line **Summary**, a **Sources** link back to the raw file, the section body, and a **`## Related pages`** block of `[[backlinks]]`. The answering loop is wiki navigation, not vector search: each question carries `suggested_entry_pages`, a lightweight router picks the most relevant pages from compact briefs, verbatim snippets are extracted, and the answer must cite spans that actually appear on those pages. The result is transparent, debuggable, and grounded — you can open any claim's citation and read the exact sentence in Obsidian.

---

## The self-improving loop

A BUY/HOLD/AVOID verdict is a falsifiable prediction. AlphaAnalyst treats it that way and learns from being right or wrong:

```
 aa analyze TSLA ─► report ─► snapshot_prediction
                                   │  records {verdict, timing, entry price} → predictions.jsonl
                                   ▼
 (next run, any ticker) ─► back-test ALL past calls vs LIVE prices (Finnhub)
                                   │  LLM grades each call: right / wrong / on-track / too-early
                                   ▼
                          distill bounded "LESSONS" → reflections.md
                                   │
                                   ▼
            injected into the answer / checklist / verdict / report prompts on the next run
```

- Runs automatically at the start of every `aa analyze` (disable with `--no-reflect`), or on demand with **`aa reflect`**.
- Back-tests **every** ticker you've ever analyzed, not just the current one.
- Lessons are written to `reflections.md` (a `<!-- LESSONS -->` block) and fed back into prompts so the agent asks sharper questions, weighs sources better, and analyzes more carefully over time.
- **Optional**: needs a free [Finnhub](https://finnhub.io/) API key. Without it, prices are recorded as `null` and the reflection pass simply no-ops — nothing breaks.
- The fine-tuned question-generation model is deliberately left untouched by injected lessons (to avoid degrading the fine-tune).

---

## Prerequisites

- **Python 3.11+**
- An **OpenAI API key**
- A **SEC user-agent string** with a real contact email (required by [SEC EDGAR's fair-access policy](https://www.sec.gov/os/webmaster-faq#developers))
- **[Obsidian](https://obsidian.md)** (free) — the output is an Obsidian vault; install it to browse the wiki, backlinks, and graph view
- A free **[Finnhub](https://finnhub.io/)** API key to enable the self-improving back-test loop

---

## Installation

```bash
git clone https://github.com/<your-username>/alphaanalyst-llmwiki.git
cd alphaanalyst-llmwiki

python3 -m venv .venv
.venv/bin/pip install -e .

# IR scraping drives a headless browser:
.venv/bin/playwright install chromium

# Put the `aa` command on your PATH globally:
./scripts/install.sh
```

If you skip `scripts/install.sh`, run the CLI as `.venv/bin/aa ...` (or activate the venv with `source .venv/bin/activate` and use `aa ...`).

For development tooling (tests + linter):

```bash
.venv/bin/pip install -e ".[dev]"
```

---

## Configuration

Copy the example env file and fill it in:

```bash
cp .env.example .env
```

| Variable | Required | Purpose |
|----------|:--------:|---------|
| `OPENAI_API_KEY` | ✅ | OpenAI access for all LLM calls |
| `QGEN_MODEL_ID` | ✅ | Question-generation model — see note below |
| `SEC_USER_AGENT` | ✅ | e.g. `aa2 Jane Doe <jane@example.com>` (SEC policy) |
| `FINNHUB_API_KEY` | ⬜ | Enables the self-improving price back-test |
| `AA2_MODEL_FAST` / `_MEDIUM` / `_REASONING` / `_CHECKLIST` / `_ANSWER` / `_ANSWER_ROUTER` / `_REPORT_KO` | ⬜ | Override the model used at each tier |
| `AA2_CHROME_USER_DATA_DIR` / `AA2_CHROME_PROFILE` | ⬜ | Reuse a logged-in Chrome profile for headed IR scraping |
| `AA2_VAULT_ROOT` / `AA2_CACHE_ROOT` / `AA2_REFLECTIONS_PATH` | ⬜ | Where vaults, the download cache, and reflections.md live |
| `AA2_NUM_QUESTIONS`, `AA2_MAX_MENTIONED_COMPANIES`, `AA2_MAX_PARALLEL_QUESTIONS`, … | ⬜ | Limits & tuning (see `.env.example`) |

> **About `QGEN_MODEL_ID`:** the question generator was designed for a *fine-tuned* model, but its output schema is fully specified in the system prompt — so you can point `QGEN_MODEL_ID` at a capable base model (e.g. `gpt-5`) to get started, and swap in your own fine-tune later for higher-quality questions.

---

## Quickstart

```bash
# 1. Check your environment is set up (shows which keys are set)
aa start

# 2. Analyze a company (by ticker or name)
aa analyze NVDA
aa analyze micron

# 3. Read the memo
#    vaults/NVDA/REPORT.md      (English)
#    vaults/NVDA/REPORT.ko.md   (Korean)
```

Useful `analyze` options:

| Flag | Effect |
|------|--------|
| `--ticker MU` | Skip the candidate prompt; use this exact ticker |
| `--non-interactive` | Fail loudly instead of prompting (CI-friendly) |
| `--depth 0` | Primary ticker only (skip the mentioned-company hop) |
| `--headed` | Run the IR-scrape browser headed (uses your Chrome profile) |
| `--from-step <stage>` / `--only-step <stage>` | Resume from / run a single stage using cached checkpoints |
| `--list-checkpoints` | Show which stages are cached |
| `--no-reflect` | Skip the pre-run back-test pass |

Other commands:

```bash
aa reflect                 # back-test all past predictions vs live prices, refresh reflections.md
aa clean MU --all          # delete a ticker's vault AND its cached downloads
aa clean --cache           # purge the entire download cache (all tickers)
aa version
```

`aa clean` granularities: `--logs-only` (default), `--manifests`, `--checkpoints`, `--all` (whole vault + that ticker's cache), `--keep-cache` (with `--all`, keep downloads), `--cache` (purge cached downloads; with a ticker → just that ticker). Every deletion shows a preview and asks for confirmation (or `--yes`).

---

## Viewing results in Obsidian

The output is a real Obsidian vault, so the best way to explore a report — with backlinks and the graph view — is in Obsidian.

1. **Download & install Obsidian** (free): https://obsidian.md
2. Open Obsidian → **"Open folder as vault"**.
3. Select this repo's **`vaults/`** folder — e.g. `…/alphaanalyst-llmwiki/vaults`.
   - ⚠️ Open **`vaults/`**, *not the repo root* — that keeps source code and the download cache out of your vault view.
4. Browse each ticker's `wiki/` pages, click `[[backlinks]]`, open the graph view, and read `REPORT.md` / `REPORT.ko.md`.

---

## Project layout

```
aa2/
├── cli.py                 # `aa` command (analyze / reflect / clean / start / version)
├── graph.py               # LangGraph wiring of the 14 stages
├── steps.py               # @step decorator: checkpoints, post-conditions, run log
├── llm.py                 # OpenAI client + retry/error classification
├── config.py              # .env-driven configuration
├── prices.py              # Finnhub live-quote client
├── ledger.py              # append-only predictions/evaluations ledger
├── reflect.py             # cross-ticker back-test + reflection pass
├── reflection_guidance.py # inject distilled LESSONS into prompts
└── nodes/                 # one module per pipeline stage (edgar, ir_scraper, wiki, answer, …)
vaults/                    # generated per-ticker Obsidian vaults  (gitignored)
cache/                     # SEC download cache, keyed by CIK        (gitignored)
tests/                     # pytest suite
```

---

## Development

```bash
.venv/bin/pip install -e ".[dev]"
.venv/bin/pytest                 # run the test suite
.venv/bin/ruff check aa2 tests   # lint
```

Each pipeline stage is independently testable and re-runnable in isolation; tests mock all network and LLM calls (no live API access required).

---

## Disclaimer

AlphaAnalyst produces **automated, model-generated research** from public filings. It can be wrong, incomplete, or out of date, and its BUY/HOLD/AVOID verdicts are **not financial, investment, legal, or tax advice**. Nothing here is a recommendation to buy or sell any security. Do your own research and consult a licensed professional before making investment decisions.

---

## License

Proprietary © Jinwoo Park. All rights reserved.
