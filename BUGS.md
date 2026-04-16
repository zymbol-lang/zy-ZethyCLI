# ZethyCLI — Bugs Confirmados

> Comportamientos incorrectos o inconsistentes del compilador/intérprete.
> Estos son errores que deben corregirse — el lenguaje debería funcionar diferente.

---

## BUG-01 — Funciones privadas del módulo no pueden llamarse dentro del mismo módulo

**Severidad**: CRÍTICA
**Descubierto**: Phase 1 de ZethyCLI
**Archivo afectado**: `lib/history.zy`
**Estado**: ✅ RESUELTO en Sprint 2

Una función definida en un módulo (sin exportar) no puede ser invocada por otra función del mismo módulo. Incluso funciones EXPORTADAS no pueden llamar a otras exportadas del mismo módulo.

**Reproduce**:
```zymbol
# mimodulo
#> { foo, bar }

foo() { <~ bar() }     // runtime error: undefined function 'bar'
bar() { <~ 42 }
```

**Comportamiento esperado**: Las funciones del mismo módulo deben ser visibles entre sí.
**Workaround**: Inlinear toda la lógica — no hay reutilización de código dentro del módulo.
**Impacto real**: En `history.zy` se duplican 5 líneas idénticas en `add_user` y `add_assistant`.

**Fix implementado**: `LoadedModule` ahora guarda `all_functions` (todas las funciones del módulo, exportadas + privadas). Al entrar en una llamada de módulo, `self.functions` se reemplaza temporalmente con `module.all_functions` y se restaura al finalizar. Archivos: `interpreter/src/modules.rs`, `interpreter/src/functions_lambda.rs`.

---

## BUG-02 — Variables con prefijo `_` no accesibles en bloques internos

**Severidad**: SIGNIFICATIVA
**Descubierto**: Phase 1 de ZethyCLI
**Archivo afectado**: `main.zy`
**Estado**: ✅ POR DISEÑO — documentado en `interpreter/MANUAL.md` § 4

Una variable `_nombre` declarada en un scope externo no puede ser reasignada desde un bloque `? { }` o `@ { }` interno. El error indica "strictly local to declaration block".

**Reproduce**:
```zymbol
_r = fn1()
? condicion {
    _r = fn2()   // error: cannot access underscore variable from inner scope
}
```

**Resolución**: Las variables `_` tienen **scope de bloque exacto** — solo son visibles dentro
del bloque donde se declararon, ni hacia adentro ni hacia afuera. Este comportamiento es
intencional y está verificado por tests en `zymbol-semantic/tests/underscore_semantics.rs`.
El patrón correcto es pre-declarar como variable regular antes del bloque condicional:

```zymbol
cmd  = ""           // variable regular, visible en bloques internos
args = ""
? has_space {
    cmd  = input$[1..p-1]
    args = input$[p+1..-1]
}
```

**Aplicado en**: `main.zy` líneas 37–38 (workaround ya en producción).
**Documentado en**: `interpreter/MANUAL.md` § 4 — subsección "Underscore Variables (`_name`)".

---

## BUG-03 — BashExec como statement sin asignación no es válido

**Severidad**: SIGNIFICATIVA
**Descubierto**: Phase 1 de ZethyCLI
**Archivo afectado**: `lib/file.zy`, `lib/history.zy`
**Estado**: ✅ RESUELTO en Sprint 1

Un BashExec `<\ cmd \>` usado como statement de efecto secundario (sin asignar el resultado) produce error de parser.

**Reproduce**:
```zymbol
<\ mkdir "-p" /tmp/test \>   // error: unexpected token BashCommand
```

**Comportamiento esperado**: BashExec debería ser válido como statement cuando se usa solo para efectos secundarios (crear directorio, borrar archivo, escribir a archivo).
**Workaround**: `disc = <\ mkdir "-p" /tmp/test \>`.

**Fix implementado**: Agregado arm `TokenKind::BashCommand(_)` en `parse_statement` del parser. Archivo: `zymbol-parser/src/lib.rs`.

---

## BUG-04 — Literal `{` en strings causa error de interpolación

**Severidad**: CRÍTICA
**Descubierto**: Phase 1 de ZethyCLI
**Archivos afectados**: `lib/ollama.zy`, `lib/history.zy`
**Estado**: ✅ RESUELTO en Sprint 1

Un carácter `{` en un string literal es siempre interpretado como inicio de interpolación. Si no va seguido de un identificador válido y `}`, el parser falla con "unterminated interpolation".

**Reproduce**:
```zymbol
ob = "{"        // error: unterminated interpolation in string
json = "{}"     // error: unterminated interpolation
```

**Comportamiento esperado**: Necesita un escape para `{` literal, como `\{` o `{{`.
**Workaround**: Obtener `{` via BashExec: `ob = <\ printf '\x7B' \>` (no implementado aún; en Phase 1 se usó Python `dict()` para evitar JSON literal en strings).
**Impacto**: Imposible construir JSON, shells scripts con `${}`, o cualquier string que contenga `{` sin interpolación.

**Fix implementado**: `{{` en un string produce `{` literal (sin activar interpolación). La interpolación `{varname}` sigue funcionando normalmente junto al escape. Archivo: `zymbol-lexer/src/literals.rs`.

---

## BUG-06 — `>> "literal" (expr)` crash: literal interpretado como callable

**Severidad**: SIGNIFICATIVA
**Descubierto**: ZethyCLI Sprint 4 — `/tmp/zytest/main.zy`
**Archivo afectado**: `zymbol-parser/src/expressions.rs` — `parse_output_item_postfix`
**Estado**: ✅ RESUELTO (2026-04-14)

En una sentencia `>>`, un literal seguido de `(expr)` era parseado como una llamada de función con el literal como callable. El runtime fallaba con "expression is not callable".

**Reproduce**:
```zymbol
ok = (#1 == #1)
>> "Result: " (ok) ¶          // crash: "expression is not callable"
>> "Value: " (1 + 2) ¶        // crash
>> "Flag: " (#1 == #0) ¶      // crash
```

**Comportamiento esperado**: `"literal" (expr)` en contexto `>>` son dos ítems de output
separados — el literal y la expresión entre paréntesis. No una llamada de función.

**Workaround** (antes del fix): asignar la expresión a variable antes de imprimir:
```zymbol
result = (1 + 2)
>> "Value: " result ¶
```

**Causa raíz**: `parse_output_item_postfix` trataba `(` como postfix de llamada de función
sin verificar si el `expr` base era invocable. Los literales (string, int, bool, char, float)
nunca son invocables.

**Fix implementado**: Guardado `is_callable` antes del arm `LParen`:
```rust
let is_callable = matches!(&expr,
    Expr::Identifier(_) | Expr::MemberAccess(_) | Expr::FunctionCall(_)
);
if !is_callable { break; }
```
Si el base no es callable, el `(` rompe el loop postfix — el parser lo deja para el siguiente
ítem de output. Archivo: `zymbol-parser/src/expressions.rs`, función `parse_output_item_postfix`.

---

## BUG-07 — `}}` en strings produce dos `}` literales (asimetría con `{{`)

**Severidad**: SIGNIFICATIVA
**Descubierto**: ZethyCLI — `lib/config.zy`, función `save()`
**Estado**: ✅ RESUELTO

En Zymbol, `{{` dentro de un string literal produce `{` (escape de llave). Sin embargo
`}}` **no** produce `}` — produce dos `}` literales. La asimetría es una trampa: el
programador asume que si `{{` escapa `{`, entonces `}}` escapa `}`.

**Reproduce**:
```zymbol
// Intención: generar el string  '{"key":"val"}'
s = "'{{\"key\":\"val\"}}'"
// Resultado real:  '{"key":"val"}}'   ← doble } al final
// Resultado esperado: '{"key":"val"}'
```

**Impacto real en ZethyCLI**: `config.zy` `save()` generaba un filtro jq inválido
`'{"model":$model,...}}'` (doble `}`) que hacía fallar jq silenciosamente, dejando
`~/.ZethyCLI/ZethyCLI.conf` vacío. El modelo no se persistía entre ejecuciones.

**Por qué es difícil de detectar**: BashExec no expone stderr al programador Zymbol.
El comando shell falla sin ningún mensaje visible — el único síntoma es un resultado
vacío o un archivo sin contenido.

**Fix implementado**: Eliminado el mecanismo `{{` → `{` del lexer por completo.
Diseño final del lexer de strings:

- `{varname}` → interpolación de variable
- `\{` → llave `{` literal (escape)
- `\}` → llave `}` literal (escape)
- `\n`, `\t`, `\r`, `\"`, `\\` → escapes estándar

```zymbol
// Antes (mecanismo eliminado):
<\ "jq -n '{{\"key\":\"val\"}}'" \>   // producía {"key":"val"}}  ← bug

// Ahora (correcto):
<\ "jq -n '\{\"key\":\"val\"\}'" \>   // produce {"key":"val"}  ← correcto

// Interpolación normal (sin cambios):
name = "Alice"
>> "Hola {name}!" ¶                   // → Hola Alice!
```

Archivo: `zymbol-lexer/src/literals.rs` — reemplazado el bloque `{{ → {`
por `\{ → {` y `\} → }` en el match de secuencias de escape.

---

## BUG-05 — `@>` (continue) en bucles infinitos

**Severidad**: MENOR
**Descubierto**: Phase 1 de ZethyCLI
**Archivo afectado**: `main.zy`
**Estado**: ✅ NO ES BUG — confirmado por análisis estático del intérprete

`@>` dentro de `@ { }` (loop infinito) funciona correctamente. El flujo de control se
propaga hacia arriba a través de bloques intermedios (cada `execute_block` detiene la
ejecución de sentencias siguientes al detectar `is_control_flow_pending()`), y
`handle_loop_control` en `loops.rs` limpia el `Continue` y devuelve `false` —
lo que hace que el loop continúe con la siguiente iteración sin romper.

El scope también se limpia correctamente: `execute_block` llama `pop_scope()` antes de
retornar, independientemente de si hubo control flow pendiente.

**Verificado**: análisis estático de `interpreter/crates/zymbol-interpreter/src/loops.rs`
y `lib.rs` (funciones `handle_loop_control`, `execute_block`, `execute_block_no_scope`).
**Test de regresión**: `interpreter/tests/loops/02_infinite_continue.zy` (11/11 PASS).
