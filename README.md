# Alternatic Forgejo

Este repositorio configura una instancia propia de [Forgejo](https://forgejo.org/), una plataforma para alojar repositorios Git y colaborar en ellos. Docker Compose ejecuta Forgejo y PostgreSQL; los datos de ambos servicios se guardan en el equipo anfitrión.

## Componentes

| Servicio | Función | Acceso |
| --- | --- | --- |
| `server` (`forgejo`) | Interfaz web, gestión de repositorios y servidor Git por HTTP y SSH. | Web: `http://localhost:<FORGEJO_HTTP_PORT>`; SSH: puerto `<FORGEJO_SSH_PORT>` del anfitrión. |
| `db` (`forgejo-db`) | Base de datos PostgreSQL 15 de Forgejo. | Solo dentro de la red Docker `forgejo-net`; no publica un puerto al anfitrión. |

La configuración de Compose está en [`docker-compose.yml`](./docker-compose.yml). Los puertos publicados se definen en `.env`:

- `FORGEJO_HTTP_PORT` se conecta al puerto 3000 del contenedor para la interfaz web.
- `FORGEJO_SSH_PORT` se conecta al puerto 22 del contenedor para SSH.
- `POSTGRES_PORT` es el puerto interno de PostgreSQL en la red Docker; no se publica en el anfitrión.

## Requisitos

- Docker Desktop (Windows/macOS) o Docker Engine con el complemento Docker Compose.
- Libres en el equipo los puertos elegidos para `FORGEJO_HTTP_PORT` y `FORGEJO_SSH_PORT`.

## Iniciar Forgejo

Desde PowerShell, abre la carpeta del repositorio. Crea tu archivo local `.env` desde la plantilla y edítalo:

```powershell
if (-not (Test-Path .env)) { Copy-Item .env.example .env }
notepad .env
```

Configura una contraseña larga y única en `POSTGRES_PASSWORD`. Después inicia los servicios:

```powershell
docker compose up -d
docker compose ps
```

Cuando los contenedores estén en ejecución, visita `http://localhost:<FORGEJO_HTTP_PORT>` (por defecto `http://localhost:3000`) y termina la configuración inicial de Forgejo si aparece el asistente. Crea una cuenta y, desde Forgejo, crea un repositorio nuevo. El usuario y la contraseña de Forgejo son independientes de las credenciales de PostgreSQL configuradas para uso interno.

Comandos útiles:

```powershell
docker compose logs -f server  # Seguir los registros de Forgejo
docker compose logs -f db      # Seguir los registros de PostgreSQL
docker compose restart         # Reiniciar los servicios
docker compose down            # Detenerlos sin borrar los datos
docker compose up -d           # Volver a iniciarlos
```

## Datos y configuración

El archivo `.env` y los directorios `forgejo-data/` y `postgres-data/` están excluidos de Git mediante `.gitignore` porque contienen configuración local, credenciales o estado de la instancia:

- `.env`: usuarios, contraseña de PostgreSQL y puertos. Se crea localmente a partir de [`.env.example`](./.env.example), que sí está versionado y sirve de plantilla.
- `forgejo-data/`: repositorios Git, configuración, adjuntos, paquetes, claves y otros datos de Forgejo.
- `postgres-data/`: archivos de la base de datos PostgreSQL.

Docker Compose monta estos directorios en los contenedores. Por eso, `docker compose down` no borra los datos y al volver a iniciar los servicios se conservan usuarios y repositorios. **No borres esos directorios si quieres conservar la instancia.** Para una copia de seguridad consistente, detén primero los servicios y copia ambos directorios y el archivo `.env` a una ubicación segura y privada; después puedes iniciarlos de nuevo.

La configuración persistente de Forgejo suele encontrarse en `forgejo-data/gitea/conf/app.ini`. La instalación usa PostgreSQL dentro de la red Docker. Si vas a acceder desde otros equipos o publicar el servicio, configura la URL pública (`ROOT_URL`) y el dominio SSH (`SSH_DOMAIN`) para que coincidan con tu dominio y con los puertos publicados.

### Seguridad

No compartas ni subas `.env`: contiene credenciales y `.gitignore` lo excluye del repositorio. `.env.example` es solo una plantilla; antes de iniciar una instalación nueva, reemplaza su contraseña de ejemplo por una contraseña larga y única. El archivo `.env` local que acompaña a esta instalación conserva las credenciales que ya usa su base de datos; no las cambies sin actualizar también la contraseña del usuario dentro de PostgreSQL, ya que cambiar `POSTGRES_PASSWORD` por sí solo no modifica una base de datos ya inicializada. Para desplegar en Internet, configura además HTTPS, dominio y firewall. No publiques copias de seguridad ni archivos de datos: pueden contener información sensible.

## Conectar un repositorio Git

En Forgejo, abre el repositorio y copia su URL de clonación. Puedes trabajar por HTTPS o por SSH.

### HTTPS

Para una instancia local con los puertos predeterminados, la URL suele tener este formato (si cambias `FORGEJO_HTTP_PORT`, sustituye `3000`):

```text
http://localhost:3000/USUARIO/REPOSITORIO.git
```

Clona el repositorio y entra en su carpeta:

```powershell
git clone http://localhost:3000/USUARIO/REPOSITORIO.git
cd REPOSITORIO
```

Para subir cambios, autentícate con tu cuenta de Forgejo cuando Git lo solicite. Si tienes la autenticación de dos factores habilitada, utiliza un token de acceso en lugar de la contraseña, si así lo requiere tu configuración.

### SSH

SSH permite autenticarte con una clave en vez de introducir la contraseña en cada operación.

1. Si aún no tienes una clave SSH, créala en PowerShell:

   ```powershell
   ssh-keygen -t ed25519 -C "tu-correo@example.com"
   ```

   Acepta la ubicación propuesta o elige otra y, si quieres, protege la clave privada con una frase de paso.

2. Copia **la clave pública** y añádela en Forgejo, en la configuración de tu cuenta → **Claves SSH**:

   ```powershell
   Get-Content $env:USERPROFILE\.ssh\id_ed25519.pub | Set-Clipboard
   ```

   No compartas ni subas el archivo `id_ed25519` (sin `.pub`): es la clave privada.

3. Clona usando el puerto SSH publicado por este Compose (por defecto `2222`; si cambias `FORGEJO_SSH_PORT`, usa su nuevo valor):

   ```powershell
   git clone ssh://git@localhost:2222/USUARIO/REPOSITORIO.git
   ```

   Reemplaza `localhost` por el dominio o la dirección IP del servidor si te conectas desde otro equipo. Compose configura el puerto SSH anunciado por Forgejo con `FORGEJO_SSH_PORT`, que debe coincidir con el puerto publicado al exterior, no con el puerto interno 22 del contenedor.

## Flujo habitual de trabajo con Git

Dentro de la carpeta del repositorio:

```powershell
git status
git pull
# Edita o crea archivos
git add .
git commit -m "Describe el cambio"
git push
```

- `git status`: muestra los archivos modificados y la rama actual.
- `git pull`: trae e integra los cambios remotos antes de trabajar.
- `git add .`: prepara los cambios del directorio actual para el próximo commit. Puedes usar `git add archivo` para seleccionar archivos concretos.
- `git commit -m "..."`: guarda un commit local con un mensaje descriptivo.
- `git push`: publica los commits en Forgejo.

Para asociar un proyecto local existente con un repositorio vacío de Forgejo, desde la carpeta del proyecto ejecuta:

```powershell
git init
git add .
git commit -m "Primer commit"
git branch -M main
git remote add origin http://localhost:3000/USUARIO/REPOSITORIO.git
git push -u origin main
```

Si cambiaste `FORGEJO_HTTP_PORT`, reemplaza `3000` por su valor. También puedes usar la URL SSH en `git remote add origin` (reemplaza `2222` si cambiaste `FORGEJO_SSH_PORT`):

```powershell
git remote add origin ssh://git@localhost:2222/USUARIO/REPOSITORIO.git
```

`origin` es el nombre convencional del remoto. Si ya existe, actualiza su URL en vez de volver a añadirlo:

```powershell
git remote -v
git remote set-url origin NUEVA_URL
```

## Solución de problemas

- **No abre la interfaz web:** comprueba `docker compose ps` y `docker compose logs -f server`; verifica que el puerto configurado en `FORGEJO_HTTP_PORT` esté libre y que exista `.env`.
- **No conecta por SSH:** comprueba que Forgejo esté activo, que el puerto configurado en `FORGEJO_SSH_PORT` esté permitido por el firewall y que la URL SSH use ese puerto. Confirma también que añadiste la clave pública correcta a tu cuenta.
- **La aplicación no conecta con PostgreSQL:** consulta `docker compose logs -f db` y `docker compose logs -f server`. En esta red, el servidor debe conectarse a `db:5432`; `localhost` dentro del contenedor no apunta al servicio de base de datos.
- **No aparecen los datos previos:** verifica que estás ejecutando Compose desde la carpeta correcta y que existen `forgejo-data/` y `postgres-data/`.
