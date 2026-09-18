# ADR 004 — Modos de Entrega: Didáctico y Pedagógico Conmutables

**Estado:** Aceptada · Septiembre 2026

**Contexto**

El proyecto tiene un objetivo doble, declarado en la **Filosofía** de `CLAUDE.md`: *desarrollar y guiar al usuario para implementar el sistema*. No basta con que el sistema corra; el usuario debe quedar capaz de construirlo.

La versión original de `CLAUDE.md` decía *"Modo por defecto: didáctico + pedagógico"*, y definía el pedagógico como "no modifico archivos; doy ruta, bloque y línea, y espero confirmación". Aplicado como estado permanente, ese modo tiene un costo alto y mal repartido: el usuario teclea a mano el `Dockerfile`, el `.gitignore` y diecisiete `__init__.py` vacíos — trabajo que no enseña nada — y llega cansado a lo que sí enseña, que son los modelos, el núcleo y las políticas RLS.

**Alternativas Evaluadas**

1. **Pedagógico permanente** (descartada)
   - Máximo aprendizaje en teoría
   - En la práctica gasta la atención del usuario en andamiaje inerte
   - El proyecto tiene ~17 entidades: el ritmo importa

2. **Didáctico permanente, sin pedagógico** (descartada)
   - Rápido, pero el usuario nunca teclea el código
   - Leer código y escribirlo no producen el mismo aprendizaje
   - Quita la mitad "guiar para implementar" de la filosofía

3. **Modos conmutables con defecto explícito (ELEGIDA ✓)**
   - Didáctico ON siempre: la explicación nunca se negocia
   - Pedagógico bajo demanda: el usuario decide dónde quiere teclear
   - El control del ritmo queda en quien está aprendiendo

**Decisión**

Tres modos independientes, con estado persistente hasta que el usuario lo cambie:

| Modo | Defecto | Activar | Desactivar |
|---|---|---|---|
| Didáctico | **ON** | `/didactico on` | `/didactico off` |
| Pedagógico | **OFF** | `/pedagogico on` | `/pedagogico off` |
| Verboso | **OFF** | `/verboso on` | `/verboso off` |

**Didáctico:** explicar el flujo lógico y el porqué arquitectónico antes de escribir código. Cuando Ponytail recorta algo, explicar qué se eliminó y por qué no hacía falta.

**Pedagógico:** no modificar archivos; entregar ruta completa, bloque exacto y línea de inserción, y esperar confirmación por paso. **Al activarlo se advierte el costo antes de aplicarlo** y se espera el sí — de lo contrario el usuario lo enciende sin saber que acaba de duplicar su trabajo.

**Verboso:** comentarios en el código explicando la lógica de negocio, nunca lo obvio de la sintaxis.

**Regla de equilibrio con Ponytail:** Ponytail recorta el **código**, jamás la **explicación**. Un recorte no explicado no es simplicidad, es pérdida de información.

**Punto de detención:** al cierre de cada **etapa**, no de cada fase. Una fase puede tomar días; esperar hasta su final para verificar el rumbo es descubrir tarde los errores.

**Consecuencias**

*Positivas:*
- El usuario controla el ritmo sin renegociarlo cada vez
- La explicación está garantizada en todo momento
- Las etapas de andamiaje avanzan rápido; las de aprendizaje real pueden ir lentas

*Negativas:*
- Los comandos no son slash commands reales del harness: son convención de este proyecto. Si se quieren descubribles, hay que crear skills en `.claude/skills/` (no se hace ahora: YAGNI)
- El estado de los modos vive en la conversación, no en disco. Tras un `/clear` vuelven al defecto — que es precisamente por qué el defecto está escrito en `CLAUDE.md`

**Supera**

- `CLAUDE.md` §"Documentación y entrega", que declaraba ambos modos activos por defecto

**Referencias**

- `CLAUDE.md` §Filosofía y §Modos de entrega
- ADR 005: Bitácora de desarrollo (la columna `porque` cumple la misma función que el modo didáctico, pero persistida)
