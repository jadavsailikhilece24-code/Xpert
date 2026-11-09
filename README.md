# Xpert Cold Start

Solution for the Xpert Winter Internship 2026 "Cold Start" challenge.
Given a few weeks of one person's WhatsApp history, the program works out:

1. People: each person's circle (spouse, parent, staff, client, ...), their organisations, and who a staff member looks after.
2. What they care about: goals with evidence (area, type, horizon).
3. What's open: open loops (commitment, followup, deadline, question), their status (open, done, dropped), and what the assistant should do (assist, remind, leave).

Everything is written to one file, `output.json`, in the format of `schema.json`.

## Run it

```bash
./run.sh <input_dir> <output.json>
```

`<input_dir>` contains `chats.jsonl`, `messages.jsonl` and `seed.json`. Python 3.9+, standard library only (no installs needed).

On Windows, where `./run.sh` is unavailable:

```
python solution.py <input_dir> <output.json>
```

### Configuration

The program reads an OpenAI-compatible endpoint from the environment (or from a local `.env`, see `.env.example`):

| Variable | Meaning |
|---|---|
| `LLM_BASE_URL` | Base URL of the OpenAI-compatible API |
| `LLM_API_KEY` | API key |
| `LLM_MODEL` | Model id |
| `XPERT_WORKERS` | Parallel calls (default: auto; use 2 on free tiers) |
| `XPERT_MAX_RPM` | Requests per minute limit (default: unlimited) |
| `XPERT_LEAN` | `1` = fewer, larger calls (for strict free tiers) |
| `XPERT_DEADLINE_S` | Stop model calls after this many seconds (default 1560) |
| `LLM_PRICE_IN_PER_M`, `LLM_PRICE_OUT_PER_M` | Optional USD per 1M tokens, for the cost report |

Never commit `.env` or the data. Both are in `.gitignore`.

Check the connection first:

```bash
python tools/ping_llm.py
```

### Dev set

Unzip the dev data into `data/` so that `data/dev/` exists, then:

```bash
python solution.py data/dev out.json
python check.py data/dev out.json
```

### Tests

```bash
python -m unittest discover -s tests -t .
```

The tests run offline with a mock model.

## How it works

Plain Python handles the deterministic work (parsing, ordering, windowing, noise filtering, id validation, de-duplication, schema build). The model only does the semantic steps, in batches and in parallel:

| Stage | File | What it does |
|---|---|---|
| People | `src/people_analyzer.py` | Circle per person; seed circles are kept as given |
| Organisations | `src/organisation_analyzer.py` | Organisations, their type/side/role, and runs/supports links |
| Chats | `src/chat_classifier.py` | Ties every chat to one person or organisation |
| Goals | `src/goal_extractor.py` | Goals, kept only if backed by real message ids |
| Open loops | `src/open_loop_detector.py` | Finds loops with a source message, owner and due date |
| Status | `src/loop_tracker.py` | Open, done or dropped, with the evidence message |
| Where to help | `src/assistance_decider.py` | assist, remind or leave for each open loop |
| Output | `src/output_builder.py`, `src/validator.py` | Builds and validates `output.json` |

Safety rules live in code, not only in prompts:

- A loop needs a verifiable source message and owner.
- Done or dropped needs a later evidence message in the same chat.
- Tasks owed by other people are set to `leave`.
- `assist` without a next step, or `remind` without a valid time, falls back to `leave`.
- Goals without valid evidence are dropped.
- Any failed model call falls back to a conservative default and is logged.

Finished model calls are cached in `.cache/`, so a re-run resumes where it stopped. A time guard stops model calls before the 30-minute limit so the output file is always written. A per-stage token report is written to `ai-logs/usage_report.json`.

## Repository layout

```
run.sh               one-command entry point
solution.py          Python entry point (same as run.sh)
src/                 pipeline stages
tests/               offline tests with a mock model
tools/               helper scripts (ping_llm.py)
check.py             official checker (shape + dev score)
schema.json          output schema
example/             tiny worked example
WRITEUP.md           approach, cost and expected failures
ai-logs/             transcripts of AI-assisted sessions
.env.example         configuration template
```


