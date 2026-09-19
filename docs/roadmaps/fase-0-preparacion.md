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

## Etapa 0.2 — Entorno local ✅

- [x] `uv` 0.12.16 instalado en `~/.local/bin`
- [x] Python 3.13.15 instalado por `uv`, en paralelo al 3.14.4 del sistema
- [x] Node 22.22.1 — ya presente, es LTS, sirve en vez del 20 previsto
- [x] git 2.53.0 — ya presente
- [x] Versiones decididas: **Python 3.13 + Django 5.2 LTS** (ADR 006)
- [x] Decisión de versionado: se versiona todo salvo `.claude/settings.local.json`
- [x] **Docker 29.8.0 y Compose v5.5.1** — vía Docker Desktop con integración WSL
- [x] **Usuario en el grupo `docker`** — lo agregó Docker Desktop
- [x] Imagen `postgres:16-alpine` descargada

### Cómo quedó Docker en este equipo

Se usó **Docker Desktop en Windows con integración WSL** activada para la distro
`Ubuntu`. El motor corre en Linux (VM WSL2); solo la gestión es de Windows.

> Docker Desktop → Settings → Resources → WSL Integration
> → activar el interruptor general y marcar `Ubuntu` → Apply & Restart

**Trampa:** la pertenencia a grupos se resuelve al crear la sesión. Docker
Desktop agrega el usuario al grupo `docker`, pero una sesión de WSL ya abierta
sigue con la lista vieja y el socket responde `permission denied`. Hay que hacer
`wsl --shutdown` desde PowerShell y reabrir; cerrar la ventana no basta, porque
WSL mantiene la distro viva en segundo plano.

### Alternativa nativa, si en el otro equipo no hay Docker Desktop

Ubuntu 26.04 (`resolute`) trae versiones suficientes en sus propios
repositorios. **No hace falta el repositorio de terceros de Docker** ni su llave
GPG, que es el camino largo que casi toda guía recomienda.

```bash
sudo apt update
sudo apt install -y docker.io docker-compose-v2 docker-buildx
sudo usermod -aG docker "$USER"
sudo systemctl enable --now docker   # este equipo tiene systemd=true en /etc/wsl.conf
```

Luego `wsl --shutdown` y reabrir, por la misma razón de arriba.

Ventaja de la nativa: los contenedores corren en la **misma** distro, así que los
bind mounts de la Etapa 1.6 (recarga en caliente) no cruzan entre distros.
No mezclar ambas: si Docker Desktop tiene la integración activa, su atajo
intercepta el comando `docker` y se acaba hablando con el motor equivocado.

### Verificación de la 0.2 — resultado

```
docker             29.8.0        ✅ (mínimo 24)
compose            5.5.1         ✅ v2+
uv                 0.12.16       ✅
python (uv)        3.13.15       ✅ el del proyecto
python (sistema)   3.14.4        ✅ intacto
node               v22.22.1      ✅ LTS
git                2.53.0        ✅
grupo docker       OK            ✅
docker run hello-world           ✅
postgres:16-alpine descargada    ✅
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
