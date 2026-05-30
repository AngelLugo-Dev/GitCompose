# Despliegue Forgejo profesional — Clever Cloud

Guia operativa para desplegar Forgejo en DigitalOcean (Ubuntu) con TLS automatico (Caddy), acceso **solo por invitacion por correo** y migracion desde Bitbucket. Cada paso lleva: **Que / Por que / Validar / Rollback**.

> **Doble fuente de verdad:**
> - **Este archivo (`DEPLOY_DIGITALOCEAN_UBUNTU.md`)** = manual de referencia. Lista los 16 pasos canonicos.
> - **`EXECUTED_LOG.md`** = bitacora cronologica de lo que realmente se ejecuto en el droplet `gitserver`, con fechas y resultados. Si necesitas saber "donde nos quedamos hoy", abre el log. Si necesitas saber "como hacer el paso N", abre este manual.

> **Modelo de acceso:** No hay registro publico. Solo entran usuarios a quienes el admin invita por correo (manual o desde una organizacion). El usuario recibe un correo, hace clic en el link, define su contrasena y entra. Sin restriccion de dominio: cualquier correo (Workspace, Gmail personal, otra empresa) funciona si fue invitado por el admin.

---

## Estado y supuestos

- Droplet Ubuntu 22.04 o 24.04 (recomendado 4 vCPU / 8 GB RAM, disco 80 GB+).
- Acceso SSH ya disponible: `angel@gitserver`.
- Dominio `gitserver.clevercloud.app` debe tener un A record apuntando al IP del droplet (verificar con `dig +short gitserver.clevercloud.app`).
- **SMTP funcional** (Gmail Workspace con App Password recomendado). Sin esto NO se mandan invitaciones — es bloqueante.
- Token de Bitbucket con scopes `read:repository`, `read:workspace`.

---

## 1) Firewall en DigitalOcean (panel web) y en Ubuntu

**Que:** Abrir puertos 22 (admin SSH), 80 (HTTP, ACME), 443 (HTTPS), 222 (Git por SSH).

**Por que:** Caddy necesita 80 para emitir certificados Let's Encrypt y 443 para servir HTTPS. 222 es el puerto donde Forgejo expone su SSH para `git clone/push`.

**Hacer en el panel de DigitalOcean (Networking → Firewalls):**

- Inbound: TCP 22, 80, 443, 222 desde `Any IPv4 / Any IPv6`.
- Outbound: All TCP, All UDP, All ICMP.

**Y en Ubuntu (UFW como segunda capa):**

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y ca-certificates curl gnupg ufw
sudo ufw allow OpenSSH
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw allow 222/tcp
sudo ufw --force enable
sudo ufw status verbose
```

**Validar:** `sudo ufw status` debe listar 22, 80, 443, 222 en estado `ALLOW`.

**Rollback:** `sudo ufw disable` desactiva todas las reglas (deja al droplet expuesto solo al firewall de DO).

---

## 2) Instalar Docker + Compose v2

**Que:** Instalar Docker Engine y plugin `compose`.

**Por que:** Toda la pila (Caddy, Forgejo, Postgres) corre en contenedores con `docker compose`.

```bash
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER
newgrp docker
docker --version
docker compose version
```

**Validar:** Ambos `--version` imprimen una version (Docker 24+ y Compose v2.x). `docker run --rm hello-world` corre sin error.

**Rollback:** `sudo apt purge -y docker-ce docker-ce-cli containerd.io docker-compose-plugin && sudo rm -rf /var/lib/docker /etc/docker`.

---

## 3) Crear directorio de despliegue

**Que:** Crear `/opt/forgejo` con permisos del usuario `angel`.

**Por que:** Convencion: stacks productivos viven bajo `/opt/<servicio>`. Asi separamos del home del usuario y evitamos perder datos si se borra `~`.

```bash
sudo mkdir -p /opt/forgejo
sudo chown -R $USER:$USER /opt/forgejo
cd /opt/forgejo
```

**Validar:** `ls -ld /opt/forgejo` muestra `angel angel` como owner.

**Rollback:** `sudo rm -rf /opt/forgejo` (solo si nunca levantaste el stack).

---

## 4) Subir archivos del proyecto al droplet

**Que:** Dejar `docker-compose.prod.yml` y `infra/caddy/Caddyfile` dentro de `/opt/forgejo` en el droplet.

**Por que:** Son los manifiestos que `docker compose` y Caddy van a leer.

Hay dos opciones, elige una.

### Opcion A — `scp` desde tu PC (rapido si tienes los archivos locales)

Desde una terminal de tu PC (PowerShell o WSL), parado en `C:\Users\hannm\OneDrive\Escritorio\Clever-Forgejo`:

```bash
scp docker-compose.prod.yml angel@64.23.137.164:/opt/forgejo/
ssh angel@64.23.137.164 "mkdir -p /opt/forgejo/infra/caddy"
scp infra/caddy/Caddyfile angel@64.23.137.164:/opt/forgejo/infra/caddy/
```

> Nota: NO subimos `.env.example` ni `.env`. El `.env` con secretos reales se crea directo en el droplet en el paso 5.

### Opcion B — pegar contenido en el droplet con `nano` (sin salir de la SSH)

Desde la sesion SSH del droplet:

```bash
cd /opt/forgejo
nano docker-compose.prod.yml
# Pegar el contenido del archivo. Ctrl+O Enter para guardar, Ctrl+X para salir.

mkdir -p /opt/forgejo/infra/caddy
nano /opt/forgejo/infra/caddy/Caddyfile
# Pegar el contenido del Caddyfile. Ctrl+O Enter, Ctrl+X.
```

### Validar (cualquier opcion)

```bash
ls -la /opt/forgejo
ls -la /opt/forgejo/infra/caddy
docker compose -f /opt/forgejo/docker-compose.prod.yml config >/dev/null && echo "YAML valido"
```

> El comando `compose config` puede avisar de variables vacias (`POSTGRES_USER`, `FORGEJO_DOMAIN`, etc.). Es esperado: el `.env` no existe todavia, se crea en el siguiente paso.

**Rollback:** `rm -f /opt/forgejo/docker-compose.prod.yml /opt/forgejo/infra/caddy/Caddyfile` y volver a crear.

---

## 5) Generar secretos y crear `.env`

**Que:** Crear `/opt/forgejo/.env` desde el ejemplo, sustituyendo placeholders por secretos reales.

**Por que:** Los secretos NO se commitean. Si se filtra `SECRET_KEY` o `INTERNAL_TOKEN` un atacante puede falsificar sesiones. `LFS_JWT_SECRET` firma tokens de archivos grandes; cambiarlo invalida sesiones activas pero los datos en disco siguen.

```bash
cd /opt/forgejo
cp .env.example .env

# Generar 3 secretos (uno por linea)
openssl rand -hex 32   # FORGEJO_SECRET_KEY
openssl rand -hex 32   # FORGEJO_INTERNAL_TOKEN
openssl rand -hex 32   # FORGEJO_LFS_JWT_SECRET

nano .env
# Pegar los 3 secretos en sus variables.
# Cambiar POSTGRES_PASSWORD por uno fuerte (>=24 chars, sin comillas).
# Llenar SMTP_USER/SMTP_PASS (App Password de Gmail Workspace).

chmod 600 .env
```

**Validar:** `cat .env | grep CHANGE_ME` no devuelve nada (todos los placeholders sustituidos).

**Rollback:** `cp .env.example .env` y rehacer.

---

## 6) (Ya hecho en paso 4) Verificar Caddyfile

Si seguiste paso 4, el Caddyfile ya quedo con tu dominio real. Solo verifica:

```bash
head -1 /opt/forgejo/infra/caddy/Caddyfile
dig +short gitserver.clevercloud.app
```

La primera linea del Caddyfile debe coincidir con el dominio que devuelve `dig`.

---

## 7) Levantar stack

**Que:** Levantar Caddy + Forgejo + Postgres.

**Por que:** Caddy emite el certificado TLS al primer arranque y Forgejo queda listo para crear la cuenta admin.

```bash
cd /opt/forgejo
docker compose -f docker-compose.prod.yml pull
docker compose -f docker-compose.prod.yml up -d
docker compose -f docker-compose.prod.yml ps
docker compose -f docker-compose.prod.yml logs -f --tail=100
```

Espera a que veas `Listen: http://0.0.0.0:3000` en el log de `server` y `certificate obtained successfully` en el log de `caddy`. Sal con Ctrl+C (no detiene los contenedores, solo el `logs -f`).

**Validar:**

- `curl -I https://gitserver.clevercloud.app` devuelve `HTTP/2 200` o `303` con header `server: Forgejo`.
- En el navegador, abrir `https://gitserver.clevercloud.app` muestra la home de Forgejo.

**Rollback:** `docker compose -f docker-compose.prod.yml down` (deja datos en `forgejo_data` y `postgres_data` intactos). Si algo en datos esta mal: `docker compose -f docker-compose.prod.yml down -v` y `sudo rm -rf forgejo_data postgres_data infra/caddy/data infra/caddy/config`.

---

## 8) Crear el primer admin

**Que:** Crear cuenta admin desde el CLI de Forgejo. Es la cuenta principal del operador, la que usa para administrar el servidor, invitar gente y configurar la organizacion.

**Por que:** El registro publico esta desactivado, asi que la unica forma de meter el primer usuario es por CLI.

```bash
cd /opt/forgejo
set -a && source .env && set +a

docker compose -f docker-compose.prod.yml exec -u 1000 server \
  forgejo admin user create \
    --admin \
    --username clever-root \
    --email devops@clevercloud.mx \
    --random-password \
    --must-change-password
```

Copia el password aleatorio que imprime y guardalo en un password manager.

**Validar:** Login en `https://gitserver.clevercloud.app/user/login` con `clever-root` + el password. Forgejo te pide cambiarlo al entrar. Despues:

1. **Activa 2FA** inmediatamente: Settings → Security → Two-Factor Authentication.
2. Verifica el correo saliente: Site Administration → Configuration → "Test Mailer" envia un mail a tu inbox.

**Rollback:** Si te equivocas o pierdes la contrasena:

```bash
docker compose -f docker-compose.prod.yml exec -u 1000 server \
  forgejo admin user delete --username clever-root
```

Y vuelve a crear.

---

## 9) Verificar que SMTP funciona (CRITICO antes de invitar)

**Que:** Probar que Forgejo si esta mandando correo. Si no, las invitaciones nunca van a llegar y vas a creer que el sistema esta roto.

**Por que:** El acceso a la plataforma depende 100% del correo. Si SMTP esta mal, ningun invitado puede entrar.

Como `clever-root`:

1. **Site Administration → Configuration → Mailer Configuration**: revisa que muestre el SMTP que pusiste en `.env`.
2. Boton **"Send Testing Email"** → ingresa tu propio correo → envia.
3. Llega a tu inbox en menos de 1 minuto.

**Si no llega:**

- Revisa logs: `docker compose -f docker-compose.prod.yml logs --tail=200 server | grep -i mail`.
- Errores tipicos:
  - `535 5.7.8 Username and Password not accepted` → la App Password de Gmail esta mal o no es App Password (es contrasena normal de la cuenta). Genera una nueva en `https://myaccount.google.com/apppasswords`.
  - `connection timed out` → el droplet bloquea egress en 587. DigitalOcean a veces filtra; abrir ticket o usar relay alterno (SendGrid, Mailgun, AWS SES).

**Rollback:** Editar `.env`, reiniciar `server`:

```bash
docker compose -f docker-compose.prod.yml up -d --force-recreate server
```

Y volver a probar.

---

## 10) Crear la organizacion CleverCloud (antes de invitar)

**Que:** Crear la organizacion donde van a vivir todos los repos.

**Por que:** Las invitaciones al equipo se hacen a nivel **organizacion**, no a nivel servidor. Si invitas a un externo a la organizacion, Forgejo le manda un correo con un link que le **permite registrarse** (aunque el registro publico este desactivado), y queda autoasignado al team que elegiste.

Como `clever-root` desde la UI:

1. Click en el avatar (arriba derecha) → **+ New Organization**.
   - Owner: `clever-root`
   - Organization Name: `CleverCloud`
   - Visibility: `Private`
2. Dentro de la organizacion → **Teams → New Team**, crear:
   - `Owners` — Permission: Owner (admin total de la org). Solo 2-3 personas.
   - `Developers` — Permission: Write. Acceso write a todos los repos.
   - `QA` — Permission: Read. Read + triage de issues.
   - `ReleaseManagers` — Permission: Write. Crear releases/tags.
3. Organization → Settings:
   - **Default repository permission**: None (los repos privados por defecto).

**Validar:** `https://gitserver.clevercloud.app/org/CleverCloud/teams` lista los 4 teams.

**Rollback:** Settings de cada team → Delete. O Settings de la org → Delete Organization (al final del todo).

---

## 11) Invitar usuarios por correo

**Que:** Para cada empleado o residente, mandar invitacion por correo desde Forgejo. El usuario recibe el correo, hace clic en el link, define su contrasena y entra. Sin OAuth, sin que el admin tenga que crear cuenta a mano.

**Por que:** Esta es la implementacion del modelo "solo entran los invitados". Funciona igual para `@clevercloud.mx`, Gmail personales, o cualquier otro correo.

### 11a) Invitacion via organizacion (recomendado)

Como `clever-root`, en la UI:

1. **Organization CleverCloud → Members → Invite User**.
2. Pega el correo del invitado (uno por uno o varios separados por coma).
3. Selecciona el **team** al que va (Developers / QA / etc). Su rol queda fijado desde el dia uno.
4. Click "Add invitation".

Forgejo manda automaticamente un correo con el link. El invitado:

1. Abre el correo, click en el link.
2. Pagina de Forgejo: define `username` (sugerido) y contrasena.
3. Confirma el correo (link adicional si `REGISTER_EMAIL_CONFIRM=true`).
4. Queda con sesion iniciada y ya esta dentro del team.

**Validar:** `Organization → Members` ya muestra al usuario. Si aun no acepto la invitacion, aparece como "pending invitation".

**Rollback de una invitacion:**

- Si todavia no la aceptaron: Members → Invitations → boton de borrar al lado de la invitacion pendiente.
- Si ya entraron y quieres sacarlos: Members → "Remove from organization", o Site Administration → Users → Delete user.

### 11b) Crear usuario directo desde el panel admin (alternativa)

Si necesitas crear un usuario que no va a ninguna organizacion (caso raro), o quieres preconfigurar todo:

1. **Site Administration → Users → Create User Account**.
2. Llena: username, full name, email.
3. Marca: `User must change password` y `Send notification email` (esto manda un correo de bienvenida).
4. Save.

**Validar:** Te llega el correo a la direccion del usuario.

### 11c) Crear usuario desde CLI (caso de emergencia)

```bash
docker compose -f docker-compose.prod.yml exec -u 1000 server \
  forgejo admin user create \
    --username juan-perez \
    --email juan@externo.com \
    --random-password \
    --must-change-password
```

Imprime un password temporal. Compartelo de forma segura. El usuario lo cambia al primer login.

---

## 12) Mover usuarios entre equipos

**Que:** Cambiar el rol de alguien (subir a Owners, bajar a QA, etc).

Como `clever-root`:

1. `Organization → CleverCloud → Teams → <team destino> → Add team member`.
2. Buscar por username y agregar.
3. Si quieres quitarlo del team anterior: ese team → boton "Remove" al lado del usuario.

> Nota: un usuario puede pertenecer a varios teams al mismo tiempo. Forgejo aplica el permiso mas alto entre todos sus teams.

---

## 13) Migracion desde Bitbucket

**Token de Bitbucket** (Atlassian Account → API tokens):
- Scopes: `read:repository:bitbucket`, `read:workspace:bitbucket`, `read:user:bitbucket`.

**Por repositorio** (estrategia con espejo, sin downtime):

1. En Forgejo, dentro de la org `CleverCloud`: **+ → New Migration → Git**.
2. Migration source: `https://bitbucket.org/<workspace>/<repo>.git`
3. Authorization:
   - Username: `x-bitbucket-api-token-auth`
   - Password: el API token.
4. Owner: `CleverCloud`. Repo Name: igual que en Bitbucket.
5. **Marcar "This repository will be a mirror"** (pull mirror cada N minutos).
6. Items: Issues / Milestones / Labels / Pull Requests / Releases / Wiki segun aplique.

Durante el periodo de transicion el espejo sincroniza Bitbucket → Forgejo. Cuando llegue el corte:

1. En Bitbucket: marcar el repo como read-only (Repo settings → Branch permissions → bloquear push a todos).
2. En Forgejo: en el repo, **Settings → Mirror → Sync now** (jalar lo ultimo).
3. **Settings → Mirror → Convert to regular repository** (apaga el espejo).
4. Comunicar a los equipos el nuevo remoto: `git remote set-url origin git@gitserver.clevercloud.app:222/CleverCloud/<repo>.git` (puerto 222 porque ese es el SSH de Forgejo).

**Validar:** `git ls-remote origin` desde la maquina de un dev devuelve refs.

**Rollback de un repo:** Si la migracion sale mal, borrar el repo en Forgejo (`Settings → Delete`) y reintentar. Los datos en Bitbucket no se tocan.

---

## 14) Backups

**Que respaldar:**

- `/opt/forgejo/forgejo_data` (repos + attachments + LFS).
- `/opt/forgejo/postgres_data` (issues, PRs, users, invitaciones).
- `/opt/forgejo/.env` (en password manager, no en backup publico).

**Backup minimo (cron diario a las 03:00):**

```bash
sudo tee /usr/local/bin/forgejo-backup.sh >/dev/null <<'EOF'
#!/usr/bin/env bash
set -euo pipefail
BACKUP_DIR=/var/backups/forgejo
STAMP=$(date +%Y%m%d-%H%M%S)
mkdir -p "$BACKUP_DIR"
cd /opt/forgejo
docker compose -f docker-compose.prod.yml exec -T db \
  pg_dump -U "$POSTGRES_USER" "$POSTGRES_DB" \
  | gzip > "$BACKUP_DIR/db-$STAMP.sql.gz"
tar -czf "$BACKUP_DIR/forgejo_data-$STAMP.tar.gz" -C /opt/forgejo forgejo_data
find "$BACKUP_DIR" -name 'db-*.sql.gz'         -mtime +30 -delete
find "$BACKUP_DIR" -name 'forgejo_data-*.tar.gz' -mtime +30 -delete
EOF
sudo chmod +x /usr/local/bin/forgejo-backup.sh

# leer .env al ejecutar
sudo crontab -e
# agregar:
# 0 3 * * * cd /opt/forgejo && set -a && . ./.env && set +a && /usr/local/bin/forgejo-backup.sh
```

Despues subir esos `.tar.gz` y `.sql.gz` a DO Spaces con `s3cmd` o `rclone` (offsite).

**Validar:** Ejecutar `sudo /usr/local/bin/forgejo-backup.sh` a mano y verificar archivos en `/var/backups/forgejo/`.

---

## 15) Cheatsheet de operacion

```bash
# Estado
docker compose -f docker-compose.prod.yml ps

# Logs (server / db / caddy)
docker compose -f docker-compose.prod.yml logs -f --tail=200 server
docker compose -f docker-compose.prod.yml logs -f --tail=200 caddy

# Reiniciar solo Forgejo (no toca DB)
docker compose -f docker-compose.prod.yml up -d --force-recreate server

# Actualizar imagenes (mismo tag :9, parches)
docker compose -f docker-compose.prod.yml pull
docker compose -f docker-compose.prod.yml up -d
docker image prune -f

# Shell dentro del contenedor de Forgejo
docker compose -f docker-compose.prod.yml exec -u 1000 server bash

# Crear usuario / cambiar contrasena de un usuario / hacerlo admin
docker compose -f docker-compose.prod.yml exec -u 1000 server forgejo admin user create --username X --email Y --random-password --must-change-password
docker compose -f docker-compose.prod.yml exec -u 1000 server forgejo admin user change-password --username X --password NUEVO
docker compose -f docker-compose.prod.yml exec -u 1000 server forgejo admin user must-change-password X

# Restaurar DB desde dump (DESTRUYE la DB actual)
gunzip -c /var/backups/forgejo/db-XXXX.sql.gz \
  | docker compose -f docker-compose.prod.yml exec -T db psql -U "$POSTGRES_USER" "$POSTGRES_DB"
```

---

## 16) Rollback global de emergencia

Si algo se rompio gravemente y quieres volver a estado limpio:

```bash
cd /opt/forgejo
docker compose -f docker-compose.prod.yml down            # detiene, conserva volumenes
# o, si quieres tambien borrar datos (PERDIDA TOTAL):
# docker compose -f docker-compose.prod.yml down -v
# sudo rm -rf forgejo_data postgres_data infra/caddy/data infra/caddy/config
```

Antes de cualquier rollback destructivo: **corre el script de backup** del paso 14.
