# Red Sinosófica — Reglas de desarrollo

Sistema de gestión clínica multidisciplinaria, multi-tenant, para redes de profesionales de la salud enfocadas en atención integral de mujeres cis.

---

## Filosofía

> **Desarrollar y guiar al usuario para implementar el sistema.**

Las dos mitades pesan igual. No es "yo lo construyo y tú lo recibes", ni "tú lo construyes y yo te corrijo": se desarrolla y a la vez se guía para que el usuario pueda implementarlo por su cuenta.

La prueba de que funcionó no es que el sistema corra. Es que al llegar a la entidad número diez, el usuario la construya sin asistencia.

De este principio se derivan los modos de entrega, la columna `porque` de la bitácora, los ADR de `docs/decisiones/` y `docs/patron-entidad.md`. Todos existen para que el conocimiento sobreviva a la sesión.

**Criterio de conflicto:** si la velocidad y el aprendizaje se estorban, gana el aprendizaje. Excepción: en lo declarado en **Lo que está blindado**, gana la corrección y la explicación viene después.

---

## Precedencia

Las invariantes de arquitectura ganan sobre el ladder YAGNI de Ponytail.

Ponytail decide el CÓMO (mínimo, reutilizar, una línea).
Este archivo decide el QUÉ existe.

Si el ladder sugiere eliminar algo listado abajo, la respuesta es no: el criterio del proyecto es costo de retrofit, no necesidad inmediata.

---

## Invariantes (no sujetas a YAGNI)

- Multi-tenancy por esquema desde el día uno, aunque haya un solo cliente.
- `AUTH_USER_MODEL` propio con PK UUID, antes del primer migrate.
- Auditoría conectada desde la primera entidad.
- `consulta_bloques` separado de `consultas`. No colapsar a un JSONB.
- Versionado de plantillas, aunque no haya constructor visual.
- Monolito en capas, NO modular estricto. Las apps se importan entre sí con FK directas. No crear fachadas ni eventos entre `apps/`.
- Dos etapas, dos directorios raíz: `feminist_net_back/` y `feminist_net_front/`. `docs/` y este archivo viven en la raíz porque describen el sistema, no una etapa.
- Cada etapa lleva `dev-log/dev_log.csv`, append-only. Un renglón por evento relevante, escrito al cierre de cada etapa junto con el commit.
- Una carpeta por entidad, agrupadas en familias (`apps/<familia>/<entidad>/`). La familia es organización de archivos, NO frontera arquitectónica: las FK cruzan familias sin fachadas.
- Patrón de entidad: Modelo → Serializer → ViewSet plano. La lógica de negocio vive en `nucleo/`. Sin repositorios envolviendo el ORM. `services.py` solo con orquestación de efectos, y se justifica.
- `feminist_net_back/nucleo/` NUNCA importa Django.
- Tablas de tenant NUNCA llevan `organizacion_id`. El aislamiento es por `search_path`.
- Todo modelo hereda `BaseModel` (UUID + timestamps) + `SoftDeleteMixin`.
- JWT lleva claim `tenant` y se valida contra el `Host` del request.
- Consulta con `finalizada_at` no se modifica. Solo notas de enmienda.
- Lectura del expediente nunca se bloquea por límite de plan ni por falta de pago.
- Pacientes no se limitan por número. Solo profesionales activas.
- Plantilla publicada: `clave` y `tipo` de campo son inmutables. Se depreca, no se edita.
- Autorización nueva = política RLS + test que la verifique por SQL directo.

---

## Nivel de Ponytail por tipo de trabajo

- **CRUD, serializers, viewsets:** `/ponytail full`
- **Fase 6 (RLS, consentimiento, privacidad):** `/ponytail lite`
- **Migraciones estructurales:** `/ponytail off`

---

## Deuda técnica

Los atajos se marcan con comentario `ponytail:` en el código y se cosechan con `/ponytail-debt` al cerrar cada fase.

```python
# ponytail: temporal, resolver en etapa 2 cuando X esté completo
```

---

## Modos de entrega

Derivados de la **Filosofía**. Cada modo se conmuta por separado y el estado persiste hasta que el usuario lo cambie.

| Modo | Defecto | Activar | Desactivar |
|---|---|---|---|
| Didáctico | **ON** | `/didactico on` | `/didactico off` |
| Pedagógico | **OFF** | `/pedagogico on` | `/pedagogico off` |
| Verboso | **OFF** | `/verboso on` | `/verboso off` |

**Didáctico** — explicar antes de escribir:
- Explico el flujo lógico antes del código.
- Explico el porqué de cada decisión arquitectónica, no solo el qué.
- Cuando Ponytail recorta algo, explico qué se eliminó y por qué no hacía falta. Ponytail recorta el código, jamás la explicación.

**Pedagógico** — el usuario implementa:
- NO modifico archivos. Doy ruta completa, nombre de archivo, bloque exacto y línea de inserción.
- Espero confirmación de que el usuario copió e implementó antes de seguir.
- **Al activarlo, advertir primero el costo** (sin escritura de archivos, confirmación paso por paso) y esperar el sí.

**Verboso** — comentarios en el código explicando la lógica de negocio, nunca lo obvio de la sintaxis.

**Punto de detención:** al cierre de cada **etapa**, no de cada fase. Se escribe el renglón en `dev-log/dev_log.csv`, se hace el commit, y se espera confirmación antes de continuar.

---

## Control de versiones

- **Sin autoría de IA.** Jamás `Co-authored-by`.
- **Cero referencias a Claude/IA en cualquier artefacto versionado:** mensajes de commit, descripciones de PR, nombres de rama, código y comentarios. Antes de cualquier `git commit`, revisar el diff y el mensaje y confirmar que no aparece "Claude", "Anthropic", "AI", "IA" ni menciones equivalentes.
- **En español,** describiendo la acción técnica.
- **Granularidad a demanda:** por paso, fase o etapa completada.

---

## Flujo de trabajo

Lee `docs/flujo-desarrollo.md` antes de cualquier feature. Para una entidad nueva, lee además `docs/patron-entidad.md`: ahí está la receta completa y no se reinventa por entidad.

Por cada feature nueva:
1. Crea o lee `docs/roadmaps/<feature>.md`
2. Marca tareas con `[x]` conforme avanzan
3. Abre `docs/specs/<tema>.md` con `/interview-me` si hay ambigüedad
4. Sigue el flujo para el tipo de tarea (entidad, regla de dominio, cambio de privacidad)
5. Registra el evento en `dev-log/dev_log.csv` al cierre de cada etapa
6. Cosecha deuda al cierre con `/ponytail-debt`

---

## Comandos de Ponytail

| Comando | Qué hace |
|---|---|
| `/ponytail [lite \| full \| ultra \| off]` | Fijar intensidad o reportar la actual |
| `/ponytail-review` | Revisar el diff actual por sobre-ingeniería |
| `/ponytail-audit` | Auditar el repo completo por sobre-ingeniería |
| `/ponytail-debt` | Cosechar los `ponytail:` marcados en un registro |
| `/ponytail-help` | Referencia rápida |

`/ponytail-audit` escanea el repo completo sin conocer las invariantes de este archivo. Va a proponer recortes que aquí están prohibidos: colapsar `consulta_bloques` a un JSONB, eliminar el versionado de plantillas, quitar el multi-tenancy por esquema mientras haya un solo cliente. Antes de ejecutarlo, pásale la sección **Invariantes** como contexto y descarta de entrada cualquier hallazgo que toque algo listado ahí o en **Lo que está blindado**.

---

## Lo que está blindado

RLS, consentimiento, auditoría, validación en fronteras de confianza, manejo de pérdida de datos y seguridad nunca están en la lista de recortes de Ponytail. Si ves que lo sugiere, señálalo antes de documentar.
