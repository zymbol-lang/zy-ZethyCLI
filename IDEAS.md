# ZethyCLI — Ideas de Implementación para Zymbol

> Características nuevas identificadas durante el desarrollo de ZethyCLI.
> No son errores — son capacidades que harían al lenguaje más expresivo y poderoso.
> Prioridad: Alta / Media / Baja

---

## IDEA-01 — `std/net`: Módulo de red nativo

**Prioridad**: ALTA
**Gap documentado**: G1
**Origen**: `lib/http.zy` y `lib/ollama.zy`

Zymbol necesita un módulo de red para hacer peticiones HTTP sin depender de `curl` via BashExec. Esto eliminaría la dependencia de herramientas externas y permitiría manejo real de errores.

**API propuesta**:
```zymbol
<# std/net <= net

response = net::get("http://example.com")           // GET
response = net::post(url, headers, body)             // POST
response = net::post_json(url, json_str)             // POST JSON
>> response.status ¶       // código HTTP
>> response.body   ¶       // cuerpo de respuesta
>> response.headers ¶      // cabeceras
```

**Impacto en ZethyCLI**: `lib/http.zy` y `lib/ollama.zy` quedarían en <10 líneas de Zymbol puro.

---

## IDEA-02 — `std/json`: Módulo JSON nativo

**Prioridad**: ALTA
**Gap documentado**: G2
**Origen**: `lib/json.zy`, `lib/ollama.zy`, `lib/history.zy`

Serialización y deserialización JSON sin depender de `jq` o Python.

**API propuesta**:
```zymbol
<# std/json <= json

// Encode
str = json::encode(value)          // any Zymbol value → JSON string
str = json::encode_str("hello")    // "\"hello\""

// Decode
obj = json::decode(json_str)       // JSON string → Zymbol value
val = json::get(obj, "key")        // extract field
val = json::get_path(obj, "a.b.c") // nested access

// Build
arr_str = json::array(items)       // Zymbol array → JSON array string
obj_str = json::object(pairs)      // pairs array → JSON object string
```

**Impacto en ZethyCLI**: Elimina toda dependencia de Python en `history.zy` y `ollama.zy`.

---

## IDEA-03 — `std/io`: Módulo de I/O de archivos nativo

**Prioridad**: ALTA
**Gap documentado**: G3
**Origen**: `lib/file.zy`, `lib/history.zy`

Lectura y escritura de archivos sin `cat`, `printf`, `rm` via BashExec.

**API propuesta**:
```zymbol
<# std/io <= io

content = io::read("/path/to/file")           // string
io::write("/path/to/file", content)           // overwrite
io::append("/path/to/file", content)          // append
exists  = io::exists("/path/to/file")         // bool
io::delete("/path/to/file")
lines   = io::read_lines("/path/to/file")     // array of strings
io::mkdir("/path/to/dir")
```

**Nota crítica**: La implementación debe manejar contenido arbitrario (incluyendo `'`, `"`, `\n`, null bytes) correctamente — exactamente lo que BashExec no puede garantizar (GAP G4).

---

## IDEA-04 — Escape para `{` literal en strings: `\{` o `{{`

**Prioridad**: ALTA
**Bug relacionado**: BUG-04
**Origen**: `lib/ollama.zy`, `lib/history.zy`

Actualmente imposible tener `{` como carácter literal en un string Zymbol. Necesita una sintaxis de escape.

**Opciones**:
```zymbol
// Opción A — backslash escape
s = "valor: \{dato\}"     // → "valor: {dato}" literal

// Opción B — doble llave (como Python format strings)
s = "valor: {{dato}}"     // → "valor: {dato}" literal
```

**Recomendación**: Opción B (`{{`) es más natural para quien viene de Python/JavaScript/Rust.

**Impacto en ZethyCLI**: `lib/ollama.zy` y `lib/history.zy` podrían construir JSON directamente en Zymbol sin Python.

---

## IDEA-05 — BashExec como statement (sin asignación)

**Prioridad**: ALTA
**Bug relacionado**: BUG-03
**Origen**: `lib/file.zy`, `lib/history.zy`

Permitir `<\ cmd \>` como statement de efecto secundario sin requerir asignación.

**Propuesta**:
```zymbol
<\ mkdir "-p" /tmp/test \>       // válido — efecto secundario
<\ rm "-f" /tmp/file.txt \>      // válido — efecto secundario
```

**Implementación**: El parser debe aceptar BashCommand como statement expresión (equivalente a `_ = <\ cmd \>`).

---

## IDEA-06 — `arr$join(separator)`: Operador join para arrays

**Prioridad**: MEDIA
**Gap documentado**: G5
**Origen**: `lib/history.zy` (construcción de JSON array)

Unir elementos de un array con un separador es una operación fundamental que hoy requiere un loop manual.

**Propuesta**:
```zymbol
parts = ["a", "b", "c"]
result = parts$join(", ")     // → "a, b, c"
result = parts$join("")       // → "abc"
```

---

## IDEA-07 — `str$split(sep)` y `str$split(sep, limit)`: Split nativo

**Prioridad**: MEDIA
**Gap documentado**: G6
**Origen**: `main.zy` (parsing de comandos `/comando args`)

Hoy se hace con `$??` + slice manual, que es verboso y frágil.

**Propuesta**:
```zymbol
parts = "hello world foo"$split(" ")          // → ["hello", "world", "foo"]
parts = "/mode fast"$split(" ", 2)            // → ["/mode", "fast"]
head  = "/mode fast"$split_head(" ")          // → "/mode"
tail  = "/mode fast"$split_tail(" ")          // → "fast"
```

---

## IDEA-08 — BashExec con interpolación segura `${var}` para shell

**Prioridad**: ALTA
**Gap documentado**: G4
**Origen**: `lib/json.zy`, `lib/file.zy`

La interpolación actual `{var}` en BashExec no es shell-safe: si `var` contiene `'` o `"`, el comando shell falla.

**Propuesta**: Modo de interpolación segura que envuelva el valor en comillas shell:
```zymbol
// Actual (unsafe para contenido arbitrario):
<\ printf '%s' '{content}' > /tmp/out \>

// Propuesto (shell-safe):
<\ printf '%s' $[content] > /tmp/out \>
// Zymbol escapa content apropiadamente para shell
```

**Alternativa más simple**: Que BashExec siempre use variables de entorno para pasar valores:
```zymbol
// Zymbol exporta ZYMBOL_VAR_content=... antes del comando
<\ printf '%s' "$ZYMBOL_content" > /tmp/out \>
```

---

## IDEA-09 — BashExec sin trailing `\n`

**Prioridad**: MEDIA
**Gap documentado**: G7
**Origen**: Todos los módulos que usan BashExec

Todo resultado de BashExec incluye `\n` final. Hoy se hace `result$~~["\n":""]` en cada llamada.

**Propuesta**: BashExec retorna string sin `\n` por defecto. Modo raw para cuando se necesita:
```zymbol
result = <\ date \>           // sin \n
result = <\raw date \>        // con \n (modo raw explícito)
```

---

## IDEA-10 — Funciones de módulo visibles entre sí (intra-module calls)

**Prioridad**: CRÍTICA
**Bug relacionado**: BUG-01
**Origen**: `lib/history.zy`

Las funciones del mismo módulo deben poder llamarse. Esto es comportamiento estándar en cualquier lenguaje modular.

**Estado actual**: `add_user()` no puede llamar a `get_count()` ni `append_msg()` del mismo módulo.
**Impacto**: Impide DRY (Don't Repeat Yourself) dentro de módulos. Todo debe estar inlineado.

---

## IDEA-11 — Variable descarte nativa

**Prioridad**: MEDIA
**Bug relacionado**: BUG-02, BUG-03
**Origen**: `main.zy`

Necesidad de descartar valores de retorno de funciones usadas por efectos secundarios.

**Propuesta A** — Wildcard `_` como expresión de descarte:
```zymbol
_ = ui::show_info("hello")   // descarta el valor retornado
```

**Propuesta B** — Funciones void (sin retorno obligatorio):
```zymbol
// Funciones que no retornan valor usan ~> en lugar de <~
banner() {
    >> "Hello" ¶
    ~>    // void return — no value
}
// Llamadas a funciones void son statements válidos:
banner()   // válido sin asignación
```

**Propuesta B** es más ergonómica y elimina el problema de raíz.
