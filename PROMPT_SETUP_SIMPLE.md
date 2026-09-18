# Prompt Simple para Estructurar Red Sinosófica

Copia esto y pégalo en Claude Code.

---

## Prompt

```
Estoy iniciando "Red Sinosófica". He descargado 14 archivos en mi carpeta de descargas.

**Tarea:**
1. Crea las carpetas: docs/decisiones y .claude/
2. Mueve los archivos a su lugar:
   - Raíz: CLAUDE.md, README.md, SETUP.md, PROMPT_SETUP.md, .gitignore, .env.example
   - docs/: plan.md, plan_desarrollo_red_sinosofica_v2.md, flujo-desarrollo.md, guia_tecnica_red_sinosofica.md
   - docs/: red_sinosofica_schema_v2.dbml → renombra a esquema.dbml
   - docs/decisiones/: 001-multitenancy-por-esquema.md, 002-auth-user-model-uuid.md
   - Ignora: red_sinosofica_schema.dbml (respaldo)

3. Git:
   - git init
   - git config user.name "Alonso"
   - git config user.email "tu-email@tudominio.mx"
   - git add .
   - git commit -m "Fase 0: documentación y planificación"

4. Muestra:
   - Árbol de carpetas (tree -L 2)
   - git log --oneline

**Estructura final:**
```
redsinosofica/
├── CLAUDE.md
├── README.md
├── SETUP.md
├── PROMPT_SETUP.md
├── .gitignore
├── .env.example
├── .claude/
├── docs/
│   ├── plan.md
│   ├── plan_desarrollo_red_sinosofica_v2.md
│   ├── flujo-desarrollo.md
│   ├── guia_tecnica_red_sinosofica.md
│   ├── esquema.dbml
│   └── decisiones/
│       ├── 001-multitenancy-por-esquema.md
│       └── 002-auth-user-model-uuid.md
```

Sé pedagógico: explica cada paso antes de hacerlo.
```

---

## Alternativa: Manual (Línea de Comandos)

```bash
# Crear carpetas
mkdir -p docs/decisiones .claude

# Mover archivos (ajusta ~/Downloads según donde descargaste)
cp ~/Downloads/CLAUDE.md .
cp ~/Downloads/README.md .
cp ~/Downloads/SETUP.md .
cp ~/Downloads/PROMPT_SETUP.md .
cp ~/Downloads/.gitignore .
cp ~/Downloads/.env.example .
cp ~/Downloads/plan.md docs/
cp ~/Downloads/plan_desarrollo_red_sinosofica_v2.md docs/
cp ~/Downloads/flujo-desarrollo.md docs/
cp ~/Downloads/guia_tecnica_red_sinosofica.md docs/
cp ~/Downloads/red_sinosofica_schema_v2.dbml docs/esquema.dbml
cp ~/Downloads/001-multitenancy-por-esquema.md docs/decisiones/
cp ~/Downloads/002-auth-user-model-uuid.md docs/decisiones/

# Git
git init
git config user.name "Alonso"
git config user.email "tu-email@tudominio.mx"
git add .
git commit -m "Fase 0: documentación y planificación"

# Verificar
tree -L 2
git log --oneline
```

---

Eso es. Descarga, copia el prompt o ejecuta los comandos, y listo. ✅
