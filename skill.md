---
name: cloudcostops
description: A self-hosted FinOps CLI that lets engineers ask plain-English questions about cloud costs. Use this skill whenever the user is working on the CloudCostOps project — including adding or modifying `ccops` commands, changing how the Bridge talks to OptScale or Gauss, adjusting the AI prompt, debugging mock vs real mode, troubleshooting OptScale or Gauss connectivity, fixing API endpoint mismatches, adding new configuration, writing tests, explaining the architecture, or planning the V2 roadmap (vector memory, Slack bot, multi-source data, auto-remediation). Trigger this skill even when the user only mentions one piece — "the Bridge", "the ccops CLI", "the prompt builder", "MOCK_MODE", "Gauss", or "OptScale integration" — because all of those live inside this project.
---

# CloudCostOps

A small, self-hosted system that lets engineers ask questions like
*"why did our bill go up last week?"* from the terminal and get clear,
plain-English answers. It runs entirely inside the user's corporate
network.

## The 30-second architecture

Four components, all inside one trust boundary:

```
   CLI ($ ccops)  ──►  Bridge Service  ──►  OptScale   ──►  Cloud billing APIs
                            │
                            └───►  Gauss (internal LLM)
```

- **OptScale** — open-source FinOps tool. Pulls and stores cloud cost data.
  It is *not* part of this codebase — the user installs it separately.
- **Bridge** — a thin Python service (~500 lines, the only thing this
  project actually builds). Orchestrates everything.
- **Gauss** — an internal LLM (Samsung's). Also *not* part of this codebase —
  the Bridge just calls its API.
- **CLI** (`ccops`) — what engineers type. Lives in this codebase.

## File map

Read this once so you know where everything is.

```
cloudcostops/
├── README.md                   ← top-level orientation (read first time only)
├── SKILL.md                    ← this file
├── docs/                       ← end-user setup guides (01 → 06, in order)
│   ├── 01-prerequisites.md
│   ├── 02-install-optscale.md
│   ├── 03-configure-optscale.md
│   ├── 04-setup-bridge.md
│   ├── 05-run-and-test.md
│   └── 06-troubleshooting.md
├── bridge/                     ← all the actual code
│   ├── ccops                   ← CLI entry point (argparse + rich output)
│   ├── bridge.py               ← orchestrator class
│   ├── optscale_client.py      ← ONLY file that knows OptScale's API
│   ├── gauss_client.py         ← ONLY file that knows Gauss's API
│   ├── prompt_builder.py       ← builds [SYSTEM][CONTEXT][USER] prompt
│   ├── config.py               ← loads .env into a Config class
│   ├── .env.example            ← config template (the real .env is gitignored)
│   ├── requirements.txt
│   ├── README.md               ← code-level orientation
│   └── tests/test_bridge.py    ← pytest tests, all in mock mode
├── scripts/healthcheck.sh
└── .gitignore
```

## Non-negotiable design principles

These are the rules that keep the codebase clean. Preserve them in any change.

1. **One source of truth per external API.** `optscale_client.py` is the *only*
   file that contains OptScale-specific HTTP calls. `gauss_client.py` is the
   *only* file that contains Gauss-specific HTTP calls. If you find yourself
   wanting to add a `requests.get(...)` anywhere else, stop — it belongs in one
   of those two clients.

2. **The CLI is a thin layer.** `ccops` parses arguments and prints results.
   It must not contain business logic. All real work lives in `bridge.py`.

3. **`MOCK_MODE` must keep working.** Every client class has both a real path
   and a mock path. When adding any new API call, also add a mock equivalent
   that returns realistic-looking data. This is what lets the user develop
   without OptScale or Gauss being live, and what lets tests run anywhere.

4. **Secrets live only in `.env`.** Never hardcode tokens or URLs. `.env` is
   gitignored; only `.env.example` (the template) belongs in version control.

5. **`# VERIFY:` comments are sacred.** Any line in `optscale_client.py` or
   `gauss_client.py` that depends on the external API's exact shape (URL path,
   request body, response field) is marked `# VERIFY:`. Preserve these markers
   when editing — they're how the user knows where to check things if real
   mode breaks.

6. **The Bridge audits every question.** Every call to `Bridge._run()` writes
   to `ccops_audit.log`. Don't bypass `_run()` when adding a new command.

## How to do common tasks

### Add a new `ccops` command

Say the user wants `ccops forecast` to project next month's spend.

1. **Add a `handle_forecast` method to `Bridge`** in `bridge.py`. It must:
   - Fetch the data it needs from `self.optscale` (add a new client method if needed — see below).
   - Build a `question` string describing what to ask the AI.
   - Call `self._run(question, data)` and return the result. **Always use `_run` — it does the prompt building, the Gauss call, and the audit log.**

2. **Add the subcommand to `ccops`** following the same pattern as the existing ones:
   - A `cmd_forecast(args, bridge)` function that calls `bridge.handle_forecast(...)` and `show_answer(result)`.
   - A subparser block in `build_parser()` defining `ccops forecast` and its flags.

3. **Add a test** in `tests/test_bridge.py` that calls `bridge.handle_forecast()` in mock mode and asserts the answer is non-empty.

### Add a new OptScale data fetch

Say `handle_forecast` needs historical monthly totals.

1. In `optscale_client.py`, add a method `get_monthly_history(months=12)`:
   - At the top: `if self.mock: return self._mock_monthly_history(months)`.
   - For the real branch, use `self._get(...)` with the endpoint path, and mark the path with a `# VERIFY:` comment so the user knows to confirm it against their OptScale docs.
2. Add a `_mock_monthly_history` method that returns realistic sample data shaped like what OptScale really returns.
3. That's it — `bridge.py` can now call `self.optscale.get_monthly_history()`.

### Change how the AI is prompted (tone, instructions, formatting)

Edit **only** `prompt_builder.py`. The `SYSTEM_PROMPT` string at the top is the
AI's "job description" — adjusting it changes every answer the system gives.
This file is intentionally tiny so the user can iterate on prompt quality
without touching anything else.

### Fix an OptScale or Gauss endpoint mismatch (real mode failing)

This is the most likely real-mode issue. The Bridge's mock mode will keep
working, but real calls error out.

1. Run `./ccops doctor` to confirm the service is reachable.
2. Open the relevant client (`optscale_client.py` or `gauss_client.py`).
3. Find the `# VERIFY:` comments — each marks something that could differ by
   version: the URL path, the request body shape, or where the answer text
   sits in the response.
4. Compare those to the user's actual API docs and adjust.
5. **Do not** introduce ad-hoc HTTP calls elsewhere as a workaround.

### Add a new configuration setting

1. Add it to `.env.example` with a comment explaining what it does.
2. Add it as a class attribute on `Config` in `config.py`, reading from
   `os.getenv(...)` with a sensible default.
3. If it's required in real mode, add a check to `Config.validate()`.
4. Use `Config.YOUR_SETTING` wherever needed.

### Switch a user from mock to real mode

In `.env`: set `MOCK_MODE=false` and fill in `OPTSCALE_URL`, `OPTSCALE_TOKEN`,
`OPTSCALE_ORG_ID`, `GAUSS_URL`, `GAUSS_API_KEY`, `GAUSS_MODEL`. Then run
`./ccops doctor`. If it complains, the message tells you exactly which field
is missing or unreachable.

## How to test changes

Always validate in this order:

1. `pytest` from inside `bridge/`. All tests run in mock mode and exercise the
   core wiring. If these break, the change has a logic bug, not an integration
   bug.
2. `./ccops doctor` — confirms config is valid and (in real mode) that both
   external services are reachable.
3. `./ccops ask "..."` and the other commands — end-to-end smoke test in
   whichever mode the user is in.

When debugging a real-mode issue, always reproduce it in mock mode first.
If mock mode works and real mode doesn't, the problem is integration
(network, auth, endpoint paths) — not the Bridge code.

## Common pitfalls

- **The user edits `.env.example` instead of `.env`.** Settings won't take
  effect. The fix is `cp .env.example .env` then edit `.env`.
- **`./ccops: Permission denied`.** Tell them: `chmod +x ccops`, or run with
  `python3 ccops ...`.
- **`rich` not installed.** Output falls back to plain text — the tool still
  works. `pip install -r requirements.txt` fixes it.
- **OptScale shows no cost data.** Cloud account credentials are wrong, or the
  first import is still running (can take hours). This is an OptScale
  configuration issue, not a Bridge issue — point them to
  `docs/03-configure-optscale.md` and `docs/06-troubleshooting.md`.
- **`401 Unauthorized` from OptScale or Gauss.** The token or API key in
  `.env` is wrong or expired. Regenerate it from the source.
- **The user wants to add a chatbot, web UI, or Slack integration.** That is
  the V2 roadmap, not V1. The whole point is that the Bridge service is the
  reusable backend — a new interface should call `Bridge.handle_*()` exactly
  the way `ccops` does. Don't duplicate the OptScale/Gauss logic into a new
  frontend.

## The V2 roadmap (informational)

If the user mentions any of these, they're future phases that build *on top
of* the current Bridge — none of them require rewriting V1:

- **Vector memory (ChromaDB)** — Bridge stores past queries and retrieves
  relevant history when building prompts. Most natural place: a new
  `memory_client.py` plus a hook in `prompt_builder.py`.
- **Slack / Teams bot** — a new frontend (`slack_bot.py`) that calls the same
  `Bridge.handle_*()` methods. The Bridge itself does not change.
- **Multi-source data** — additional clients (`kubecost_client.py`,
  `steampipe_client.py`) alongside `optscale_client.py`. The Bridge decides
  which to call per question.
- **Auto-remediation** — a new `executor.py` that can apply Cloud Custodian
  policies or Kubernetes VPA changes when the Bridge identifies a fix.

Push back if the user wants to short-circuit V1 to jump straight to one of
these — V1's value is in shipping small and reusing the Bridge.

## When to read more

- For end-user setup (OptScale install, cloud account connection, real-mode
  config): the `docs/` folder, in order 01 → 06. The user follows these; you
  reference them when answering setup questions.
- For code-level orientation when starting a new task: `bridge/README.md`.
- For the top-level "what is this" pitch: `README.md`.

## Tone when helping the user

The user explicitly described themselves as new to this. Default to:

- Concrete next steps over abstract advice.
- Showing the exact file and line to edit, not just "modify the client".
- Explaining *why* a convention exists when introducing it, so the user
  internalizes the design rather than just copying.
- Suggesting mock mode whenever they're stuck — it isolates code issues from
  integration issues and is the fastest path to a working state.
