# Project instructions for Claude Code

This repo is a learn-and-build project. Read `PLAN.md` first; it is the source of truth for scope, design, and the day-by-day schedule. Read `PROGRESS.md` to see where things stand.

## Who is doing what

- Aditya is a full-stack engineer (Django/DRF, Next.js, PostgreSQL) learning AI engineering through this project.
- **Claude writes** setup, UI, API plumbing, and other routine code.
- **Aditya writes** the ML code (`backend/ml/`) and the LLM code (`backend/assistant/`). For these, do not write the full solution. Explain the idea, let him write it, then review what he wrote.

## How to teach

- Explain in plain, everyday language with a small example first. Introduce a technical term only after the idea it names is clear.
- Keep explanations short. One idea at a time.
- At the end of each day's work, ask two or three questions to check understanding before moving on.
- If his approach is wrong, say so directly and explain why.

## Project rules

- Follow the current day in `PLAN.md` section 11. Do not build ahead or add features outside the plan without asking.
- Never commit price data or secrets. `data/` and `.env` stay in `.gitignore`.
- Download price data once, save it under `data/`, and load the database from that file. Do not re-download in loops.
- No real orders, ever. No public hosting of market data.
- Keep the TradingView attribution logo visible on the chart.
- Results go in the README as they are, good or bad.

## Keeping progress in sync

At the end of every working session:

1. Tick the finished boxes in `PLAN.md`.
2. Add a dated entry to `PROGRESS.md`: what was built, what Aditya learned, any decisions that differ from the plan, open questions, and what comes next.
3. Commit and push.
