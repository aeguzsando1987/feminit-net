# ADR 003 — Estructura en Dos Etapas y Apps Agrupadas por Familia

**Estado:** Aceptada · Septiembre 2026

**Contexto**

El proyecto tiene dos etapas claramente separadas en el tiempo: backend (Fases 0–6) y frontend (Fase 7). `docs/plan_desarrollo_red_sinosofica_v2.md` proponía `backend/` y `frontend/` como hermanos, con `apps/` plano por dentro:

```
backend/apps/{usuarios,profesionales,pacientes,agenda,consultas,plantillas,privacidad}/
```

Dos problemas con ese árbol plano. Primero, el esquema de `docs/esquema.dbml` tiene ~17 entidades de tenant, no 7: al repartirlas quedan apps de Django con tres y cuatro modelos sin relación entre sí (`privacidad/` cargando consentimientos y auditoría, `consultas/` cargando citas, consultas y bloques). Segundo, no hay un lugar evidente donde poner lo que pertenece a una entidad concreta, así que los archivos de una entidad se dispersan entre los de otra.

**Alternativas Evaluadas**

1. **Árbol plano del v2** (descartada)
   - Apps con múltiples entidades no relacionadas
   - Agregar la entidad 18 obliga a decidir en qué app cabe: es una decisión de diseño nueva cada vez
   - Lo que se busca es que sea mecánico

2. **Monolito modular estricto: un módulo por dominio con fachada** (descartada)
   - Contradice la invariante de `CLAUDE.md`
   - Se paga con varios equipos en paralelo o con intención de extraer a servicios. Ninguna aplica
   - Renunciar a las FK de Postgres en datos clínicos es una pérdida neta: el expediente es intrínsecamente relacional

3. **Una carpeta por entidad, agrupadas en familias (ELEGIDA ✓)**
   - Todo lo de una entidad vive junto: modelo, serializer, viewset, urls, tests
   - La familia agrupa entidades afines sin crear frontera arquitectónica
   - Agregar una entidad es copiar el patrón, no diseñar

**Decisión**

Raíz del repositorio, una carpeta por etapa:

```
SINOSOFIC/
├── CLAUDE.md                  # reglas, comunes a ambas etapas
├── docs/                      # esquema, ADR, roadmaps — describen el SISTEMA
├── feminist_net_back/         # etapa 1
└── feminist_net_front/        # etapa 2, no se crea hasta la Fase 7
```

`docs/` y `CLAUDE.md` quedan en la raíz porque describen el sistema completo, no una etapa. El `dev-log/` sí es por etapa (ver ADR 005).

Dentro del backend, apps agrupadas en cuatro familias:

```
feminist_net_back/apps/
├── identidad/    {usuarios, especialidades, profesionales}
├── expediente/   {pacientes, antecedentes, ciclo}
├── atencion/     {agenda, plantillas, consultas}
└── privacidad/   {consentimientos, auditoria}
```

Las carpetas de familia son **paquetes Python vacíos**, no apps de Django: solo `__init__.py`. Cada app anidada declara su ruta completa:

```python
# apps/expediente/pacientes/apps.py
from django.apps import AppConfig

class PacientesConfig(AppConfig):
    name = "apps.expediente.pacientes"   # label por defecto: "pacientes"
```

Los labels resultantes son únicos en todo el proyecto, así que `TENANT_APPS` sigue siendo legible y las FK se escriben `"pacientes.Paciente"` como siempre.

**La familia NO es una frontera arquitectónica.** `apps/atencion/consultas/models.py` importa directamente `apps/expediente/pacientes/models.py` con FK real. No hay fachadas ni eventos entre familias: eso sería el monolito modular estricto que la invariante rechaza. La familia es organización de archivos y nada más.

**Consecuencias**

*Positivas:*
- Agregar una entidad es mecánico: copiar `docs/patron-entidad.md`
- Todo lo de una entidad está en una carpeta; nada que buscar en cuatro lugares
- Las dos etapas no se contaminan: cada una tiene su bitácora y sus dependencias
- Borrar una entidad es borrar una carpeta

*Negativas:*
- `AppConfig.name` explícito es obligatorio en cada app. Olvidarlo produce un error de arranque confuso
- Rutas de import más largas
- Se aparta del árbol del v2: ese documento queda superado en este punto

**Supera**

- `docs/plan_desarrollo_red_sinosofica_v2.md` §5 "Estructura": el árbol plano de `apps/`
- ADR 002 §"Cambios en Procedimiento de Desarrollo": donde dice `apps/usuarios/models.py`, léase `apps/identidad/usuarios/models.py`. La decisión del UUID no cambia; solo la ruta
- `docs/flujo-desarrollo.md`: donde dice `backend/nucleo/`, léase `feminist_net_back/nucleo/`

**Referencias**

- ADR 001: Multi-tenancy por esquema
- ADR 004: Modos de entrega
- ADR 005: Bitácora de desarrollo
- `docs/patron-entidad.md`
- [Django AppConfig.name](https://docs.djangoproject.com/en/5.1/ref/applications/#django.apps.AppConfig.name)
