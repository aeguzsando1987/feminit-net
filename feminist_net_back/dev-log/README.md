# Bitácora de desarrollo — `dev_log.csv`

Diccionario de columnas y vocabulario. Decisión completa en
`docs/decisiones/005-bitacora-dev-log.md`.

---

## Reglas

1. **Append-only.** Nunca se edita ni se borra un renglón pasado. Si algo se
   revierte, se agrega un renglón de tipo `reversion`.
2. **Se escribe al cierre de cada etapa**, junto con el commit. Nunca en bloque
   al final de la fase: un log escrito de memoria es ficción.
3. **Vocabulario cerrado.** Si hace falta un valor de `tipo` que no está en la
   lista, se agrega a este archivo **en el mismo commit**.

---

## Columnas

| # | Columna | Formato | Para qué |
|---|---|---|---|
| 1 | `fecha` | `YYYY-MM-DD` | Trazabilidad temporal; ordena el archivo |
| 2 | `hora` | `HH:MM` 24h | Distingue eventos del mismo día |
| 3 | `fase` | `0`–`8` | Filtrar por fase |
| 4 | `etapa` | `3.2`, `0.1`, vacío | Granularidad dentro de la fase |
| 5 | `tipo` | ver abajo | La columna que hace consultable el archivo |
| 6 | `componente` | ruta, o `infra` / `proyecto` / `raiz` | Qué parte se tocó |
| 7 | `descripcion` | texto | Qué se hizo, una línea |
| 8 | `porque` | texto, puede ir vacío | **La razón de la decisión** |
| 9 | `ponytail` | `off` `lite` `full` `ultra` | Nivel activo al hacer el cambio |
| 10 | `commit` | SHA corto (7) o vacío | Puente al historial de git |
| 11 | `estado` | `hecho` `parcial` `bloqueado` `revertido` | Detectar lo que quedó a medias |

`porque` es el campo que justifica el archivo entero. En seis meses vale más que
el `qué`.

---

## Vocabulario de `tipo`

| Valor | Cuándo |
|---|---|
| `decision` | Se eligió una opción sobre otras. Si es grande, además va un ADR |
| `infra` | Docker, CI, VPS, DNS, cuentas |
| `entidad` | Modelo nuevo o cambio de modelo |
| `migracion` | Migración de esquema o de datos |
| `regla-dominio` | Lógica en `nucleo/` |
| `rls` | Política de Row Level Security |
| `auditoria` | Triggers y registro de auditoría |
| `test` | Pruebas añadidas o cambiadas |
| `bug` | Defecto encontrado o corregido |
| `refactor` | Cambio sin alterar comportamiento |
| `deuda` | Atajo deliberado, marcado con `# ponytail:` en el código |
| `doc` | Documentación, ADR, roadmap |
| `bloqueo` | Algo impide avanzar. Espera decisión o insumo externo |
| `reversion` | Se deshizo algo registrado antes |

---

## Formato

UTF-8 · encabezado en la primera línea · separador coma · campos con espacios
entre comillas dobles · **sin comas dentro de `descripcion` ni `porque`** (usar
punto y coma) · sin saltos de línea dentro de un campo.

---

## Consultas útiles

```bash
# Integridad: todos los renglones con 11 campos
awk -F',' 'NF!=11 {print NR": "NF" campos"}' dev-log/dev_log.csv

# Qué quedó sin cerrar
grep -E ',(bloqueado|parcial)$' dev-log/dev_log.csv

# Toda la deuda declarada
grep ',deuda,' dev-log/dev_log.csv

# Qué pasó en la Fase 3
awk -F',' '$3==3' dev-log/dev_log.csv

# Solo las decisiones, en orden
grep ',decision,' dev-log/dev_log.csv | cut -d',' -f1,6,7
```
