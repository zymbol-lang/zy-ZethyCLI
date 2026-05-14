# ZethyCLI — Design Document v3

> **Revisado para v0.0.5 — 2026-05-12**

> A real-world AI chat CLI written entirely in Zymbol.
> Primary goal: stress-test the language and map its real strengths and gaps.
> No external Python bridges — everything via Zymbol modules + BashExec.

## Models & Host

Models and the Ollama host are fully configurable at runtime — no hardcoded presets required.

| Command | Effect |
|---------|--------|
| `/model llama3:8b` | Switch to any installed model (think disabled) |
| `/mode fast` | Preset shortcut → `gemma3:4b`, no thinking |
| `/mode think` | Preset shortcut → `deepseek-r1:1.5b`, reasoning |
| `/host 192.168.1.1` | Switch Ollama to remote host (port 11434 assumed) |
| `/host 192.168.1.1:9999` | Switch to remote host with explicit port |

Preset models (convenience constants in `ollama.zy`, not enforced):

| Preset | Value | Notes |
|--------|-------|-------|
| `fast` | `gemma3:4b` | ~17s, no thinking |
| `think` | `deepseek-r1:1.5b` | ~43s, reasoning |

Model strings are inlined directly in `main.zy` — no exported constants in `ollama.zy`.

---

## Project Structure

```
ZethyCLI/
  main.zy               Entry point — REPL loop, inline command dispatcher
  lib/
    config.zy           Persistent config — ~/.ZethyCLI/ZethyCLI.conf (JSON via jq)
    ui.zy               Terminal output (box-drawing, formatting) — pure Zymbol
    history.zy          Conversation state — volatile session buffer (msgs.json)
    memory.zy           Persistent cross-session memory — ~/.ZethyCLI/memory.txt
    sessions.zy         Incremental session archive — ~/.ZethyCLI/sessions/*.json
    json.zy             JSON encode/decode via jq + BashExec
    http.zy             HTTP client via curl + BashExec
    ollama.zy           Ollama-specific API (imports http + json)
  DESIGN.md
  GAPS.md               Living document — gaps found during development
  BUGS.md               Runtime bugs discovered and fixed
```

Commands and code execution are handled inline in `main.zy` — no `commands.zy`
or `executor.zy` modules. This keeps the module graph shallow.

### File storage — two tiers

**Persistent** (`~/.ZethyCLI/`) — survives reboots and session restarts:

| Path | Contents | Writable by |
|------|----------|-------------|
| `~/.ZethyCLI/ZethyCLI.conf` | Config (JSON): model, mode, think, host, tmp_dir | `/model` `/mode` `/host` |
| `~/.ZethyCLI/memory.txt` | Cross-session memory: facts + session summaries | `/remember` `/forget` + auto on `/exit` |
| `~/.ZethyCLI/sessions/YYYY-MM-DD_HH-MM-SS.json` | Incremental session archive — written per turn, crash-safe | `sessions.zy` (auto) |

**Volatile** (`{tmp_dir}/`, default `/tmp/ZethyCLI/`) — current session only, lost on reboot:

| Path | Contents | Configurable |
|------|----------|--------------|
| `{tmp_dir}/msgs.json` | Active conversation (JSON array of messages) | via `tmp_dir` in config |
| `{tmp_dir}/count.txt` | Turn counter for current session | via `tmp_dir` in config |
| `{tmp_dir}/req.json` | Ollama request body (rebuilt each turn) | via `tmp_dir` in config |
| `{tmp_dir}/resp.json` | Ollama raw response | via `tmp_dir` in config |
| `{tmp_dir}/content.txt` | Extracted assistant text | via `tmp_dir` in config |
| `{tmp_dir}/tags.json` | Model list cache | via `tmp_dir` in config |

Default `tmp_dir` = `/tmp/ZethyCLI`. Change it by editing `~/.ZethyCLI/ZethyCLI.conf` and restarting.

**Design rule**: anything the user expects to survive a reboot lives in `~/.ZethyCLI/`.
Anything that is only relevant to the running session lives in `tmp_dir`.

---

## Architecture

```
                    ┌─────────────────────────────────────┐
                    │           main.zy (REPL)             │
                    │                                      │
                    │  running = ollama::is_running()      │
                    │  mem_ctx = memory::get_context()     │
                    │  ? running → hist::init(sys_prompt,  │
                    │                         mem_ctx)     │
                    │             sessions::start()        │
                    │  @ {                                 │
                    │      << "> " raw_input               │
                    │      first = input$[1..1]            │
                    │      ? first == "/" →                │
                    │          ?? cmd { ... }  (inline)    │
                    │      _ → chat turn (inline)          │
                    │          sessions::append_user()     │
                    │          sessions::append_assistant()│
                    │  }                                   │
                    └───────────┬──────────────────────────┘
                                │
         ┌──────────────────────┼──────────────────────────┐
         │                      │                          │
   ┌─────▼──────┐  ┌────────────▼───┐  ┌───────────────────▼──┐
   │  ui.zy     │  │  history.zy    │  │  ollama.zy           │
   │            │  │                │  │                      │
   │ banner()   │  │ init(sys,mem)  │  │ chat()               │
   │ show_*()   │  │ add_user()     │  │ summarize()          │
   │            │  │ add_asst()     │  │ is_running()         │
   │ pure Zymbol│  │ get_msgs_file()│  │                      │
   │ no BashExec│  │ get_count()    │  │ imports:             │
   └────────────┘  │ clear()        │  │ http.zy  json.zy     │
                   │                │  └──────────────────────┘
                   │ volatile:      │
                   │ /tmp/*.json    │
                   └───────┬────────┘
                           │
              ┌────────────┴──────────────┐
              │                           │
        ┌─────▼──────────┐  ┌─────────────▼──────────┐
        │  memory.zy     │  │  sessions.zy            │
        │                │  │                         │
        │ get_context()  │  │ start()                 │
        │ save_fact()    │  │ set_tmp_dir(dir)         │
        │ save_summary() │  │ append_user(content)    │
        │ clear()        │  │ append_assistant()      │
        │                │  │ list()                  │
        │ persistent:    │  │                         │
        │ ~/.ZethyCLI/     │  │ persistent (per-turn):  │
        │ memory.txt     │  │ ~/.ZethyCLI/sessions/     │
        └────────────────┘  │ YYYY-MM-DD_HH-MM-SS.json│
                            └─────────────────────────┘
```

---

## Module Visibility Model (G13)

Zymbol modules have two kinds of top-level declarations:

| Declaration | Access | Persists | Exportable |
|-------------|--------|----------|------------|
| `name := value` | `:=` constant | immutable | ✅ via `#>` |
| `name = value` | private variable | ✅ write-back after each call | ✗ silently excluded from `#>` |
| `fn(args) { }` | function | — | ✅ via `#>` |

```zymbol
# counter {
    #> { increment, get_value, STEP }

    STEP  := 1          // exported constant — accessible as alias.STEP
    count = 0           // private mutable state — persists between calls

    increment() { count = count + STEP }
    get_value()  { <~ count }
}
```

```zymbol
// From main.zy:
c::increment()          // count → 1
c::increment()          // count → 2
n = c::get_value()      // n = 2
step = c.STEP           // step = 1   (exported constant)
x = c.count             // Runtime error: no constant 'count'
```

**Design rule**: a module is a toolbox, not a class. Private state is allowed
for mutable module-level values (caches, counters) but is always singleton
and cannot be accessed except through exported functions.

---

## `#>` Export Block Position (G14)

The export block can appear in two positions — both are valid:

```zymbol
// Position 1 — exports before imports:
# module_name {
    #> { fn, CONST }
    <# ./dep <= dep
    code...
}

// Position 2 — imports before exports (MANUAL recommended order):
# module_name {
    <# ./dep <= dep
    #> { fn, CONST }
    code...
}
```

Comments and function definitions can appear anywhere inside the block.

---

## Module Designs

> **v0.0.4**: All modules use closed block syntax — `# name { ... }`.
> The code blocks below show interface design; see the actual `lib/*.zy` files for the current implementation.

### lib/config.zy

Reads and writes `~/.ZethyCLI/ZethyCLI.conf` using jq. All values have in-code
defaults so the file is optional — created on first `/model`, `/mode` or `/host`.

Private mutable state: `cfg_model`, `cfg_mode`, `cfg_think`, `cfg_host`, `cfg_tmp_dir`.
All five are set by `load()` and persisted by `save()`.

```
Config file format (JSON):
{
  "model":   "gemma3:4b",
  "mode":    "fast",
  "think":   false,
  "host":    "http://localhost:11434",
  "tmp_dir": "/tmp/ZethyCLI"
}
```

`tmp_dir` is only applied at startup — changing it via the config file and
restarting ZethyCLI is the intended workflow (no `/tmp` command).

---

### main.zy

State variables (main scope):
```
mode       = "fast"             active mode label (preset name or model name)
model      = ollama.MODEL_FAST  active model name
think      = #0                 thinking mode toggle
sys_prompt = "..."              system prompt text
```

Startup sequence:
```
conf::load()                        // read ~/.ZethyCLI/ZethyCLI.conf (or defaults)
mode/model/think/tmp_dir ← conf     // populate main-scope vars
ollama::set_host(conf::get_host())  // apply saved host
ollama::set_tmp_dir(tmp_dir)        // apply saved tmp dir
hist::set_tmp_dir(tmp_dir)          // apply saved tmp dir
ui::banner()
running = ollama::is_running()      // health check via HTTP 200
? running → hist::init(sys_prompt)
         → enter REPL @ loop
_ → show error + exit
```

Command routing (inline `??` match on `cmd` string):
```
/exit    → @!
/help    → ui::show_help()
/clear   → hist::clear() + hist::init(sys_prompt)
/mode    → preset shortcut: fast | think (model + think flag)
/model   → set any model by name; think = #0; mode = model name
/host    → ollama::set_host(args); verify connectivity
/models  → ollama::models() + ui::show_models()
/info    → >> model, mode, host, turns
```

Chat turn:
```
hist::add_user(input)
msgs_file = hist::get_msgs_file()
response  = ollama::chat(model, msgs_file, think)
hist::add_assistant(response)
ui::show_response(response, model)
```

---

### lib/ui.zy

Pure Zymbol output — no BashExec, no gaps. Demonstrates `>>` strength.

```zymbol
# ui

#> {
    banner
    divider
    show_response
    show_error
    show_info
    show_mode
    show_models
    show_help
}

banner() {
    >> "╔═══════════════════════════════════════════╗" ¶
    >> "║   ZethyCLI — Zymbol AI Engine  v0.1      ║" ¶
    >> "║   /help for commands · /exit to quit      ║" ¶
    >> "╚═══════════════════════════════════════════╝" ¶
}

show_response(text, model) {
    >> ¶
    >> "  [" model "] › " text ¶
    >> ¶
}

show_error(msg) { >> "[error] " msg ¶ }
show_info(msg)  { >> "[info]  " msg ¶ }
show_mode(mode, model) { >> "[mode]  " mode " → " model ¶ }
show_models(list)      { >> "[models] " list ¶ }
```

**Strength**: `>>` multi-value output with explicit `¶` newline is concise and
readable. No helpers needed for terminal formatting.

---

### lib/memory.zy  ← NEW

Persistent cross-session memory stored in `~/.ZethyCLI/memory.txt`.
Plain text file — one entry per line, human-readable and editable.
Injected into the system prompt at startup so the model always has context.

Two types of entries coexist in the same file:

```
- El usuario trabaja en Zymbol, un lenguaje de programación simbólico en Rust
- Prefiere respuestas concisas y directas
[2026-04-15 14:32] Discutimos el diseño de ZethyCLI y la persistencia de memoria.
  - Diseñamos tres capas: memoria manual, auto-resumen y rolling history
  - El módulo memory.zy guarda en ~/.ZethyCLI/, no en /tmp/
```

`save_fact()` — appends a user-defined bullet (`- <fact>`).
`save_summary()` — appends a dated block `[YYYY-MM-DD HH:MM] <summary>` generated by the AI on `/exit`.

```zymbol
# memory

#> {
    get_context
    save_fact
    save_summary
    clear
}

mem_file = "~/.ZethyCLI/memory.txt"

// Returns full memory content for injection into system prompt.
// Returns "" if file does not exist yet.
get_context() {
    exists = <\ "test -f ~/.ZethyCLI/memory.txt && echo 1 || echo 0" \>
    ? (exists$~~["\n":""] == "1") {
        content = <\ "cat ~/.ZethyCLI/memory.txt" \>
        <~ content
    }
    <~ ""
}

// Append a user-defined fact as a bullet point.
save_fact(fact) {
    <\ "printf '\\n- " fact "' >> ~/.ZethyCLI/memory.txt" \>
}

// Append a dated summary block (called automatically on /exit).
save_summary(summary) {
    date = <\ "date '+%Y-%m-%d %H:%M'" \>
    date = date$~~["\n":""]
    <\ "printf '\\n[" date "]\\n' >> ~/.ZethyCLI/memory.txt" \>
    <\ "printf '%s\\n' '" summary "' >> ~/.ZethyCLI/memory.txt" \>
}

// Erase all memory. User confirms intent via /forget command.
clear() {
    <\ "rm -f ~/.ZethyCLI/memory.txt" \>
}
```

**Memory injection**: `hist::init(sys_prompt, mem_ctx)` appends `mem_ctx` to the
system prompt before writing `msgs.json`. If `mem_ctx` is empty, the system prompt
is unchanged — no visible difference for first-time users.

```
System prompt sent to model (when memory exists):
  "You are ZethyCLI, a helpful AI assistant...

   Memory from previous sessions:
   - El usuario trabaja en Zymbol...
   [2026-04-15 14:32] ..."
```

---

### lib/history.zy — init() signature change

`init()` gains a second parameter `mem_ctx` to receive the memory content:

```zymbol
init(system_prompt, mem_ctx) {
    full_prompt = system_prompt
    ? (mem_ctx <> "") {
        full_prompt = system_prompt $++ "\n\nMemory from previous sessions:\n" mem_ctx
    }
    safe = full_prompt$~~["'":"(APOS)"]
    <\ "jq -n --arg c '" safe "' '[\{role:\"system\",content:$c}]' > " msgs_file \>
    <\ "printf '1' > " count_file \>
}
```

All other functions in `history.zy` are unchanged.

---

### lib/ollama.zy — summarize() function

New exported function. Takes the current `msgs_file`, appends an internal
summarization request, sends it to the model, and returns the summary text.
The summarization request is NOT added to the real conversation history.

```zymbol
// Generate a summary of the conversation in msgs_file.
// Builds a one-shot request: all history + a fixed summary instruction.
// Returns the model's response text.
summarize(model, msgs_file) {
    sum_url  = host_base $+ "/api/chat"
    sum_req  = tmp_dir $+ "/sum_req.json"
    sum_resp = tmp_dir $+ "/sum_resp.json"
    sum_out  = tmp_dir $+ "/sum_content.txt"
    instr    = "Summarize this conversation in 3-5 concise bullet points. Focus on key topics, decisions, and facts about the user. Be brief."
    <\ "(cat " msgs_file " | jq '. + [\{role:\"user\",content:\"" instr "\"}]') | jq '\{model:\"" model "\",messages:.,stream:false}' > " sum_req \>
    <\ "curl -s -X POST -H 'Content-Type: application/json' -d '@" sum_req "' " sum_url " > " sum_resp \>
    <\ "jq -r '.message.content // empty' " sum_resp " > " sum_out " 2>/dev/null" \>
    result = <\ "cat " sum_out \>
    <~ result
}
```

The summarization call uses `tmp_dir` files (volatile) — the summary itself is
saved to `~/.ZethyCLI/memory.txt` by `memory::save_summary()` in `main.zy`.

---

### main.zy — new commands and /exit change

**New commands** added to the `??` dispatcher:

```
/remember <fact>    Append a fact to ~/.ZethyCLI/memory.txt
/memory             Display current memory content
/forget             Clear all memory (with confirmation message)
```

**`/exit` updated** — auto-summary if session had turns:

```zymbol
"/exit" : {
    turns = hist::get_count()
    ? (turns > 2) {
        ui::show_info("Summarizing session...")
        summary = ollama::summarize(model, hist::get_msgs_file())
        memory::save_summary(summary)
        ui::show_info("Session summary saved.")
    }
    ui::show_info("Goodbye.")
    @!
}
```

**Startup sequence updated** to load memory before `hist::init`:

```zymbol
mem_ctx = memory::get_context()
hist::init(sys_prompt, mem_ctx)
```

---

### lib/sessions.zy  ← NEW (Phase 4)

Incremental session archive. Each session is stored as a JSON array in a dated
file under `~/.ZethyCLI/sessions/`. Messages are appended per turn — if the
program crashes, everything up to the last completed turn is already on disk.

**File naming**: `YYYY-MM-DD_HH-MM-SS.json` — one file per program run.
Old sessions are never overwritten; the directory accumulates history.

```
~/.ZethyCLI/sessions/
  2026-04-15_14-32-07.json    ← session 1 (40 messages)
  2026-04-15_18-07-22.json    ← session 2 (12 messages)
  2026-04-16_09-11-54.json    ← today's session (growing per turn)
```

Each file is a JSON array with the same schema as `msgs.json`:
```json
[
  {"role": "system",    "content": "You are ZethyCLI..."},
  {"role": "user",      "content": "What is Zymbol?"},
  {"role": "assistant", "content": "Zymbol is a symbolic language..."},
  ...
]
```

```zymbol
# sessions

#> {
    start
    set_tmp_dir
    append_user
    append_assistant
    list
}

// Private mutable state
tmp_dir      = "/tmp/ZethyCLI"
session_file = ""                // set by start()

set_tmp_dir(dir) {
    tmp_dir = dir
}

// Create a new session file with the system message from msgs.json.
// Called once at startup — filename is the current timestamp.
start() {
    ts = <\ "date '+%Y-%m-%d_%H-%M-%S'" \>
    ts = ts$~~["\n":""]
    <\ "mkdir -p ~/.ZethyCLI/sessions" \>
    session_file = "~/.ZethyCLI/sessions/" $++ ts $+ ".json"
    // Seed the session file with an empty array — system message
    // will be appended immediately by the first append_user call.
    <\ "printf '[]' > " session_file \>
}

// Append a user message. Called after hist::add_user().
// G4: sanitize single quotes before BashExec interpolation.
append_user(content) {
    safe = content$~~["'":"(APOS)"]
    <\ "jq --arg c '" safe "' '. += [\{role:\"user\",content:$c}]' " session_file " > " tmp_dir "/sess_tmp.json && mv " tmp_dir "/sess_tmp.json " session_file \>
}

// Append the assistant response from {tmp_dir}/content.txt.
// Uses --rawfile to avoid G4 entirely — AI content is never shell-quoted.
append_assistant() {
    content_file = tmp_dir $+ "/content.txt"
    <\ "jq --rawfile c " content_file " '. += [\{role:\"assistant\",content:$c}]' " session_file " > " tmp_dir "/sess_tmp.json && mv " tmp_dir "/sess_tmp.json " session_file \>
}

// List all session files, newest-first. Returns a formatted string.
// Each line: "  YYYY-MM-DD HH:MM:SS  (N messages)"
list() {
    result = <\ "ls -t ~/.ZethyCLI/sessions/*.json 2>/dev/null | while read f; do n=$(jq 'length' \"$f\" 2>/dev/null || echo '?'); base=$(basename \"$f\" .json); echo \"  ${base/_/ } ($n messages)\"; done" \>
    <~ result
}
```

**Crash safety**: `append_user` is called immediately before `ollama::chat()`.
If the program crashes during the API call, the user message is already archived.
`append_assistant` is called right after the response is received. The worst-case
data loss is one assistant response — the prompt that triggered it is already saved.

**Write path per turn**:
```
user types input
  → hist::add_user(input)          // volatile /tmp/ZethyCLI/msgs.json
  → sessions::append_user(input)   // persistent ~/.ZethyCLI/sessions/SESS.json ← CRASH SAFE HERE
  → ollama::chat()                 // blocks until response
  → hist::add_assistant(response)  // volatile
  → sessions::append_assistant()   // persistent ← complete turn
```

---

### lib/http.zy

HTTP client via curl + BashExec. G12 BashExec syntax — arguments are
Zymbol expressions, concatenated before being passed to the shell.

```zymbol
# http

#> { get, post_json_file, get_status }

// Simple GET request
get(url) {
    result = <\ "curl -s " url \>
    <~ result
}

// POST JSON body from file — avoids shell quoting issues (GAP G4)
post_json_file(url, file_path) {
    result = <\ "curl -s -X POST -H 'Content-Type: application/json' -d '@" file_path "' " url \>
    <~ result
}

// Get HTTP status code (for health checks)
// %\{http_code} → %{http_code} after Zymbol escaping (\{ → {, } is literal)
get_status(url) {
    result = <\ "curl -s --connect-timeout 4 --max-time 5 -o /dev/null -w '%\{http_code}' " url \>
    <~ result
}
```

**Note on `\{`**: In Zymbol strings, `\{` is the escape sequence for a literal `{`
and `\}` for a literal `}`. So `%\{http_code}` → `%{http_code}` — the correct curl
format string.

---

### lib/json.zy

JSON encode/decode via jq. Documents G2 (no native `std/json`).

```zymbol
# json

#> { encode, decode, trim_nl }

// Encode raw string as JSON string (with surrounding quotes)
// GAP G4: replaces ' with (APOS) before BashExec interpolation
encode(raw) {
    safe = raw$~~["'":"(APOS)"]
    result = <\ "printf '%s' '" safe "' | jq -Rs ." \>
    <~ result$~~["\n":""]
}

// Extract a dotted-path field from a JSON object string
// e.g. decode(json, "message.content") → field value
decode(json_str, key) {
    safe = json_str$~~["'":"(APOS)"]
    <\ "printf '%s' '" safe "' > /tmp/ZethyCLI_dec.json" \>
    result = <\ "jq -r '." key "' /tmp/ZethyCLI_dec.json" \>
    <~ result$~~["\n":""]
}

// Remove trailing newline (G7 — auto-trimmed by interpreter since Sprint 1)
trim_nl(s) { <~ s$~~["\n":""] }
```

**G4 workaround**: content with `'` is sanitized to `(APOS)` before any
BashExec interpolation. This loses the original character. Full fix requires
`std/json` or `std/io`.

---

### lib/ollama.zy

Ollama API client. Uses `http.zy` and `json.zy` via imports.
Model presets exposed as exported `:=` constants. The active host is private
mutable state (`host_base`) — changed via `set_host()`, read by all API functions.

```zymbol
# ollama

#> {
    chat
    models
    is_running
    set_host
    get_host
    MODEL_FAST
    MODEL_THINK
}

<# ./http <= http
<# ./json <= json

MODEL_FAST  := "gemma3:4b"
MODEL_THINK := "deepseek-r1:1.5b"

host_base = "http://localhost:11434"   // private mutable — G13 write-back

// Change Ollama host. Normalizes to http://host:port
// "192.168.1.1"       → "http://192.168.1.1:11434"
// "192.168.1.1:9999"  → "http://192.168.1.1:9999"
// "http://..."        → unchanged
set_host(addr) {
    proto_pos = addr$?? "://"
    ? (proto_pos$# == 0) {
        colon_pos = addr$?? ":"
        ? (colon_pos$# == 0) {
            host_base = "http://" $++ addr ":11434"
        } _ {
            host_base = "http://" $++ addr
        }
    } _ {
        host_base = addr
    }
}

get_host() { <~ host_base }

// Send chat request. Builds URL from host_base each call.
chat(model, msgs_file, think) {
    base      = host_base
    chat_url  = base $+ "/api/chat"
    think_val = "false"
    req_model = model
    req_msgs  = msgs_file
    ? think { think_val = "true" }
    <\ "(printf '{{\"model\":\"%s\",\"messages\":' '" req_model "'; cat " req_msgs "; printf ',\"stream\":false,\"think\":" think_val "}') > /tmp/ZethyCLI_req.json" \>
    <\ "curl -s -X POST -H 'Content-Type: application/json' -d '@/tmp/ZethyCLI_req.json' " chat_url " > /tmp/ZethyCLI_resp.json" \>
    <\ "jq -r '.message.content // empty' /tmp/ZethyCLI_resp.json > /tmp/ZethyCLI_content.txt 2>/dev/null" \>
    content = <\ "cat /tmp/ZethyCLI_content.txt" \>
    <~ content
}

models() {
    base     = host_base
    tags_url = base $+ "/api/tags"
    <\ "curl -s " tags_url " > /tmp/ZethyCLI_tags.json" \>
    result = <\ "jq -r '[.models[].name] | join(\", \")' /tmp/ZethyCLI_tags.json 2>/dev/null" \>
    <~ result$~~["\n":""]
}

is_running() {
    base     = host_base
    base_url = base $+ "/"
    status   = http::get_status(base_url)
    <~ status == "200"
}
```

---

### lib/history.zy

Conversation history stored in `/tmp/ZethyCLI_msgs.json` (JSON array) and
`/tmp/ZethyCLI_count.txt` (turn counter).

File-based storage is used here instead of G13 module state because:
- History must survive program restarts
- The file is passed directly to curl via `--slurpfile` (no re-serialization)

```zymbol
# history

#> {
    init
    add_user
    add_assistant
    get_msgs_file
    get_count
    clear
}

MSGS_FILE  := "/tmp/ZethyCLI_msgs.json"
COUNT_FILE := "/tmp/ZethyCLI_count.txt"

get_msgs_file() { <~ MSGS_FILE }

get_count() {
    raw = <\ "cat " COUNT_FILE " 2>/dev/null || echo 0" \>
    <~ #|raw$~~["\n":""]|
}

// Initialize conversation with a system prompt
init(system_prompt) {
    safe = system_prompt$~~["'":"(APOS)"]
    <\ "jq -n --arg c '" safe "' '[{{role:\"system\",content:$c}}]' > " MSGS_FILE \>
    <\ "printf '1' > " COUNT_FILE \>
}

// Append a user message
add_user(raw_content) {
    safe = raw_content$~~["'":"(APOS)"]
    <\ "jq --arg c '" safe "' '. += [{{role:\"user\",content:$c}}]' " MSGS_FILE " > /tmp/ZethyCLI_tmp.json && mv /tmp/ZethyCLI_tmp.json " MSGS_FILE \>
    raw_n = <\ "cat " COUNT_FILE " 2>/dev/null || echo 0" \>
    n = #|raw_n$~~["\n":""]|
    <\ "printf '" (n + 1) "' > " COUNT_FILE \>
}

// Append an assistant message
add_assistant(raw_content) {
    safe = raw_content$~~["'":"(APOS)"]
    <\ "jq --arg c '" safe "' '. += [{{role:\"assistant\",content:$c}}]' " MSGS_FILE " > /tmp/ZethyCLI_tmp.json && mv /tmp/ZethyCLI_tmp.json " MSGS_FILE \>
    raw_n = <\ "cat " COUNT_FILE " 2>/dev/null || echo 0" \>
    n = #|raw_n$~~["\n":""]|
    <\ "printf '" (n + 1) "' > " COUNT_FILE \>
}

// Reset conversation history
clear() {
    <\ "rm -f " MSGS_FILE " " COUNT_FILE \>
}
```

**jq `{{` note**: inside BashExec string literals, `{{` → `{` after Zymbol escaping.
jq sees the correct `{role:...}` object syntax.

---

## BashExec Syntax (G12)

The `<\ \>` construct tokenizes its contents as normal Zymbol expressions.
Arguments are evaluated and concatenated (no implicit separator).

```zymbol
// Before G12 (old — opaque string with {var} micro-syntax):
result = <\ curl "-s" {url} \>

// After G12 (current — Zymbol expressions):
result = <\ "curl -s " url \>           // string literal + variable
result = <\ "curl -s " url " -X POST" \> // three args concatenated
```

String interpolation inside string literals still works normally:
```zymbol
file_path = "/tmp/out.json"
result = <\ "cat {file_path}" \>        // → cat /tmp/out.json
```

Use `\{` and `\}` to include literal braces in a string:
```zymbol
// jq object syntax inside BashExec string:
<\ "jq -n '\{key:\"val\"\}'" \>         // → jq -n '{key:"val"}'
```

---

## Gap Status

| ID | Gap | Status | Workaround / Fix |
|----|-----|--------|------------------|
| G1 | No `std/net` HTTP client | CRITICAL (open) | curl via BashExec in `http.zy` |
| G2 | No `std/json` encode/decode | CRITICAL (open) | jq via BashExec in `json.zy` |
| G3 | No `std/io` file read/write | CRITICAL (open) | cat/printf via BashExec |
| G4 | BashExec shell-safety (single quotes) | SIGNIFICANT (open) | sanitize `'` → `(APOS)` |
| G5 | String join | — | ✅ `,` operator already exists |
| G6 | String split | — | ✅ `str / char` operator |
| G7 | BashExec trailing `\n` | — | ✅ auto-trimmed by interpreter |
| G8 | BashExec not usable as expression | — | ✅ resolved by G12 redesign |
| G9 | Module `.CONST` access broken | — | ✅ TypeChecker fix |
| G11 | `alias::fn()` as void statement | — | ✅ parser fix |
| G12 | BashExec opaque `{var}` syntax | — | ✅ redesigned — Zymbol expressions |
| G13 | Module private mutable state | — | ✅ write-back after function calls |
| G14 | `#>` must immediately follow `# name` | — | ✅ any position inside `# name { }` block |

---

## Bug Status

| ID | Bug | Status |
|----|-----|--------|
| BUG-06 | `>> "literal" (expr)` crashes — literal treated as callable | ✅ fixed — `is_callable` guard in parser |

---

## Persistent Memory — Design Summary

### Context lifecycle

```
Startup:
  memory::get_context()          → reads ~/.ZethyCLI/memory.txt (or "")
  hist::init(sys_prompt, mem_ctx) → writes /tmp/ZethyCLI/msgs.json
                                    sys_prompt + memory injected as role:system

Per turn (unchanged):
  hist::add_user(input)          → appends to /tmp/ZethyCLI/msgs.json
  ollama::chat(model, msgs, think) → sends full msgs array to model
  hist::add_assistant(response)  → appends to /tmp/ZethyCLI/msgs.json

/exit:
  ? turns > 2:
    ollama::summarize()          → one-shot call, NOT added to hist
    memory::save_summary()       → appends [date] block to ~/.ZethyCLI/memory.txt
```

### Memory file anatomy

```
~/.ZethyCLI/memory.txt (plain text, human-editable)

- El usuario trabaja en Zymbol, un lenguaje de programación en Rust
- Prefiere respuestas concisas y sin relleno
[2026-04-15 14:32]
- Discutimos el diseño de persistencia de ZethyCLI
- Se acordó usar ~/.ZethyCLI/ para datos persistentes y /tmp/ZethyCLI/ para volátiles
- El módulo memory.zy será nuevo en lib/
[2026-04-15 18:07]
- ...siguiente sesión...
```

### Memory growth

Over time `memory.txt` grows with each session summary. The model receives
the entire file as part of the system prompt. Practical limit depends on the
model's context window:

| Model | Context window | ~sessions before limit |
|-------|---------------|----------------------|
| `gemma3:4b` | 8k tokens | ~40–60 sessions |
| `deepseek-r1:1.5b` | 32k tokens | ~200+ sessions |

A future `/compress-memory` command could ask the model to consolidate the
entire memory file into a shorter version. Not in scope for Phase 2.

---

## Development Phases

### Phase 1 — Core chat ✅ complete
Files: `main.zy`, `lib/ui.zy`, `lib/json.zy`, `lib/http.zy`, `lib/ollama.zy`, `lib/history.zy`
Goal: plain text multi-turn conversation with model/mode/host switching

### Phase 2 — Persistent memory ✅ complete
Files: `lib/memory.zy` (new), `lib/history.zy` (init signature), `lib/ollama.zy` (summarize), `main.zy` (commands + exit)
Goal: cross-session memory via `~/.ZethyCLI/memory.txt`

New commands: `/remember`, `/memory`, `/forget`
Changed commands: `/exit` (auto-summary), `/clear` (keeps memory, resets only session history)

### Phase 3 — /file command ✅ complete
Goal: `/file <path>` reads a file and sends its content as a user message — model
responds immediately, file stays in conversation context for follow-up questions.

**UX flow:**
```
> /file src/main.rs
[info]  Sending file: src/main.rs (1.2 KB)
[info]  ...
  [gemma3:4b] › This Rust file defines the entry point for...

> What does the error handling look like?
  [gemma3:4b] › The error handling uses...
```

**Message format injected into history:**
```
File: src/main.rs
```
<file content here>
```
```

**Implementation:**

`lib/history.zy` — new `add_user_from_file(label, content_file)`:
Uses `--rawfile` (same pattern as `add_assistant`) to avoid shell quoting of
arbitrary file content (G4 full mitigation). Builds label + fenced block in a
temp file, then jq-appends as a user message.

```zymbol
add_user_from_file(label, content_file) {
    msg_file = tmp_dir $+ "/file_msg.txt"
    <\ "{ printf 'File: " label "\\n```\\n'; cat '" content_file "'; printf '\\n```'; } > " msg_file \>
    <\ "jq --rawfile c " msg_file " '. += [\{role:\"user\",content:$c}]' " msgs_file " > " tmp_dir "/tmp.json && mv " tmp_dir "/tmp.json " msgs_file \>
    raw_n = <\ "cat " count_file " 2>/dev/null || echo 0" \>
    n = #|raw_n$~~["\n":""]|
    new_n = n + 1
    <\ "printf '" new_n "' > " count_file \>
}
```

`main.zy` — `/file` handler:
1. Validates file exists
2. Shows file size for feedback
3. Calls `hist::add_user_from_file()`
4. Immediately calls `ollama::chat()` and shows response
5. Adds assistant response to history — file + response are now in context

```zymbol
"/file" : {
    ? (args == "") {
        ui::show_error("Usage: /file <path>")
    } _ {
        exists = <\ "test -f '" args "' && echo 1 || echo 0" \>
        ? (exists$~~["\n":""] == "1") {
            size = <\ "du -sh '" args "' | cut -f1" \>
            size = size$~~["\n":""]
            info = "Sending file: " $++ args " (" size ")"
            ui::show_info(info)
            hist::add_user_from_file(args, args)
            msgs_file = hist::get_msgs_file()
            ui::show_info("...")
            response = ollama::chat(model, msgs_file, think)
            hist::add_assistant(response)
            ui::show_response(response, model)
        } _ {
            ui::show_error("File not found: " $++ args)
        }
    }
}
```

**Limitations:**
- Paths with single quotes will fail (G4). Paths with spaces work (wrapped in `'`).
- Very large files will fill the model's context window. No size cap enforced.
- Binary files are sent as-is — garbled output is expected behavior.

### Phase 4 — Session persistence + /history (next)
Files: `lib/sessions.zy` (new), `main.zy` (startup + per-turn + `/history`), `lib/ui.zy` (help)
Goal: crash-safe session archive + `/history` to review past conversations.

**Crash recovery**: every user message is written to `~/.ZethyCLI/sessions/` before
the API call — the session survives any crash without requiring `/exit`.

**New command**: `/history`

```
> /history
  2026-04-15 14-32-07  (40 messages)
  2026-04-15 18-07-22  (12 messages)
  2026-04-16 09-11-54  (6 messages)
```

Shows all archived sessions newest-first. Each entry shows the timestamp and
message count. Sessions are read-only — viewing is the only action supported.
A future `/resume <date>` command could reload a session (Phase 5+).

**main.zy startup additions**:
```zymbol
sessions::set_tmp_dir(tmp_dir)
sessions::start()
```

**main.zy per-turn additions** (chat branch):
```zymbol
hist::add_user(input)
sessions::append_user(input)        // ← crash safe from here
msgs_file = hist::get_msgs_file()
response  = ollama::chat(model, msgs_file, think)
hist::add_assistant(response)
sessions::append_assistant()        // ← complete turn archived
ui::show_response(response, model)
```

**`/history` handler**:
```zymbol
"/history" : {
    listing = sessions::list()
    ? (listing$~~["\n":""] == "") {
        ui::show_info("No sessions archived yet.")
    } _ {
        >> "[sessions]" ¶
        >> listing ¶
    }
}
```

**`/file` handler update**: `sessions::append_user(args)` called after
`hist::add_user_from_file()` to log the file send event to the session archive.

### Phase 5 — /run command (future)
Goal: `/run <path>` executes a file via BashExec, sends stdout+stderr as context,
model sees the output and can explain errors or suggest fixes.

### Phase 6 — Full polish (future)
Goal: production-ready ZethyCLI, `/compress-memory`, rolling history (last N turns),
`/resume <date>` to reload an archived session, complete gap documentation for stdlib roadmap.
