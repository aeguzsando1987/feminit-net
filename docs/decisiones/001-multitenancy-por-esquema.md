# ADR 001 — Multi-tenancy por Esquema PostgreSQL

**Estado:** Aceptada · Septiembre 2026

**Contexto**

El sistema necesita servir a múltiples organizaciones de forma simultánea, con aislamiento garantizado de datos. Una organización hoy (Red Sinosófica), N después.

Además, debe permitir que un cliente crítico escale a una instancia VPS dedicada si lo requiere, sin cambiar el código de la aplicación — solo migrando datos.

**Alternativas Evaluadas**

1. **Columna `organizacion_id` en cada tabla** (descartada)
   - Simple, pero depende de recordar filtrar cada query
   - Un bug en un queryset -> fuga de datos entre orgs
   - Imposible garantizar aislamiento a nivel de BD
   - Migraciones a VPS dedicado requieren limpiar esa columna en 100+ tablas

2. **Instancia de Django + BD por cliente** (descartada)
   - Seguridad garantizada (procesos aislados)
   - Aber: N desarrolladores, N deployments, N backups, N monitoreos
   - Costo de infraestructura lineal con cantidad de clientes
   - No viable para un solo desarrollador

3. **Esquema PostgreSQL por organización** (ELEGIDA ✓)
   - `django-tenants` abstrae la complejidad
   - Aislamiento a nivel de BD: un query no puede salir del esquema
   - Migración a VPS: `pg_dump -n slug > org.sql` — dos comandos
   - Rendimiento: idéntico al de columna `organizacion_id`
   - Escalable: 1 o 100 clientes sin cambiar arquitectura

**Decisión**

Usar `django-tenants` para manejar multi-tenancy por esquema PostgreSQL.

- Base de datos única, servidor único (Hetzner 4 GB)
- Esquema `public`: planes, organizaciones, dominios
- Un esquema `<slug>` por organización con todas las tablas de negocio
- Middleware fija `connection.schema_name` según `Host`
- Migraciones: `makemigrations` → `migrate_schemas`

**Consecuencias**

*Positivas:*
- Aislamiento garantizado. Un bug en un queryset no puede fugar datos.
- Fácil de respaldar: `pg_dump -n org > backup.sql`
- Fácil de restaurar en otro servidor
- Escalable: agregar un cliente = un comando
- Cumple LFPDPPP: datos de una org nunca entran en otra sesión de BD

*Negativas:*
- Complejidad inicial: entender `search_path` y `TenantMixin`
- Las tablas de tenant NO llevan `organizacion_id`
- Autenticación requiere UUID en PK (no autoincremental)
- Tests necesitan crear dos orgs para verificar aislamiento

**Implicaciones Arquitectónicas**

1. `AUTH_USER_MODEL` propio con PK UUID (no autoincremental)
2. JWT lleva claim `tenant`, validado contra `Host`
3. Middleware: `TenantMainMiddleware` antes que `SessionMiddleware`
4. Tablas de tenant: sin columna `organizacion_id`, aislamiento por `search_path`
5. Herencias en BD: usar `multi_table` en Django, ambos in schema (automático)
6. Comandos: crear_organizacion (nunca manual)

**Blindajes**

- Test de aislamiento: leer como ORG1, verificar que ORG2 no ve nada
- Test de token cruzado: JWT de ORG1 en Host de ORG2 → rechazado
- Migraciones estructurales verificadas en ambos esquemas

**Referencias**

- [django-tenants documentation](https://django-tenants.readthedocs.io/)
- [PostgreSQL Schemas](https://www.postgresql.org/docs/16/ddl-schemas.html)
- Plan de desarrollo: `docs/plan.md`
- Arquitectura: `CLAUDE.md`

---

*Aprobado por: Alonso, arquitecto del sistema*
*Fecha: Septiembre 2026*
