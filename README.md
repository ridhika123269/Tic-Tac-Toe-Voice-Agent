# Tic-Tac-Toe-Voice-Agent - Using Vocal Bridge

A Jupyter notebook lesson that connects a Vocal Bridge voice agent to a
tic-tac-toe UI, using **Client Actions** to sync voice input, clicks, and
the game board over the same real-time channel.

## What's in this project

| File | Purpose |
|---|---|
| `helpers.py` | Custom Python glue code for this lesson (not an official Vocal Bridge package). Wraps the `vb` CLI, calls Vocal Bridge's token API, and renders the in-notebook voice widget. |
| `prompt.md` | The agent's system prompt — tic-tac-toe rules, personality, and instructions for firing Client Actions. |
| `actions.json` | The Client Actions contract — the named events the agent and the app are allowed to send each other. |
| `.env` | Local secrets: your API key and the agent ID. **Never commit this file or share it.** |

## How the pieces fit together

```
Vocal Bridge dashboard          Your terminal              Your notebook
┌────────────────────┐         ┌───────────────┐          ┌──────────────────┐
│ Agent config:       │◄────────┤ vb prompt set │          │ mint_token()      │
│  - prompt.md        │         │ vb config set │          │  → TokenData      │
│  - actions.json     │         └───────────────┘          │                    │
└────────────────────┘                                     │ voice_widget()     │
                                                             │  → renders React  │
                                                             │    UI in browser  │
                                                             └──────────────────┘
```

1. **The agent** lives on Vocal Bridge's servers, configured with a prompt
   and a set of Client Actions it's allowed to fire.
2. **Python (`helpers.py`)** never touches audio. It mints a short-lived
   session token via Vocal Bridge's REST API, and generates the HTML/JS
   for the voice widget.
3. **The browser widget** (React, loaded from a CDN inside the notebook
   output) connects to the agent over WebRTC/LiveKit, and listens for /
   sends Client Actions to keep the board in sync with the conversation.

## Client Actions used in this lesson

**Agent → App**
- `show_tic_tac_toe` — start a new game
- `place_mark` — the agent's move
- `user_move` — the user's move, when *spoken* (not clicked)
- `clear_ui` — reset the board

**App → Agent**
- `user_placed_mark` (`respond`) — user clicked a cell; agent should reply
- `board_sync` (`notify`) — silent, authoritative board state after every move

## One-time setup

```bash
cd ~/Desktop/Python/Voice_in_app

# Authenticate the CLI with your API key
vb auth login vb_your_api_key_here

# Confirm your agent
vb agent

# Push the lesson's prompt and actions to your dashboard-created agent
vb prompt set --file prompt.md
vb config set --client-actions-file actions.json

# Verify
vb prompt show
vb config get client-actions
```

Add your agent ID to `.env` so the notebook reuses it instead of trying
to create a new one:
```
VOCAL_BRIDGE_API_KEY=vb_your_api_key_here
VOCAL_BRIDGE_AGENT_ID_L2=your-agent-id-here
```

## Running the lesson

1. Open the notebook in a **real browser tab** via `jupyter notebook` or
   `jupyter lab` (not VS Code's built-in notebook viewer — its sandboxed
   webview can silently block microphone access).
2. Restart the kernel and run all cells from the top.
3. Click **Connect** in the rendered widget, allow microphone access.
4. Say **"let's play tic-tac-toe"** to start a game.
5. Speak a move ("center", "top right") or click a cell directly.

## Troubleshooting quick reference

| Symptom | Likely cause |
|---|---|
| `Agent creation via API requires a paid plan` | `VOCAL_BRIDGE_AGENT_ID_L2` isn't set/loaded — the notebook fell into the `agent create` branch. Set it in `.env` and restart the kernel. |
| Agent says it "can't show the board" | `actions.json` wasn't pushed to the agent yet. Run `vb config set --client-actions-file actions.json`. |
| `Connection failed` in the widget, no console detail | Likely running inside VS Code's notebook viewer — try a real browser tab via `jupyter notebook`. |
| `403 Forbidden` on `/api/v1/token` | Check the response body — it's often a **usage limit**, not a bad key (e.g. free tier minutes/year exceeded), not something fixable from code. |
| `.env` changes don't seem to take effect | Restart the kernel. `load_dotenv()` only re-reads the file when explicitly called, and env vars are cached in the running kernel process. |

