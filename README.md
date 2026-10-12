# ZethyCLI

🌐 [zymbol-lang.org](https://zymbol-lang.org) · ▶ [playground](https://zymbol-lang.org/playground.html)

> **Updated for Zymbol v0.0.8 — 2026-08-31**

> A real-world AI chat CLI written entirely in Zymbol.
> Primary goal: stress-test the language and map its real strengths and gaps.

> **Validation project for Zymbol.** Built across v0.0.3–v0.0.7 to push the
> language into territory where it had gaps — and to drive the standard library
> that filled them.

---

## What is ZethyCLI?

ZethyCLI is a multi-turn conversational AI REPL written exclusively in the
[Zymbol programming language](https://zymbol.dev). It talks to **either** of two
providers, selectable at runtime:

- **`ollama`** — a local [Ollama](https://ollama.com) instance (models on your machine)
- **`gemini`** — Google's Gemini web API (cloud, needs an API key)

It manages conversation history, cross-session memory, and session archives, and
provides slash-commands for everything.

---

## The Exercise

ZethyCLI is a **language validation instrument**.

The methodology was: build a real feature in pure Zymbol → hit the wall where the
standard library was missing → work around it with the `<\ BashExec \>` escape hatch
→ document the gap → and use that to prioritize the stdlib roadmap.

That loop **worked**: the three biggest gaps it documented — `std/net` (G1),
`std/json` (G2), and `std/io` (G3) — were implemented in Zymbol **v0.0.7**, and
ZethyCLI has now been **migrated off the workarounds onto the real standard
library**. The `curl`/`jq`/`cat`/`printf` plumbing is gone; HTTP, JSON, and file
I/O are native.

Remaining `<\ \>` calls are only for legitimate, non-gap cases: the `ollama` CLI
(`command -v`, `ollama list`), resolving `$HOME` (`printenv HOME` — `std/io` does
not expand `~`), and timestamps (`date`).

> **Requires the Zymbol v0.0.7 interpreter** (with `std/net`, `std/json`, `std/io`
> and `net` header support). Build/install it from `interpreter/`:
> ```bash
> bash interpreter/install-zymbol.sh   # symlinks the v0.0.7 build into /usr/bin
> ```

---

## How to Run

```bash
# From the ZethyCLI/ directory:
zymbol run main.zy
```

**Requirements**

- Zymbol **v0.0.7** (`zymbol -V`)
- For the **ollama** provider: a running Ollama daemon + at least one model pulled
- For the **gemini** provider: a free API key from <https://aistudio.google.com/apikey>

No `jq` / `curl` needed anymore — those were the old workarounds.

On first run (no config) ZethyCLI defaults to the **ollama** provider and asks you
to pick from `ollama list`. The choice is saved to `~/.ZethyCLI/ZethyCLI.conf`.

---

## Choosing local vs cloud

The provider and model are two separate settings (both persisted):

```text
# Local (Ollama)
/provider ollama
/model gemma3:4b

# Cloud (Gemini)
/provider gemini
/key YOUR_API_KEY        # gemini-flash-latest is the default model
```

The banner and `/info` always show the active provider.

---

## Commands

| Command | Effect | Saved |
|---------|--------|-------|
| `/provider <ollama\|gemini>` | Switch provider (local vs cloud) | ✅ |
| `/key <api_key>` | Set the API key for the cloud provider | ✅ |
| `/model <name>` | Switch model (validated against `ollama list` for ollama) | ✅ |
| `/model` | Show `ollama list` (ollama) / current model (gemini) | — |
| `/mode think` | Enable reasoning mode (ollama: verified via `/api/show`) | ✅ |
| `/mode fast` | Disable reasoning mode | ✅ |
| `/host <addr>` | Change Ollama host (ollama only) | ✅ |
| `/models` | List installed Ollama models | — |
| `/info` | Show provider, model, mode, host, tmp_dir, turn count | — |
| `/file <path>` | Send a file's content to the model as context | — |
| `/remember <fact>` | Save a fact to persistent memory | — |
| `/memory` | Show saved memory | — |
| `/forget` | Clear persistent memory | — |
| `/history` | List archived sessions with message counts | — |
| `/clear` | Reset conversation history (memory preserved) | — |
| `/reset` | Clear temp files + config, restart | — |
| `/help` | Show command list | — |
| `/exit` | Quit (auto-saves a session summary to memory) | — |

---

## Configuration

Persistent config at `~/.ZethyCLI/ZethyCLI.conf` (JSON):

```json
{
  "model":    "gemma3:4b",
  "mode":     "fast",
  "think":    false,
  "host":     "http://localhost:11434",
  "tmp_dir":  "/tmp/ZethyCLI",
  "provider": "ollama",
  "api_key":  ""
}
```

`provider` selects local (`ollama`) vs cloud (`gemini`); `api_key` is used by the
cloud provider. `tmp_dir` is applied only at startup.

---

## Project Structure

```
ZethyCLI/
  main.zy          Entry point — REPL, command dispatch, provider routing
  lib/
    config.zy      Persistent config (~/.ZethyCLI/ZethyCLI.conf)
    ui.zy          Terminal output — pure Zymbol
    history.zy     Conversation state — std/io + std/json
    memory.zy      Cross-session memory — std/io
    sessions.zy    Per-run session archive — std/io + std/json
    ollama.zy      Local provider — std/net + std/json + std/io
    gemini.zy      Cloud provider — std/net (auth header) + std/json + std/io
  README.md · DESIGN.md · BUGS.md · GAPS.md
```

Both providers expose the same `chat` / `summarize` contract, so `main.zy` routes
to either one through a single `do_chat` / `do_summarize` dispatch.

> The old workaround modules `lib/http.zy` (curl), `lib/json.zy` (jq), and
> `lib/file.zy` (cat/printf) were **removed** in the v0.0.7 migration.

---

## Language Findings

### Standard library gaps — now resolved (v0.0.7)

| Gap | Was | Now |
|-----|-----|-----|
| **G1** `std/net` (HTTP) | `curl` via BashExec (`lib/http.zy`) | ✅ `net::get` / `net::post_json` (with optional auth headers) |
| **G2** `std/json` | `jq` via BashExec (`lib/json.zy`) | ✅ `json::encode` / `json::decode` (object↔NamedTuple) |
| **G3** `std/io` | `cat`/`printf` via BashExec (`lib/file.zy`) | ✅ `io::read` / `write` / `append` / `list` / `delete` |
| **G4** shell-quoting of content | sanitize `'`→`(APOS)` | ✅ mostly gone — content no longer interpolated into shell strings |

Driving these three modules into the language and then migrating onto them is the
project's headline result. See `GAPS.md` for the full log.

### What works well

- **`>>` juxtaposition with `¶`** — terminal UI is remarkably readable.
- **`??` match** — flat, expressive command dispatcher.
- **Module private mutable state** — `host_base`, provider config, session file as
  singleton state via setters works as expected.
- **String operators** `$~~` `$??` `$[..]` `$#` — expressive once learned.
- **Soft error values** — `std/net` returns failures as `Value::Error`, checked with
  `$!`, so a dropped connection or bad key never crashes the REPL.

### Still-open friction

- **Concat inside function arguments** — juxtaposition concatenation is rejected as a
  function-call argument; you must build the string in a variable first
  (`url = base model; net::get(url)`).
- **No safe field access on decoded JSON** — accessing a missing NamedTuple field
  errors; `config.zy` is deliberately kept on `jq` for its tolerant `// empty` defaults.
- **Module-level vars must be literals** — can't init from an expression/BashExec at
  module top level (resolve `$HOME` in a per-call helper instead).

See `GAPS.md` and `BUGS.md` for the complete, versioned findings.

---

## See Also

- `DESIGN.md` — architecture decisions and module designs
- `BUGS.md` — confirmed runtime bugs with reproduction steps
- `GAPS.md` — language gaps (G1–G16) with status and fixes
- `interpreter/GUIDE.md` — authoritative Zymbol language reference
- `interpreter/IMPL_V007.md` — the v0.0.7 stdlib that filled G1/G2/G3
