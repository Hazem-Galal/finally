# Review of PLAN.md

## Findings

1. **High - The "single Docker command" first-launch contract is not currently actionable.**  
   `PLAN.md:15`, `PLAN.md:85-117`, and `PLAN.md:374-418` describe `frontend/`, `scripts/`, `test/`, top-level `db/`, `Dockerfile`, and `docker-compose.yml` as project structure or launch artifacts. The current repo has `backend/` and `planning/`, but those launch/scaffold files are absent. This makes the plan risky as an agent contract: a frontend, deployment, or QA agent may assume files and directories already exist, while a user following the first-launch story cannot run the app. Either mark this tree explicitly as target state, or add the minimal scaffold now.

2. **High - Required LLM configuration conflicts with the zero-friction first-launch story.**  
   `PLAN.md:15-20` says the user runs one command and immediately sees an AI chat panel ready to assist, but `PLAN.md:121-140` and `PLAN.md:286` make `OPENROUTER_API_KEY` required and depend on a `.env` file that is not committed. If the key is truly required, first launch needs an explicit setup step and a committed `.env.example`. If the demo must run with one command, the backend should degrade to `LLM_MOCK=true` or an unavailable-state chat panel when no key is present.

3. **High - SSE payload shape is inconsistent with the implemented market-data contract.**  
   `PLAN.md:176-180` says each SSE event contains a ticker, price, previous price, timestamp, and direction, which reads like one event per ticker. The implemented endpoint batches all prices into one SSE `data:` payload keyed by ticker (`backend/app/market/stream.py:30-33`, `backend/app/market/stream.py:80-83`). Frontend agents need a single canonical payload shape. Update the plan to show the exact batch JSON shape, or change the endpoint contract before frontend work starts.

4. **Medium - The UI asks for daily change %, but market data only defines tick-to-tick change.**  
   `PLAN.md:355` requires a watchlist "daily change %". The implemented `PriceUpdate.change_percent` is calculated from `previous_price`, i.e. the prior update, not prior close or session open (`backend/app/market/models.py:23-28`). For simulator data, no day-open baseline is specified. Decide whether the UI should show tick change, session change since page load, or true daily change, then add the needed field/source.

5. **Medium - Database ownership wording contradicts the listed schema.**  
   `PLAN.md:196` says all tables include `user_id`, but `users_profile` is then defined with `id` as the user identifier and no `user_id` field (`PLAN.md:198-201`). This is small but important for agents implementing repositories and joins. Reword to "all user-owned tables except `users_profile` include `user_id`" or add a `user_id` column consistently.

6. **Medium - Chat action result shape is underspecified.**  
   `PLAN.md:301-319` defines the LLM's requested action schema, and `PLAN.md:328` says validation failures are included in the chat response, but there is no response schema for executed trades, rejected trades, watchlist failures, or partial success. `PLAN.md:361` also expects inline confirmations in the frontend. Add a backend API response schema such as `message`, `actions_requested`, `actions_executed`, and `errors` so backend and frontend agents do not invent incompatible formats.

7. **Medium - Portfolio cost-basis behavior needs one more level of precision.**  
   `PLAN.md:210-226` defines `positions.avg_cost` and `trades`, while `PLAN.md:256-261` defines trade endpoints, but the plan does not specify how partial sells affect `avg_cost`, whether zero-quantity rows are deleted, rounding rules for fractional shares, or how to handle tiny float residues. These choices directly affect P&L, tests, and chat validation. Document the intended accounting rules before portfolio implementation.

8. **Low - `.env` loading is ambiguous outside Docker.**  
   `PLAN.md:140` says the backend reads `.env` from the project root. The current market-data factory reads `MASSIVE_API_KEY` from `os.environ` (`backend/app/market/factory.py:24`) and the backend dependencies do not include a dotenv/settings loader. Docker `--env-file` satisfies this, but local `uv run` development will not unless a loader is added. Clarify "Docker passes `.env` via `--env-file`" or add explicit application settings loading.

## Open Questions

- Is `PLAN.md` intended to describe the current repository state or the target end state? The current wording mixes both, which is the source of several risks above.
- Should the production demo be usable without an OpenRouter key, with chat mocked or disabled, or is key setup a mandatory part of the course exercise?

## Summary

The plan is directionally coherent and the completed market-data slice mostly matches it. The highest-value fixes are to separate target-state structure from existing files, make first-run setup honest, and lock down JSON contracts for SSE and chat before frontend/backend agents build against different assumptions.
