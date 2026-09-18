# Setup Simple — Red Sinosófica

Pasos rápidos para estructurar el proyecto.

---

## Paso 1: Descargar Archivos (14 total)

Descarga estos 14 archivos de la salida anterior.

**Raíz (6):** CLAUDE.md, README.md, SETUP.md, PROMPT_SETUP.md, .gitignore, .env.example

**docs/ (5):** plan.md, plan_desarrollo_red_sinosofica_v2.md, flujo-desarrollo.md, guia_tecnica_red_sinosofica.md, red_sinosofica_schema_v2.dbml

**docs/decisiones/ (2):** 001-multitenancy-por-esquema.md, 002-auth-user-model-uuid.md

**Respaldo (ignorar):** red_sinosofica_schema.dbml

---

## Paso 2: Crear Carpetas

```bash
cd redsinosofica
mkdir -p docs/decisiones
mkdir -p .claude
```

---

## Paso 3: Mover Archivos

Suponiendo que descargaste en `~/Downloads`:

```bash
# Raíz
cp ~/Downloads/CLAUDE.md .
cp ~/Downloads/README.md .
cp ~/Downloads/SETUP.md .
cp ~/Downloads/PROMPT_SETUP.md .
cp ~/Downloads/.gitignore .
cp ~/Downloads/.env.example .

# docs/
cp ~/Downloads/plan.md docs/
cp ~/Downloads/plan_desarrollo_red_sinosofica_v2.md docs/
cp ~/Downloads/flujo-desarrollo.md docs/
cp ~/Downloads/guia_tecnica_red_sinosofica.md docs/
cp ~/Downloads/red_sinosofica_schema_v2.dbml docs/esquema.dbml

# docs/decisiones/
cp ~/Downloads/001-multitenancy-por-esquema.md docs/decisiones/
cp ~/Downloads/002-auth-user-model-uuid.md docs/decisiones/
```

---

## Paso 4: Git

```bash
git init
git config user.name "Alonso"
git config user.email "tu-email@tudominio.mx"
git add .
git commit -m "Fase 0: documentación y planificación"
```

---

## Paso 5: Verificar

```bash
tree -L 2
git log --oneline
```

Deberías ver 13 archivos en su lugar + 1 commit.

---

Listo para Fase 1. ✅
