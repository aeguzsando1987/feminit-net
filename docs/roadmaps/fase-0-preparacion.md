# Fase 0 — Preparación

Sin código. Cierra cuando el entorno local levanta contenedores y las reglas del
proyecto están sanjadas.

Plan maestro: Fases 0→6, alcance backend. Ver `docs/patron-entidad.md` y los ADR.

---

## Etapa 0.1 — Sanjar las reglas ✅

- [x] `CLAUDE.md`: filosofía al inicio, modos de entrega reescritos, 5 invariantes nuevas
- [x] `docs/decisiones/003-estructura-por-familias.md`
- [x] `docs/decisiones/004-modos-de-entrega.md`
- [x] `docs/decisiones/005-bitacora-dev-log.md`
- [x] `docs/patron-entidad.md` — esqueleto con los 8 pasos *(ejemplos de código pendientes, Fase 3)*
- [x] `red_sinosofica_schema.dbml` → `.OBSOLETO.dbml.bak` (difería de `docs/esquema.dbml`)

## Etapa 0.1b — Raíz de la etapa backend ✅

- [x] `feminist_net_back/dev-log/dev_log.csv` — bitácora append-only
- [x] `feminist_net_back/dev-log/README.md` — diccionario de columnas
- [x] `feminist_net_front/` **no** se crea hasta la Fase 7

## Etapa 0.2 — Entorno local 🔶 parcial

- [x] `uv` 0.12.16 instalado en `~/.local/bin`
- [x] Python 3.13.15 instalado por `uv`, en paralelo al 3.14.4 del sistema
- [x] Node 22.22.1 — ya presente, es LTS, sirve en vez del 20 previsto
- [x] git 2.53.0 — ya presente
- [x] Versiones decididas: **Python 3.13 + Django 5.2 LTS** (ADR 006)
- [x] Decisión de versionado: se versiona todo salvo `.claude/settings.local.json`
- [ ] **Docker 24+ y Compose v2** — bloqueado, requiere `sudo` con contraseña
- [ ] **Usuario en el grupo `docker`** — depende de lo anterior

### Comandos pendientes de la 0.2

Requieren contraseña, así que los ejecuta el usuario. Ubuntu 26.04 (`resolute`)
ya trae versiones suficientes en sus propios repositorios: Docker 29.1.3 y
Compose v2.40.3. **No hace falta agregar el repositorio de terceros de Docker**
ni su llave GPG, que es el camino largo que casi toda guía recomienda.

```bash
sudo apt update
sudo apt install -y docker.io docker-compose-v2 docker-buildx

# Poder usar docker sin sudo
sudo usermod -aG docker "$USER"

# WSL2: arrancar el servicio
sudo service docker start
```

Después de `usermod`, **cerrar la sesión de WSL y volver a entrar** (`exit` y
reabrir la terminal, o `wsl --shutdown` desde PowerShell). El grupo no se aplica
a una sesión ya abierta.

### Verificación de la 0.2

```bash
docker --version           # >= 24
docker compose version     # v2.x
docker run --rm hello-world
id -nG | tr ' ' '\n' | grep -qx docker && echo "grupo docker OK"
uv --version
uv python list --only-installed | grep 3.13
node --version
```

## Etapa 0.3 — Cuentas y dominio ⬜

- [ ] GitHub, Cloudflare, registrador de dominio, Resend/Brevo, Sentry
- [ ] Dominio registrado, nameservers en Cloudflare
- [ ] Formato de slug: minúsculas, `a-z0-9-`, 3–30 caracteres
- [ ] Reservar subdominios: `admin api www app staging dev mail static`

## Etapa 0.4 — Bloqueadores no técnicos ⬜

Arrancan ahora porque tardan semanas.

- [ ] Aviso de privacidad LFPDPPP: transferencia internacional (VPS fuera de
      México), Cloudflare como subencargado que ve tráfico descifrado, R2 como
      almacén
- [ ] Por escrito: ¿de quién es el expediente si una profesional deja la Red?
      ¿de quién es el código?

---

## Al retomar en otro equipo

```bash
git clone <remoto> && cd SINOSOFIC

# 1. Herramientas (uv no necesita sudo)
curl -LsSf https://astral.sh/uv/install.sh | sh
source "$HOME/.local/bin/env"
uv python install 3.13

# 2. Docker: ver "Comandos pendientes de la 0.2" arriba

# 3. Leer, en este orden
#    CLAUDE.md
#    docs/decisiones/00{1..6}-*.md
#    docs/patron-entidad.md
#    feminist_net_back/dev-log/dev_log.csv   ← el porqué de cada decisión
```

Siguiente paso real: terminar la 0.2 (Docker) y seguir con la 0.3.
