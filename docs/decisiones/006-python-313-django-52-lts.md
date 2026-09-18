# ADR 006 — Python 3.13 y Django 5.2 LTS

**Estado:** Aceptada · Septiembre 2026

**Contexto**

`docs/plan_desarrollo_red_sinosofica_v2.md` §2 fijaba **Django 5.1 + Python 3.12** como decisión cerrada. Al preparar el entorno local (Fase 0.2) esa combinación resultó inviable por dos motivos, ninguno previsible cuando se escribió el plan:

1. **Django 5.1 salió de soporte.** No recibe parches de seguridad y ya no aparece en la matriz de compatibilidad de `django-tenants`.
2. **El equipo local trae Python 3.14.4** (Ubuntu 26.04 LTS). Django 5.1 nunca soportó 3.14.

Estado real de las dependencias al momento de decidir, consultado en PyPI:

```
django          6.1.1     Python 3.12, 3.13, 3.14     requires_python >=3.12
django-tenants  3.14.0    Python 3.10 … 3.13          Django 5.2, 6.0, 6.1
```

El dato decisivo: **`django-tenants` no declara soporte para Python 3.14.** No es una dependencia cualquiera — es el eje del que cuelga el multi-tenancy por esquema, que es la invariante fundacional del sistema. Si se rompe, no hay proyecto.

**Regla que se deriva:** cuando la dependencia crítica es un proyecto pequeño que va detrás del core, manda su matriz de compatibilidad, no la del framework. Django soporta 3.14; da igual, porque `django-tenants` no.

La intersección de ambas matrices es **Python 3.13**.

**Alternativas Evaluadas**

1. **Python 3.14 + Django 6.1** (descartada)
   - Lo más nuevo de ambos lados
   - `django-tenants` no declara soporte para 3.14: apostar el multi-tenancy a que funcione "aunque no lo diga" es apostar la invariante fundacional
   - Un fallo aquí no se manifiesta como error de import, sino como comportamiento sutil en el manejo de `search_path`

2. **Python 3.13 + Django 6.1** (descartada)
   - Soportada por ambos según clasificadores
   - No es LTS: ventana de soporte corta, con presión de actualizar cada pocos meses sobre un sistema clínico en producción
   - Menos rodaje con `django-tenants`

3. **Python 3.13 + Django 6.0** (descartada)
   - Punto intermedio sin ventaja propia
   - Toma la presión de actualización sin ganar el soporte extendido

4. **Python 3.13 + Django 5.2 LTS (ELEGIDA ✓)**
   - LTS: parches de seguridad por años sin forzar migración
   - La combinación con más rodaje en producción junto a `django-tenants`
   - Todo el ecosistema previsto (DRF, `django-otp`, `pgcrypto`, `pg_jsonschema`) está probado ahí

**Decisión**

- **Python 3.13** — gestionado por `uv`, no por el sistema
- **Django 5.2 LTS**
- `django-tenants` 3.14.x

**Por qué LTS en este proyecto concreto:** el sistema maneja datos clínicos sensibles bajo LFPDPPP y tiene vida útil de años. En ese contexto, **poder NO actualizar es una función, no una carencia**. Cada actualización mayor de framework sobre un sistema en producción con expedientes reales es una ventana de riesgo. LTS compra el derecho a elegir cuándo abrirla.

**El Python del sistema no se toca.** Ubuntu 26.04 trae 3.14.4 y varias herramientas del sistema dependen de él; degradarlo rompería la distribución. `uv` instala y gestiona el 3.13 del proyecto en paralelo, aislado. Lo mismo que hace el contenedor en producción, pero para que el editor encuentre las dependencias al escribir código.

**Consecuencias**

*Positivas:*
- Soporte extendido sin presión de upgrade
- La dependencia crítica opera dentro de su matriz declarada
- El Python del sistema queda intacto
- Paridad: el contenedor también fija 3.13

*Negativas:*
- Sin las novedades de Django 6.x
- Habrá que planear la migración a la siguiente LTS antes de que expire el soporte de la 5.2
- `docs/plan_desarrollo_red_sinosofica_v2.md` §2 queda desactualizado en dos renglones de su tabla de decisiones cerradas

**Supera**

- `docs/plan_desarrollo_red_sinosofica_v2.md` §2 "Decisiones cerradas": donde dice `Django 5.1 + DRF`, léase **Django 5.2 LTS + DRF**
- `docs/flujo-desarrollo.md` §"Herramientas y Setup": donde dice `Python 3.12+`, léase **Python 3.13**
- El mismo documento describe el setup con `python -m venv` + `pip install -r requirements.txt`. Se usa `uv` en su lugar

**Criterio para reabrir**

Cuando `django-tenants` declare soporte para una Python más nueva **y** exista una LTS de Django posterior a la 5.2. Antes no: no hay motivo nuevo.

**Referencias**

- [Django supported versions](https://www.djangoproject.com/download/#supported-versions)
- [django-tenants en PyPI](https://pypi.org/project/django-tenants/)
- ADR 001: Multi-tenancy por esquema
- `docs/plan_desarrollo_red_sinosofica_v2.md` §2
