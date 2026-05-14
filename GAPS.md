# ZethyCLI — Language Gaps Log

> Living document. Each gap is discovered during development of ZethyCLI.
> Goal: feed these findings directly into Zymbol's standard library roadmap.

---

## Gap Classification

- **CRITICAL** — Feature cannot be implemented in pure Zymbol; BashExec bridge required
- **SIGNIFICANT** — Implementable but requires verbose workarounds
- **MINOR** — Cosmetic or style limitation, acceptable workaround exists

---

## G1 — No `std/net` (HTTP client)

**Status**: propuesto v0.0.6 — diseñado en `IMPL_V007.md` (Step 10)
**Nature**: Intentional — documented to define stdlib roadmap priority
**Module affected**: `lib/http.zy`

ZethyCLI was built without a native HTTP client by design. The goal was to
validate that the language is expressive enough to bridge external networking
via BashExec/curl, and to document precisely which HTTP capabilities would
need to be native. This was not a surprise gap — the absence was known from
the start and accepted as a deliberate trade-off for v0.0.3.

Constraints confirmed by the exercise:
- Networking is tied to shell availability (acceptable for a CLI tool, not for portable scripts)
- No timeout control from Zymbol code
- No access to response headers or HTTP status codes from Zymbol logic
- SSL/auth options require raw curl flags as strings

**Workaround used**: `<\ curl "-s" "-X" "POST" ... {url} \>`
**Roadmap**: `std/net` module — `net::post(url, headers, body)` returning `(status, body)`

---

## G2 — No `std/json` (JSON encode/decode)

**Status**: propuesto v0.0.6 — diseñado en `IMPL_V007.md` (Step 9)
**Nature**: Intentional — documented to define stdlib roadmap priority
**Module affected**: `lib/json.zy`, `lib/ollama.zy`, `lib/history.zy`

ZethyCLI was built without native JSON serialization by design. The goal was to
validate that JSON workflows (Ollama API payload construction, response field
extraction, history serialization) are achievable via BashExec/jq, and to
document precisely which JSON operations would need to be native. This was not
a surprise gap — the absence was known from the start and accepted as a
deliberate trade-off for v0.0.3.

Operations confirmed as necessary:
- No `json::encode(value)` → string
- No `json::decode(string)` → typed value
- No `json::get(string, key)` → field extraction

**Workaround used**: `<\ jq "-Rs" "." file \>` for encoding, `<\ jq "-r" ".field" file \>` for decoding
**Roadmap**: `std/json` module — `json::encode(v)`, `json::decode(s)`, `json::get(s, key)`

---

## G3 — No `std/io` (file read/write)

**Status**: propuesto v0.0.6 — diseñado en `IMPL_V007.md` (Step 8)
**Nature**: Intentional — documented to define stdlib roadmap priority
**Module affected**: `lib/file.zy`, `lib/history.zy`

ZethyCLI was built without native file I/O by design. The goal was to validate
that file-based workflows (chat history persistence, temp payload delivery to
curl) are achievable via BashExec, and to document precisely which I/O
primitives would need to be native. This was not a surprise gap — the absence
was known from the start and accepted as a deliberate trade-off for v0.0.3.

Operations confirmed as necessary:
- Read file to string: requires `<\ "cat " path \>`
- Write string to file: requires `<\ "printf '%s' '" safe_content "' > " path \>`
- Append to file: requires `<\ "echo '" safe_content "' >> " path \>`

**Workaround used**: `<\ "cat " path \>` for read, `<\ "printf '%s' '" content "' > " path \>` for write
**Roadmap**: `std/io` module — `io::read(path)`, `io::write(path, content)`, `io::append(path, content)`

---

## G4 — BashExec shell-safety (single quotes in content)

**Status**: workaround — sanitizar `'` con `$~~`; fix completo cuando `std/io` + `std/net` (v0.0.6) eliminen BashExec de la mayoría de casos
**Discovered**: Design phase
**Module affected**: `lib/json.zy`, `lib/file.zy`, `lib/history.zy`

When a variable contains single quotes (`'`), its interpolation inside
a BashExec string command can break shell syntax.

Example:
```zymbol
msg = "it's broken"
<\ "printf '%s' '" msg "' > /tmp/out.txt" \>
// Shell sees: printf '%s' 'it's broken'  ← syntax error
```

**Workaround**: Pre-sanitize user input before any BashExec interpolation:
```zymbol
safe = raw$~~["'":"(APOS)"]   // loses fidelity — documented limitation
```

**Note**: The old `{var}` interpolation mechanism was replaced in the BashExec
redesign (see below). Single-quote safety is still a concern when embedding
variable content inside shell command strings.

**Future fix**: `std/io::write` + `std/net` make BashExec unnecessary for most I/O.

---

## G5 — Join: `,` operator entre strings

**Status**: ✅ YA EXISTE — documentado
**Discovered**: Design phase

Join de strings se hace con el operador `,` entre expresiones de tipo string.
La filosofía del lenguaje es simbólica — no existen keywords como `join`.

```zymbol
s = "hello" , " " , "world" , "!"   // → "hello world!"
msg = "User: " , username , " logged in"
```

El operador `,` concatena strings sin separador implícito.
Para separadores variables, simplemente se incluyen como string literal entre los operandos.

---

## G6 — Split: `/` operador sobre strings

**Status**: ✅ YA EXISTE — documentado (falso warning corregido)
**Discovered**: Design phase

Split de string se hace con el operador `/` entre un string y un char.
Retorna un array de strings. La filosofía es simbólica — divisón de string = dividir en partes.

```zymbol
parts = "a,b,c,d" / ','          // → [a, b, c, d]
words = "hello world" / ' '      // → [hello, world]
path  = "/home/user/f.zy" / '/'  // → [, home, user, f.zy]
```

El analizador semántico emitía un falso warning "arithmetic on non-numeric"
al usar `/` sobre `String`. Corregido: el type checker ahora reconoce
`String / Char → Array(String)` como operación válida sin advertencia.
**Fix**: `zymbol-semantic/src/type_check.rs` — arm `BinaryOp::Div`.

---

## G7 — BashExec output always includes trailing `\n`

**Status**: SIGNIFICANT → ✅ RESUELTO en Sprint 1
**Discovered**: Design phase
**Module affected**: Everywhere BashExec result is used in string context

Every `<\ cmd \>` result ends with `\n`. When used in string interpolation
or comparisons this causes subtle bugs (e.g. `response == "200"` fails
because response is `"200\n"`).

**Workaround used**: `trimmed = raw$~~["\n":""]` after every BashExec call

**Fix implementado**: `eval_bash_exec` ahora aplica `trim_end_matches('\n')` antes de retornar. Comportamiento idéntico a shell command substitution `$(...)`. Newlines internos se preservan; solo el trailing final se elimina. Archivo: `zymbol-interpreter/src/script_exec.rs`.

---

## G8 — `??` match arms cannot contain BashExec

**Status**: SIGNIFICANT → ✅ RESUELTO por G12
**Discovered**: Design phase
**Module affected**: `lib/executor.zy`

El rediseño G12 convirtió BashExec en `BashExecExpr { args: Vec<Expr> }` — nodo de
expresión en el AST. Ahora puede usarse directamente en match arms y assignments.

```zymbol
// ✅ Funciona tras G12:
output = ?? lang {
    "py" : <\ "python3 " tmp \>
    "sh" : <\ "bash " tmp \>
    _    : "unsupported"
}
```

**Fix**: G12 — `zymbol-lexer`, `zymbol-ast`, `zymbol-parser`, `zymbol-interpreter`.

---

## G9 — Module constant access via `.` broken

**Status**: MINOR → ✅ RESUELTO (2026-04-14)
**Discovered**: Known — documentado en ROADMAP.md y MANUAL.md L3
**Module affected**: `lib/ollama.zy` (`OLLAMA_URL`, `MODEL_FAST`, `MODEL_THINK`)

`alias.CONST` fallaba con "undefined variable 'alias'" antes de que el intérprete
pudiera evaluar el acceso. La causa raíz estaba en el **TypeChecker**, no en el
intérprete: `infer_expr` sobre `MemberAccess` llamaba `infer_expr(Identifier("u"))`
sin saber que `u` es un alias de módulo, emitiendo un error semántico fatal que
bloqueaba la ejecución.

```zymbol
// ✅ Funciona:
<# ./config <= cfg
url = cfg.BASE_URL
model = cfg.DEFAULT_MODEL
>> url ¶
```

**Fix implementado**: `TypeChecker` ahora registra los import aliases de `program.imports`
en `module_aliases: HashSet<String>` antes de los tres passes de análisis. En `infer_expr`
para `Identifier`, si el nombre está en `module_aliases` retorna `ZymbolType::Any` sin error.
Archivo: `zymbol-semantic/src/type_check.rs`.

---

## Benchmark Results (2026-04-12)

Models validated on this machine for ZethyCLI:

| Model | Size | Time (simple prompt) | Thinking | Role |
|-------|------|---------------------|----------|------|
| `gemma3:4b` | 3.3 GB | 17.5s | ❌ | **fast** (default) |
| `qwen2.5:3b` | 1.9 GB | 18.8s | ❌ | fast alt |
| `phi4-mini` | 2.5 GB | 19.6s | ❌ | fast alt |
| `llama3.2:3b` | 2.0 GB | 21.2s | ❌ | fast alt |
| `deepseek-r1:1.5b` | 1.1 GB | 43.3s | ✅ 1.2k chars | **think** |
| `qwen3:1.7b` | 1.4 GB | 47.5s | ✅ 1.0k chars | think alt |
| `deepseek-r1:7b` | — | ~9 min | ✅ | removed (inviable) |

**Selected for ZethyCLI switch:**
- `/mode fast` → `gemma3:4b`
- `/mode think` → `deepseek-r1:1.5b`

---

## G11 — `alias::fn()` no puede usarse como void statement

**Status**: SIGNIFICANT → ✅ RESUELTO en Sprint 3
**Discovered**: Phase 1 de ZethyCLI
**Module affected**: `main.zy`

Una llamada de función de módulo `alias::fn()` no puede usarse como statement sin asignación. El parser la interpreta como una expression inválida al nivel de statement.

**Reproduce**:
```zymbol
ui::banner()          // error: parse error at statement level
ui::show_info("hi")  // error: parse error at statement level
```

**Workaround used**: `disc = ui::banner()` — asignación forzada a variable descartable.

**Fix implementado**: Agregado chequeo `peek_ahead(1) == ScopeResolution` en `parse_statement` que despacha al path `parse_function_call_statement`. Archivo: `zymbol-parser/src/lib.rs`.

---

## G12 — BashExec diseño opaco — `{var}` raw string

**Status**: CRITICAL → ✅ REDISEÑADO

El diseño original de `<\ \>` consumía el contenido como un string crudo opaco.
Las variables se referenciaban con `{var}` — un micro-lenguaje interno invisible
para el lexer, parser, semantic analyzer, type checker y LSP.

Consecuencias:
- Variables usadas en BashExec marcadas como "unused" por el analizador
- Sin autocompletado LSP dentro de `<\ \>`
- Sin type checking de los valores interpolados
- Sintaxis `{var}` inconsistente con el resto del lenguaje

**Fix implementado**: BashExec rediseñado — `<\` y `\>` son tokens delimitadores.
El contenido es tokenizado normalmente como expresiones Zymbol:
```zymbol
// Antes (opaco):
result = <\ curl "-s" {url} \>

// Después (expresiones Zymbol):
result = <\ "curl -s " url \>           // concatenación de string + variable
result = <\ "curl -s {url}" \>          // string con interpolación Zymbol
result = <\ "ls" ' ' dir \>             // tres expresiones: str + char + var
```

Las variables son ahora `Identifier` normales — visibles para el análisis semántico,
type checking y LSP. String interpolation `{var}` dentro de strings funciona
normalmente. Sin separador implícito — usar char/string literales para espacios.

**Archivos modificados**:
- `zymbol-lexer/src/lib.rs` — `BashOpen`/`BashClose` reemplazan `BashCommand(String)`
- `zymbol-lexer/src/io.rs` — `bash_depth` tracking para detectar `\>`
- `zymbol-ast/src/script_exec.rs` — `BashExecExpr { args: Vec<Expr> }`
- `zymbol-parser/src/script_exec.rs` — parseo de expresiones entre `BashOpen`/`BashClose`
- `zymbol-interpreter/src/script_exec.rs` — evalúa cada arg, concatena
- `zymbol-semantic/src/variable_analysis.rs` — traversal de `args`
- `zymbol-semantic/src/def_use.rs` — traversal de `args`
- `zymbol-formatter/src/visitor.rs` — format args con espacios
- `zymbol-compiler/src/lib.rs` — compila args a registros
- `zymbol-vm/src/lib.rs` — trim `\n` agregado (parity con tree-walker)

---

## G13 — Module state does not persist across calls

**Status**: ✅ RESUELTO (2026-04-14) — módulo state privado implementado
**Discovered**: ZethyCLI Sprint 4 — `history.zy` rewrite
**Module affected**: `lib/history.zy`

Module-level mutable variables now persist across calls via write-back after each
function execution. The design enforces strict privacy:

- Variables declared with `=` at module level → **private state**, persists across calls,
  NOT accessible from outside (not in `constants`), only via exported getter/setter functions
- Variables declared with `:=` at module level → **exported constants**, immutable,
  accessible externally as `alias.CONST`
- `#>` block → only exports functions and `:=` constants; mutable `=` variables are
  silently excluded even if listed

```zymbol
# counter
#> { increment, get_value }

count = 0          // private mutable state — persists between calls

increment() { count = count + 1 }
get_value() { <~ count }
```

```zymbol
// From outside:
c::increment()         // ✅ — count becomes 1
c::increment()         // ✅ — count becomes 2
n = c::get_value()     // ✅ — n = 2
x = c.count            // ✗ Runtime error: Module 'c' has no constant 'count'
```

**Fix implementado**:
- `zymbol-interpreter/src/functions_lambda.rs` — write-back post-ejecución:
  after function body, before `restore_call_state`, reads modified values for all
  keys that existed in `module.all_variables` and writes them back to `LoadedModule`.
  Only module-level keys are candidates — parameters and function-local vars are
  excluded automatically (they were not in `all_variables` at load time).
- `zymbol-interpreter/src/modules.rs` — export filter: `#>` items now check
  `is_const(name)` before adding to `constants`; mutable variables are silently skipped.
- `zymbol-interpreter/src/lib.rs` — `is_const` promoted to `pub(crate)`.

---

## G14 — `#>` export block must immediately follow `# module_name`

**Status**: ✅ RESUELTO (2026-04-14)
**Discovered**: ZethyCLI Sprint 4 — `ollama.zy` rewrite

`#>` ahora puede aparecer en cualquiera de las dos posiciones:
1. Inmediatamente después de `# module_name` (posición original)
2. Después de los `<#` imports (posición nueva)

Ambas son válidas. Comentarios entre `# name` y `#>` siguen siendo permitidos
(el lexer los descarta antes del parser).

```zymbol
// ✅ Forma 1 — #> antes de imports:
# module_name
#> { fn, CONST }
<# ./dep <= dep
code...

// ✅ Forma 2 — #> después de imports (G14 fix):
# module_name
<# ./dep <= dep
#> { fn, CONST }
code...

// ✅ Forma 3 — comentarios entre # y #> (siempre funcionó):
# module_name
// descripción del módulo
#> { fn, CONST }
code...
```

**Fix implementado**: `parse()` en `zymbol-parser/src/lib.rs` — después del loop de imports,
si el siguiente token es `ExportBlock` y el módulo aún no tiene export block, se parsea y
adjunta al `module_decl`. Si ya había un `#>` en posición 1, este segundo check se salta.

---

## G15 — Operador `!=` no existe — descubribilidad del operador `<>`

**Status**: ✅ RESUELTO
**Nature**: Deliberate probe — used `!=` intentionally to observe error quality and confirm the fix was required
**Context**: ZethyCLI — `lib/config.zy`

`!=` is not a valid Zymbol operator — the correct operator is `<>` (not equal),
consistent with the language's symbolic design. The use of `!=` in `lib/config.zy`
was deliberate: the goal was to observe exactly what error experience a programmer
encounters when reaching for the familiar C/Python not-equal operator, and to
confirm whether the existing diagnostic was actionable enough or a targeted fix
was required.

**Result of the probe**: The original error was not actionable — it did not name
`<>` or suggest the correct operator. This confirmed that a lexer-level diagnostic
with a direct help message was required, not just "nice to have."

**Fix implementado**: El lexer detecta `!=` como una secuencia específica y emite
un diagnostic con mensaje directo y help accionable, antes de que el parser vea
nada. Archivo: `zymbol-lexer/src/lib.rs`.

```zymbol
? (m != "") { }
// error: '!=' is not a valid Zymbol operator
//   --> file.zy:1:6
//    |  ? (m != "") { }
//    |       ^^
// help: use '<>' for not-equal  →  a <> b
```

**Operador correcto**:
```zymbol
? (m <> "") { cfg_model = m }   // ✅
```

`!=` no se agrega como alias — Zymbol mantiene un símbolo por operación.
El diagnostic es suficiente para guiar al programador.

---

## G16 — Parámetros de función no visibles en BashExec sin copia local

**Status**: ✅ RESUELTO (implícito por G12)
**Discovered**: ZethyCLI — `lib/ollama.zy`, `lib/history.zy`

El bug existía antes del rediseño G12 de BashExec, cuando el contenido de `<\ \>`
era un string opaco y las variables se referenciaban con `{var}` interno. En ese
esquema, la resolución de variables era diferente al scope normal de Zymbol.

Tras G12, BashExec evalúa cada argumento como una expresión Zymbol completa.
Los parámetros de función son variables normales en el scope del intérprete —
visibles en `<\ \>`, `? { }`, `@ { }` y `?? { }` sin ninguna copia.

```zymbol
// ✅ Funciona directamente (sin copias):
chat(model, msgs_file, think) {
    chat_url = host_base $+ "/api/chat"   // módulo var usado directamente
    ? think { think_val = "true" }
    <\ "curl ... '" model "' ... " msgs_file " ..." \>  // params directos
}

// ✅ También en ?? y @:
greet_match(lang) {
    ?? lang {
        "es" : { r = <\ "echo hola " lang \> }
        _ :    { r = <\ "echo hello " lang \> }
    }
}
```

**Fix aplicado**: Eliminadas todas las copias de boilerplate `req_model = model`,
`mf = msgs_file`, `cf = count_file`, `tmp = tmp_dir` en `ollama.zy` y `history.zy`.
Parámetros y variables de módulo se usan directamente donde se necesitan.

---

## G17 — Sin funciones top-level en archivos script

**Status**: ✅ RESUELTO (2026-04-15)
**Discovered**: ZethyCLI — `main.zy`

Un archivo script (sin header `# module_name`) puede definir y llamar funciones
top-level. El parser y el tree-walker ya las soportaban — el bug real era que
`take_call_state()` borraba `import_aliases`, y las funciones de módulo restauraban
sus propios aliases desde `LoadedModule`, pero las funciones de script no tenían
mecanismo de restauración. Al llamar `pick_model()` desde `main.zy`, se perdían los
aliases `ollama::`, `ui::`, etc., haciendo fallar cualquier `alias::fn()` dentro
del cuerpo.

**Causa raíz**: En `functions_lambda.rs`, el bloque `else` para funciones non-módulo
no restauraba los import aliases. El fix es minimal: clonar los aliases del caller
antes de `take_call_state()` y restaurarlos para la ejecución de la función.

**Fix implementado**: `zymbol-interpreter/src/functions_lambda.rs` — en el bloque
`else` (función de script, sin `module_info`):
```rust
} else {
    // G17 fix: Script-level function inherits caller's import aliases
    self.import_aliases = saved.import_aliases.clone();
    None  // function table unchanged
};
```

**Resultado en ZethyCLI**: `pick_model()` definida una sola vez en `main.zy`.
Los dos bloques duplicados (first-run y `/reset`) reemplazados por `model = pick_model()`.

```zymbol
pick_model() {
    raw_list = <\ "ollama list" \>
    >> raw_list ¶
    chosen = ""
    @ {
        << "Model name: " chosen
        chosen = chosen$~~["\n":""]
        ? (ollama::model_exists(chosen)) { @! }   // ✅ alias::fn() funciona dentro
        ui::show_error("Model not found — pick one from the list above")
    }
    <~ chosen
}

// Uso:
model = pick_model()   // ✅ — retorna el modelo elegido
```

---

## G18 — Concatenación de strings en argumentos de función

**Status**: ✅ YA EXISTE — falso GAP corregido
**Discovered**: ZethyCLI — `main.zy`

`$++` encadena múltiples operandos en secuencia — no requiere variable como base.
`>>` acepta expresiones separadas por espacio sin operador explícito.
La "limitación" era desconocimiento del lenguaje, no un GAP real.

```zymbol
// ✅ $++ encadena directamente desde literal:
warn = "Model " $++ model " does not support think"
ui::show_error(warn)

// ✅ $++ también desde string vacío (concatenación pura):
warn2 = "" $++ "Model " model " does not support think"

// ✅ >> acepta múltiples expresiones separadas por espacio:
>> "Model " model " does not support think" ¶

// ✅ concat() implementable en puro Zymbol si se necesita join con separador:
concat(val) {
    sum = ""
    @ p : val {
        sum = sum $++ p ' '
    }
    <~ sum
}
val = ["Model", model, "does not support think"]
>> concat(val) ¶
```

El workaround de variable temporal sigue siendo válido por legibilidad,
pero no es obligatorio.

---

## G19 — `<#` no acepta rutas absolutas ni `~`

**Status**: ✅ RESUELTO (2026-04-15)
**Discovered**: ZethyCLI — observado al intentar importar módulos de librerías compartidas
**Affected**: `zymbol-parser/src/modules.rs` → `parse_module_path`, `zymbol-interpreter/src/modules.rs` → `resolve_module_path`

El parser de `<#` solo reconoce dos formas de ruta:

| Forma | Ejemplo | Funciona |
|-------|---------|----------|
| Relativa actual | `<# ./lib/ollama <= m` | ✅ |
| Relativa padre | `<# ../shared/utils <= m` | ✅ |
| Absoluta | `<# /home/user/libs/utils <= m` | ✗ |
| Home `~` | `<# ~/zymbol-libs/utils <= m` | ✗ |

**Causa raíz — parser** (`parse_module_path`):
El parser espera que la ruta empiece con `.` (`Dot`) o `..` (`DotDot`).
Si encuentra `/` (ruta absoluta) o `~` (home), emite `"expected module path"` y
el resto de la línea se parsea como statements normales, produciendo errores en cascada.

```
error: expected module path
  --> test.zy:1:4
  |  <# /tmp/mymod <= m
  |     ^
```

**Causa raíz — resolver** (`resolve_module_path`):
Incluso si el parser aceptara el path, el resolver siempre parte de `current_dir`
y no tiene rama para rutas absolutas (`is_absolute`) ni expansión de `~`.

```rust
// Actual — siempre parte de current_dir:
let mut resolved = current_dir.to_path_buf();
if module_path.is_relative {
    // navega con parent_levels
}
for component in &module_path.components { resolved.push(component); }
// → ruta absoluta /foo/bar → current_dir/foo/bar  ✗
```

**Fix propuesto**:

1. **AST** — agregar `is_absolute: bool` a `ModulePath`:
```rust
pub struct ModulePath {
    pub components: Vec<String>,
    pub is_relative: bool,
    pub is_absolute: bool,   // ← nuevo
    pub parent_levels: usize,
    pub span: Span,
}
```

2. **Parser** — reconocer `/` y `~` al inicio de la ruta:
```rust
// Ruta absoluta: /foo/bar
if matches!(self.peek().kind, TokenKind::Slash) {
    self.advance(); // consume leading /
    is_absolute = true;
    // parse components...
}
// Ruta home: ~/foo/bar
if matches!(self.peek().kind, TokenKind::Tilde) {
    self.advance(); // consume ~
    self.expect(TokenKind::Slash)?;
    is_absolute = true;
    home_relative = true;
    // parse components...
}
```

3. **Resolver** — arrancar desde `/` o `$HOME` cuando `is_absolute`:
```rust
let mut resolved = if module_path.is_absolute {
    if module_path.home_relative {
        PathBuf::from(std::env::var("HOME").unwrap_or_default())
    } else {
        PathBuf::from("/")
    }
} else {
    current_dir.to_path_buf()
};
```

**Fix implementado**:

1. **AST** (`zymbol-ast/src/modules.rs`) — `ModulePath` tiene dos campos nuevos:
   `is_absolute: bool` y `home_relative: bool`. Constructor `new_absolute()` para crearlos.

2. **Parser** (`zymbol-parser/src/modules.rs`) — `parse_module_path` reconoce:
   - `Slash` al inicio → ruta absoluta (`/foo/bar`)
   - `Tilde` + `Slash` → home-relativa (`~/foo/bar`)

3. **Resolver intérprete** (`zymbol-interpreter/src/modules.rs`) — cuando `is_absolute`,
   arranca desde `/` (absoluta) o `$HOME` (home-relativa) en lugar de `current_dir`.

4. **Resolver semántico** (`zymbol-semantic/src/modules.rs`) — misma lógica para
   que `zymbol check` valide rutas absolutas correctamente.

```zymbol
// ✅ Las cuatro formas funcionan:
<# ./lib/ollama    <= ollama   // relativa actual
<# ../shared/utils <= u        // relativa padre
<# /opt/zymbol-libs/json <= j  // absoluta
<# ~/zymbol-libs/http    <= h  // home-relativa
```

---

## G20 — `zymbol run` oculta errores de módulos: archivo y mensajes ausentes

**Status**: ✅ RESUELTO (2026-04-15)
**Discovered**: ZethyCLI — al correr `zymbol run main.zy` con errores en módulos importados
**Affected**: `zymbol-interpreter/src/modules.rs` → `load_module`

Cuando un módulo importado (`<#`) tenía errores de lexer o parser, `zymbol run`
solo reportaba un conteo genérico sin contexto útil:

```
Runtime error: failed to parse module: 3 lexer errors in module
```

El programador no sabía:
- **Qué archivo** causó el error
- **Qué línea y columna** del archivo
- **Cuál es el error** exactamente
- **El help** con sugerencia de corrección

Mientras que `zymbol check` sí mostraba toda esa información porque usa el
renderizador completo de diagnósticos. La discrepancia hacía que los errores en
módulos solo fueran visibles con `check` pero no con `run` — un fallo silencioso
en producción.

**Causa raíz**: `load_module` usaba `lex_diagnostics.len()` para el mensaje y
`format!("{:?}", e)` (debug format) para los errores del parser, sin incluir el
`file_path` en ningún caso.

**Fix implementado**: `zymbol-interpreter/src/modules.rs` — tanto el bloque de
lexer errors como el de parser errors ahora formatean cada diagnóstico con:
- Ruta del archivo (`file_path.display()`)
- Línea y columna (`span.start.line`, `span.start.column`)
- Mensaje del error (`d.message`)
- Help si existe (`d.help`)

```
# Antes:
Runtime error: failed to parse module: 3 lexer errors in module

# Después:
Runtime error: failed to parse module: 1 lexer error(s) in 'lib/memory.zy'
  lib/memory.zy:40:5: invalid character in string interpolation
    help: interpolation must be {identifier} — use \{ for a literal brace
```

**Impacto**: `zymbol run` ahora da el mismo nivel de detalle que `zymbol check`
cuando un módulo importado tiene errores.

---

## G21 — `\{` escape no suprimía interpolación — double-interpolation bug

**Status**: ✅ RESUELTO como BUG (no era GAP)
**Discovered**: ZethyCLI — `lib/sessions.zy` → `list()`

Dentro de un string literal en Zymbol, `{identifier}` activa la interpolación.
Esto aparenta conflicto con bash `${...}`, pero el comportamiento de `\{` lo resuelve:

**Regla**: `\{` suprime el error de interpolación cuando el contenido NO es un
identificador válido — produciendo `{` literal. La mayoría de bash parameter
expansions contienen chars inválidos (`/`, `:-`, `#`, `%`) y funcionan con `\{`:

| Bash idiom | En Zymbol string | Resultado |
|------------|-----------------|-----------|
| `${var/_/ }` | `$\{var/_/ }` | ✅ shell recibe `${var/_/ }` |
| `${var:-default}` | `$\{var:-default}` | ✅ `:` y `-` inválidos → fallback literal |
| `${#var}` | `$\{#var}` | ✅ `#` inválido → fallback literal |
| `${var%pattern}` | `$\{var%pattern}` | ✅ `%` inválido → fallback literal |
| `${var}` simple | `$\{var\}` | ❌ si `var` existe en Zymbol → **interpolará** |
| `$(cmd)` | sin cambio | ✅ `$` sin `{` no activa interpolación |

**Caso límite**: `$\{var\}` (simple expansion sin chars especiales) — si `var`
es también una variable Zymbol definida, **sigue interpolando**. Workaround:
inyectar el valor vía `{var}` Zymbol como asignación bash previa:

```zymbol
// ❌ ${base} simple — si base es variable Zymbol, interpolará base:
<\ "echo $\{base\}" \>   // → echo $hello_world  (no lo que queremos)

// ✅ Inyectar valor Zymbol como variable bash:
base_val = "hello_world"
<\ "v={base_val}; echo $\{v\}" \>   // → v=hello_world; echo ${v}  ✓
// (v no es variable Zymbol, así que \{v\} no interpola)

// ✅ Uso correcto con chars inválidos (forma habitual):
<\ "base={base_val}; echo $\{base/_/ }" \>   // → base=hello_world; echo ${base/_/ }
```

**Root cause**: el parser convertía `StringInterpolated([Variable("foo")])` de vuelta
a `Literal::String("{foo}")` — indistinguible de un `Literal::String("{foo}")` producido
por `\{foo\}`. El intérprete re-interpolaba ambos en runtime.

**Fix**: nuevo variant `Literal::InterpolatedString(String)` en `zymbol-common`.
- Parser: `TokenKind::String` → `Literal::String` (nunca interpolado)
- Parser: `TokenKind::StringInterpolated` → `Literal::InterpolatedString` (interpolado en runtime)
- Intérprete: solo `InterpolatedString` llama a `interpolate_string`; `String` es texto plano

**Archivos**: `zymbol-common/src/lib.rs`, `zymbol-parser/src/lib.rs`,
`zymbol-interpreter/src/literals.rs`, `zymbol-interpreter/src/match_stmt.rs`,
`zymbol-compiler/src/lib.rs`, `zymbol-semantic/src/type_check.rs`,
`zymbol-formatter/src/visitor.rs`

**Fix en `lib/sessions.zy`**: `${base/_/ }` → `$\{base/_/ }` (un solo `\`).

---

## Summary Table

| ID | Gap | Severity | Workaround | Future module |
|----|-----|----------|------------|---------------|
| G1 | No HTTP client | intentional — stdlib roadmap | curl via BashExec | `std/net` (IMPL_V007.md Step 10) |
| G2 | No JSON encode/decode | intentional — stdlib roadmap | jq via BashExec | `std/json` (IMPL_V007.md Step 9) |
| G3 | No file read/write | intentional — stdlib roadmap | cat/printf via BashExec | `std/io` (IMPL_V007.md Step 8) |
| G4 | BashExec shell-safety (single quotes) | workaround | sanitize `'` | `std/io` + `std/net` (v0.0.6) |
| G5 | Join strings | — | ✅ `,` operator | ya existe |
| G6 | Split string | — | ✅ `str / char` | falso warning corregido |
| G7 | BashExec trailing `\n` | SIGNIFICANT | `$~~["\n":""]` | ✅ trim by default |
| G8 | BashExec not expression | SIGNIFICANT | guard chain | ✅ resuelto por G12 |
| G9 | Module `.CONST` broken | MINOR | getter function | ✅ resuelto (TypeChecker fix) |
| G11 | `alias::fn()` void call | SIGNIFICANT | `disc = alias::fn()` | ✅ resuelto |
| G12 | BashExec opaco `{var}` | CRITICAL | — | ✅ rediseñado (expresiones Zymbol) |
| G13 | Module state non-persistent | — | ✅ private mutable state + write-back | resuelto |
| G14 | `#>` must follow `# name` immediately | — | ✅ resuelto — ambas posiciones válidas | — |
| G15 | `!=` no existe — error no orienta a `<>` | deliberate probe — fix confirmed required | ✅ diagnostic en lexer | — |
| G16 | Parámetros función invisibles en BashExec | SIGNIFICANT | ✅ resuelto por G12 | — |
| G17 | Sin funciones top-level en scripts | SIGNIFICANT | ✅ resuelto — alias restaurados en functions_lambda.rs | — |
| G18 | Concat de strings en args de función | — | ✅ `$++` encadena múltiples; `>>` acepta expresiones directas | falso GAP |
| G19 | `<#` no acepta rutas absolutas ni `~` | SIGNIFICANT | ✅ resuelto — AST + parser + resolver (intérprete y semántico) | — |
| G20 | `zymbol run` oculta errores de módulos | SIGNIFICANT | ✅ resuelto — `load_module` ahora muestra archivo, línea y mensaje | — |
| G21 | `\{` no suprimía interpolación — double-interpolation bug | — | ✅ resuelto — `Literal::InterpolatedString` separado de `Literal::String` | — |
