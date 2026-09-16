# Red Sinosófica

Sistema de gestión clínica multidisciplinaria para redes de profesionales de la salud enfocadas en atención integral de mujeres cis.

## Estado

**Fase 0** — Planificación y setup de herramientas.

- [x] Análisis de requerimientos
- [x] Decisiones de arquitectura
- [x] Diseño de base de datos
- [ ] Credenciales y dominios
- [ ] Aviso de privacidad (LFPDPPP)

## Documentación

- Reglas de arquitectura y desarrollo — invariantes del proyecto (config local, no versionada)
- **[`docs/plan.md`](./docs/plan.md)** — Plan completo de fases y entregables
- **[`docs/flujo-desarrollo.md`](./docs/flujo-desarrollo.md)** — Flujo de trabajo por tipo de tarea
- **[`docs/esquema.dbml`](./docs/esquema.dbml)** — Diseño de base de datos
- **[`docs/decisiones/`](./docs/decisiones/)** — ADRs (decisiones arquitectónicas registradas)
- **[`docs/specs/`](./docs/specs/)** — Especificaciones de features (generadas con `/interview-me`)
- **[`docs/roadmaps/`](./docs/roadmaps/)** — Roadmaps por feature

## Stack

| Área | Tecnología |
|---|---|
| Backend | Django 5.1 + DRF |
| Base de datos | PostgreSQL 16 |
| Multi-tenancy | `django-tenants` (esquema por organización) |
| Frontend | Vite + React + TypeScript + TanStack Query |
| Ruteo | Cloudflare Tunnel + subdominio por tenant |
| Archivos | Cloudflare R2 |
| Correo | Resend o Brevo |
| Infra | VPS Hetzner 4 GB, Docker Compose |

## Arquitectura

Monolito en capas con dominio aislado:

```
ADAPTADORES (DRF)  → serializers, viewsets, urls
NÚCLEO (py puro)   → reglas clínicas, tests sin BD
PERSISTENCIA       → models, migraciones, RLS
```

Multi-tenancy por esquema PostgreSQL:
- Base de datos única
- Esquema `public`: organizaciones, planes, dominios
- Un esquema `<slug>` por organización con todas las tablas de negocio
- Middleware fija `search_path` según Host

## Primeros pasos (Fase 1)

1. Clona el repo
2. Copia `.env.example` a `.env` y completa las credenciales
3. Corre `docker-compose up` (será en Fase 1)
4. Ejecuta migraciones y carga datos de prueba

## Reglas de desarrollo

Lee las reglas de arquitectura del proyecto (config local) antes de escribir cualquier línea.

**Modo por defecto:** didáctico + pedagógico. Ponytail activa en full.

## Contacto

Alonso — desarrollo backend, arquitectura del sistema.

---

*Red Sinosófica © 2026. Proyecto de sistema de salud integral.*
