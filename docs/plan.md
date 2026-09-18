# Plan de Desarrollo — Red Sinosófica

Sistema de gestión clínica multidisciplinaria, multi-tenant, para redes de profesionales de la salud.

Desarrollador: proveedor independiente. El sistema es también un producto comercial multi-tenant futuro.
Cliente inicial: Red Sinosófica (una organización).

---

## Contexto

Redes de profesionales (médicas, nutricionistas, psicólogas, etc.) enfocadas en atención integral de mujeres cis. Expedientes centralizados, consultas multidisciplinarias, planes de tratamiento compartidos, consentimientos de paciente con granularidad por profesional.

El sistema debe:
- Aislar datos de cada organización a nivel de base de datos (esquema)
- Permitir crecer desde 1 cliente a N sin cambiar la arquitectura
- Hacer la lectura del expediente nunca más lenta cuando se alcanza límite de plan
- Cumplir LFPDPPP (datos sensibles de paciente, almacenamiento en México)

---

## Stack Elegido

| Área | Decisión | Razón |
|---|---|---|
| Backend | Django 5.1 + DRF | ORM maduro, migraciones, admin out-of-box |
| API | REST (GraphQL después) | MVP rápido sobre los mismos models |
| BD | PostgreSQL 16 | Multi-tenancy nativa con esquemas, RLS, JSONB |
| Multi-tenancy | `django-tenants` (un esquema por org) | Aislamiento garantizado, fácil portabilidad |
| Ruteo | Subdominio (`<org>.tudominio.mx`) en middleware | Escalable, standard multi-tenant |
| Ingreso | Cloudflare Tunnel | Cero puertos abiertos, sin VPN del proveedor |
| Frontend | Vite + React + TypeScript + TanStack Query | SPA rápida, sin SSR al inicio |
| Archivos | Cloudflare R2 | Escalable, URLs firmadas, CORS simple |
| Correo | Resend o Brevo | SMTP confiable, transaccional de bajo costo |
| Infra | VPS Hetzner 4 GB, Docker Compose | Dev ≈ prod, fácil de reproducir |
| Auth | OAuth 2 (JWT con claim `tenant`) | Aislamiento validado en cada request |
| Autenticación multifactor | django-otp + TOTP | Requerido Fase 8 |
| Trabajo asíncrono | Diferido a etapa 2 | MVP síncrono, Redis/Celery después |

---

## Decisiones Cerradas

- **No Supabase**, no Firebase. Base de datos en VPS controlada.
- **No GraphQL todavía.** REST MVP, GraphQL sobre los mismos modelos en etapa 2.
- **No constructor visual de formularios en etapa 1.** Plantillas sembradas por migración.
- **No portal de pacientes en etapa 1.** Profesionales primero.
- **No reportes ni gráficas en etapa 1.** Solo datos brutos.
- **Sin Redis/Celery hasta fase 2.** El MVP es síncrono.
- **UUID como PK.** Nunca autoincremental dentro de un esquema de tenant (colisión entre orgs).

---

## Arquitectura

### Monolito en capas

```
┌─────────────────────────────────┐
│  ADAPTADORES (DRF)              │  Serializers, viewsets, urls, admin
├─────────────────────────────────┤
│  NÚCLEO (py puro)               │  Reglas clínicas, sin Django
├─────────────────────────────────┤
│  PERSISTENCIA (Django ORM)      │  Models, migraciones, RLS, triggers
└─────────────────────────────────┘
```

Modularidad **horizontal** (capas), no vertical (no crear paquetes con frontera dura).
Las apps se importan entre sí con FK directas. No crear fachadas ni eventos.

### Multi-tenancy: cómo funciona

Base de datos única. Dos niveles de esquemas:

**Esquema `public` (compartido):**
- `planes` — planes de suscripción
- `organizaciones` — datos de organización + TenantMixin
- `dominios` — Host → organización
- `organizacion_uso` — counters de límites actuales

**Esquema `<slug>` (por organización):**
- `usuarios` — AUTH_USER_MODEL, PK UUID
- `profesionales`, `especialidades`, `profesionales_especialidades`
- `pacientes`, `antecedentes_generales`, `antecedentes_ginecobstetricos`
- `registros_ciclo` — serie temporal
- `plantillas_formulario`, `plantilla_versiones`, `campos_definicion`
- `citas`, `consultas`, `consulta_bloques`
- `consentimientos_acceso`
- `auditoria` — append-only
- `adjuntos`, `planes_tratamiento`

**Cómo aisla:**

1. Middleware lee `Host` → busca en `public.dominios` → fija `connection.schema_name`
2. Todas las queries se ejecutan dentro de ese esquema (`SET search_path TO <slug>`)
3. Las tablas del otro esquema no existen en la sesión: un bug en un queryset no puede fugar datos

**Cómo migrar:**

```bash
pg_dump -n sinosofica_org1 > org1.sql
# Restaurar en otro servidor
```

---

## Autenticación

Una tabla `usuarios` (AUTH_USER_MODEL) **por esquema de tenant**, con PK UUID.

**Por qué UUID:**

Por defecto, `auth_user` usa entero autoincremental. El usuario id=5 existe en dos organizaciones, pero son personas distintas.
Un JWT que dice "soy el 5" presentado en otro subdominio autenticaría como alguien diferente.

Con UUID no hay colisión posible: `123e4567-e89b-12d3-a456-426614174000` solo existe una vez en el planeta.

**Medidas obligatorias:**

1. `AUTH_USER_MODEL = "usuarios.Usuario"` con `UUIDField` como PK — antes del primer `migrate`
2. Claim `tenant` en el JWT, validado contra `Host` del request
3. Middleware orden: `TenantMainMiddleware` → `SessionMiddleware` → `AuthenticationMiddleware`

---

## Invariantes Arquitectónicas (CLAUDE.md)

- Monolito en capas, NO modular. Sin fachadas entre apps.
- `nucleo/` NUNCA importa Django.
- Tablas de tenant NUNCA llevan `organizacion_id`. Aislamiento por `search_path`.
- `AUTH_USER_MODEL` propio con PK UUID.
- JWT lleva `tenant` y se valida contra Host.
- Todo modelo hereda `BaseModel` (UUID + timestamps) + `SoftDeleteMixin`.
- Consulta con `finalizada_at` no se modifica. Solo notas.
- Lectura del expediente nunca se bloquea por límite de plan.
- Pacientes sin límite. Solo profesionales activas.
- Plantilla publicada: clave y tipo inmutables.
- Autorización nueva = política RLS + test por SQL.

---

## Estructura del Proyecto (después de Fase 1)

```
redsinosofica/
├── CLAUDE.md
├── docker-compose.yml            # idéntico a producción
├── docker-compose.override.yml   # solo local
├── Dockerfile / pyproject.toml
│
├── backend/
│   ├── config/settings/{base,local,produccion}.py
│   ├── nucleo/                   # sin imports de Django
│   │   ├── ciclo/
│   │   ├── plantillas/
│   │   ├── consentimiento/
│   │   ├── planes/
│   │   └── __init__.py (test CI: falla si importa Django)
│   ├── compartido/
│   │   ├── organizaciones/
│   │   └── dominios/
│   ├── comun/
│   │   ├── base.py               # BaseModel, SoftDeleteMixin
│   │   ├── auditoria/
│   │   ├── permisos.py
│   │   └── middleware.py
│   └── apps/
│       ├── usuarios/             # ★ AUTH_USER_MODEL, antes del 1er migrate
│       ├── profesionales/
│       ├── pacientes/
│       ├── agenda/
│       ├── consultas/
│       ├── plantillas/
│       └── privacidad/
│
├── frontend/src/{api,componentes,paginas,hooks}/
│
├── infra/
│   ├── caddy/Caddyfile
│   ├── cloudflared/config.yml
│   └── scripts/
│
├── docs/
│   ├── plan.md                   # ← este archivo
│   ├── flujo-desarrollo.md
│   ├── esquema.dbml
│   ├── decisiones/
│   │   ├── 001-multitenancy.md
│   │   ├── 002-auth-uuid.md
│   │   └── ...
│   ├── specs/
│   │   └── <tema>.md             # generado por /interview-me
│   └── roadmaps/
│       └── <feature>.md          # por cada feature
│
└── .claude/
    └── commands/
        ├── entidad.md
        ├── dominio.md
        └── privacidad.md
```

---

## Fases de Desarrollo

### Fase 0 — Planificación (esta semana)

**Entregables:**
- [ ] Repo con `.gitignore`, `README.md`, `CLAUDE.md`, `docs/plan.md`
- [ ] ADRs principales en `docs/decisiones/`
- [ ] Esquema DBML en `docs/esquema.dbml`
- [ ] Aviso de privacidad (LFPDPPP) — quién es dueño del expediente, del código, transferencia internacional
- [ ] Cartas firmadas de consentimiento de datos

### Fase 1 — Esqueleto Backend (semana 1–2)

**Qué se construye:**
- Repo con `docker-compose.yml` idéntico a producción
- Django 5.1 + DRF configurado
- `django-tenants` integrado
- `apps/usuarios/` con UUID antes del primer `migrate`
- `BaseModel` + `SoftDeleteMixin` (compartidos)
- Endpoint `/healthz`
- CI que falla si `nucleo/` importa Django

**No se construye:**
- Frontend (eso es Fase 7)
- Autorización (eso es Fase 6)
- Trabajo asíncrono (eso es etapa 2)

**Commits:**
1. Repo inicial + Docker
2. Django + settings base/local/prod
3. django-tenants + middleware
4. AUTH_USER_MODEL con UUID
5. BaseModel + migrations
6. CI + /healthz

### Fase 2 — Esquema Público (semana 2–3)

**Qué se construye:**
- Tabla `Organizacion` (TenantMixin, auto_drop_schema=False)
- Tabla `Plan` con campos de límites
- Tabla `Dominio` (DomainMixin)
- Comando `crear_organizacion` (nunca manual)
- Lógica de límites en `nucleo/planes/limites.py` (dataclass, sin Django)

**Commits:**
1. Models + migraciones de public
2. Comando crear_organizacion
3. Lógica de límites en núcleo
4. Tests de isolamiento entre organizaciones

### Fase 3 — Primera Entidad de Tenant (semana 3–4)

**Qué se construye:**
- Profesional + Especialidad (M2M con cédula y verificación)
- Test de aislamiento entre dos orgs distintas
- Test de token cruzado → rechazado
- Auditoría por trigger desde aquí
- ModelViewSet plano sin lógica

**Commits:**
1. Models de profesional + especialidad
2. Migraciones per schema
3. Tests de aislamiento
4. Trigger de auditoría
5. Serializer + viewset

### Fase 4 — Pacientes y Ciclo (semana 4–5)

**Qué se construye:**
- Paciente (identidad independiente)
- Antecedentes (1-1 JSONB)
- RegistroCiclo (serie temporal, unique por paciente+fecha)
- Cálculo de fase en `nucleo/ciclo/` — devuelve INDETERMINADA o NO_APLICA, nunca asume 28 días
- Cifrado selectivo (pgcrypto): CURP, teléfono, domicilio

**Commits:**
1. Models de paciente + antecedentes
2. RegistroCiclo con índices
3. Función de cálculo de fase (núcleo)
4. Tests del ciclo
5. Migración de cifrado selectivo

### Fase 5 — Plantillas y Consultas (semana 5–6)

**Qué se construye:**
- PlantillaFormulario (definiciones relacionales)
- PlantillaVersion (inmutable)
- CamposDefinicion (qué campos, en qué orden, validaciones)
- Consulta (contenedor SOAP + fase denormalizada)
- ConsultaBloque (metricas_especialidad JSONB + campos_custom, índices GIN)
- Plantillas sembradas por migración (sin constructor visual aún)

**Commits:**
1. Models de plantilla + campo
2. Migraciones de plantilla_version
3. Model Consulta + bloque
4. Índices GIN en consulta_bloques
5. Serializers + viewsets
6. Seed de plantillas

### Fase 6 — Consentimiento, RLS y Auditoría (semana 6–8)

⚠️ **Revisión manual obligatoria.**

**Qué se construye:**
- Tabla `ConsentimientosAcceso` (paciente × profesional × ámbito, vigencia)
- RLS policies (aísla profesionales dentro de la misma org)
- Middleware fija `app.profesional_id`
- Auditoría append-only con `intento_denegado`
- Tests por SQL directo para cada nueva política

**Commits:**
1. Model ConsentimientosAcceso
2. RLS policies (con tests SQL)
3. Middleware de profesional_id
4. Tabla de auditoría append-only
5. Tests de negación
6. Documentación de cada política

### Fase 7 — Frontend SPA (semana 8–10)

**Qué se construye:**
- Vite + React + TypeScript
- TanStack Query (caché + sincronización)
- Componente renderizador de bloques dinámicos (plantillas)
- URLs de correo desde `connection.tenant`
- Formulario de login con multi-tenancy
- No constructor visual (Fase 2 de etapa posterior)

**Commits:**
1. Setup Vite + React
2. TanStack Query + API client
3. Componentes base
4. Renderizador de bloques
5. Flujo de login + redirect por tenant
6. Formularios CRUD de profesional, paciente

### Fase 8 — Endurecimiento (semana 10–12)

**Qué se construye:**
- MFA con django-otp (TOTP)
- Rate limiting en endpoints críticos
- Cloudflare Access en `/admin/`
- Sentry con filtro PII
- Headers de seguridad (CSP, HSTS, X-Frame-Options)
- Respaldo cifrado a R2
- **Restauración de prueba real** (backup → restaurar en VPS test → verificar)
- Tests de penetración básicos

**Commits:**
1. MFA setup + serializer
2. Rate limiting por IP + user
3. Sentry + filtro PII
4. Headers de seguridad
5. Script de backup a R2
6. Script de restauración
7. Test de restauración

---

## Etapa 2 (Después de MVP)

Insertable en cualquier momento, pero recomendado después de Fase 8:

- Constructor visual de formularios
- Portal de pacientes (solo lectura del expediente)
- Recordatorios (Celery + Redis)
- Adjuntos (upload a R2)
- Reportes y gráficas de correlación
- GraphQL sobre los mismos models
- Cobranza automatizada
- Instancia dedicada por cliente (`pg_dump -n slug` → otro servidor)

---

## Límites de Planes (Multi-tenant Comercial)

Tabla `Plan` en esquema `public`. Límites resueltos en `nucleo/planes/limites.py` (dataclass, sin Django).

Pacientes: sin límite numérico.
Profesionales activas: sí tienen límite.

**Bloqueo:**
- Aviso al 80% del límite
- Bloqueo de creación al 100%
- Lectura del expediente: **nunca bloqueada**

**Plan "dedicado":**
- Upsell: VPS dedicado
- Migración: `pg_dump -n slug > org.sql` + `docker-compose` nuevo en otro servidor
- Misma imagen, otro compose

---

## Bloqueadores No Técnicos (Fase 0)

- [ ] Aviso de privacidad LFPDPPP
  - Transferencia internacional de datos (a Cloudflare, a R2)
  - Cloudflare es subencargado de datos
  - Retención, derechos de acceso, derecho al olvido
  
- [ ] Cartas firmadas
  - Quién es dueño del expediente (la organización)
  - Quién es dueño del código (tú, o la organización)
  - Permisos de uso de datos para mejora del producto

---

## Comandos de Desarrollo (.claude/commands/)

Viven en Ponytail, se activan con `/` en Claude Code:

- **`/entidad`** — Checklist para nueva tabla de tenant (modelo, test, serializer, viewset)
- **`/dominio`** — Checklist para nueva regla en `nucleo/` (test primero, sin BD, adaptador Django)
- **`/privacidad`** — Checklist para cambio que toca RLS (política primero, test SQL, revisión manual obligatoria)

---

## Métricas y Señales

- **Cobertura de tests:** 80%+ (requerido en CI)
- **Tiempo de request GET:** <100ms (p99, sin query N+1)
- **Restauración de backup:** <5 min (probado cada Fase)
- **Uptime:** 99.9% (de aquí en adelante, con load balancer)

---

## Diferido Explícitamente

- No multi-región (una VPS)
- No instancia dedicada por cliente (Fase 2)
- No GraphQL (Fase 2)
- No portal de pacientes (Fase 2)
- No constructor visual (Fase 2)
- No Celery (Fase 2)
- No searchable PII en expedientes (nunca, cifrado siempre)

---

## Rollout (después de Fase 8)

1. Ambiente de prueba interno (todo el flujo)
2. Invitación a Red Sinosófica para pruebas
3. Feedback y fixes (máximo 2 semanas)
4. Producción en viernes (rollback preparado)
5. Monitoreo 24h primer fin de semana
6. Soporte on-call primer mes

---

*Última actualización: 2026-09*
*Responsable: Alonso, arquitecto del sistema*
