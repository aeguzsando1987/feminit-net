# Plan maestro de ejecución — Red Sinosófica (backend, Fases 0→6)

> **Estado:** aprobado · vigente.
>
> Este documento descompone el trabajo en Fase → Etapa → Paso. El avance real no
> se marca aquí, sino en `docs/roadmaps/fase-N-*.md`, uno por fase.
>
> **Precedencia:** donde este plan discrepe de un ADR de `docs/decisiones/`,
> gana el ADR — es posterior y trae el motivo del cambio. A la fecha, el ADR 006
> sustituye Python 3.12 + Django 5.1 por **Python 3.13 + Django 5.2 LTS**.
>
> Supera a `docs/plan.md` y a `docs/plan_desarrollo_red_sinosofica_v2.md` en todo
> lo que sea calendario y ejecución. Esos dos siguen valiendo como fuente de
> contexto, stack y razonamiento estratégico.

## Contexto

El repositorio está en Fase 0: dos commits, solo documentación. Existen el plan
estratégico (`docs/plan_desarrollo_red_sinosofica_v2.md`), el esquema completo
(`docs/esquema.dbml`, 873 líneas, ~17 entidades de tenant + 4 de `public`) y las
invariantes (`CLAUDE.md`). Lo que **no** existe es el puente entre ese *qué* y la
ejecución diaria: una descomposición Fase → Etapa → Paso con puntos de detención,
y un patrón de entidad repetible para que agregar la entidad número 18 sea
mecánico en vez de una decisión de diseño nueva.

Este plan produce ese puente. El objetivo declarado por el usuario es doble:
construir el sistema **y aprender en el proceso**. Por eso el modo didáctico
(explicar el flujo y el porqué arquitectónico antes del código) es el estado por
defecto, y se equilibra con ponytail para que el pragmatismo no se pierda.

Alcance: **backend completo, Fases 0 a 6**. Frontend (Fase 7), endurecimiento
(Fase 8) y VPS (Fase X) se planean por separado cuando toque.

---

## Filosofía del proyecto

> **Desarrollar y guiar al usuario para implementar el sistema.**

Las dos mitades pesan igual. No es "yo lo construyo y tú lo recibes", ni "tú lo
construyes y yo te corrijo": **desarrollo y a la vez te guío para que puedas
implementarlo tú**. La prueba de que funcionó no es que el sistema corra — es que
al llegar a la entidad número diez la construyas sin mí.

Qué implica en la práctica, y cómo se verifica en cada artefacto del plan:

| La filosofía exige | Cómo se materializa |
|---|---|
| Que entiendas antes de que exista el código | Modo **didáctico ON por defecto**: explico el flujo lógico y el porqué arquitectónico *antes* de escribir. |
| Que puedas tomar el control cuando quieras | Modo **pedagógico bajo demanda** (`/pedagogico on`): dejo de escribir, te doy ruta + bloque + línea, y tú implementas. |
| Que el conocimiento sobreviva a la sesión | Columna **`porque`** en `dev_log.csv` y ADRs en `docs/decisiones/`. Lo que solo vive en un chat, se pierde. |
| Que aprendas el patrón, no cada caso | **`docs/patron-entidad.md`**: se construye despacio una vez en la Fase 3, y de ahí en adelante tú lo repites. |
| Que la simplicidad también se enseñe | Ponytail recorta el **código**, jamás la **explicación**. Cuando elimine algo, te digo qué y por qué no hacía falta. |
| Que puedas detener y corregir el rumbo | Punto de detención **al cierre de cada etapa**, no de cada fase. Confirmas antes de que sigamos. |

**Criterio de conflicto:** si en algún momento la velocidad y el aprendizaje se
estorban, gana el aprendizaje — salvo en lo que `CLAUDE.md` declara blindado
(RLS, consentimiento, auditoría, fronteras de confianza), donde gana la
corrección, y la explicación viene después.

---

## Decisiones tomadas en esta sesión

Estas cuatro quedan cerradas y se escriben en `CLAUDE.md` en el Paso 0.1.

| # | Decisión | Implicación |
|---|---|---|
| 1 | **Didáctico ON por defecto, pedagógico OFF por defecto.** Se conmutan con `/didactico on\|off` y `/pedagogico on\|off`. Al activar pedagógico, se advierte el costo antes de aplicarlo. | Yo escribo archivos y explico el porqué; si activas pedagógico, dejo de escribir y te doy ruta + bloque + línea, esperando confirmación por paso. |
| 2 | **Patrón plano: Modelo → Serializer → ViewSet.** Lógica de negocio en `nucleo/`. Sin repositorios. `services.py` solo cuando hay orquestación con efectos. | Respeta las tres invariantes de `CLAUDE.md`. Lo repetible se logra con plantilla documentada, no con capas. |
| 3 | **Una carpeta por entidad, agrupadas en familias.** Nivel extra de directorio por familia de entidades. | Cambio respecto a la estructura plana de `apps/` que propone el v2. Requiere `AppConfig.name` explícito por app anidada. |
| 4 | **Arranque desde cero.** Sin Docker, sin toolchain, sin dominio. | La Fase 0 se ejecuta completa, incluidos los bloqueadores no técnicos. |
| 5 | **Dos etapas = dos directorios raíz**: `feminist_net_back/` y `feminist_net_front/`. Cada uno con su propio `dev-log/dev_log.csv`. | Cambia la estructura de raíz respecto al v2. La bitácora es obligatoria, no opcional. |

**Fuentes de verdad** (los demás archivos son históricos):
`CLAUDE.md` · `docs/plan_desarrollo_red_sinosofica_v2.md` · `docs/esquema.dbml` ·
`docs/flujo-desarrollo.md`. El archivo `red_sinosofica_schema.dbml` de la raíz es
respaldo obsoleto y difiere de `docs/esquema.dbml`: se marca como tal o se borra.

---

## Estructura de directorios definitiva

### Raíz — dos etapas, dos directorios

```
SINOSOFIC/
├── CLAUDE.md                    # reglas, comunes a ambas etapas
├── docs/                        # esquema, plan, decisiones — compartidos
│   ├── esquema.dbml
│   ├── flujo-desarrollo.md
│   ├── patron-entidad.md
│   ├── decisiones/
│   ├── roadmaps/
│   └── specs/
├── feminist_net_back/           # ★ ETAPA 1 — empezamos aquí
│   ├── dev-log/
│   │   ├── dev_log.csv          # ★ bitácora obligatoria
│   │   └── README.md            # diccionario de columnas y vocabulario de `tipo`
│   ├── pyproject.toml
│   ├── Dockerfile
│   ├── docker-compose.yml
│   ├── docker-compose.override.yml
│   ├── config/
│   ├── nucleo/
│   ├── comun/
│   ├── compartido/
│   └── apps/
└── feminist_net_front/          # ETAPA 2 — no se crea hasta la Fase 7
    ├── dev-log/dev_log.csv
    └── src/{api,componentes,paginas,hooks}/
```

`docs/` queda en la raíz porque el esquema, las decisiones y los roadmaps
describen **el sistema**, no una etapa. El `dev-log/` sí es por etapa: registra el
trabajo de esa etapa, y quien audite el backend no debe tener que filtrar eventos
del frontend.

### Dentro de `feminist_net_back/`

```
feminist_net_back/
├── config/settings/{base,local,produccion}.py
├── nucleo/                      # ★ py puro, NUNCA importa Django
│   ├── ciclo/                   # cálculo de fase
│   ├── plantillas/              # validación contra campos_definicion
│   ├── consentimiento/          # resolución de acceso por ámbito
│   └── planes/                  # límites de plan
├── comun/                       # BaseModel, SoftDeleteMixin, paginación, filtros
├── compartido/
│   └── organizaciones/          # SHARED_APPS — Plan, Organizacion, Dominio, Uso
└── apps/                        # TENANT_APPS, agrupadas por familia
    ├── identidad/
    │   ├── usuarios/            # ★ AUTH_USER_MODEL, antes del 1er migrate
    │   ├── especialidades/      # catálogo
    │   └── profesionales/       # + through ProfesionalEspecialidad
    ├── expediente/
    │   ├── pacientes/
    │   ├── antecedentes/        # generales + ginecobstétricos
    │   └── ciclo/               # RegistroCiclo
    ├── atencion/
    │   ├── agenda/              # Cita
    │   ├── plantillas/          # PlantillaFormulario, Version, CampoDefinicion
    │   └── consultas/           # Consulta + ConsultaBloque
    └── privacidad/
        ├── consentimientos/
        └── auditoria/
```

Las carpetas de familia (`identidad/`, `expediente/`, `atencion/`, `privacidad/`)
son **paquetes Python vacíos**, no apps de Django: solo `__init__.py`. Cada app
anidada declara su ruta completa:

```python
# apps/expediente/pacientes/apps.py
class PacientesConfig(AppConfig):
    name = "apps.expediente.pacientes"   # label por defecto: "pacientes"
```

Los labels resultantes son únicos, así que `TENANT_APPS` sigue siendo legible y
las FK entre apps se escriben `"pacientes.Paciente"` como siempre.

> **Nota didáctica a cubrir al construirlo:** la familia es organización de
> archivos, no frontera arquitectónica. `apps/atencion/consultas/` importa
> directamente `apps/expediente/pacientes/models.py` con FK real. No hay fachadas
> entre familias — eso sería el monolito modular estricto que `CLAUDE.md` rechaza.

---

## El patrón de entidad (la receta que se repite ~17 veces)

Se define **una sola vez** en la Fase 3 con `Profesional`, se documenta en
`docs/patron-entidad.md`, y de ahí en adelante toda entidad nueva lo sigue.

**Archivos por entidad:**

```
apps/<familia>/<entidad>/
├── __init__.py
├── apps.py          # AppConfig con name completo
├── models.py        # hereda BaseModel + SoftDeleteMixin
├── serializers.py
├── views.py         # ModelViewSet plano
├── urls.py          # router.register
├── admin.py
├── migrations/
├── services.py      # ⚠ solo si hay orquestación con efectos. Justificar.
└── tests/
    ├── test_modelo.py
    ├── test_aislamiento.py   # ★ obligatorio: dos organizaciones
    └── test_api.py
```

**Los 8 pasos, siempre en este orden:**

1. **Leer el DBML** de la tabla en `docs/esquema.dbml`, incluidas sus `Note`.
   Las notas contienen reglas de negocio, no adorno.
2. **¿Hay lógica clínica?** Si sí → la regla va a `nucleo/`, con test sin BD
   *antes* del modelo. Si es CRUD puro → saltar al paso 3.
3. **Modelo** + `makemigrations` + `migrate_schemas`.
4. **Test de aislamiento**: crear dos organizaciones, escribir en una, verificar
   que la otra no ve nada. Por SQL directo, no solo por ORM.
5. **Serializer** — validación en la frontera de confianza.
6. **ViewSet + urls** — plano. Si pasa de ~150 líneas, lo que sobra es dominio
   disfrazado y pertenece a `nucleo/`.
7. **Auditoría**: verificar que el trigger cubre la tabla nueva.
8. **`/ponytail-review`** sobre el diff + commit en español.

**Regla de equilibrio ponytail ↔ didáctico:** ponytail recorta el *código*, nunca
la *explicación*. Cuando ponytail elimine algo, se explica qué se eliminó y por
qué no hacía falta — eso también es aprendizaje. Niveles según
`docs/flujo-desarrollo.md`: `full` en CRUD, `lite` en lógica clínica y Fase 6,
`off` en migraciones estructurales.

---

## Bitácora de desarrollo — `dev-log/dev_log.csv`

Regla obligatoria de ambas etapas. Un renglón por evento relevante, en orden
cronológico, nunca se edita ni se borra un renglón pasado: si algo se revierte,
se agrega un renglón nuevo de tipo `reversion`. Es append-only, igual que la
tabla `auditoria` del sistema — y por la misma razón.

**Columnas propuestas:**

| # | Columna | Tipo / formato | Para qué sirve |
|---|---|---|---|
| 1 | `fecha` | `YYYY-MM-DD` | Trazabilidad temporal. Ordena el archivo. |
| 2 | `hora` | `HH:MM` 24h | Distingue eventos del mismo día sin depender del orden de línea. |
| 3 | `fase` | `0`–`6` | Filtrar por fase: "¿qué pasó en la Fase 3?". |
| 4 | `etapa` | `3.2`, `0.1`, vacío | Granularidad fina dentro de la fase. |
| 5 | `tipo` | vocabulario cerrado (abajo) | **La columna que hace consultable el archivo.** |
| 6 | `componente` | `apps/identidad/profesionales`, `nucleo/ciclo`, `infra` | Qué parte del sistema se tocó. |
| 7 | `descripcion` | texto, entrecomillado | Qué se hizo, en español, una línea. |
| 8 | `porque` | texto, entrecomillado, puede ir vacío | **El campo pedagógico.** La razón de la decisión. En seis meses esto vale más que el `qué`. |
| 9 | `ponytail` | `off` · `lite` · `full` · `ultra` | Nivel activo al hacer el cambio. Explica por qué un código es más escueto que otro. |
| 10 | `commit` | SHA corto (7) o vacío | Puente al historial de git. |
| 11 | `estado` | `hecho` · `parcial` · `bloqueado` · `revertido` | Permite detectar lo que quedó a medias. |

**Vocabulario cerrado de `tipo`** (si hace falta uno nuevo, se agrega al
`README.md` del `dev-log/` en el mismo commit):

`decision` · `infra` · `entidad` · `migracion` · `regla-dominio` · `rls` ·
`auditoria` · `test` · `bug` · `refactor` · `deuda` · `doc` · `bloqueo` ·
`reversion`

**Ejemplo:**

```csv
fecha,hora,fase,etapa,tipo,componente,descripcion,porque,ponytail,commit,estado
2026-09-16,11:40,0,0.1,decision,raiz,"Estructura en dos etapas: feminist_net_back y feminist_net_front","Cada etapa audita su propio trabajo sin filtrar el de la otra",off,,hecho
2026-09-17,09:15,1,1.5,migracion,apps/identidad/usuarios,"AUTH_USER_MODEL con PK UUID antes del primer migrate","Cambiarlo despues es de las migraciones mas dolorosas de Django",off,a3f9c21,hecho
2026-09-17,14:02,1,1.4,entidad,comun,"BaseModel y SoftDeleteMixin","Lo heredan las 17 entidades; hacerlo mal se paga 17 veces",lite,7b2e440,hecho
```

**Convenciones de formato:** UTF-8, encabezado en la primera línea, separador
coma, todo campo con comas o espacios va entre comillas dobles, y **sin comas
dentro de `descripcion` ni `porque`** — usar punto y coma. Es un CSV que se lee
con `grep` y con pandas indistintamente; mantenerlo simple lo conserva útil.

**Cuándo se escribe:** al cierre de cada etapa, junto con el `[x]` del roadmap y
el commit. Nunca en bloque al final de la fase — un log escrito de memoria es
ficción.

---

## Fase 0 — Preparación (sin código)

**Etapa 0.1 — Sanjar las reglas.** Archivos a modificar:

- `CLAUDE.md`: encabezar con la **filosofía** (*"desarrollar y guiar al usuario
  para implementar el sistema"*) como principio rector del que se derivan los
  modos; reescribir la sección *"Documentación y entrega"* con la tabla de
  modos (didáctico ON, pedagógico OFF) y sus comandos; añadir a las invariantes
  la estructura en dos etapas, la agrupación por familias (aclarando que no es
  modularidad estricta) y la **obligación de la bitácora `dev-log/dev_log.csv`**.
- `docs/decisiones/003-estructura-por-familias.md`: ADR nuevo.
- `docs/decisiones/004-modos-de-entrega.md`: ADR nuevo.
- `docs/decisiones/005-bitacora-dev-log.md`: ADR nuevo.
- `docs/patron-entidad.md`: esqueleto; se llena en la Fase 3.
- `red_sinosofica_schema.dbml`: eliminar o renombrar a `.bak`.

**Etapa 0.1b — Crear la raíz de la etapa 1.** `feminist_net_back/` con su
`dev-log/dev_log.csv` (encabezado + el primer renglón, que es esta misma
decisión) y `dev-log/README.md` con el diccionario de columnas. El
`feminist_net_front/` **no se crea todavía**: una carpeta vacía durante seis
fases es ruido.

**Etapa 0.2 — Entorno local.** Docker 24+ y Compose v2, usuario en grupo
`docker`, Python 3.13 (ADR 006), `uv`, Node 20 o superior.  Cada instalación se explica: qué hace y por
qué esa versión.

**Etapa 0.3 — Cuentas y dominio.** GitHub, Cloudflare (nameservers apuntando),
registrador, Resend/Brevo, Sentry. Formato de slug (minúsculas, `a-z0-9-`, 3–30)
y lista de subdominios reservados: `admin api www app staging dev mail static`.

**Etapa 0.4 — Bloqueadores no técnicos.** Arrancar ya, tardan semanas: aviso de
privacidad LFPDPPP (declara transferencia internacional por el VPS, Cloudflare
como subencargado que ve tráfico descifrado, R2 como almacén) y el acuerdo por
escrito sobre propiedad del expediente y del código.

**Salida:** `docker compose version` responde · `uv --version` responde ·
nameservers propagados · CLAUDE.md actualizado y commiteado.

---

## Fase 1 — Esqueleto

**Objetivo:** `docker compose up` levanta el proyecto y `/healthz` responde.

| Etapa | Contenido |
|---|---|
| 1.1 | `feminist_net_back/`: `pyproject.toml`, `Dockerfile`, `.gitignore`, `.env.example`. **Didáctico:** por qué `uv` y no `pip`, por qué imagen única dev/prod. |
| 1.2 | Árbol de directorios completo (el de arriba), todo con `__init__.py`. |
| 1.3 | `config/settings/{base,local,produccion}.py` + `django-tenants`. **Didáctico:** por qué `TenantMainMiddleware` va primero, por qué `django.contrib.auth` está en SHARED **y** TENANT. |
| 1.4 | `comun/models.py`: `BaseModel` (UUID + timestamps) y `SoftDeleteMixin`. Se usa en las 17 entidades: vale la pena hacerlo despacio. |
| 1.5 | **`apps/identidad/usuarios/` con `AUTH_USER_MODEL` de PK UUID — antes del primer `migrate`.** Punto de no retorno. |
| 1.6 | `docker-compose.yml` (db, web, caddy) + `docker-compose.override.yml` con hot reload. Paridad local/prod obligatoria. |
| 1.7 | Endpoint `/healthz`. |
| 1.8 | GitHub Actions: `pytest` + test que falla si `nucleo/` importa Django. |

**Ponytail:** `off` en 1.3 y 1.5 (settings y migración inicial son críticos),
`full` en el resto.

**Salida:** `/healthz` responde · pytest en CI · `grep -r "import django"
backend/nucleo/` vacío · el primer `migrate` corrió con `usuarios.Usuario`.

---

## Fase 2 — Esquema público

**Objetivo:** crear organizaciones con un comando, sin pasos manuales.

| Etapa | Contenido |
|---|---|
| 2.1 | `compartido/organizaciones/models.py`: `Plan`, `Organizacion(TenantMixin)`, `Dominio(DomainMixin)`, `OrganizacionUso`. `auto_create_schema=True`, `auto_drop_schema=False`. |
| 2.2 | `nucleo/planes/limites.py`: dataclass `EstadoPlan` con `puede_agregar` y `requiere_aviso` (80%). **Test primero, sin BD** — es el primer ejercicio del flujo "Regla de dominio". |
| 2.3 | Comando `crear_organizacion`: valida slug → crea esquema → migra → siembra especialidades y plantillas base → registra dominio → crea administradora. |

**Reglas de límite:** usuario activo = `activa=true` (no login reciente) ·
pacientes sin límite numérico · aviso al 80%, bloqueo de creación al 100% ·
**la lectura nunca se bloquea**.

**Ponytail:** `lite` en 2.2 (regla de negocio), `full` en 2.1 y 2.3.

**Salida:** dos organizaciones creadas, `\dn` muestra ambos esquemas · tests de
`nucleo/planes/` pasan sin BD · alta sin un solo paso manual.

---

## Fase 3 — Primera entidad de tenant (define el patrón)

**Objetivo:** `Profesional` + `Especialidad` de punta a punta. Esta fase es la más
importante del plan: lo que se decida aquí se copia 15 veces. Va despacio.

| Etapa | Contenido |
|---|---|
| 3.1 | `apps/identidad/especialidades/` — catálogo. La entidad más simple, sirve de calentamiento del patrón. |
| 3.2 | `apps/identidad/profesionales/` — `Profesional` + through `ProfesionalEspecialidad` con `cedula_profesional` y `verificada_at`. |
| 3.3 | Admin de ambas. |
| 3.4 | Serializers con especialidades anidadas. **Didáctico:** por qué el through model no se serializa como M2M simple. |
| 3.5 | ViewSets. El límite de plan se valida llamando a `nucleo/planes/` desde `perform_create` — primer adaptador núcleo↔Django. |
| 3.6 | **Auditoría por trigger** — se conecta aquí, en la primera entidad, como manda `CLAUDE.md`. |
| 3.7 | Paginación, filtros y orden resueltos en `comun/`, **una sola vez**. |
| 3.8 | Test de aislamiento entre dos organizaciones + test de rechazo de JWT cruzado. |
| 3.9 | **Escribir `docs/patron-entidad.md`** con lo que acabamos de hacer, y `.claude/commands/entidad.md` que lo automatice. |

**Salida:** CRUD contra `redsinosofica.localhost:8080` · límite de plan bloquea al
máximo · aislamiento en verde · JWT de la org A rechazado en el subdominio de B ·
auditoría registra alta, cambio y borrado lógico · **patrón documentado**.

---

## Fase 4 — Pacientes y ciclo

De aquí en adelante cada entidad sigue los 8 pasos del patrón; el plan solo
señala lo que se sale de la receta.

| Etapa | Entidad | Lo no rutinario |
|---|---|---|
| 4.1 | `pacientes/Paciente` | Identidad separada de lo clínico: las tablas clínicas solo referencian `paciente_id`. Cifrado pgcrypto **solo** en CURP, teléfono y domicilio — una columna cifrada no se indexa ni se busca. |
| 4.2 | `antecedentes/AntecedentesGenerales` | 1-1, JSONB. |
| 4.3 | `antecedentes/AntecedentesGinecobstetricos` | 1-1 pero **ámbito de acceso distinto** — prepara el terreno de la Fase 6. |
| 4.4 | `ciclo/RegistroCiclo` | Serie temporal, único por `(paciente_id, fecha)`. |
| 4.5 | `nucleo/ciclo/calculo_fase.py` | **Test primero, sin BD.** Debe devolver `INDETERMINADA` o `NO_APLICA` en vez de inventar día de ciclo: asumir 28 días regulares falla a muchas pacientes, y falla en silencio. |

**Ponytail:** `lite` en 4.5 y en el cifrado de 4.1. `full` en el resto.

**Salida:** tests de fase cubren irregular, amenorrea, anticoncepción continua y
posmenopausia · query de correlación métrica ↔ fase funciona.

---

## Fase 5 — Plantillas y consultas

**Patrón central:** definiciones relacionales, valores en JSONB.

| Etapa | Contenido |
|---|---|
| 5.1 | `plantillas/`: `PlantillaFormulario`, `PlantillaVersion` (inmutable), `CampoDefinicion`. |
| 5.2 | `nucleo/plantillas/validacion.py` — valida la captura contra las definiciones. Test primero, sin BD. |
| 5.3 | `CHECK` con `jsonb_matches_schema` (extensión `pg_jsonschema`). Doble validación: núcleo **y** base de datos. |
| 5.4 | `atencion/consultas/Consulta` — contenedor: SOAP, signos vitales, fase del ciclo denormalizada. Con `finalizada_at` no se modifica: solo notas de enmienda. |
| 5.5 | `ConsultaBloque` — uno por especialidad, con `metricas_especialidad` y `campos_custom_usuario`. **No colapsar a un JSONB en `consultas`** (invariante). |
| 5.6 | Índices GIN en ambas columnas JSONB. |
| 5.7 | `agenda/Cita`. |
| 5.8 | Sembrar plantillas base por especialidad, vía migración de datos. |

**Reglas:** cada bloque guarda `plantilla_version_id`, así editar un formulario no
reinterpreta el histórico · publicada una versión, `clave` y `tipo` son
inmutables: se crea campo nuevo y se depreca el viejo · **sin constructor visual
en etapa 1**, el motor queda completo pero las plantillas se siembran.

**Salida:** consulta con dos bloques de especialidades distintas · captura
inválida rechazada por núcleo **y** por CHECK · editar plantilla no altera
consultas anteriores.

---

## Fase 6 — Consentimiento, RLS y auditoría

**Fase de mayor riesgo. Revisión manual obligatoria línea por línea. Ponytail
`lite` en toda la fase, y lo blindado de `CLAUDE.md` no se recorta jamás.**

Se sigue el flujo *"Cambio que toca privacidad"* de `docs/flujo-desarrollo.md`:
**la política RLS se escribe antes del endpoint**, y el test de denegación por SQL
directo va antes de la implementación.

| Etapa | Contenido |
|---|---|
| 6.1 | `privacidad/consentimientos/ConsentimientoAcceso` — paciente × profesional × ámbito, con vigencia y evidencia. |
| 6.2 | `nucleo/consentimiento/reglas.py` — resolución de acceso. Test sin BD. |
| 6.3 | Middleware: `set_config('app.profesional_id', ..., true)` por request. El `true` (local a transacción) es obligatorio con pgbouncer en modo `transaction`. |
| 6.4 | **Políticas RLS** sobre las tablas clínicas, con su test SQL de denegación previo. |
| 6.5 | Triggers de auditoría en todas las tablas clínicas. |
| 6.6 | Registro de `intento_denegado`. |
| 6.7 | Auditoría append-only: revocar UPDATE y DELETE a todos los roles. |

**Didáctico obligatorio:** explicar las dos capas de aislamiento — el *esquema*
aísla organizaciones entre sí, la *RLS* aísla profesionales dentro de una misma
organización. Son mecanismos distintos resolviendo problemas distintos.

**Salida:** profesional sin consentimiento de ámbito no lee el bloque, ni por API
ni por SQL directo con su rol · revocar corta el acceso de inmediato · intentos
denegados quedan en `auditoria` · **ninguna política depende de que la app filtre
bien**.

---

## Verificación end-to-end

Al cierre de cada fase, y obligatoriamente al cerrar la Fase 6:

```bash
cd feminist_net_back/

# 1. Todo levanta
docker compose up -d && curl -f localhost:8080/healthz

# 2. Dos organizaciones independientes
python manage.py crear_organizacion --nombre "Red Sinosófica" --slug redsinosofica \
  --plan base --admin-email coordinacion@ejemplo.mx
python manage.py crear_organizacion --nombre "Colectiva Flor" --slug colectivaflor \
  --plan base --admin-email hola@ejemplo.mx
psql -c '\dn'          # ambos esquemas presentes

# 3. La invariante del núcleo
grep -r "import django\|from django" nucleo/    # debe salir vacío

# 4. Suite completa
pytest                                  # todo
pytest nucleo/ --no-cov -p no:django    # núcleo sin BD, debe correr solo

# 5. Aislamiento por SQL directo, no por ORM
psql -c "SET search_path TO colectivaflor; SELECT count(*) FROM profesionales;"

# 6. RLS con rol de otra profesional (Fase 6)
psql -c "SET app.profesional_id = '<uuid-ajeno>'; SELECT * FROM consulta_bloques;"
# debe devolver 0 filas
```

**Cierre de cada fase:** `/ponytail-debt` para cosechar los `ponytail:` marcados
(mínimo una cosecha por fase) · actualizar `docs/roadmaps/fase-N-*.md` con `[x]` ·
renglón de cierre en `dev-log/dev_log.csv` · commit en español, sin ninguna
referencia a IA en mensaje, rama ni código.

**Salud de la bitácora**, verificable en cualquier momento:

```bash
# Columnas correctas en todos los renglones (11 campos)
awk -F',' 'NF!=11 {print NR": "NF" campos"}' feminist_net_back/dev-log/dev_log.csv

# Qué quedó sin cerrar
grep -E ',(bloqueado|parcial)$' feminist_net_back/dev-log/dev_log.csv

# Toda la deuda declarada
grep ',deuda,' feminist_net_back/dev-log/dev_log.csv
```

---

## Cómo trabajaremos, paso a paso

Es la filosofía convertida en rutina:

1. Al abrir cada **fase**, escribo `docs/roadmaps/fase-N-<nombre>.md` con sus
   etapas y pasos en casillas `[ ]`.
2. Al abrir cada **etapa**, explico el flujo lógico y el porqué arquitectónico
   **antes** de tocar código (modo didáctico).
3. Ejecuto los pasos, marco `[x]` conforme avanzan.
4. Al cerrar cada **etapa**: renglón en `dev-log/dev_log.csv`, commit en español,
   y me detengo a esperar tu confirmación antes de seguir.
5. Si en cualquier momento dices `/pedagogico on`, dejo de escribir archivos y
   paso a darte ruta + bloque + línea de inserción, con confirmación por paso.
