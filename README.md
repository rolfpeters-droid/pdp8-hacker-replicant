# PiDP-8/I Hacker Replicant

An AI tutor/copilot that sits next to Oscar Vermeulen's PiDP-8/I replica: a
web page with the real terminal on the left, and a chat panel on the right
that knows OS/8, PAL-8 and FOCAL — and, if you let it, can type into the
terminal itself.

This is part of a small family of "Hacker Replicant" packages built for
different PDP/VAX/HP replicas; this one is PiDP-8/I-specific, but the
pattern (a tiny `screen`-reading boot-API + a chat panel + an LLM backend)
is reusable for other machines too.

## How it works

```
Browser
 │
 ├─ Left panel:  <iframe> → your own existing web console for the PiDP-8/I
 │      (ttyd, or whatever web-UI you already run -- see "Bring your own
 │      console" below. Not bundled in this repo.)
 │
 └─ Right panel: chat → POST /api/chat (SSE stream)
      │  this Node service (server.js)
      │
      ├─ LLM backend: any OpenAI-compatible /chat/completions API
      │  (local Ollama by default, or a hosted provider -- see below)
      │
      └─ when the control toggle is ON, two tools only:
           send_keys   → POST  <BOOTAPI_BASE>/api/keys    {"text": "..."}
           read_screen → GET   <BOOTAPI_BASE>/api/screen
           (served by pdp8-bootapi.py, included in this repo)
```

There's no SIMH API involved anywhere. The machine's terminal runs inside a
GNU `screen` session; the chat agent's tools talk to that session via two
simple, stateless primitives:

- `screen -X stuff <text>` — types text into the session, as if someone
  sat down and typed it.
- `screen -X hardcopy <file>` — writes a plain-text snapshot of the
  current screen contents to a file, which the API reads back.

That's the entire "control" surface for the agent: no shell access, no
filesystem access, nothing beyond what's visible in the same terminal your
own web console already shows. `pdp8-bootapi.py` also exposes `GET
/api/status` and `GET /api/boot/<n>` for querying/restarting the simulator
with a given boot script — the chat agent only ever uses `/api/keys` and
`/api/screen`.

## Bring your own console

Unlike some other packages in this family, this repo does **not** bundle a
web console for driving the machine. Most PiDP-8/I builds already have one
running (`ttyd`, a VNC-to-web bridge, or something along those lines) —
building a machine-specific console from scratch was out of scope for this
release. Point `CONSOLE_URL` at whatever you already have, and the left
panel (and the "back" link) will just use it. If you don't have one yet,
`ttyd` pointed at the same `screen` session `pdp8-bootapi.py` talks to is
the simplest option.

## LLM-agnostic by design

The chat backend talks to **any backend that speaks the OpenAI-compatible
`/chat/completions` API with tool/function calling** — that's one code
path for both:

- **Local** (default, free): point it at a local [Ollama](https://ollama.com)
  install. Ollama has spoken this API since mid-2024, nothing extra to
  configure beyond pulling a model that supports tool calling (`qwen3`,
  `llama3.1`+, `mistral-nemo`, etc. all do).
- **Cloud** (optional, for a faster/smarter option): point it at OpenAI,
  Groq, Together, Fireworks, DeepSeek, Mistral, OpenRouter, a hosted
  vLLM/TGI endpoint — anything OpenAI-compatible. See `.env.example` for
  examples.

No vendor SDK, no hard dependency on any single provider, no API key
required at all if you only use the local backend.

**A heads-up on local "thinking" models:** Ollama's OpenAI-compatible
endpoint (what the local backend uses) doesn't currently expose a way to
turn off a reasoning model's internal "thinking" step the way Ollama's own
native API does (`think: false`) — tested directly against Ollama 0.x as of
Nov 2026, passing `think: false` in the request body had no effect via
`/v1/chat/completions`. For `qwen3` specifically this means every reply
carries a few hundred tokens of hidden reasoning, adding roughly 10-20
extra seconds per turn compared to what you'd get calling Ollama's native
API directly. `LLM_TIMEOUT_MS` already accounts for this (default 180s), so
nothing breaks, it just feels slower than it has to. If snappier replies
matter more to you than using a reasoning model locally, point
`LOCAL_MODEL` at a non-thinking model instead (`llama3.1`, `mistral-nemo`,
etc. all support tool calling without this overhead).

## Setup

### Prerequisites

- GNU `screen`, installed on the Pi (`apt install screen` on Debian/Raspberry
  Pi OS if it isn't already). `pdp8-bootapi.py` shells out to it directly.
- A PiDP-8/I running a SIMH-based PDP-8 simulator in a `screen` session, plus
  whatever web console you already use to view/drive it (see "Bring your own
  console" above).
- Node.js 18+ and Python 3 on that same host (or reachable over the network).
- [Ollama](https://ollama.com) running somewhere reachable (for the free
  local backend), or an API key for a cloud provider.

Both pieces (boot-API, chat backend) are plain Python/Node processes — they
can run on the PiDP-8/I's own Pi, or the chat backend can run elsewhere and
just point `CONSOLE_URL`/`BOOTAPI_BASE` at the Pi over the network. The
boot-API needs to run on the Pi itself (it shells out to `screen` directly).

### Install

```bash
git clone <this-repo-url>
cd pdp8-hacker-replicant
npm install
cp .env.example .env
# edit .env: at minimum set CONSOLE_URL and BOOTAPI_BASE to match your setup
```

### Run the boot-API (on the Pi, bridges the chat agent to the console)

```bash
python3 pdp8-bootapi.py
```

Edit the `BOOTDIR`/`SIM`/`WORKDIR` constants at the top of `pdp8-bootapi.py`
if your simulator binary or boot scripts live somewhere other than the
default `/opt/pidp8i/...` layout.

### Run the chat backend (right panel)

```bash
npm start
```

Open `http://<host>:8100/` (or whatever `PORT` you set) — `CONSOLE_URL`
should point at the web console you already run for the machine.

### Running both as services

See `docs/pdp8-bootapi.service.example` and
`docs/pdp8-hacker-replicant.service.example` for ready-to-adapt systemd
units.

## Configuration reference

All configuration is environment variables — see `.env.example` for the
full list with examples. The short version:

| Variable | What |
|---|---|
| `PORT` | Port the chat backend listens on |
| `CONSOLE_URL` | Your existing web console, shown in the left panel |
| `BOOTAPI_BASE` | Where `pdp8-bootapi.py` is reachable |
| `ACTION_LIMIT` | Tool calls per turn before the agent must ask to continue |
| `LOCAL_BASE_URL` / `LOCAL_API_KEY` / `LOCAL_MODEL` | Local (default) backend |
| `CLOUD_BASE_URL` / `CLOUD_API_KEY` / `CLOUD_MODEL` | Optional cloud backend |

## Safety model

- The control toggle is **off** on every page load — by default the agent
  only talks, it never touches the machine.
- When on, the agent gets exactly two tools (`send_keys`, `read_screen`),
  nothing else — no shell, no filesystem, no network access beyond the
  boot-API.
- After `ACTION_LIMIT` tool calls in one turn, the agent is cut off from
  further tool calls and must ask in plain text whether it may continue —
  no silent, unbounded action streak. A hard iteration cap (20) sits
  underneath as a defensive backstop against a broken loop.
- Everything the agent types is visible live in the same terminal session
  your own web console shows — no black box.
- This does **not** protect you from the agent typing something wrong (it's
  an LLM, it can make mistakes) — the boundary is on *what* it can touch,
  not on *how well* it types. Use a local/free backend if you want to
  experiment without any cost risk.

## Repository layout

```
server.js               Chat backend (Node/Express, SSE, LLM-agnostic)
public/                  Chat panel UI (right side)
pdp8-bootapi.py          screen -X stuff/hardcopy bridge for the chat agent's tools
skills/pdp8-tutor.md     System prompt: OS/8, PAL-8, FOCAL knowledge + behavior rules
docs/                    Example systemd units (not installed automatically)
```

## Adapting this for a different replica

`pdp8-bootapi.py` is deliberately small and machine-specific only in its
paths (`BOOTDIR`, `SIM`, `WORKDIR`) and session-naming convention — it's a
reasonable starting point for wiring up the same pattern on a different
PDP/VAX/HP/whatever replica: same two primitives (`screen -X stuff` /
`screen -X hardcopy`), same tiny HTTP wrapper. `skills/pdp8-tutor.md` is the
only file that's actually PDP-8/OS-8-specific content; swap it for a system
prompt about your own machine and point `server.js` at the new file.

This repo doesn't ship a web console at all (see "Bring your own console"
above) — for a different replica, point `CONSOLE_URL` at whatever web
console that machine already has (most PiDP replicas ship with something
along these lines); the chat panel and the boot-API don't care what's
actually driving the left panel.

## Credits

Built by Rolf Peters with Seb (his homelab AI assistant), in collaboration
with Oscar Vermeulen (designer of the PiDP replica series). Inspired by
Oscar's own `agent-pdp1` project for the PiDP-1.

## License

MIT — see `LICENSE`.
