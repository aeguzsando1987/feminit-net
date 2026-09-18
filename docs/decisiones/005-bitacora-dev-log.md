# ADR 005 — Bitácora de Desarrollo en CSV Append-Only

**Estado:** Aceptada · Septiembre 2026

**Contexto**

El proyecto abarca siete fases y ~17 entidades, con decisiones tomadas a lo largo de meses. El historial de git registra *qué cambió*, pero no *por qué se decidió así* ni *qué se intentó antes*. Los ADR cubren las decisiones grandes; no cubren el día a día: qué se probó, qué se bloqueó, qué se revirtió, con qué nivel de Ponytail se escribió un módulo.

Sin ese registro, el conocimiento vive solo en la conversación con el asistente y desaparece con ella. Eso contradice directamente la **Filosofía** del proyecto: si la meta es que el usuario quede capaz de implementar el sistema, el rastro del razonamiento debe sobrevivir a la sesión.

**Alternativas Evaluadas**

1. **Solo mensajes de commit** (descartada)
   - Registran el qué, no el porqué
   - No admiten eventos sin código: decisiones, bloqueos, cuentas pendientes
   - No se filtran por fase ni por tipo

2. **Bitácora en Markdown narrativo** (descartada)
   - Cómoda de escribir, imposible de consultar
   - Sin estructura, a los tres meses es un muro de texto
   - No se responde "¿qué quedó bloqueado?" sin leerlo entero

3. **CSV con vocabulario cerrado (ELEGIDA ✓)**
   - Consultable con `grep`, `awk` y pandas indistintamente
   - Sin dependencias, sin servicio, sin base de datos
   - El vocabulario cerrado de `tipo` lo mantiene filtrable

**Decisión**

Cada etapa del proyecto lleva `dev-log/dev_log.csv` en su directorio raíz:

```
feminist_net_back/dev-log/dev_log.csv
feminist_net_front/dev-log/dev_log.csv
```

Es **por etapa** y no global porque quien audite el backend no debe filtrar eventos del frontend.

**Once columnas:**

| # | Columna | Formato | Para qué |
|---|---|---|---|
| 1 | `fecha` | `YYYY-MM-DD` | Trazabilidad temporal; ordena el archivo |
| 2 | `hora` | `HH:MM` 24h | Distingue eventos del mismo día |
| 3 | `fase` | `0`–`8` | Filtrar por fase |
| 4 | `etapa` | `3.2`, `0.1`, vacío | Granularidad dentro de la fase |
| 5 | `tipo` | vocabulario cerrado | La columna que hace consultable el archivo |
| 6 | `componente` | ruta o `infra` | Qué parte del sistema se tocó |
| 7 | `descripcion` | texto | Qué se hizo, una línea |
| 8 | `porque` | texto, puede ir vacío | **La razón de la decisión** |
| 9 | `ponytail` | `off`·`lite`·`full`·`ultra` | Nivel activo al hacer el cambio |
| 10 | `commit` | SHA corto (7) o vacío | Puente al historial de git |
| 11 | `estado` | `hecho`·`parcial`·`bloqueado`·`revertido` | Detectar lo que quedó a medias |

Tres columnas merecen justificación:

- **`porque`** es el campo pedagógico, y la razón de ser del archivo. En seis meses vale más que el `qué`. Es la Filosofía persistida en disco.
- **`ponytail`** explica por qué un módulo es escueto y otro verboso, sin que parezca inconsistencia del código.
- **`estado`** permite encontrar con un `grep` lo que quedó a medias, que es exactamente lo que se olvida.

**Vocabulario cerrado de `tipo`:**

`decision` · `infra` · `entidad` · `migracion` · `regla-dominio` · `rls` · `auditoria` · `test` · `bug` · `refactor` · `deuda` · `doc` · `bloqueo` · `reversion`

Si hace falta un valor nuevo, se agrega a `dev-log/README.md` **en el mismo commit**. Un vocabulario que crece sin registro degenera en sinónimos y el archivo deja de ser filtrable.

**Append-only.** Nunca se edita ni se borra un renglón pasado. Si algo se revierte, se agrega un renglón de tipo `reversion`. Es la misma regla que la tabla `auditoria` del sistema, por la misma razón: un registro que se puede reescribir no es un registro.

**Cuándo se escribe:** al cierre de cada etapa, junto con el `[x]` del roadmap y el commit. Nunca en bloque al final de la fase — un log escrito de memoria es ficción.

**Consecuencias**

*Positivas:*
- El razonamiento sobrevive al `/clear` y a los meses
- Consultable sin herramientas: `grep ',bloqueado' dev_log.csv`
- Sirve de insumo para la planeación de la etapa 2 (Anexo B del plan v2)
- Cero dependencias

*Negativas:*
- Disciplina manual: nada obliga a escribirlo. Si se abandona, miente por omisión
- CSV es frágil con comas y saltos de línea; se mitiga con las convenciones de formato
- Duplica parcialmente el historial de git. Es deliberado: git guarda el diff, esto guarda el porqué

**Convenciones de formato**

UTF-8 · encabezado en la primera línea · separador coma · campos con espacios entre comillas dobles · **sin comas dentro de `descripcion` ni `porque`** (usar punto y coma) · sin saltos de línea dentro de un campo.

**Verificación**

```bash
# Todos los renglones con 11 campos
awk -F',' 'NF!=11 {print NR": "NF" campos"}' dev-log/dev_log.csv

# Qué quedó sin cerrar
grep -E ',(bloqueado|parcial)$' dev-log/dev_log.csv

# Toda la deuda declarada
grep ',deuda,' dev-log/dev_log.csv
```

**Referencias**

- `CLAUDE.md` §Filosofía y §Invariantes
- `feminist_net_back/dev-log/README.md` — diccionario de columnas
- ADR 003: Estructura en dos etapas
- ADR 004: Modos de entrega
