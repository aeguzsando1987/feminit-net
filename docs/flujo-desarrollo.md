# Flujo de Desarrollo — Red Sinosófica

Cómo trabajar en diferentes tipos de tarea, con checkpoints y revisiones.

---

## Regla General

Por cada feature nueva:

1. **Lee** `docs/plan.md` y `CLAUDE.md`
2. **Crea o actualiza** `docs/roadmaps/<feature>.md` (lista de tareas con `[ ]`)
3. **Crea o lee** `docs/specs/<tema>.md` si hay ambigüedad (generado con `/interview-me`)
4. **Sigue el flujo** según el tipo de tarea (abajo)
5. **Cosecha deuda** al cierre con `/ponytail-debt`
6. **Commit** en español, sin autoría de IA

---

## Flujo: Entidad Nueva de Tenant

Para una tabla nueva que pertenece a un esquema de tenant.

```
┌─────────────────────────────────────────┐
│ 1. ¿Es un CRUD puro? Sáltate la spec.  │
│    ¿Tiene lógica clínica? Escribe spec.│
└─────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────┐
│ 2. Modelo + migración                   │
│    $ python manage.py makemigrations    │
│    $ python manage.py migrate_schemas   │
└─────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────┐
│ 3. Test de aislamiento entre dos orgs   │
│    Lee datos como ORG1, verifica que    │
│    ORG2 no ve nada. Test SQL directo.   │
└─────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────┐
│ 4. Serializer, viewset, urls            │
│    ModelViewSet plano. Sin lógica.      │
└─────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────┐
│ 5. /ponytail-review                     │
│    Revisar sobre-ingeniería.            │
│    Marcar atajos con ponytail:          │
└─────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────┐
│ 6. /review [commit si aplica]           │
│    Revisión manual antes de merge.      │
└─────────────────────────────────────────┘
```

**Commits:**
1. Models + migraciones
2. Tests de aislamiento
3. Serializer + viewset
4. Registrar atajos en /ponytail-debt

---

## Flujo: Regla de Dominio (Núcleo)

Para lógica que vive en `backend/nucleo/` (sin Django).

```
┌─────────────────────────────────────────┐
│ 1. Test primero. SIN base de datos.     │
│    Import desde nucleo/, no models.     │
│    Ej: from nucleo.ciclo import fase   │
└─────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────┐
│ 2. Implementar función / clase          │
│    Dataclass, métodos puros.            │
│    Ningún import de Django.             │
└─────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────┐
│ 3. Test pasa (sin BD).                  │
│    Ejecuta: pytest backend/nucleo/      │
└─────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────┐
│ 4. Adaptador Django                     │
│    View/serializer llama la regla.      │
│    Traduce entrada/salida.              │
└─────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────┐
│ 5. Test de integración (opcional)       │
│    Si el adaptador toca BD.             │
└─────────────────────────────────────────┘
```

**Commits:**
1. Función + test unitario
2. Adaptador Django
3. Tests de integración (si aplica)

---

## Flujo: Cambio que Toca Privacidad/RLS

**⚠️ Revisión manual obligatoria. No aceptar de agente sin leer línea por línea.**

```
┌─────────────────────────────────────────┐
│ 1. Escribe la política RLS primero.     │
│    CREATE POLICY ... con rol específico │
│    ANTES del endpoint.                  │
└─────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────┐
│ 2. Test de denegación (SQL directo)     │
│    Ej: SET ROLE otra_profesional_id     │
│    SELECT paciente_id FROM consultas    │
│    → debe devolver 0 filas.             │
└─────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────┐
│ 3. Implementar endpoint                 │
│    Middleware fija app.profesional_id   │
│    Serializer verifica consentimiento   │
└─────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────┐
│ 4. Test de integración                  │
│    Leyendo como profesional A           │
│    de un paciente de profesional B      │
│    → debe devolver 403 o 0 resultados   │
└─────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────┐
│ 5. Documentación de la política         │
│    Qué permite, a quién, por qué.       │
│    Registrar en CHANGELOG.md            │
└─────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────┐
│ 6. /review (MANUAL, línea por línea)    │
│    No aceptar cambios recomendados por  │
│    agente sin leer el SQL.              │
└─────────────────────────────────────────┘
```

**Commits:**
1. Política RLS + tests SQL
2. Modelo (si es nuevo)
3. Endpoint + serializer
4. Tests de integración
5. Documentación

---

## Flujo: Requisito Ambiguo

Cuando no está claro qué se pide.

```
┌─────────────────────────────────────────┐
│ 1. /interview-me                        │
│    Conversa, pregunta casos de uso      │
│    Escribe docs/specs/<tema>.md         │
└─────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────┐
│ 2. /clear                               │
│    Limpia contexto.                     │
└─────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────┐
│ 3. Modo plan sobre specs/<tema>.md      │
│    "Basándote en lo que escribimos en   │
│    specs/requisito.md, diseña el...     │
│    Divide en pasos."                    │
└─────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────┐
│ 4. Detente tras paso 3 y espera         │
│    confirmación antes de continuar.     │
└─────────────────────────────────────────┘
```

---

## Ponytail: Cuándo Cada Nivel

| Situación | Nivel | Razón |
|---|---|---|
| CRUD, serializers, viewsets | `/ponytail full` | Máxima compresión sin riesgo |
| Lógica clínica nueva | `/ponytail lite` | Conservador, no eliminar reglas |
| Fase 6 (RLS, consentimiento) | `/ponytail lite` | Blindado: seguridad primero |
| Migraciones estructurales | `/ponytail off` | Levantar schema es crítico |

**Cambiar nivel en medio de una tarea:**

```
/ponytail lite
[implementar RLS]
/ponytail full
[serializers después]
```

---

## Deuda Técnica y Atajos

Marcación estándar en el código:

```python
# ponytail: temporal, resolver cuando X esté completo
# Contexto: Y fue rápido pero frágil para MVP.
# Prioridad: media
```

Cosecha automática:

```bash
/ponytail-debt
```

Devuelve un registro en `CHANGELOG.md` con:
- Ubicación (archivo + línea)
- Descripción del atajo
- Por qué es temporal
- Prioridad
- Estimación de esfuerzo

**Política:** mínimo una cosecha por Fase.

---

## Checklist Pre-Commit

Antes de hacer `git commit`:

- [ ] Tests pasan (`pytest`)
- [ ] CI pasa (no hay import de Django en `nucleo/`)
- [ ] Formato (`black`, `ruff`)
- [ ] Migraciones nombradas bien (`0001_initial.py`, `0002_add_field.py`)
- [ ] Mensaje en español, describiendo la acción técnica
- [ ] Sin `Co-authored-by` ni firma de IA
- [ ] Archivos modificados documentados (si hay cambio de arquitectura)

---

## Revisión Manual (/review)

Qué buscar en el diff:

1. **¿Hay un bug funcional visible?** Señalar antes de documentar.
2. **¿Se viola una invariante de CLAUDE.md?** Rechazar.
3. **¿RLS es nuevo y el SQL da miedo?** Leer línea por línea.
4. **¿Over-engineering claro?** Proponer /ponytail-review.
5. **¿Está blindada la frontera de confianza?** (Validación, error handling, autenticación)

No busques:
- Style (lo hace `ruff`)
- Typos (lo hace el editor)
- Comentarios faltantes (ponytail no comenta)

---

## Flujo de Deploy (después de Fase 8)

```
Viernes 18:00 UTC
↓
1. Backup automático a R2
2. Tests de smoke (health, login, CRUD básico)
3. Deploy a producción (docker-compose up -d)
4. Rollback preparado (compose down && restore from backup)
5. Monitoreo Sentry + logs en tiempo real
6. On-call durante fin de semana
```

**Rollback = 5 minutos.**

---

## Herramientas y Setup

- **Python 3.12+** (venv local)
- **PostgreSQL 16** en Docker
- **Docker Compose** (dev ≈ prod)
- **pytest** (tests)
- **black + ruff** (formato)
- **git** (VCS)
- **Ponytail** (optimización)

Instalación primera sesión:

```bash
git clone <repo>
cd redsinosofica
python -m venv .venv
source .venv/bin/activate
pip install -r backend/requirements.txt
cp .env.example .env
# Editar .env con credenciales locales
docker-compose up -d  # PostgreSQL local
cd backend
python manage.py migrate_schemas
pytest
```

---

## Emergencias

Si RLS está rota:

```bash
# Aislar
git revert <commit>
docker-compose up -d
python manage.py migrate_schemas
# Verificar
SELECT * FROM consultas WHERE profesional_id = otra_prof;
# Debe devolver 0 resultados
```

Si la migración falla:

```bash
docker-compose down
docker-compose up -d
python manage.py migrate_schemas --fake <app> 0001
# Fijar la migración en VCS
# Notificar
```

Si hay una fuga de datos:

1. Cierra la conexión de esa organización (middleware rechaza Host)
2. Toma backup
3. Investiga en log de auditoría
4. Notifica a la organización
5. Restaura si es necesario

---

*Última actualización: 2026-09*
*Mantener actualizado cada Fase.*
