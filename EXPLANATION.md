# The AbletonMCP Project

## The big-picture analogy

Imagine Ableton Live is a professional recording studio, and Claude is a producer who wants to work in that studio — but Claude is at home, on the phone. The studio doesn't have a phone. So you install **a receptionist with a phone in the studio** (the Remote Script) and give the producer **a personal assistant who knows how to talk to that receptionist** (the MCP Server). Claude tells the assistant "make a hi-hat track"; the assistant phones the receptionist; the receptionist walks across the studio and pushes the right buttons on the mixing console.

That's the entire architecture. Two processes, one phone line, lots of translation.

## The diagram

```
┌─────────────────┐    MCP/stdio     ┌──────────────────┐    TCP socket     ┌───────────────────┐
│                 │  (JSON-RPC over  │                  │    port 9877      │                   │
│   Claude /      │  stdin/stdout)   │   MCP_Server/    │  (JSON command    │  Ableton Live     │
│   Cursor        │◄────────────────►│   server.py      │◄─────────────────►│  Remote Script    │
│   (the LLM)     │                  │   (FastMCP)      │   {type, params}) │  (__init__.py)    │
└─────────────────┘                  └──────────────────┘                   └─────────┬─────────┘
                                              ▲                                       │
                                              │                                       │ schedule_message(0, fn)
                                              │ exposes @mcp.tool()                   ▼
                                              │ functions:                  ┌───────────────────┐
                                              │  • get_session_info         │  Live API         │
                                              │  • create_midi_track        │  (self._song,     │
                                              │  • add_notes_to_clip        │   tracks, clips,  │
                                              │  • load_instrument...       │   browser, ...)   │
                                              │  • create_stems / status    └───────────────────┘
                                              ▼
                                        Claude sees a
                                        "hammer" toolkit
```

Two processes, two protocols, one shared idea: "send a JSON envelope, get a JSON envelope back."

## Walking through a single call

Take "create a MIDI track" end-to-end:

**1. Claude picks a tool.** `MCP_Server/server.py:288` — `create_midi_track(ctx, index)` is decorated with `@mcp.tool()`, so FastMCP advertises it to Claude as a callable tool.

**2. The MCP server forwards it.** It calls `ableton.send_command("create_midi_track", {"index": -1})` (`server.py:298`). The `AbletonConnection` class (`server.py:15-163`) is a thin wrapper around a TCP socket to `localhost:9877`. It JSON-encodes the command, sends it, and then reads chunks until it can parse a complete JSON object back (`receive_full_response`, `server.py:46-91`) — a clever hack to handle the fact that big responses arrive in pieces.

**3. The Remote Script receives it.** Inside Ableton, `AbletonMCP_Remote_Script/__init__.py` is loaded as a "MIDI Remote Script" by Ableton itself. On startup it spins up a TCP server thread (`start_server`, line 78) that accepts connections and dispatches each client to a handler thread (`_handle_client`, line 136).

**4. The command is dispatched — but carefully.** `_process_command` (line 213) inspects `command_type`. For *read-only* commands (`get_session_info`, `get_track_info`) it just runs them directly. For *state-modifying* commands it does something subtle: it wraps the work in `schedule_message(0, main_thread_task)` (line 305) and waits on a `queue.Queue` for the result. **Why?** Live's API can only be touched from the main UI thread; calling `song.create_midi_track()` from the socket thread would crash Ableton.

**5. Live actually does the work.** `_create_midi_track` (line 429) calls `self._song.create_midi_track(index)` — that's the real Ableton Live Python API. Result goes back through the queue → the socket → the MCP server → Claude.

## The interesting bit: stem recording

Most tools are one-shot calls. `create_stems` (`server.py:655`, remote line 816) is the standout — it's *asynchronous*. The flow:

1. Claude calls `create_stems(scene_index=0, bars=8)`.
2. The Remote Script creates one new audio track per source track, sets each to "Resampling" input, then starts recording the **first** one with all other tracks muted (line 917–919). This guarantees clean isolation — no bleed.
3. Returns immediately with `status: "recording_started"`.
4. Claude polls `get_stems_status` until it sees `"completed"`. Then it gets back `.wav` file paths on disk.
5. By default the temporary stem tracks are cleaned out of the Live set afterward — the recent commit `6700a88` made that the default.

The state machine lives in a single dict, `self._stems_state` (line 43, 870), which is what `get_stems_status` reads.

## The gotcha

**Two threads, one Live API.** The most common mistake when extending this codebase is to add a new state-modifying command and forget the `schedule_message` dance. If you do, your tool will *appear* to work in tests — until it randomly hangs or crashes Ableton Live, because you touched `self._song` from a socket thread. The way to add a new mutating tool is:

1. Add it to the dispatch list at `__init__.py:232-236` (the `if command_type in [...]` block).
2. Add an `elif` inside the `main_thread_task()` closure (around line 244–294).
3. Implement the actual `_my_new_thing()` method that touches `self._song`.
4. Add the matching `@mcp.tool()` wrapper in `server.py`, and remember to also list the command name in the `is_modifying_command` list at `server.py:104` so the timeout/delay handling is right.

A second, smaller gotcha: the protocol assumes the **whole response is one JSON object**. Both sides loop reading chunks until `json.loads` stops raising. If you ever return a non-JSON-decodable byte stream (binary, or a partial write), the receiver will hang until its 15-second timeout. Keep results small and JSON-clean.

---

That's the project: ~2,000 lines of code total (~1,350 in the Remote Script, ~720 in the MCP server), one TCP socket between them, one careful threading rule, and a thin layer of async polling for the only long-running operation.

## Tools reference

All tools are defined in `MCP_Server/server.py` and decorated with `@mcp.tool()`.

### Session / project info

| Tool | Parameters | Description |
|---|---|---|
| `get_session_info` | — | Tempo, time signature, track count, and clip info for the whole Live set |
| `get_song_file_path` | — | Path of the currently open `.als` file |
| `get_track_info` | `track_index` | Name, type, devices, and clips for a single track |

### Transport

| Tool | Parameters | Description |
|---|---|---|
| `start_playback` | — | Press Play |
| `stop_playback` | — | Press Stop |
| `set_tempo` | `tempo` | Set BPM |

### Tracks

| Tool | Parameters | Description |
|---|---|---|
| `create_midi_track` | `index` (default `-1`) | Insert a new MIDI track; `-1` appends at the end |
| `set_track_name` | `track_index`, `name` | Rename a track |

### Clips

| Tool | Parameters | Description |
|---|---|---|
| `create_clip` | `track_index`, `clip_index`, `length` (default `4.0`) | Create an empty MIDI clip in a session slot |
| `add_notes_to_clip` | `track_index`, `clip_index`, `notes` | Write MIDI notes into a clip |
| `set_clip_name` | `track_index`, `clip_index`, `name` | Rename a clip |
| `fire_clip` | `track_index`, `clip_index` | Launch a clip |
| `stop_clip` | `track_index`, `clip_index` | Stop a clip |

### Browser / instruments

| Tool | Parameters | Description |
|---|---|---|
| `get_browser_tree` | `category_type` (default `"all"`) | Browse the Live library tree (instruments, sounds, etc.) |
| `get_browser_items_at_path` | `path` | List items at a specific browser path |
| `load_instrument_or_effect` | `track_index`, `uri` | Load a device onto a track by its browser URI |
| `load_drum_kit` | `track_index`, `rack_uri`, `kit_path` | Load a Drum Rack and a specific kit into it |

### Stems

| Tool | Parameters | Description |
|---|---|---|
| `create_stems` | `track_indices`, `output_dir`, `duration` | Solo-record each specified track to a separate WAV file (async — returns immediately) |
| `get_stems_status` | — | Check progress of an in-flight stems recording |

## Python libraries used

### Third-party (declared in `pyproject.toml`)

- **`mcp[cli]>=1.3.0`** — the only declared dependency. Specifically `mcp.server.fastmcp` (`FastMCP`, `Context`) is what turns Python functions decorated with `@mcp.tool()` into tools Claude can call. The `[cli]` extra adds the `mcp` command-line entry point.

### Ableton-side (provided by Ableton Live's bundled Python)

- **`_Framework.ControlSurface`** — Ableton's internal MIDI Remote Script base class. Not on PyPI; it ships inside Ableton Live and is only importable from a script loaded by Live itself. `AbletonMCP` subclasses `ControlSurface` to get hooks like `self.song()`, `self.log_message()`, `self.show_message()`, and `self.schedule_message()` (the main-thread scheduler).

### Python standard library (everything else)

| Module | Used for |
|---|---|
| `socket` | TCP server in the Remote Script (port 9877) and the client in `MCP_Server/server.py` |
| `json` | The wire format on both ends |
| `threading` | Server thread + per-client handler threads in the Remote Script |
| `queue` (`Queue` on Py2) | Bridging the socket thread → main-thread Live API → back |
| `time` | Small `sleep()` calls around state-modifying commands |
| `traceback` | Logging full stack traces from inside Ableton |
| `logging` | Server-side structured logging |
| `dataclasses` | `AbletonConnection` dataclass |
| `contextlib.asynccontextmanager` | `server_lifespan` for FastMCP startup/shutdown |
| `typing` | Type hints on tool signatures |
| `__future__` | Py2/Py3 compatibility shim in the Remote Script (older Ableton versions ran Python 2) |

### What's notable

- **Only one third-party dependency.** The whole project is essentially "stdlib + MCP SDK." The heavy lifting (the Live API, `_Framework`) comes from Ableton itself, not pip.
- **The Remote Script still carries Python 2 compatibility code** (`from __future__ import ...`, the `try: import Queue` fallback, the byte/string juggling in `_handle_client`). Ableton 11+ uses Python 3, so this is dead weight in modern installs but harmless.
- **No async/concurrent libraries beyond `threading` + `queue`.** The MCP server is async (FastMCP runs on asyncio under the hood), but the socket call to Ableton is synchronous and blocking — which is why timeouts are set so carefully in `send_command`.
