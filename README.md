# DeepBridge

A self-hosted coding-agent console that runs on DeepSeek's free web chat instead of the API.
It opens a real Firefox window (via [Camoufox](https://github.com/daijro/camoufox)), points it
at chat.deepseek.com, and drives the consumer UI like an LLM endpoint: prompts go into the
composer, streamed replies get captured, and a local loop feeds them through a text-based
tool-calling protocol so the model can read and edit files, grep, and run shell commands in a
sandboxed workspace.

No API key. No billing. Just browser automation.

> **Warning:** automating chat.deepseek.com's web UI rather than using an official API almost
> certainly violates DeepSeek's Terms of Service. This is a personal/local experiment, not a
> product backend. Your account can be flagged or banned. Use at your own risk.

## How it works

```
 browser ──HTTP──▶ Flask app (app.py, :8080)
                      │
                      ├─▶ BridgeWorker ── one dedicated thread that owns all
                      │     │             Playwright calls (thread-affinity)
                      │     ▼
                      │   DeepSeekBridge ── Camoufox/Firefox ──▶ chat.deepseek.com
                      │                         │
                      │                         ├─ SSE capture of /chat/completion (preferred)
                      │                         └─ DOM scraping fallback + stability window
                      │
                      ├─▶ tools.py ........ sandboxed tool execution (one folder per session)
                      ├─▶ live_progress.py  worker writes progress, poll endpoint reads it
                      └─▶ token_tracking.py token/cost estimates per session

 static/app.js ......... vanilla-JS dashboard, polls the JSON API
```

The agent loop (`generate_agent_turn` in `app.py`):

1. Send `system prompt + user message` to DeepSeek through the bridge.
2. If the reply contains a ` ```tool_call ` block, execute it locally against the session's
   sandbox folder, append a step to the turn trace, and send the result back as the next
   message.
3. Repeat until DeepSeek replies with no tool call — that reply is the final answer.

There is no cap on loop iterations; a turn runs as long as the model keeps calling tools.

## Requirements

- Python **3.10+** (uses PEP 604 unions at runtime)
- Windows (developed/tested here; nothing is deliberately POSIX-incompatible)
- A display — the browser window is real and visible on purpose, so you can log in by hand
  and watch what the automation is doing

## Setup

```bat
pip install -r requirements.txt
python -m camoufox fetch   :: downloads the hardened Firefox binary (~150 MB, once)
python app.py              :: or just run begin.bat
```

Then open http://localhost:8080.

## First run

1. Click **Connect** in the header. A visible Firefox window opens on chat.deepseek.com.
2. Log in inside that window by hand. Login state persists across restarts in
   `~/.deepbridge-dashboard/deepseek-profile`.
3. Create a session, pick its workspace folder (or let it create `sessions/session-N`),
   and send a prompt.

Sending is blocked until you've connected explicitly — no prompt will silently spawn a
browser window.

## Tools

Exposed to the model via a fenced-block convention taught in `TOOLS_SYSTEM_PROMPT`
(`tools.py`), since the web UI has no native function-calling channel:

| Tool | Purpose |
|---|---|
| `read_file` | Read a file, optionally by line range / head / tail (200 KB limit) |
| `create_file` | New file; refuses to clobber without `overwrite: true` |
| `edit_file` | Find/replace (must match exactly once) or line-range replace |
| `list_dir` | Directory listing, optional recursive/depth-limited |
| `search_code` | Regex grep with context lines |
| `run_command` | Shell command, cwd confined to the session folder |
| `path_list` | Enumerate executables on PATH |
| `move_file` / `rename_file` / `copy_file` | File ops within the sandbox |
| `delete_file` | Moves to `.trash/` inside the session folder — never permanent |
| `quiz_user` | Pauses the turn and asks the human a question via a modal |

Every path argument resolves against the session root and rejects anything escaping it
(`..`, absolute paths, symlinks). `run_command` does a heuristic string check for outside
references and requires `allow_outside: true` to bypass — it's a convention check, not a
real sandbox; don't point sessions at directories you care about.

### quiz_user

`quiz_user` suspends the agent mid-turn (full loop state is kept), shows a modal in the UI
with single-choice / multi-choice / free-text questions, and resumes the same turn with the
answers fed back as a tool result.

## Usage and cost tracking

The web UI exposes no usage data, so everything in the Logs tab is an estimate:

- tokens ≈ characters ÷ 4, counted over every message actually sent (initial prompt plus
  each tool-result round trip) and the final reply;
- pricing uses two hardcoded presets, *V4 Fast* / *V4 Pro* off-peak rates. These numbers are
  constructed assumptions, not copied from any published DeepSeek price sheet.

Switching presets re-prices the entire session history immediately, and (best-effort) clicks
the matching Fast/Expert toggle in the live browser. Note that DeepSeek only allows changing
mode on a fresh conversation, so a successful switch starts a new chat there.

## HTTP API

All JSON. The dashboard uses these; they're equally scriptable.

| Method | Path | Purpose |
|---|---|---|
| GET/POST | `/api/sessions` | List / create sessions |
| GET/DELETE | `/api/sessions/:id` | Fetch / delete session |
| PATCH | `/api/sessions/:id/system-prompt` | Set per-session system prompt |
| POST | `/api/sessions/:id/messages` | Send user message; blocks until the full agent turn finishes |
| GET | `/api/sessions/:id/progress` | Live phase/label/steps while a turn is in flight (poll ~600 ms) |
| POST | `/api/sessions/:id/quiz-answer` | Answer a pending `quiz_user` and resume |
| GET | `/api/sessions/:id/file-tree` | Sandbox file tree annotated with +N/−N change stats |
| GET | `/api/sessions/:id/file-content?path=` | Inline file preview (200 KB cap) |
| GET | `/api/sessions/:id/logs` | Usage entries, totals, current pricing |
| PATCH | `/api/sessions/:id/logs/model` | Switch pricing preset (+ flips the browser's Fast/Expert toggle) |
| GET/POST | `/api/bridge/status` · `/connect` · `/disconnect` | Manage the shared browser |
| POST | `/api/bridge/new-chat` | Reset DeepSeek's own conversation |
| GET | `/api/tools` | Tool reference parsed from docstrings |

## Project layout

```
deepseekgen/
├── app.py                Flask server: REST API + the agent tool-loop
├── deepseek_bridge.py    Camoufox automation of chat.deepseek.com
│                         (SSE capture, DOM fallback, mode switching)
├── bridge_worker.py      Single-slot job queue on one thread — all Playwright
│                         calls happen there, Flask threads never touch it
├── tools.py              Sandboxed tools + tool_call protocol parser + system prompt
├── token_tracking.py     Token/cost estimation, per-session usage logs
├── live_progress.py      Thread-safe in-memory progress state for polling
├── templates/index.html
├── static/app.js         Dashboard frontend (vanilla JS, ~2200 lines)
├── static/style.css
├── sessions/             Created at runtime; per-session workspaces
├── backup/               Historical snapshots/zips, ignore
├── begin.bat             Launcher
└── requirements.txt
```

## Known limitations

- **Fragile by construction.** Selectors target hashed class names (`textarea._27c9245`,
  `div._4f9bf79`) and DOM structure that DeepSeek can change at any time; when they break,
  `_find_composer` / `_find_send_button` / `_extract_latest_reply` fallbacks are where to look.
- **One shared browser** for the whole process — all sessions talk to the same DeepSeek tab.
  Sessions are concurrent only from the dashboard's perspective.
- **In-memory state.** Sessions, turns, file-change stats, and usage logs vanish on restart.
  Files written to the workspace survive; the dashboard history doesn't.
- **Slow replies are normal.** Browser automation plus DeepSeek's own anti-bot throttling
  means a turn takes seconds to minutes. Progress polling exists precisely because of this.
- **Flask dev server** with `debug=True` and `use_reloader=False`, bound to 0.0.0.0:8080.
  Fine for localhost use; not hardened, not production anything.
- No tests, no lint config. The `backup/` directory is the version control, such as it is.
