# Red Sinosófica — Plan de desarrollo

Sistema de gestión clínica multidisciplinaria, multi-tenant, para redes de profesionales de la salud enfocadas en atención integral de mujeres cis.

Este documento es el prompt maestro del proyecto. Se usa junto con `red_sinosofica_schema_v2.dbml`.

---

## 1. Contexto

- **Cliente inicial:** una organización (Red Sinosófica). Arquitectura multi-tenant desde el día uno.
- **Proveedor:** desarrollador independiente. El sistema es también producto comercial.
- **Usuarias:** profesionales de la salud con especialidades híbridas (una misma persona puede ejercer Fitness + Nutrición y capturar ambas en una sesión).
- **Etapa 1 = CRUD grande + reglas de negocio básicas.** Todo lo demás se difiere.

---

## 2. Decisiones cerradas

No reabrir sin motivo nuevo.

| Área | Decisión |
|---|---|
| Backend | Django 5.1 + DRF |
| API | REST. GraphQL se puede agregar después sobre los mismos modelos. |
| BD | PostgreSQL 16 |
| Multi-tenancy | Un esquema por organización (`django-tenants`) |
| Ruteo de tenant | Subdominio, resuelto por `Host` en middleware |
| Frontend | Vite + React + TypeScript + TanStack Query, SPA estática |
| Ingreso | Cloudflare Tunnel, cero puertos abiertos |
| Arquitectura | MVT/DRF plano + núcleo de dominio aislado en Python puro |
| Archivos | Cloudflare R2 |
| Correo | Resend o Brevo |
| Infra | VPS Hetzner 4 GB, Docker Compose |
| Sin Celery/Redis | No hay trabajo asíncrono hasta etapa 2 |

---

## 3. Invariantes

Contenido de `CLAUDE.md` en la raíz. Aplican siempre.

```markdown
## Invariantes
- Monolito en capas, NO modular estricto. Las apps se importan entre sí
  con FK directas. No crear fachadas ni eventos entre apps/.
- backend/nucleo/ NUNCA importa Django.
- Tablas de tenant NUNCA llevan organizacion_id. El aislamiento es por search_path.
- Todo modelo hereda BaseModel (UUID + timestamps) + SoftDeleteMixin.
- AUTH_USER_MODEL propio con PK UUID. Nunca el auth_user entero de Django.
- Todo JWT lleva claim `tenant` y se valida contra el Host del request.
- Consulta con finalizada_at no se modifica. Solo notas de enmienda.
- Lectura del expediente nunca se bloquea por límite de plan ni por falta de pago.
- Autorización nueva = política RLS + test que la verifique por SQL directo.
- Pacientes no se limitan por número. Solo profesionales activas.
- Plantilla publicada: clave y tipo de campo son inmutables. Se depreca, no se edita.
```

**Obligación en CI, no solo en el archivo:** test que falle si `nucleo/` importa Django.

---

## 4. Flujos de trabajo

Contenido de `docs/flujo-desarrollo.md`. Por tipo de tarea, no por fase.

```markdown
## Entidad nueva de tenant
1. /spec (sáltalo si es CRUD puro)
2. Modelo + migración + migrate_schemas
3. Test de aislamiento entre 2 organizaciones
4. Serializer, viewset, urls
5. /review

## Regla de dominio
1. Vive en nucleo/. Test primero, sin BD.
2. El adaptador Django solo traduce.

## Cambio que toca privacidad
1. Política RLS antes del endpoint
2. Test leyendo con rol de otra profesional, por SQL directo
3. Revisión manual obligatoria. No aceptar de agente sin leer.

## Requisito ambiguo
1. /interview-me → escribe docs/specs/<tema>.md
2. /clear
3. Modo plan sobre ese archivo
```

Comandos en `.claude/commands/`, uno por flujo. Con checkpoint:

```markdown
---
description: Entidad nueva de tenant
---
Implementa $ARGUMENTS según docs/flujo-desarrollo.md, sección
"Entidad nueva de tenant". Detente tras el paso 3 y espera confirmación.
```

---

## 5. Arquitectura

**Estilo: monolito en capas con dominio aislado, multi-tenant por esquema.**

No es un monolito modular estricto. La modularidad es horizontal (capas), no vertical (módulos de negocio con fronteras entre sí).

| Sí | No |
|---|---|
| FK directas entre apps (`ConsultaBloque` → `Profesional`) | Fachadas o eventos entre `apps/` |
| `apps/consultas/` importa modelos de `apps/pacientes/` | Módulos verticales aislados entre sí |
| Núcleo aislado por regla de dirección | Repositorios envolviendo el ORM |

**Por qué no modular estricto:** paga con varios equipos en paralelo o si se prevé extraer módulos a servicios. Ninguna aplica. Además las FK de Postgres son integridad referencial de datos clínicos; renunciar a ellas sería una pérdida neta. Este sistema no se parte en microservicios: el expediente es intrínsecamente relacional y transaccional.

**La única frontera dura es la del núcleo**, y es de dirección: `nucleo/` no importa Django; todo lo demás puede importar `nucleo/`.

**Degradación a vigilar:** que las FK crucen apps es normal; que la *lógica de negocio* de una app viva en otra, no. Si un viewset pasa de ~150 líneas, lo que sobra suele ser dominio disfrazado y pertenece a `nucleo/`.

Tres capas. Regla de dependencia: **el núcleo no importa Django**.

```
ADAPTADORES (DRF)  →  serializers, viewsets, urls, admin
NÚCLEO (py puro)   →  reglas clínicas, testeable sin BD
PERSISTENCIA       →  models, migraciones, RLS
```

**Va en el núcleo:** validación contra `campos_definicion`, resolución de consentimiento, cálculo de fase de ciclo, conflictos entre planes de tratamiento, límites de plan.

**No va en el núcleo:** CRUD de pacientes, citas, catálogos. Eso es `ModelViewSet` plano. Sin repositorios envolviendo el ORM.

### Estructura

```
redsinosofica/
├── CLAUDE.md
├── docker-compose.yml            # idéntico a producción
├── docker-compose.override.yml   # solo local
├── Dockerfile
├── pyproject.toml
├── backend/
│   ├── config/settings/{base,local,produccion}.py
│   ├── nucleo/                   # ★ sin imports de Django
│   │   ├── ciclo/ plantillas/ consentimiento/ planes/
│   ├── compartido/organizaciones/  # SHARED_APPS
│   ├── comun/                      # BaseModel, auditoría, permisos
│   └── apps/                       # TENANT_APPS
│       ├── usuarios/               # ★ AUTH_USER_MODEL, antes del 1er migrate
│       ├── profesionales/ pacientes/ agenda/
│       └── consultas/ plantillas/ privacidad/
├── frontend/src/{api,componentes,paginas,hooks}/
├── infra/{caddy,cloudflared,scripts}/
├── docs/{flujo-desarrollo.md,specs/}
└── .claude/{commands,skills}/
```

### django-tenants

```python
SHARED_APPS = ["django_tenants", "compartido.organizaciones",
               "django.contrib.contenttypes", "django.contrib.auth",
               "django.contrib.staticfiles", "rest_framework"]

TENANT_APPS = ["django.contrib.contenttypes", "django.contrib.auth",
               "apps.usuarios", "apps.profesionales", "apps.pacientes",
               "apps.agenda", "apps.consultas", "apps.plantillas",
               "apps.privacidad"]

TENANT_MODEL = "organizaciones.Organizacion"
TENANT_DOMAIN_MODEL = "organizaciones.Dominio"
AUTH_USER_MODEL = "usuarios.Usuario"

MIDDLEWARE = [
    "django_tenants.middleware.main.TenantMainMiddleware",   # ← PRIMERO
    "django.contrib.sessions.middleware.SessionMiddleware",
    "django.contrib.auth.middleware.AuthenticationMiddleware",
]
```

`django.contrib.auth` va en SHARED_APPS **y** en TENANT_APPS: hay una tabla de usuarios por esquema.

Orden de middleware obligatorio. Si `TenantMainMiddleware` no va primero, Django busca al usuario en `public` y falla.

No se puede retrofitear sin reorganizar todas las apps. Va desde el inicio.

### Autenticación

Una tabla de usuarios **por esquema**. `Usuario` guarda credenciales, `Profesional` guarda el perfil, relación 1-1.

```python
class Usuario(AbstractBaseUser, PermissionsMixin):
    id = models.UUIDField(primary_key=True, default=uuid.uuid4, editable=False)
    email = models.EmailField(unique=True)
    USERNAME_FIELD = "email"
```

**Flujo:** POST a `redsinosofica.tudominio.mx/api/auth/login/` → middleware fija `search_path` por Host → Django autentica contra `redsinosofica.usuarios` → JWT. El login siempre ocurre dentro de un subdominio; no hay pantalla genérica que pregunte de qué organización eres.

**Por qué UUID y no el entero de `auth_user`:** el PK entero es autoincremental por esquema, así que el usuario `id=5` existe en dos organizaciones y son personas distintas. Un JWT que dice "soy el 5", presentado contra otro subdominio, autenticaría como alguien completamente diferente. Con UUID, un id de un esquema no existe en otro.

**Segunda medida, además del UUID:**

```python
token["tenant"] = connection.schema_name
# rechazar si token["tenant"] != connection.schema_name
```

**Detalles:**
- `django_session` también es por esquema. Login en una organización no da sesión en otra.
- MFA con `django-otp`; sus tablas en TENANT_APPS.
- Recuperación de contraseña: URL desde `connection.tenant`, nunca desde un `SITE_URL` fijo.
- `admin.tudominio.mx` autentica contra el esquema `public` — entrada del proveedor, detrás de Cloudflare Access.
- La misma persona en dos organizaciones son dos cuentas. El `unique` del email es por esquema.

---

## Fase 0 — Preparación

Sin código.

- [ ] Cuentas: GitHub, Cloudflare, registrador de dominio, Resend/Brevo, Sentry
- [ ] Dominio registrado, nameservers en Cloudflare
- [ ] Local: Docker 24+, Compose v2, Python 3.12, Node 20, `uv`
- [ ] Usuario en grupo `docker`
- [ ] Definir formato de slug: minúsculas, letras/números/guión medio, 3-30 chars
- [ ] Reservar: `admin api www app staging dev mail static`

**Bloqueadores no técnicos — arrancar ahora, tardan semanas:**
- [ ] Aviso de privacidad LFPDPPP: declara transferencia internacional (VPS fuera de México), Cloudflare como subencargado que ve tráfico descifrado, R2 como almacén.
- [ ] Por escrito: ¿de quién es el expediente si una profesional deja la Red? ¿de quién es el código?

---

## Fase 1 — Esqueleto

**Objetivo:** `docker compose up` levanta el proyecto.

1. Repo, `pyproject.toml`, Dockerfile
2. `CLAUDE.md`, `docs/flujo-desarrollo.md`, `.claude/commands/`
3. Settings por entorno + django-tenants configurado
4. `comun/models.py`: `BaseModel` (UUID + timestamps), `SoftDeleteMixin` (borrado lógico)
5. **`apps/usuarios/` con `AUTH_USER_MODEL` de PK UUID — antes del primer `migrate`.** Cambiarlo después es una de las migraciones más dolorosas de Django.
6. `docker-compose.yml` (db, web, caddy) + override local con hot reload
7. Endpoint `/healthz`
8. GitHub Actions: pytest + test de que `nucleo/` no importa Django

**Paridad obligatoria:** local y producción usan la misma imagen, el mismo gunicorn, el mismo Postgres 16. Solo cambian variables de entorno. Es lo que hace insertable la Fase X.

**Salida:**
- [ ] `/healthz` responde
- [ ] pytest corre en CI
- [ ] `grep -r "import django" backend/nucleo/` vacío
- [ ] `AUTH_USER_MODEL` apunta a `usuarios.Usuario` con PK UUID, y el primer `migrate` ya corrió con él

---

## Fase 2 — Esquema público

**Objetivo:** crear organizaciones por comando.

1. Modelos en `compartido/organizaciones/`: `Plan`, `Organizacion(TenantMixin)`, `Dominio(DomainMixin)`
2. `auto_create_schema = True`, `auto_drop_schema = False`
3. `nucleo/planes/limites.py`: dataclass `EstadoPlan` con `puede_agregar` y `requiere_aviso` (80%)
4. Comando `crear_organizacion`: valida slug → crea esquema → migra → siembra especialidades y plantillas base → registra dominio → crea administradora

```bash
python manage.py crear_organizacion --nombre "Red Sinosófica" \
  --slug redsinosofica --plan base --admin-email coordinacion@ejemplo.mx
```

**Reglas de límite:**
- Usuario activo = cuenta habilitada (`activa=true`), no login reciente
- Pacientes sin límite numérico
- Aviso al 80%, bloqueo de creación al 100%
- Nunca se bloquea lectura

**Salida:**
- [ ] Dos organizaciones creadas; `\dn` muestra ambos esquemas
- [ ] Tests de `nucleo/planes/` pasan sin BD
- [ ] Alta de organización sin un solo paso manual

---

## Fase 3 — Primera entidad de tenant

**Objetivo:** `Profesional` + `Especialidad` M2M de punta a punta. Es la plantilla que se copia ~15 veces. Hazla despacio.

1. Modelos: `Profesional`, `Especialidad`, `ProfesionalEspecialidad` (con `cedula_profesional`, `verificada_at`)
2. `migrate_schemas`
3. Admin
4. Serializers con especialidades anidadas
5. ViewSet; el límite de plan se valida llamando al núcleo desde `perform_create`
6. Auditoría por trigger — conéctala aquí
7. Paginación, filtros y orden resueltos en `comun/`, una vez

**Test que valida toda la arquitectura:** crear dos organizaciones y verificar que una no ve las profesionales de la otra.

**Salida:**
- [ ] CRUD contra `redsinosofica.localhost:8080`
- [ ] Límite de plan bloquea al máximo
- [ ] Test de aislamiento en verde
- [ ] Test: un JWT emitido por la organización A es rechazado en el subdominio de B
- [ ] Auditoría registra creación, actualización y borrado lógico

---

## Fase 4 — Pacientes y ciclo

1. `Paciente` — identidad separada de lo clínico; las tablas clínicas solo referencian `paciente_id`
2. `AntecedentesGenerales` (1-1, JSONB)
3. `AntecedentesGinecobstetricos` (1-1, ámbito de acceso distinto)
4. `RegistroCiclo` — serie temporal, único por `(paciente_id, fecha)`
5. `nucleo/ciclo/calculo_fase.py`

**El cálculo de fase debe devolver `INDETERMINADA` o `NO_APLICA` en vez de inventar día de ciclo.** Asumir 28 días regulares le falla a muchas pacientes y falla en silencio.

**Cifrado con pgcrypto:** solo CURP, teléfono y domicilio. Una columna cifrada no se indexa ni se busca.

**Salida:**
- [ ] Tests de fase cubren: irregular, amenorrea, anticoncepción continua, posmenopausia
- [ ] Query de correlación métrica ↔ fase funciona

---

## Fase 5 — Plantillas y consultas

**Patrón:** definiciones relacionales, valores en JSONB.

1. `PlantillaFormulario`, `PlantillaVersion` (inmutable), `CampoDefinicion`
2. `nucleo/plantillas/validacion.py` — valida captura contra definiciones
3. `CHECK` con `jsonb_matches_schema` (`pg_jsonschema`)
4. `Consulta` — contenedor: SOAP, signos vitales, fase del ciclo denormalizada
5. `ConsultaBloque` — uno por especialidad, con `metricas_especialidad` y `campos_custom_usuario`
6. Índices GIN en ambas columnas JSONB
7. Sembrar plantillas base por especialidad, vía migración

**Sin constructor visual en etapa 1.** Las plantillas se siembran; las profesionales las usan pero no las editan. El motor ya queda completo.

**Reglas:**
- Cada bloque guarda `plantilla_version_id`. Editar un formulario no reinterpreta el histórico.
- Publicada una versión, `clave` y `tipo` son inmutables. Se crea campo nuevo y se depreca el viejo.

**Salida:**
- [ ] Consulta con dos bloques de especialidades distintas
- [ ] Captura inválida rechazada por núcleo y por CHECK
- [ ] Editar plantilla no altera consultas anteriores

---

## Fase 6 — Consentimiento, RLS y auditoría

**Fase de mayor riesgo. Revisión manual obligatoria.**

1. `ConsentimientoAcceso` — paciente × profesional × ámbito, con vigencia y evidencia
2. `nucleo/consentimiento/reglas.py`
3. Middleware: `set_config('app.profesional_id', ..., true)` por request
4. Políticas RLS sobre tablas clínicas
5. Triggers de auditoría en todas las tablas clínicas
6. Registrar `intento_denegado`
7. Auditoría append-only: revocar UPDATE y DELETE a todos los roles

Dos capas: el esquema aísla organizaciones, la RLS aísla profesionales dentro de una organización.

Con pgbouncer, modo `transaction`.

**Salida:**
- [ ] Profesional sin consentimiento de ámbito no lee el bloque, ni por API ni por SQL directo con su rol
- [ ] Revocar corta el acceso de inmediato
- [ ] Intentos denegados en `auditoria`
- [ ] Ninguna política depende de que la app filtre bien

---

## Fase 7 — Frontend

Pantallas: login con MFA · agenda semanal · directorio de pacientes · expediente (timeline filtrable por especialidad) · captura de consulta (SOAP + bloques dinámicos) · registro de ciclo · administración (profesionales, plan, uso).

**Componente crítico:** el renderizador de bloques, que recibe `campos_definicion` y produce el formulario. Se reutiliza cuando llegue el constructor visual.

**Correo transaccional:** las URLs de recuperación se generan desde el tenant actual, nunca desde un `SITE_URL` fijo. Bug que solo aparece con el segundo cliente.

---

## Fase 8 — Endurecimiento y piloto

- [ ] MFA obligatoria (`django-otp`, TOTP)
- [ ] Rate limiting en login y recuperación
- [ ] Cloudflare Access sobre `admin.` y `staging.`
- [ ] Sentry con filtrado de PII
- [ ] HSTS, CSP, X-Frame-Options
- [ ] Sin PII en logs
- [ ] Respaldo nocturno cifrado a R2
- [ ] **Restauración de prueba real** — bajar dump, levantar en limpio, verificar datos
- [ ] Aviso de privacidad publicado
- [ ] Piloto con 2-3 profesionales antes de abrir a toda la Red

Un respaldo que nunca se restauró no es un respaldo.

---

## Fase X — VPS y staging (opcional, en cualquier momento)

Insertable entre cualquier par de fases gracias a la paridad de compose. Desplegar cambia variables de entorno, no el proyecto.

**Momento recomendado:** semana 1 si vas a mostrar avances a la Red. Límite práctico: antes de la Fase 7, para construir el SPA contra un backend remoto.

1. Hetzner CX/CAX 4 GB, Ubuntu 24.04. CAX (ARM) es más barato; verifica builds arm64.
2. SSH solo con llave, `PermitRootLogin no`, `ufw` sin puertos entrantes — ni el 22
3. DNS `*.tudominio.mx` → túnel
4. Ingress wildcard:

```yaml
ingress:
  - hostname: "*.tudominio.mx"
    service: http://caddy:80
  - service: http_status:404
```

5. **Access por hostname, no sobre el wildcard.** Una política por organización con su propia lista de correos.
6. Deploy: push → Actions → SSH → `docker compose pull && up -d && migrate_schemas`

**Limitaciones a conocer:**
- SSL universal cubre `*.tudominio.mx` pero no `*.*.tudominio.mx`. Subdominios de un nivel.
- 100 MB por request en plan gratuito. Archivos pesados van directo a R2 con URL firmada.
- Cloudflare ve el tráfico descifrado. Es subencargado y debe estar en el aviso de privacidad.

**Regla:** staging nunca ve datos reales de pacientes. Banner permanente, datos sembrados falsos, y dicho en voz alta.

**Salida:**
- [ ] `staging.tudominio.mx` con TLS
- [ ] `ufw status` sin puertos entrantes
- [ ] Deploy por push
- [ ] Respaldo corriendo y restauración probada

---

## Anexo A — Diferido a etapas posteriores

Criterio: no "qué es importante", sino **qué es caro de retrofitear**.

| Diferido | Por qué se puede |
|---|---|
| Constructor visual de formularios | El motor ya está; falta la interfaz |
| Portal de pacientes | App y endpoints nuevos, modelo intacto |
| Recordatorios | Agregar Celery + Redis son dos horas |
| Adjuntos | Tabla diseñada, R2 contratado |
| Reportes y correlación con ciclo | Los datos ya se capturan desde el día uno |
| GraphQL | Se monta sobre los mismos modelos |
| Cobranza automatizada | Con una organización, es construir pagos para un cliente |
| Instancia dedicada por cliente | `pg_dump -n esquema` + otro compose |

---

## Anexo B — Planificación de la etapa 2

La etapa 2 la define el uso real, no este documento. Insumos primero: qué se pidió más, dónde se atoran, qué dejaron de usar.

Entrevista solo lo que cambia modelo o seguridad: portal de pacientes, constructor de formularios, adjuntos. El resto se agrega encima.

Una sesión de `/interview-me` por feature, no por etapa. Cierra escribiendo `docs/specs/<tema>.md`, `/clear`, luego modo plan sobre ese archivo.

Lo que no puedas contestar tú, márcalo como abierto y llévalo a la Red. Un requisito inventado con seguridad aparente es peor que uno marcado como pendiente.

Construye una feature, ponla en producción, y entonces entrevista la siguiente.
