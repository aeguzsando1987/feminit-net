# Patrón de Entidad

> **Estado: esqueleto.** Los 8 pasos están decididos (ADR 003). Los ejemplos de
> código se llenan en la **Fase 3**, al construir `Especialidad` y `Profesional`.
> No se escriben antes: un patrón redactado sin haber construido la primera
> entidad es una suposición, no un patrón.

La receta que se repite en las ~17 entidades del esquema. Se define una sola vez,
despacio, y de ahí en adelante se copia.

---

## Archivos por entidad

```
apps/<familia>/<entidad>/
├── __init__.py
├── apps.py          # AppConfig con name completo: "apps.<familia>.<entidad>"
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

---

## Los 8 pasos

### 1. Leer el DBML

La tabla en `docs/esquema.dbml`, **incluidas sus `Note`**. Las notas contienen
reglas de negocio, no adorno: ahí está por qué `profesionales.activa` define el
consumo del plan, o por qué la unicidad del email es por esquema.

### 2. ¿Hay lógica clínica?

- **Sí** → la regla va a `nucleo/`, con test sin BD **antes** del modelo.
  Ver `docs/flujo-desarrollo.md` §"Regla de Dominio".
- **CRUD puro** → saltar al paso 3.

### 3. Modelo + migración

```
makemigrations <label>
migrate_schemas
```

> *Ejemplo pendiente — Fase 3.*

### 4. Test de aislamiento

Crear dos organizaciones, escribir en una, verificar que la otra no ve nada.
**Por SQL directo, no solo por ORM**: el ORM podría estar filtrando bien mientras
la base está abierta.

> *Ejemplo pendiente — Fase 3.*

### 5. Serializer

Validación en la frontera de confianza. Lo que entra por la API no se asume
correcto nunca.

> *Ejemplo pendiente — Fase 3.*

### 6. ViewSet + urls

Plano. Si el viewset pasa de ~150 líneas, lo que sobra es dominio disfrazado y
pertenece a `nucleo/`.

> *Ejemplo pendiente — Fase 3.*

### 7. Auditoría

Verificar que el trigger cubre la tabla nueva. No es opcional: la invariante dice
*auditoría conectada desde la primera entidad*.

> *Ejemplo pendiente — Fase 3.*

### 8. Revisión y cierre

1. `/ponytail-review` sobre el diff
2. Marcar atajos con `# ponytail:`
3. Renglón en `dev-log/dev_log.csv`
4. Commit en español

---

## Nivel de Ponytail

| Situación | Nivel |
|---|---|
| CRUD, serializers, viewsets | `full` |
| Lógica clínica nueva | `lite` |
| Fase 6 (RLS, consentimiento) | `lite` |
| Migraciones estructurales | `off` |

Ponytail recorta el código, jamás la explicación.

---

## Lo que NO lleva una entidad

- **Repositorio envolviendo el ORM.** El ORM ya es esa capa.
- **Fachada hacia otras familias.** Las FK cruzan familias directamente.
- **`services.py` por costumbre.** Solo con orquestación de efectos, y se
  justifica en el commit.
- **`organizacion_id`.** El aislamiento es por `search_path`.
