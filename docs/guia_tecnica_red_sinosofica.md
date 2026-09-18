# Red Sinosófica — Guía técnica de desarrollo por fases

Sistema de gestión clínica multidisciplinaria, multi-tenant, para redes de profesionales de la salud enfocadas en atención integral de mujeres cis.

Documento operativo: cada fase es ejecutable de principio a fin y deja el sistema funcionando. Ninguna fase requiere que la siguiente exista.

---

## Decisiones cerradas

Antes de las fases, el resumen de lo que ya está decidido y por qué. Si alguna de estas se reabre, la guía cambia.

| Decisión | Elección | Motivo |
|---|---|---|
| Backend | Django 5.1 + DRF | La complejidad está en las reglas de negocio, no en el CRUD. Un BaaS ahorra lo fácil y no ayuda en lo difícil. |
| API | REST (DRF), no GraphQL | Con backend propio, GraphQL cuesta resolvers, DataLoaders y permisos por campo. Se puede agregar después sin tirar nada. |
| Base de datos | PostgreSQL 16 | JSONB, RLS, pgcrypto, `pg_jsonschema`. |
| Multi-tenancy | Un esquema por organización (`django-tenants`) | Aislamiento real de datos + un solo despliegue. Extraer un tenant a su propio VPS es `pg_dump -n esquema`. |
| Ruteo de tenant | Subdominio (`org.tudominio.mx`) | Resuelto por `Host` en middleware. Una organización nueva no toca DNS ni deploy. |
| Frontend | Vite + React SPA | Todo detrás de login: SSR no aporta. Un `dist/` estático, sin proceso Node en producción. |
| Ingreso | Cloudflare Tunnel | Cero puertos abiertos en el servidor. Elimina la superficie de ataque más explotada. |
| Arquitectura | MVT/DRF plano + núcleo de dominio aislado | Hexagonal solo donde paga: las reglas clínicas. |
| Archivos | Cloudflare R2 | No autohospedar almacenamiento de expedientes. |
| Correo | Resend o Brevo (capa gratuita) | No autohospedar SMTP. |
| Infra | VPS Hetzner 4 GB, Docker Compose | Facturación mensual, sin compromiso, sin salto de renovación. |

### Dos bloqueadores que no son técnicos

Ambos tardan más de lo que se cree y ambos impiden el lanzamiento. Ponlos a caminar en la Fase 0, no en la 8.

1. **Aviso de privacidad** conforme a la LFPDPPP, declarando: transferencia internacional de datos (el VPS no está en México), Cloudflare como subencargado que ve tráfico descifrado, y R2 como almacén de archivos.
2. **Propiedad del expediente y del código.** Si una profesional deja la Red, ¿qué pasa con sus notas? Y si el sistema es además tu producto comercial, ¿quién es dueño del código? Por escrito, antes de la primera línea.

---

## Fase 0 — Preparación

No se escribe código. Es una lista de verificación.

### 0.1 Cuentas

- [ ] GitHub (repositorio privado)
- [ ] Cloudflare (DNS, Tunnel, Access, R2)
- [ ] Registrador del dominio — Cloudflare Registrar para `.com`, Akky o Neubox para `.mx`
- [ ] Resend o Brevo (correo transaccional)
- [ ] Sentry, capa gratuita (errores)
- [ ] Hetzner — **solo cuando decidas hacer la Fase X**

### 0.2 Herramientas locales

```bash
# Ubuntu
docker --version              # 24+
docker compose version        # v2
python3 --version             # 3.12
node --version                # 20 LTS
git --version

# Gestor de dependencias Python — uv es notablemente más rápido
curl -LsSf https://astral.sh/uv/install.sh | sh
```

Verifica que tu usuario esté en el grupo `docker` para no usar `sudo`:

```bash
sudo usermod -aG docker $USER   # requiere cerrar sesión
```

### 0.3 Dominio

- [ ] Dominio registrado
- [ ] Nameservers apuntando a Cloudflare
- [ ] Registro DNS `*.tudominio.mx` — se configura en Fase X, pero decide el dominio ahora

### 0.4 Decisiones de nomenclatura

El `slug` de cada organización es permanente: aparece en URLs guardadas en favoritos y en correos de recuperación. Define ahora:

- Formato: minúsculas, solo letras, números y guión medio, 3 a 30 caracteres
- Reservados: `admin`, `api`, `www`, `app`, `staging`, `dev`, `mail`, `static`

---

## Fase 1 — Esqueleto y arquitectura

Al terminar: `docker compose up` levanta el proyecto y `/healthz` responde.

### 1.1 La arquitectura, explicada

Tres capas, con una regla de dependencia: **el núcleo no importa Django**.

```
┌─────────────────────────────────────────────┐
│  ADAPTADORES (Django/DRF)                   │
│  serializers, viewsets, urls, admin         │
│  Traducen HTTP ↔ objetos de Python          │
└───────────────────┬─────────────────────────┘
                    │
┌───────────────────▼─────────────────────────┐
│  NÚCLEO DE DOMINIO (Python puro)            │
│  Reglas clínicas. Sin ORM, sin request.     │
│  Se testea sin base de datos.               │
└───────────────────┬─────────────────────────┘
                    │
┌───────────────────▼─────────────────────────┐
│  PERSISTENCIA (Django ORM + Postgres)       │
│  models.py, migraciones, RLS                │
└─────────────────────────────────────────────┘
```

**Qué va en el núcleo (y solo esto):**

- Validación de captura contra `campos_definicion`
- Resolución de consentimiento por ámbito
- Cálculo de fase de ciclo a partir de registros
- Detección de conflictos entre planes de distintas especialidades
- Verificación de límites de plan

**Qué NO va en el núcleo:** el CRUD de pacientes, citas y catálogos. Eso es Django plano con `ModelViewSet`. Nada de repositorios envolviendo el ORM — el ORM ya es tu capa de persistencia.

El motivo de aplicarlo parcial: hexagonal paga cuando la lógica es volátil o hay varios desarrolladores. Aquí la mitad del sistema es CRUD estable y eres una persona. Abstraer todo cuesta semanas en interfaces que nadie va a intercambiar nunca.

### 1.2 Estructura de carpetas

```
redsinosofica/
├── docker-compose.yml           # idéntico a producción
├── docker-compose.override.yml  # solo local (git-ignored en prod)
├── Dockerfile
├── pyproject.toml
├── .env.example
│
├── backend/
│   ├── manage.py
│   ├── config/
│   │   ├── settings/
│   │   │   ├── base.py
│   │   │   ├── local.py
│   │   │   └── produccion.py
│   │   ├── urls.py
│   │   └── wsgi.py
│   │
│   ├── nucleo/                  # ★ Python puro, sin imports de Django
│   │   ├── ciclo/
│   │   │   ├── entidades.py
│   │   │   ├── calculo_fase.py
│   │   │   └── tests/
│   │   ├── plantillas/
│   │   │   ├── validacion.py
│   │   │   └── tests/
│   │   ├── consentimiento/
│   │   │   ├── reglas.py
│   │   │   └── tests/
│   │   └── planes/
│   │       ├── limites.py
│   │       └── tests/
│   │
│   ├── compartido/              # SHARED_APPS — esquema público
│   │   └── organizaciones/
│   │       ├── models.py        # Organizacion, Dominio, Plan
│   │       ├── management/commands/crear_organizacion.py
│   │       └── admin.py
│   │
│   ├── comun/                   # mixins y utilidades transversales
│   │   ├── models.py            # BaseModel, SoftDeleteMixin
│   │   ├── auditoria.py
│   │   ├── permisos.py
│   │   └── paginacion.py
│   │
│   └── apps/                    # TENANT_APPS — replicadas por esquema
│       ├── profesionales/
│       │   ├── models.py
│       │   ├── serializers.py
│       │   ├── views.py
│       │   ├── urls.py
│       │   └── tests/
│       ├── pacientes/
│       ├── agenda/
│       ├── consultas/
│       ├── plantillas/
│       └── privacidad/          # consentimientos, auditoría
│
├── frontend/
│   ├── package.json
│   ├── vite.config.ts
│   └── src/
│       ├── api/                 # cliente HTTP, tipos
│       ├── componentes/
│       ├── paginas/
│       └── hooks/
│
├── infra/
│   ├── caddy/Caddyfile
│   ├── cloudflared/config.yml
│   └── scripts/
│       ├── respaldo.sh
│       └── restaurar.sh
│
└── .github/workflows/
    ├── tests.yml
    └── deploy.yml               # inactivo hasta Fase X
```

### 1.3 Configuración de django-tenants

Esta es la parte que **no se puede retrofitear sin dolor**. Instalarla ahora cuesta una tarde; después significa reorganizar todas las apps entre esquemas.

```python
# config/settings/base.py

DATABASES = {
    "default": {
        "ENGINE": "django_tenants.postgresql_backend",
        # ...
    }
}

DATABASE_ROUTERS = ("django_tenants.routers.TenantSyncRouter",)

# Esquema público: existe una sola vez
SHARED_APPS = [
    "django_tenants",
    "compartido.organizaciones",
    "django.contrib.contenttypes",
    "django.contrib.auth",
    "django.contrib.staticfiles",
    "rest_framework",
]

# Replicadas dentro de CADA esquema de tenant
TENANT_APPS = [
    "django.contrib.contenttypes",
    "django.contrib.auth",
    "apps.profesionales",
    "apps.pacientes",
    "apps.agenda",
    "apps.consultas",
    "apps.plantillas",
    "apps.privacidad",
]

INSTALLED_APPS = list(SHARED_APPS) + [
    a for a in TENANT_APPS if a not in SHARED_APPS
]

TENANT_MODEL = "organizaciones.Organizacion"
TENANT_DOMAIN_MODEL = "organizaciones.Dominio"

MIDDLEWARE = [
    "django_tenants.middleware.main.TenantMainMiddleware",  # ← primero
    # ... el resto
]
```

**Consecuencia importante:** las tablas de tenant **no llevan `organizacion_id`**. El aislamiento lo da el `search_path` de Postgres. Un bug en un queryset no puede filtrar datos de otra organización, porque esas tablas ni siquiera están visibles en la sesión.

### 1.4 Convenciones obligatorias

Codifícalas en mixins desde el primer modelo. Retrofitear cualquiera de estas implica migrar datos.

```python
# comun/models.py
import uuid
from django.db import models

class BaseModel(models.Model):
    id = models.UUIDField(primary_key=True, default=uuid.uuid4, editable=False)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    class Meta:
        abstract = True


class SoftDeleteMixin(models.Model):
    deleted_at = models.DateTimeField(null=True, blank=True, db_index=True)

    objects = ActivosManager()      # filtra deleted_at IS NULL
    todos = models.Manager()

    def delete(self, using=None, keep_parents=False):
        self.deleted_at = timezone.now()
        self.save(update_fields=["deleted_at"])

    class Meta:
        abstract = True
```

| Convención | Regla |
|---|---|
| Llaves primarias | `UUIDv4`. Evita enumerar pacientes por ID secuencial. |
| Tiempos | `timestamptz` siempre, UTC en base, zona horaria solo al presentar. `USE_TZ = True`. |
| Borrado | Lógico. El expediente clínico no se borra (NOM-004-SSA3). |
| Correcciones | Nota de enmienda vinculada, nunca `UPDATE` sobre consulta firmada. |
| Auditoría | Triggers desde el primer modelo. Sin esto, los primeros meses no tienen trazabilidad y no se recupera. |

### 1.5 Compose base

```yaml
# docker-compose.yml — idéntico en local y producción
services:
  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: ${DB_NAME}
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USER}"]
      interval: 10s
    networks: [interna]

  web:
    build: .
    command: gunicorn config.wsgi:application --bind 0.0.0.0:8000 --workers 3
    env_file: .env
    depends_on:
      db: {condition: service_healthy}
    networks: [interna]

  caddy:
    image: caddy:2-alpine
    volumes:
      - ./infra/caddy/Caddyfile:/etc/caddy/Caddyfile:ro
      - ./frontend/dist:/srv/frontend:ro
    networks: [interna]

networks:
  interna:
volumes:
  pgdata:
```

```yaml
# docker-compose.override.yml — solo local
services:
  web:
    command: python manage.py runserver 0.0.0.0:8000
    volumes:
      - ./backend:/app
    environment:
      DJANGO_SETTINGS_MODULE: config.settings.local
  caddy:
    ports:
      - "8080:80"      # único puerto expuesto, y solo en local
  db:
    ports:
      - "5432:5432"    # para conectar tu cliente SQL
```

**Por qué importa que sean el mismo archivo:** si desarrollas con `runserver` + SQLite y despliegas con gunicorn + Postgres, las sorpresas de entorno aparecen todas juntas en la última semana. Con paridad, aparecen el día que las causas.

Sin Celery ni Redis. No hay trabajo asíncrono en las fases 1 a 6. Cuando llegue el primer recordatorio de cita, agregarlos son dos horas.

### 1.6 Verificación de fase

- [ ] `docker compose up` levanta db, web y caddy
- [ ] `curl localhost:8080/healthz` responde `{"ok": true}`
- [ ] `pytest` corre (aunque sea un test trivial)
- [ ] GitHub Actions ejecuta los tests en cada push
- [ ] `backend/nucleo/` no contiene un solo `import django`

---

## Fase 2 — Esquema público: organizaciones y planes

Primera entidad real. Vive en el esquema público y es la que habilita todo lo demás.

### 2.1 Modelos

```python
# compartido/organizaciones/models.py
from django_tenants.models import TenantMixin, DomainMixin
from django.db import models

class Plan(models.Model):
    clave = models.SlugField(unique=True)          # base, ampliado, dedicado
    nombre = models.CharField(max_length=60)
    max_profesionales = models.PositiveIntegerField()
    max_almacenamiento_mb = models.PositiveIntegerField()
    precio_mxn_mensual = models.DecimalField(max_digits=10, decimal_places=2)
    features = models.JSONField(default=dict)
    activo = models.BooleanField(default=True)


class Organizacion(TenantMixin):
    nombre = models.CharField(max_length=120)
    slug = models.SlugField(unique=True)
    plan = models.ForeignKey(Plan, on_delete=models.PROTECT)
    contacto_email = models.EmailField()
    activa = models.BooleanField(default=True)
    created_at = models.DateTimeField(auto_now_add=True)

    auto_create_schema = True        # crea el esquema al guardar
    auto_drop_schema = False         # ★ nunca borrar esquemas clínicos


class Dominio(DomainMixin):
    pass                             # dominio + tenant + is_primary
```

Nota `auto_drop_schema = False`. Un `Organizacion.objects.delete()` accidental no debe llevarse un expediente clínico completo.

### 2.2 Regla de negocio en el núcleo

```python
# nucleo/planes/limites.py  — sin Django
from dataclasses import dataclass

@dataclass(frozen=True)
class EstadoPlan:
    max_profesionales: int
    profesionales_activas: int

    @property
    def disponibles(self) -> int:
        return max(0, self.max_profesionales - self.profesionales_activas)

    @property
    def puede_agregar(self) -> bool:
        return self.disponibles > 0

    @property
    def requiere_aviso(self) -> bool:
        """Aviso al 80%, no error sorpresa al 100%."""
        if self.max_profesionales == 0:
            return False
        return self.profesionales_activas / self.max_profesionales >= 0.8
```

Se testea sin base de datos, sin request y sin fixtures.

### 2.3 Comando de alta

Escríbelo ahora, cuando creas la primera organización. Nunca des de alta un tenant con pasos manuales: ese sería tu techo de crecimiento.

```python
# compartido/organizaciones/management/commands/crear_organizacion.py

class Command(BaseCommand):
    def handle(self, *args, **opts):
        # 1. Validar slug (formato + lista de reservados)
        # 2. Crear Organizacion → django-tenants crea el esquema
        # 3. Correr migraciones sobre el esquema nuevo
        # 4. Registrar Dominio: f"{slug}.{DOMINIO_BASE}"
        # 5. Sembrar catálogo de especialidades
        # 6. Sembrar plantillas base por especialidad
        # 7. Crear cuenta de administradora + correo de bienvenida
```

```bash
python manage.py crear_organizacion \
    --nombre "Red Sinosófica" \
    --slug redsinosofica \
    --plan base \
    --admin-email coordinacion@ejemplo.mx
```

### 2.4 Reglas de negocio de los límites

| Regla | Decisión |
|---|---|
| Qué cuenta como usuario activo | Cuentas habilitadas (`activa = true`), no logins recientes. Predecible y sin sorpresas. |
| Pacientes | **Sin límite numérico.** Limitarlas incentivaría rechazar atención. Si el volumen pesa, el límite es almacenamiento. |
| Al llegar al 80% | Aviso visible a la administradora. |
| Al llegar al 100% | Se impide crear cuentas nuevas. |
| Al exceder o ante falta de pago | **Jamás se bloquea lectura del expediente.** Solo se impide crear. |

### 2.5 Verificación de fase

- [ ] `crear_organizacion` crea esquema, migra y siembra en un solo comando
- [ ] Dos organizaciones creadas localmente; `psql \dn` muestra ambos esquemas
- [ ] Tests del núcleo de límites pasan sin tocar la base
- [ ] `auto_drop_schema = False` verificado

---

## Fase 3 — Primera entidad de tenant, de punta a punta

`Profesional` + `Especialidad` con relación muchos-a-muchos. Es la **plantilla replicable**: toda entidad posterior sigue este mismo recorrido. Hazla con calma, porque el patrón se copia unas quince veces.

### 3.1 El recorrido completo

```
models.py       → Profesional(BaseModel, SoftDeleteMixin)
                  Especialidad
                  ProfesionalEspecialidad (tabla puente con cédula)
migración       → makemigrations + migrate_schemas
admin.py        → registro para operación interna
serializers.py  → lectura, escritura, y anidado de especialidades
views.py        → ModelViewSet con permisos
urls.py         → router DRF
tests/          → unitarios del núcleo + integración de la API
```

### 3.2 Dónde va cada regla

```python
# apps/profesionales/views.py
class ProfesionalViewSet(viewsets.ModelViewSet):
    def perform_create(self, serializer):
        estado = obtener_estado_plan(self.request.tenant)   # adaptador
        if not estado.puede_agregar:                        # ← núcleo decide
            raise ValidationError(
                f"El plan permite {estado.max_profesionales} profesionales activas."
            )
        serializer.save()
```

El adaptador consulta la base y arma el dataclass. El núcleo decide. La vista traduce a HTTP. Esa separación es todo lo que significa "hexagonal" aquí — nada de repositorios ni interfaces abstractas.

### 3.3 Detalles que definen el resto del proyecto

- **Perfil híbrido:** la tabla puente lleva `cedula_profesional` y `verificada_at`. Una profesional con Fitness + Nutrición tiene dos renglones y puede abrir bloques de ambas especialidades en la misma consulta.
- **Auditoría:** conecta el trigger aquí. Si funciona para esta entidad, funciona para todas.
- **Paginación, filtros y orden:** resuélvelos en `comun/` una vez.
- **Tests de API:** crea dos organizaciones y verifica que una no ve a las profesionales de la otra. Es la prueba que valida toda la arquitectura multi-tenant.

### 3.4 Verificación de fase

- [ ] CRUD completo funcionando contra `redsinosofica.localhost:8080`
- [ ] El límite de plan bloquea al llegar al máximo
- [ ] Test de aislamiento entre organizaciones en verde
- [ ] La auditoría registra creación, actualización y borrado lógico

---

## Fase 4 — Núcleo clínico: pacientes y ciclo

### 4.1 Entidades

- `Paciente` — identidad separada de lo clínico
- `AntecedentesGenerales` — 1 a 1, JSONB
- `AntecedentesGinecobstetricos` — 1 a 1, ámbito de acceso distinto
- `RegistroCiclo` — serie temporal

### 4.2 Por qué el ciclo es tabla propia

Es la decisión que diferencia este sistema de un expediente genérico. Casi todo correlaciona con la fase del ciclo: peso, retención de líquidos, rendimiento en fuerza, calidad de sueño, estado de ánimo, sensibilidad a la insulina, resultados hormonales.

Como serie temporal independiente, cualquier métrica de cualquier especialidad se cruza con ella:

```sql
SELECT cb.metricas_especialidad->>'press_banca_1rm', rc.fase
FROM consulta_bloques cb
JOIN consultas c ON c.id = cb.consulta_id
JOIN registros_ciclo rc
  ON rc.paciente_id = c.paciente_id AND rc.fecha = c.fecha::date;
```

Enterrada en un JSONB de ginecología, esa correlación queda inaccesible justo para quienes más la necesitan: la entrenadora que periodiza cargas y la psicóloga que distingue disforia premenstrual de un cuadro depresivo de base.

### 4.3 Cálculo de fase, en el núcleo

```python
# nucleo/ciclo/calculo_fase.py
def calcular_fase(registros: list[RegistroDia], fecha: date) -> Fase:
    """
    Puro. Maneja ciclos irregulares, amenorrea, anticoncepción continua
    y posmenopausia devolviendo INDETERMINADA o NO_APLICA en vez de
    inventar un día de ciclo.
    """
```

Ese "en vez de inventar" es el requisito clínico central: un sistema que asume ciclos de 28 días regulares le falla a una parte importante de las pacientes, y el error es silencioso.

### 4.4 Cifrado selectivo

Candidatos a `pgcrypto`: CURP, teléfono, domicilio. **No cifres el expediente completo** — una columna cifrada no se indexa ni se busca por contenido. Cifra tres o cuatro campos, no treinta.

---

## Fase 5 — Plantillas dinámicas y consultas

La parte con más lógica del sistema.

### 5.1 El patrón

**Definiciones relacionales, valores en JSONB.**

```
plantillas_formulario   → propiedad, especialidad, versión actual
plantilla_versiones     → snapshot inmutable de la definición
campos_definicion       → clave, tipo, unidad, validación, opciones
```

Crear un campo nuevo es un `INSERT`, no una migración. Y como las definiciones son relacionales, puedes validar la captura, renderizar el formulario y graficar series históricas de un campo inventado por la usuaria.

### 5.2 Versionado: la trampa que casi todos pisan

Cada bloque de consulta guarda `plantilla_version_id`. Si una nutrióloga edita su formulario hoy, las consultas de hace ocho meses siguen renderizándose e interpretándose con la definición que tenían. Sin esto, el expediente pierde trazabilidad.

**Regla:** una vez publicada una versión, `clave` y `tipo` de un campo **no se pueden cambiar**. Se crea campo nuevo y se deprecia el viejo. Cambiar un campo de texto a número no convierte las capturas anteriores.

### 5.3 Consulta como contenedor

```
consultas            → SOAP genérico, signos vitales, fase del ciclo
  └── consulta_bloques  → uno por especialidad
        ├── metricas_especialidad  (JSONB, validado contra el esquema fijo)
        └── campos_custom_usuario  (JSONB, validado contra campos_definicion)
```

Una sesión de Fitness + Nutrición genera dos bloques. Con un solo JSONB en `consultas`, las métricas quedan mezcladas, el consentimiento no puede operar por ámbito, y dos profesionales no pueden capturar en la misma sesión.

### 5.4 Alcance de la fase 1

**Sin constructor visual.** Las plantillas se siembran por migración, por especialidad. Las profesionales tienen formularios funcionando; simplemente no los editan ellas todavía. El motor completo ya está: falta solo la interfaz, que es trabajo de frontend y se agrega sin tocar el backend.

### 5.5 Validación del JSONB

Sin esto, en un año tienes basura con typos en las llaves.

- Validación en el núcleo contra `campos_definicion`, en cada escritura
- Adicionalmente, `CHECK` con `jsonb_matches_schema` (extensión `pg_jsonschema`)
- Índices GIN sobre ambas columnas JSONB

---

## Fase 6 — Consentimiento, RLS y auditoría

La fase más importante del proyecto en términos de riesgo.

### 6.1 Dos capas de aislamiento

| Capa | Qué separa | Mecanismo |
|---|---|---|
| Esquema | Una organización de otra | `search_path` (django-tenants) |
| RLS | Una profesional de otra, **dentro** de la misma organización | Políticas de Postgres |

La segunda es la que impide que la especialista en fitness lea las notas de psicología de la misma paciente.

### 6.2 RLS sin Supabase

RLS es de PostgreSQL, no de ningún proveedor. Middleware que fija la identidad por request:

```python
class RLSMiddleware:
    def __call__(self, request):
        if request.user.is_authenticated:
            with connection.cursor() as c:
                c.execute(
                    "SELECT set_config('app.profesional_id', %s, true)",
                    [str(request.user.profesional.id)],
                )
        return self.get_response(request)
```

```sql
CREATE POLICY lectura_bloques ON consulta_bloques FOR SELECT
USING (EXISTS (
  SELECT 1 FROM consentimientos_acceso ca
  JOIN consultas c     ON c.id = consulta_bloques.consulta_id
  JOIN especialidades e ON e.id = consulta_bloques.especialidad_id
  WHERE ca.paciente_id   = c.paciente_id
    AND ca.profesional_id = current_setting('app.profesional_id')::uuid
    AND ca.ambito  = e.ambito
    AND ca.estado  = 'activo'
    AND ca.puede_leer
    AND (ca.vigente_hasta IS NULL OR ca.vigente_hasta > now())
));
```

**Por qué vale el trabajo extra:** si un día alguien agrega un endpoint y olvida filtrar el queryset, la base lo detiene igual. La autorización aplicada solo en la capa de aplicación se fuga por el endpoint que nadie revisó.

Si usas pgbouncer, configúralo en modo `transaction` para que `set_config(..., true)` funcione correctamente.

### 6.3 Auditoría

- Triggers `AFTER` en todas las tablas clínicas
- Lecturas registradas desde la aplicación
- Registrar también los intentos denegados: son la mejor señal temprana de abuso interno
- Append-only: revocar `UPDATE` y `DELETE` a todos los roles
- La paciente debe poder ver quién consultó su expediente

### 6.4 Verificación de fase

- [ ] Test: profesional sin consentimiento de ámbito no ve el bloque, ni por API ni por SQL directo con su rol
- [ ] Test: revocar consentimiento corta el acceso de inmediato
- [ ] Los intentos denegados aparecen en `auditoria`
- [ ] Ninguna política de RLS depende de que la aplicación filtre bien

---

## Fase 7 — Frontend

### 7.1 Base

Vite + React + TypeScript + TanStack Query. Router con rutas protegidas. El cliente HTTP deriva la URL base del subdominio actual.

### 7.2 Pantallas de la etapa 1

1. Login (con MFA)
2. Agenda — vista semanal, crear y mover citas
3. Directorio de pacientes
4. Expediente — línea de tiempo de consultas, filtrable por especialidad
5. Captura de consulta — SOAP + bloques dinámicos por especialidad
6. Registro de ciclo
7. Administración — profesionales, plan, uso

### 7.3 El componente crítico

El renderizador de bloques: recibe `campos_definicion` y produce el formulario. Es el que hace que los campos personalizados funcionen sin tocar el backend, y el que reutilizas cuando llegue el constructor visual en la etapa 2.

### 7.4 Correo transaccional

Los enlaces de recuperación deben apuntar al subdominio correcto de cada organización. Genera las URLs desde el tenant actual, **nunca** desde un `SITE_URL` fijo en settings. Es un bug que solo aparece con el segundo cliente.

---

## Fase 8 — Endurecimiento y piloto

- [ ] MFA obligatoria para todas las profesionales (`django-otp`, TOTP)
- [ ] Rate limiting en login y recuperación de contraseña
- [ ] Cloudflare Access sobre `admin.` y `staging.`
- [ ] Sentry con filtrado de PII en los eventos
- [ ] Cabeceras de seguridad: HSTS, CSP, `X-Frame-Options`
- [ ] Sin PII en logs de aplicación
- [ ] Respaldo nocturno cifrado a R2
- [ ] **Restauración de prueba real**: bajar el dump, levantarlo en limpio, verificar datos
- [ ] Aviso de privacidad publicado y aceptado
- [ ] Piloto con dos o tres profesionales antes de abrir a toda la Red

Un respaldo que nunca se restauró no es un respaldo. Esta casilla no se palomea con el script escrito: se palomea con la restauración hecha.

---

## Fase X — VPS y staging (opcional, insertable en cualquier momento)

**Se puede ejecutar entre cualquier par de fases sin romper nada.** La propiedad que lo permite es la paridad de compose: tu entorno local ya usa la misma imagen, el mismo gunicorn y el mismo Postgres que producción. Desplegar es cambiar variables de entorno, no cambiar el proyecto.

### Recomendación de momento

Si vas a mostrarle avances a la Red desde temprano, ejecútala en la semana 1. El costo de tener el VPS prendido de más son unos €15 y es el seguro más barato del proyecto: las sorpresas de despliegue son de entorno, no de código, y descubrirlas el día tres es una tarde aburrida mientras que descubrirlas en la semana catorce es una crisis.

Si prefieres posponerla, el límite práctico es **antes de empezar la Fase 7**, para construir el SPA ya apuntando a un backend remoto.

### X.1 Servidor

```bash
# Hetzner CX/CAX 4 GB, Ubuntu 24.04
# ARM (CAX) funciona bien con Django y Postgres y sale más barato;
# verifica que tus imágenes tengan build arm64.

ssh-copy-id root@IP
# /etc/ssh/sshd_config: PasswordAuthentication no, PermitRootLogin no
ufw default deny incoming && ufw default allow outgoing && ufw enable
# No se abre ningún puerto entrante. Ni el 22.
# El acceso administrativo va por Cloudflare Tunnel también.
```

### X.2 Cloudflare

1. Registro DNS `*.tudominio.mx` apuntando al túnel
2. Tunnel con ingress wildcard:

```yaml
ingress:
  - hostname: "*.tudominio.mx"
    service: http://caddy:80
  - service: http_status:404
```

3. **Access por hostname, no sobre el wildcard.** Una política por organización, con su propia lista de correos. Una política sobre `*.tudominio.mx` dejaría que cualquier correo autorizado entre a cualquier organización.

**Limitaciones a conocer:**

- El SSL universal cubre `tudominio.mx` y `*.tudominio.mx`, **no** `*.*.tudominio.mx`. Subdominios de un solo nivel.
- Límite de 100 MB por request en plan gratuito. Para PDFs de laboratorio sobra; para imagenología no. Sube adjuntos pesados directo a R2 con URL firmada desde el navegador.
- Cloudflare termina el TLS y ve el tráfico descifrado. Es un subencargado y **debe estar declarado en el aviso de privacidad**. La contrapartida es cero puertos abiertos, que para un equipo sin personal de seguridad es el mejor intercambio de riesgo disponible.

### X.3 Despliegue

`git push` → GitHub Actions → SSH → `docker compose pull && docker compose up -d && migrate_schemas`.

Nada de Kubernetes ni ArgoCD.

### X.4 Regla del staging

**Staging nunca ve datos reales de pacientes.** Banner permanente, datos sembrados evidentemente falsos, y dicho en voz alta cada vez que se lo muestres a la Red. La trampa es una profesional entusiasmada que prueba con una paciente de verdad "nada más para ver cómo se siente", y de pronto hay expedientes clínicos en una base sin respaldos y con contraseña de demo. Access por hostname ayuda, pero no sustituye decirlo.

### X.5 Verificación de fase

- [ ] `staging.tudominio.mx` responde con TLS válido
- [ ] `ufw status` no muestra puertos entrantes abiertos
- [ ] Deploy por push funcionando
- [ ] Respaldo nocturno corriendo **y una restauración probada**

---

## Anexo: lo que NO entra en la etapa 1

No por falta de importancia, sino porque se agrega después sin tocar lo existente. El criterio no es "qué es importante" sino **qué es caro de retrofitear**.

| Diferido a etapas posteriores | Por qué se puede |
|---|---|
| Constructor visual de formularios | El motor ya está; falta solo la interfaz. |
| Portal de pacientes | App y endpoints nuevos, modelo intacto. |
| Recordatorios y notificaciones | Agregar Celery + Redis son dos horas. |
| Adjuntos y estudios | Tabla ya diseñada, R2 ya contratado. |
| Reportes y gráficas de correlación con ciclo | Los datos ya se están capturando desde el día uno. |
| Capa GraphQL | Se monta encima de los mismos modelos. |
| Cobranza automatizada | Con una organización, es construir un sistema de pagos para un cliente. |
| Instancias dedicadas por cliente | `pg_dump -n esquema` + otro compose. Previsto en el diseño. |

Y lo que **sí** entra desde el día uno aunque no se vea: UUID, `timestamptz`, soft delete, auditoría, consentimientos con RLS, `consulta_bloques`, versionado de plantillas, esquema por tenant. Todo eso es estructural: agregarlo después implica migrar datos o auditar cada endpoint.
