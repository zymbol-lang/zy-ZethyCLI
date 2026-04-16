# ZethyCLI

> A real-world AI chat CLI written entirely in Zymbol.
> Primary goal: stress-test the language and map its real strengths and gaps.

---

## What is ZethyCLI?

ZethyCLI is a multi-turn conversational AI interface for [Ollama](https://ollama.com)
built exclusively in the [Zymbol programming language](https://zymbol.dev). It
connects to a local or remote Ollama instance, manages conversation history, and
provides a REPL-style interface for chatting with any installed model.

It is **not** a showcase of what Zymbol can do easily — it is a deliberate attempt
to push the language into territory where it has gaps, and to document every
friction point along the way.

---

## The Exercise

ZethyCLI is a **language validation instrument**.

Zymbol is a keyword-free symbolic programming language. Its design is intentionally
minimal — the standard library is still being built, and the language itself is still
evolving. ZethyCLI was created to answer a concrete question:

> *What happens when you build something real in Zymbol, with no shortcuts,
> no Python bridges, and no external runtime helpers?*

The methodology:

1. **Build** — implement the feature in pure Zymbol
2. **Hit the wall** — identify what is missing or broken
3. **Workaround** — find the BashExec escape hatch solution
4. **Document** — record the gap or bug with full reproduction context
5. **Repeat** — continue until the feature works

Every workaround in the codebase (`jq` for JSON, `curl` for HTTP, `cat`/`printf` for
file I/O) represents a gap in Zymbol's standard library. Every bug note in the source
comments represents a real interpreter or parser failure that was encountered and
either fixed or worked around.

---

## How to Run

```bash
# From the ZethyCLI/ directory:
zymbol run main.zy
```

**Requirements**: `ollama` running locally or on a reachable host, `jq`, `curl`.

On first run, ZethyCLI shows the list of installed models and asks you to pick one.
The selection is saved to `~/.ZethyCLI/ZethyCLI.conf`.

---

## Commands

| Command | Effect | Saved |
|---------|--------|-------|
| `/model <name>` | Switch to any installed model | ✅ |
| `/model` | Show `ollama list` | — |
| `/mode think` | Enable reasoning mode (verified via `/api/show`) | ✅ |
| `/mode fast` | Disable reasoning mode | ✅ |
| `/host <addr>` | Change Ollama host (`192.168.1.1`, `192.168.1.1:9999`) | ✅ |
| `/models` | List installed models (`ollama list`) | — |
| `/info` | Show current model, mode, host, tmp_dir, turn count | — |
| `/clear` | Reset conversation history | — |
| `/reset` | Clear all temp files, reset config, re-select model | — |
| `/help` | Show command list | — |
| `/exit` | Quit | — |

**Think mode** is validated at runtime: `/mode think` queries `/api/show` and checks
the model's `capabilities` field. If `"thinking"` is not listed, think mode is
refused with a warning. Switching models while think is active also triggers this check.

---

## Configuration

Persistent config at `~/.ZethyCLI/ZethyCLI.conf` (JSON):

```json
{
  "model":   "gemma4:latest",
  "mode":    "fast",
  "think":   false,
  "host":    "http://localhost:11434",
  "tmp_dir": "/tmp/ZethyCLI"
}
```

`tmp_dir` is only applied at startup — edit the file and restart to change it.

---

## Project Structure

```
ZethyCLI/
  main.zy          Entry point — REPL loop, command dispatcher, startup
  lib/
    config.zy      Persistent config — ~/.ZethyCLI/ZethyCLI.conf (jq + BashExec)
    ui.zy          Terminal output — pure Zymbol, no BashExec
    history.zy     Conversation state — file-based JSON, /tmp/ZethyCLI/
    json.zy        JSON encode/decode — jq via BashExec (G2 workaround)
    http.zy        HTTP client — curl via BashExec (G1 workaround)
    ollama.zy      Ollama API client — imports http + json
  README.md        This file
  DESIGN.md        Architecture, module designs, BashExec patterns
  BUGS.md          Runtime bugs found and fixed (or open)
  GAPS.md          Language gaps found during development
```

No external Python. No shell scripts. All logic is Zymbol — standard library gaps
are bridged exclusively through `<\ BashExec \>`.

---

## Language Findings

### What Works Well

**`>>` juxtaposition with `¶`** — terminal output is remarkably readable.
`ui.zy` contains zero helper functions and still produces clean box-drawing output:
```zymbol
>> "  [" model "] › " text ¶
```

**`??` match** — the command dispatcher is expressive and flat. No nested if/else chains.

**Module private mutable state (G13)** — `host_base` and `tmp_dir` in `ollama.zy`
as singleton state, changed via `set_host()` / `set_tmp_dir()`, work exactly as
expected. The module-as-toolbox design is sound.

**`$~~` `$??` `$[..]` `$#`** — string operators are expressive once learned.

**BashExec as escape hatch** — pragmatic and powerful. The entire project would be
impossible without it. This validates the design decision.

---

### Bugs Found

#### BUG-07 — `}}` produces two `}` (asymmetry with `{{`)

`{{` in a Zymbol string escapes to `{`. But `}}` is **not** an escape — it produces
two literal `}` characters. The asymmetry caused a silent failure in `config.zy`:
the jq filter received `...tmp_dir}}` (invalid), failed silently, and the redirect
`> ~/.ZethyCLI/ZethyCLI.conf` created an empty file. The model was never saved between runs.

This is hard to debug: BashExec does not expose stderr to Zymbol code.

**Status**: Open. See `BUGS.md § BUG-07`.

---

### Gaps Found

#### G15 — `!=` does not exist, error message gives no hint

`!=` is not a valid Zymbol operator. The parser misidentifies `!` as the start of
the `!?` safe-access operator, generating a misleading error that never mentions `<>`
(the correct not-equal operator). Every programmer coming from any other language will
hit this on their first conditional with inequality.

**Status**: Open. See `GAPS.md § G15`.

#### G16 — Function parameters invisible in BashExec without local copy

Parameters of a function are not accessible in `<\ \>` BashExec expressions unless
explicitly copied to a local body variable first. This is a consequence of the
write-back mechanism (G13), but it is completely undocumented and causes silent
failures. The pattern repeats in every module that uses BashExec:

```zymbol
chat(model, msgs_file) {
    req_model = model      // mandatory copy — params not visible in BashExec
    req_msgs  = msgs_file  // mandatory copy
    <\ "curl ... " req_model " ... " req_msgs \>
}
```

**Status**: Open. See `GAPS.md § G16`.

#### G17 — No top-level functions in script files

Script files (without `# module_name` header) cannot define reusable functions.
This forced the model-selection picker to be duplicated verbatim in two places
in `main.zy` (startup and `/reset` handler).

**Status**: Open. See `GAPS.md § G17`.

#### G18 — String concatenation inside function arguments

The `,` join operator is ambiguous with the argument separator in function call
context. `$++` (string builder) requires a base variable as first operand and
does not work inline. A temporary variable is always required:

```zymbol
warn = "Model " $++ model " does not support think mode"
ui::show_error(warn)   // cannot inline the concat in the argument
```

**Status**: Open. See `GAPS.md § G18`.

---

### Critical Missing Standard Library (BashExec workarounds in use)

| Missing | Workaround | Module |
|---------|------------|--------|
| `std/net` — HTTP client | `curl` via BashExec | `lib/http.zy` |
| `std/json` — encode/decode | `jq` via BashExec | `lib/json.zy` |
| `std/io` — file read/write | `cat` / `printf` via BashExec | `lib/history.zy` |

These three gaps (G1, G2, G3) account for roughly 80% of the BashExec usage in the
project. Once `std/net`, `std/json`, and `std/io` exist, the `lib/http.zy`,
`lib/json.zy` modules and most of `lib/history.zy` can be replaced with native Zymbol.

---

## See Also

- `DESIGN.md` — architecture decisions, module designs, BashExec patterns
- `BUGS.md` — confirmed runtime bugs with reproduction steps and fix notes
- `GAPS.md` — language gaps with severity, workarounds, and fix proposals
- `interpreter/MANUAL.md` — authoritative Zymbol language reference
- `interpreter/ROADMAP.md` — planned features including `std/net`, `std/json`, `std/io`
