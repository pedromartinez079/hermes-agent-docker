# Hermes Agent con Docker en Windows y Linux

Guía para instalar **Hermes Agent** mediante Docker en:

- Linux: Debian, Ubuntu, Linux Mint y distribuciones derivadas.
- Windows 10/11 mediante Docker Desktop con contenedores Linux.
- Docker Compose y Portainer.

> **Estado de esta guía:** el Compose incluido construye la imagen desde el repositorio oficial de Hermes Agent. Se evita depender de una etiqueta de imagen publicada que pueda cambiar.

## Arquitectura

La instalación ejecuta dos servicios a partir de la misma imagen:

- `hermes`: gateway de mensajería y Telegram.
- `hermes-dashboard`: dashboard web.

Los dos servicios comparten el mismo directorio persistente, montado dentro del contenedor como `/opt/data`.

```text
Host
└── directorio persistente
    └── /opt/data dentro del contenedor
        ├── config.yaml
        ├── .env
        ├── state.db
        ├── sessions/
        ├── memory/
        └── skills/
```

## Requisitos

### Hardware recomendado

- 2 CPU o más.
- 4 GB de RAM como mínimo; 8 GB recomendado si se utiliza navegador, voz o varias herramientas.
- 10 GB libres para la imagen, dependencias y Chromium.
- Conexión a Internet.

### Software

- Docker Engine y Docker Compose en Linux.
- Docker Desktop en Windows, configurado para **Linux containers**.
- Git.
- Portainer es opcional y sirve para administrar el stack después de construir la imagen.

## Consideraciones de seguridad

El dashboard puede almacenar credenciales, tokens y configuración sensible. Esta guía publica el puerto únicamente en `127.0.0.1`, por lo que no queda accesible directamente desde otros equipos de la red.

No cambies el mapeo a `0.0.0.0:9119:9119` sin colocar antes el dashboard detrás de autenticación y HTTPS mediante un proxy inverso.

El archivo `.env` contiene secretos y debe protegerse:

```text
.env   # no subir a GitHub
```

Incluye un `.gitignore` con, al menos:

```gitignore
.env
*.db
sessions/
memory/
state/
```

---

# 1. Obtener el código fuente

El repositorio oficial es:

```text
https://github.com/NousResearch/hermes-agent
```

## Linux

```bash
sudo apt update
sudo apt install -y git docker.io docker-compose-plugin
sudo systemctl enable --now docker
sudo usermod -aG docker "$USER"
```

Cierra la sesión y vuelve a entrar para aplicar el grupo `docker`.

Clona el repositorio:

```bash
sudo mkdir -p /opt
sudo git clone https://github.com/NousResearch/hermes-agent.git /opt/hermes-agent
sudo chown -R "$USER:$USER" /opt/hermes-agent
cd /opt/hermes-agent
```

## Windows

1. Instala [Docker Desktop](https://www.docker.com/products/docker-desktop/).
2. Durante la instalación, selecciona el backend **WSL 2** si está disponible.
3. Comprueba que Docker Desktop utiliza **Linux containers**.
4. Instala [Git for Windows](https://git-scm.com/download/win).
5. Clona el repositorio desde PowerShell:

```powershell
git clone https://github.com/NousResearch/hermes-agent.git C:\hermes-agent
Set-Location C:\hermes-agent
```

También puedes trabajar dentro de WSL2. En ese caso, sigue las instrucciones de Linux desde la distribución WSL y guarda los datos dentro del sistema de archivos Linux, no en `/mnt/c`, para obtener un mejor rendimiento.

---

# 2. Crear el directorio persistente

## Linux

Se recomienda usar una ruta fuera del repositorio:

```bash
sudo mkdir -p /opt/hermes-data
sudo chown -R "$(id -u):$(id -g)" /opt/hermes-data
```

Obtén el UID y GID del usuario que ejecutará Docker:

```bash
id -u
id -g
```

Normalmente ambos valores son `1000`.

## Windows

Crea una carpeta persistente, por ejemplo:

```powershell
New-Item -ItemType Directory -Force C:\hermes-data
```

En Docker Desktop, confirma que la unidad o carpeta está disponible para Docker. Las versiones actuales normalmente gestionan automáticamente el acceso mediante WSL2; si Docker informa de un error de montaje, revisa:

```text
Docker Desktop → Settings → Resources → File sharing
```

---

# 3. Crear el archivo de Compose

Crea `docker-compose.yml` en el directorio del repositorio.

> Este Compose está adaptado para funcionar de forma portable en Linux y Windows. A diferencia del Compose de desarrollo, usa una red Docker normal y publica el dashboard únicamente en el loopback del host.

```yaml
services:
  gateway:
    image: hermes-agent:local
    container_name: hermes
    restart: unless-stopped
    volumes:
      - ${HERMES_DATA_DIR:-./hermes-data}:/opt/data
    environment:
      HERMES_UID: ${HERMES_UID:-1000}
      HERMES_GID: ${HERMES_GID:-1000}
    command: ["gateway", "run"]

  dashboard:
    image: hermes-agent:local
    container_name: hermes-dashboard
    restart: unless-stopped
    depends_on:
      - gateway
    volumes:
      - ${HERMES_DATA_DIR:-./hermes-data}:/opt/data
    environment:
      HERMES_UID: ${HERMES_UID:-1000}
      HERMES_GID: ${HERMES_GID:-1000}
    ports:
      - "127.0.0.1:9119:9119"
    command: ["dashboard", "--host", "0.0.0.0", "--no-open"]
```

## Archivo `.env` de Compose

El archivo `.env` siguiente es para que **Docker Compose** sustituya variables del YAML. No debe confundirse con `/opt/data/.env`, que puede contener secretos de Hermes.

### Linux

Crea `/opt/hermes-agent/.env`:

```dotenv
HERMES_DATA_DIR=/opt/hermes-data
HERMES_UID=1000
HERMES_GID=1000
```

Si tu UID o GID son diferentes, sustitúyelos.

### Windows

Crea `C:\hermes-agent\.env`:

```dotenv
HERMES_DATA_DIR=C:/hermes-data
HERMES_UID=1000
HERMES_GID=1000
```

Usar barras `/` en `HERMES_DATA_DIR` evita problemas de escape con las rutas de Windows dentro de Compose.

> No publiques este archivo en GitHub si contiene tokens o credenciales. En este ejemplo no contiene secretos.

---

# 4. Construir la imagen

La imagen oficial se construye desde el repositorio. La primera construcción puede tardar porque instala Node.js, Python, Playwright/Chromium y dependencias del sistema.

## Linux

```bash
cd /opt/hermes-agent
docker build -t hermes-agent:local .
```

## Windows PowerShell

```powershell
Set-Location C:\hermes-agent
docker build -t hermes-agent:local .
```

Comprueba que la imagen está disponible:

```bash
docker image ls hermes-agent
```

En PowerShell:

```powershell
docker image ls hermes-agent
```

---

# 5. Arrancar Hermes

## Linux

```bash
cd /opt/hermes-agent
docker compose up -d
```

## Windows PowerShell

```powershell
Set-Location C:\hermes-agent
docker compose up -d
```

Comprueba el estado:

```bash
docker compose ps
```

Los dos servicios deben aparecer como `running`.

---

# 6. Configurar Hermes

Ejecuta el asistente de configuración dentro del contenedor del gateway:

```bash
docker exec -it hermes hermes setup
```

También puedes abrir una consola:

```bash
docker exec -it hermes /bin/bash
```

Y ejecutar dentro del contenedor:

```bash
hermes setup
```

El dashboard estará disponible localmente en:

```text
http://127.0.0.1:9119
```

En Windows, abre:

```text
http://localhost:9119
```

En Linux, abre:

```text
http://localhost:9119
```

---

# 7. Configurar Telegram

Configura Telegram usando el asistente de Hermes o desde la configuración del gateway.

El token debe guardarse de forma segura en el directorio persistente, normalmente en:

```text
/opt/data/.env
```

En el host se encontrará en:

- Linux: `/opt/hermes-data/.env`
- Windows: `C:\hermes-data\.env`

Para grupos y supergrupos, recuerda que normalmente hay que configurar también la autorización del chat. Por ejemplo:

```dotenv
TELEGRAM_GROUP_ALLOWED_CHATS=-1001234567890
TELEGRAM_REQUIRE_MENTION=true
```

Para permitir respuestas sin mencionar al bot en un chat concreto:

```dotenv
TELEGRAM_FREE_RESPONSE_CHATS=-1001234567890
```

Después de modificar `.env`, recrea los contenedores:

```bash
docker compose up -d --force-recreate
```

No basta con reiniciar una sesión de conversación; el gateway debe volver a cargar las variables de entorno.

---

# 8. Logs y diagnóstico

## Logs del gateway

```bash
docker logs -f hermes
```

## Logs del dashboard

```bash
docker logs -f hermes-dashboard
```

## Estado de los contenedores

```bash
docker compose ps
```

## Comprobar el puerto del dashboard

Linux:

```bash
ss -ltnp | grep 9119
```

Windows PowerShell:

```powershell
Get-NetTCPConnection -LocalPort 9119
```

## Reiniciar Hermes

```bash
docker compose restart
```

## Recrear tras cambiar la configuración

```bash
docker compose up -d --force-recreate
```

---

# 9. Administrar la instalación con Portainer

Portainer no sustituye a Docker Compose; administra los contenedores y stacks que ejecuta Docker.

## Opción recomendada: construir primero desde terminal

Construye la imagen desde el repositorio:

```bash
docker build -t hermes-agent:local .
```

Después, en Portainer:

1. Abre **Stacks**.
2. Selecciona **Add stack**.
3. Introduce el nombre `hermes`.
4. Selecciona **Web editor**.
5. Pega el contenido de `docker-compose.yml`.
6. Ajusta `HERMES_DATA_DIR` a la ruta persistente del host.
7. Pulsa **Deploy the stack**.

En Portainer, las variables también pueden definirse en la sección **Environment variables**:

```text
HERMES_DATA_DIR=/opt/hermes-data
HERMES_UID=1000
HERMES_GID=1000
```

En Windows, utiliza:

```text
HERMES_DATA_DIR=C:/hermes-data
```

La imagen `hermes-agent:local` debe existir previamente en el Docker Engine gestionado por Portainer.

> Si Portainer se ejecuta en otro Docker Engine o en otro servidor, la imagen debe construirse o importarse en ese Engine concreto. Las imágenes locales no se comparten automáticamente entre servidores.

---

# 10. Actualizar la instalación

No ejecutes `hermes update` dentro del contenedor si quieres mantener una instalación reproducible mediante Docker.

Actualiza desde el repositorio y vuelve a construir la imagen:

## Linux

```bash
cd /opt/hermes-agent
git pull
docker build --pull -t hermes-agent:local .
docker compose up -d
```

## Windows PowerShell

```powershell
Set-Location C:\hermes-agent
git pull
docker build --pull -t hermes-agent:local .
docker compose up -d
```

Los datos de Hermes no se pierden porque están fuera de la imagen, en el directorio persistente.

Antes de actualizar, se recomienda hacer una copia:

```bash
sudo tar -czf hermes-backup-$(date +%Y%m%d-%H%M%S).tar.gz -C /opt hermes-data
```

En Windows, copia `C:\hermes-data` a otra unidad o utiliza una herramienta de backup compatible.

---

# 11. Migrar entre Linux y Windows

Puedes copiar el directorio persistente entre sistemas, pero Hermes debe estar detenido durante la copia.

## Linux a Windows

1. Ejecuta:

```bash
docker compose down
```

2. Copia `/opt/hermes-data` a `C:\hermes-data`.
3. Ajusta el `.env` de Compose de Windows.
4. Arranca el stack en Windows.

## Windows a Linux

1. Ejecuta en PowerShell:

```powershell
docker compose down
```

2. Copia `C:\hermes-data` a `/opt/hermes-data`.
3. Corrige permisos en Linux:

```bash
sudo chown -R 1000:1000 /opt/hermes-data
```

4. Arranca el stack en Linux.

No ejecutes dos gateways con el mismo bot de Telegram al mismo tiempo. Telegram solo debe tener una instancia activa procesando el bot.

---

# 12. Copia de seguridad

Como mínimo, incluye en la copia:

```text
.env
config.yaml
auth.json
state.db
sessions/
memory/
skills/
cron/
```

Protege especialmente `.env`, ya que puede contener el token de Telegram y credenciales de proveedores.

No publiques en GitHub:

```text
.env
*.db
sessions/
auth.json
```

Una buena práctica es publicar únicamente:

```text
docker-compose.yml
.env.example
.gitignore
README.md
```

## Ejemplo de `.env.example`

```dotenv
HERMES_DATA_DIR=/opt/hermes-data
HERMES_UID=1000
HERMES_GID=1000

# Los secretos de Hermes deben configurarse en el directorio persistente,
# no en este archivo público.
```

---

# Limitaciones conocidas

- El dashboard está limitado a `localhost` por seguridad.
- La primera construcción Docker puede requerir bastante tiempo y espacio.
- El mismo bot de Telegram no debe estar activo en dos sistemas simultáneamente.
- `state.db` no debe compartirse con dos instancias activas.
- Las imágenes locales deben existir en el mismo Docker Engine que administra Portainer.
- El nombre y la ubicación exacta de algunos archivos internos pueden cambiar entre versiones; el directorio persistente completo debe tratarse como una unidad.

## Referencias

- [Repositorio oficial de Hermes Agent](https://github.com/NousResearch/hermes-agent)
- [Documentación oficial](https://hermes-agent.nousresearch.com/docs/)
- [Docker Desktop](https://www.docker.com/products/docker-desktop/)
- [Portainer](https://www.portainer.io/)

