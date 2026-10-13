# Forgejo

Este repositorio configura una instancia propia de [Forgejo](https://forgejo.org/), una plataforma para alojar repositorios Git y colaborar en ellos. Docker Compose ejecuta Forgejo y PostgreSQL; los datos de ambos servicios se guardan en el equipo anfitrión.

## Licencia

Este proyecto se distribuye bajo la licencia [MIT](./LICENSE). Puedes usarlo, copiarlo, modificarlo y redistribuirlo, incluso en proyectos privados o comerciales y cobrando por su distribución. La licencia solo exige conservar el aviso de copyright y el texto de la licencia en las copias o partes sustanciales del software. Se proporciona «tal cual», sin garantía.

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

## Manual completo de Git

Esta sección es una guía de referencia para trabajar con Git desde PowerShell y con repositorios alojados en esta instancia de Forgejo. Los ejemplos usan `main`, `origin`, `USUARIO` y `REPOSITORIO` como valores ilustrativos: sustitúyelos por los de tu proyecto.

Git tiene muchos subcomandos internos y opciones que dependen de la versión. Por eso, esta guía cubre los comandos habituales y los avanzados que suelen hacer falta; al final se explica cómo descubrir la referencia completa incluida en tu instalación.

### 1. El modelo mental: dónde están tus cambios

Git no guarda simplemente «la última versión». Cada commit representa una instantánea identificada por un hash, conectada con sus commits anteriores. En un repositorio normal hay cuatro lugares que conviene distinguir:

1. **Directorio de trabajo**: los archivos que estás editando.
2. **Área de preparación (índice o *staging area*)**: los cambios seleccionados para el siguiente commit.
3. **Repositorio local**: los commits guardados en `.git`.
4. **Remotos**: otros repositorios, como el de Forgejo. `origin/main` es una referencia local que recuerda el último estado conocido de la rama `main` en `origin`; no es una conexión en tiempo real.

El flujo básico es:

```text
editar archivos → git add → git commit → git push
```

`git add` decide **qué contenido** entrará en el commit; `git commit` lo guarda **localmente**; `git push` envía los commits al remoto. Un cambio sin preparar no entra en el commit, y un commit no se publica hasta hacer `push`.

### 2. Instalar, identificarte y configurar Git

Comprueba que Git está instalado y consulta su versión:

```powershell
git --version
git help
```

Configura el nombre y el correo que aparecerán como autor en los commits. Estos datos no son las credenciales de inicio de sesión de Forgejo:

```powershell
git config --global user.name "Tu nombre"
git config --global user.email "tu-correo@example.com"
git config --global --list
```

La opción `--global` aplica a tus repositorios de usuario. Para configurar un valor solo en el repositorio actual, quita `--global` y ejecuta el comando desde su carpeta. La configuración local tiene prioridad sobre la global.

Opciones útiles:

```powershell
git config --show-origin --get user.name
git config --global init.defaultBranch main
git config --global core.editor "code --wait"
```

- `--show-origin` indica de qué archivo procede un valor.
- `--get CLAVE` muestra una clave concreta.
- `init.defaultBranch` establece el nombre de rama inicial para futuros repositorios.
- `core.editor` elige el editor que Git abrirá al crear o editar mensajes; `code --wait` requiere que el comando `code` esté disponible.

No pongas contraseñas ni tokens en comandos `git remote`, archivos de configuración compartidos o URLs guardadas en el repositorio. Usa el gestor de credenciales del sistema o claves SSH.

### 3. Crear o descargar un repositorio

```powershell
git init
git clone URL
git clone URL carpeta-destino
git clone --branch main URL
```

- `git init` convierte la carpeta actual en un repositorio Git. No sube nada a Forgejo.
- `git clone URL` descarga el repositorio, su historial y configura normalmente el remoto `origin`.
- `git clone URL carpeta-destino` elige el nombre de la carpeta creada.
- `git clone --branch main URL` empieza en la rama indicada.

Para este Forgejo, las URLs típicas son `http://localhost:3000/USUARIO/REPOSITORIO.git` y `ssh://git@localhost:2222/USUARIO/REPOSITORIO.git`, ajustando los puertos a tu configuración.

### 4. Ver el estado y revisar cambios

```powershell
git status
git status --short
git diff
git diff --staged
git diff -- nombre-del-archivo
git diff origin/main...HEAD
```

- `git status` muestra la rama, los cambios sin preparar, los preparados y los archivos ignorados relevantes.
- `git status --short` presenta el mismo estado en formato compacto. En cada línea, la primera columna describe el índice y la segunda el directorio de trabajo.
- `git diff` compara el directorio de trabajo con el índice: enseña cambios **aún no preparados**.
- `git diff --staged` (también `--cached`) compara el índice con el último commit: enseña lo que entraría en el siguiente commit.
- `git diff -- nombre-del-archivo` limita la comparación a un archivo.
- `git diff origin/main...HEAD` muestra, normalmente, los cambios de tu rama desde su punto común con `origin/main`.

Revisa `git diff --staged` antes de cada commit. Es la forma más directa de verificar que no estás incluyendo archivos o cambios que no querías publicar.

### 5. Seleccionar archivos y crear commits

```powershell
git add README.md
git add src/
git add -p
git add -A
git commit -m "Documenta el flujo de Git"
git commit
git commit --amend
git show --stat
```

- `git add ARCHIVO` prepara un archivo específico.
- `git add CARPETA/` prepara cambios de esa carpeta.
- `git add -p` permite revisar y preparar cambios por fragmentos interactivos; resulta útil para separar modificaciones distintas hechas en un mismo archivo.
- `git add -A` prepara todos los cambios detectados en el repositorio: archivos nuevos, modificados y eliminados. Desde una subcarpeta, Git sigue considerando el repositorio entero.
- `git commit -m "mensaje"` crea un commit con lo preparado y el mensaje dado.
- `git commit` abre el editor para escribir un mensaje más extenso.
- `git commit --amend` reemplaza el último commit por otro que incorpora el índice actual y/o corrige su mensaje. Cambia su identificador; evita usarlo en un commit que ya compartiste con otras personas.
- `git show --stat` resume el commit más reciente; `git show HASH` inspecciona otro commit.

Escribe mensajes breves que describan el cambio («Añade guía de instalación»), no solo el archivo afectado («README»). Un commit debe agrupar un cambio coherente y revisable.

### 6. Ramas: crear, cambiar y eliminar

Una rama es una referencia móvil a un commit. Permite trabajar en una línea de cambios independiente sin alterar directamente otra rama.

```powershell
git branch
git branch -a
git branch nueva-funcionalidad
git switch nueva-funcionalidad
git switch -c nueva-funcionalidad
git switch main
git branch -m nombre-nuevo
git branch -d rama-terminada
git branch -D rama
```

- `git branch` lista las ramas locales; `git branch -a` muestra también las referencias remotas conocidas.
- `git branch NOMBRE` crea una rama sin cambiar a ella.
- `git switch NOMBRE` cambia de rama.
- `git switch -c NOMBRE` crea una rama y cambia a ella.
- `git switch main` vuelve a `main`. Si tienes cambios sin guardar, Git puede impedir el cambio para evitar que se pierdan.
- `git branch -m NOMBRE` renombra la rama actual.
- `git branch -d NOMBRE` elimina una rama local ya integrada.
- `git branch -D NOMBRE` fuerza la eliminación, aunque no esté integrada. Úsalo solo si estás seguro de que no necesitas sus commits.

Una convención sencilla es crear una rama por tarea, por ejemplo `docs/guia-git` o `fix/error-de-inicio`, y volver a `main` cuando el trabajo esté integrado.

### 7. Sincronizar con Forgejo: remoto, fetch, pull y push

```powershell
git remote -v
git remote add origin URL
git remote set-url origin URL
git remote remove origin
git fetch origin
git fetch --prune origin
git pull
git pull --rebase
git push
git push -u origin nueva-funcionalidad
git push origin --delete rama-antigua
```

- `git remote -v` lista los remotos y sus URLs.
- `git remote add origin URL` registra un remoto llamado `origin`. Si ya existe, usa `git remote set-url origin URL`.
- `git remote remove origin` quita esa referencia del repositorio local; no elimina el repositorio alojado en Forgejo.
- `git fetch origin` descarga commits y actualiza referencias como `origin/main`, pero **no** cambia tus archivos ni integra los cambios en tu rama actual.
- `git fetch --prune origin` también limpia referencias remotas de ramas que ya no existen en el servidor.
- `git pull` equivale, en términos generales, a traer cambios con `fetch` e integrarlos en la rama actual. Según la configuración, la integración puede ser un merge o un rebase.
- `git pull --rebase` trae los cambios y vuelve a aplicar encima tus commits locales. No reescribas commits que ya hayas compartido sin coordinarlo.
- `git push` publica la rama actual cuando ya tiene remoto y seguimiento configurados.
- `git push -u origin NOMBRE` publica la rama y configura su rama remota de seguimiento; normalmente bastará `git push` en las siguientes ocasiones.
- `git push origin --delete NOMBRE` elimina una rama del servidor. No borra la rama local.

**Diferencia clave:** `fetch` es la opción prudente para inspeccionar primero lo que llegó; `pull` además lo integra. Si el remoto rechaza un `push` porque contiene commits nuevos, ejecuta `git fetch` y revisa la integración necesaria antes de volver a publicar.

### 8. Integrar cambios: merge y rebase

```powershell
git merge main
git merge --abort
git rebase main
git rebase --continue
git rebase --abort
git cherry-pick HASH
```

- `git merge main`, estando en otra rama, integra `main` en la rama actual. Puede crear un commit de merge; conserva la historia de ambas líneas de trabajo.
- `git merge --abort` intenta cancelar un merge que está en curso y volver al estado anterior a su inicio.
- `git rebase main` reaplica los commits de tu rama actual sobre el último commit de `main`. Produce una historia lineal, pero cambia los identificadores de los commits reaplicados.
- Durante un rebase, resuelve cada conflicto, prepara los archivos resueltos con `git add` y ejecuta `git rebase --continue`. `git rebase --abort` cancela el proceso.
- `git cherry-pick HASH` aplica en la rama actual el cambio de un commit concreto y crea un commit nuevo.

No reescribas con `rebase` commits que otras personas ya estén usando, salvo que el equipo lo haya acordado. Para una rama compartida, `merge` suele ser la opción más segura.

#### Resolver conflictos

Un conflicto indica que Git no puede combinar automáticamente cambios incompatibles. `git status` identifica los archivos afectados. En ellos pueden aparecer marcadores como `<<<<<<<`, `=======` y `>>>>>>>`.

1. Abre cada archivo en conflicto y edita su contenido hasta dejar la versión final, quitando los marcadores.
2. Comprueba el resultado con `git diff`.
3. Marca cada archivo resuelto con `git add ARCHIVO`.
4. Continúa según la operación: `git merge --continue` o `git rebase --continue`; en un merge también puedes terminar con `git commit`.

Para cancelar, usa `git merge --abort` o `git rebase --abort`, según corresponda. No elijas automáticamente «ours» o «theirs» sin entender qué contenido descartarían.

### 9. Consultar historial y comparar versiones

```powershell
git log
git log --oneline --graph --decorate --all
git log -- archivo.txt
git show HASH
git diff HASH1..HASH2
git blame archivo.txt
git grep "texto"
```

- `git log` muestra el historial; `git log --oneline --graph --decorate --all` ofrece un resumen visual compacto de todas las ramas conocidas.
- `git log -- ARCHIVO` filtra el historial por archivo.
- `git show HASH` muestra los metadatos y cambios de un commit. `HEAD` significa el commit actual; `HEAD~1`, su primer padre.
- `git diff HASH1..HASH2` compara dos puntos del historial. También puedes comparar una rama con otra.
- `git blame ARCHIVO` muestra qué commit modificó por última vez cada línea. Sirve para investigar contexto, no para atribuir culpabilidad.
- `git grep "texto"` busca texto en los archivos versionados del proyecto.

Los hashes pueden abreviarse mientras sean inequívocos. `HEAD` es una referencia útil para expresar «donde estoy ahora».

### 10. Deshacer cambios sin perder trabajo

Antes de deshacer nada, consulta `git status` y, si el cambio importa, crea una copia o una rama de respaldo. Estos comandos afectan zonas distintas:

```powershell
git restore archivo.txt
git restore --staged archivo.txt
git reset --soft HEAD~1
git reset HEAD~1
git revert HASH
```

- `git restore ARCHIVO` descarta las modificaciones **sin preparar** de ese archivo y lo recupera desde el índice. Es una operación destructiva para esos cambios locales.
- `git restore --staged ARCHIVO` quita el archivo del índice, pero conserva sus modificaciones en el directorio de trabajo. Es la forma habitual de «deshacer `git add`».
- `git reset --soft HEAD~1` mueve la rama al commit anterior y mantiene los cambios preparados.
- `git reset HEAD~1` (modo `--mixed`, el predeterminado) mueve la rama y deja los cambios en el directorio de trabajo, sin preparar.
- `git reset --hard HASH` mueve la rama y reemplaza tanto el índice como los archivos de trabajo. **Puede borrar cambios locales permanentemente**; no lo uses como método rutinario para corregir commits.
- `git revert HASH` crea un commit nuevo que deshace el efecto del commit indicado. Es preferible para revertir cambios que ya se publicaron porque no borra ni reescribe la historia compartida.

En general: para quitar un archivo del próximo commit sin perder su edición, usa `git restore --staged`; para corregir historia que solo tienes local, considera `git reset` con cuidado; para deshacer historia compartida, usa `git revert`.

### 11. Guardar cambios temporalmente: stash

`stash` aparta temporalmente cambios locales para que puedas cambiar de rama o atender otra tarea:

```powershell
git stash push -m "trabajo en curso"
git stash list
git stash show -p 'stash@{0}'
git stash apply
git stash pop
git stash drop 'stash@{0}'
```

- `git stash push -m "mensaje"` guarda los cambios rastreados y deja limpio el directorio de trabajo.
- `git stash list` lista las entradas guardadas.
- `git stash show -p stash@{0}` muestra el diff de la entrada más reciente.
- `git stash apply` restaura la entrada más reciente y la conserva en la lista.
- `git stash pop` la restaura y la elimina de la lista si se aplica correctamente.
- `git stash drop stash@{0}` elimina una entrada concreta. No la elimina hasta comprobar que ya no la necesitas.

Los archivos nuevos sin seguimiento no siempre se incluyen: usa `git stash push -u -m "mensaje"` para incluirlos. Un stash no es un sustituto de un commit o una copia de seguridad.

### 12. Ignorar archivos, moverlos y borrar contenido versionado

```powershell
git rm archivo.txt
git rm --cached archivo.txt
git mv antiguo.txt nuevo.txt
git check-ignore -v archivo.txt
git ls-files
```

- `.gitignore` declara patrones de archivos que Git no debe proponer como nuevos. No deja de rastrear archivos que ya están versionados.
- `git rm ARCHIVO` elimina el archivo del disco y prepara su eliminación del próximo commit.
- `git rm --cached ARCHIVO` deja el archivo en el disco, pero lo quita del control de versiones. Añade su patrón a `.gitignore` si debe seguir siendo local.
- `git mv ORIGEN DESTINO` mueve o renombra un archivo y prepara el cambio.
- `git check-ignore -v ARCHIVO` explica qué regla está ignorando un archivo.
- `git ls-files` lista archivos rastreados.

No subas `.env`, credenciales, claves privadas ni datos de `forgejo-data/` o `postgres-data/`. Si un secreto llegó a un commit, eliminarlo en otro commit no lo retira del historial ni invalida la credencial: revócala o rótala y coordina la limpieza del historial.

### 13. Etiquetas y versiones

```powershell
git tag
git tag v1.0.0
git tag -a v1.0.0 -m "Versión 1.0.0"
git show v1.0.0
git push origin v1.0.0
git push origin --tags
git tag -d v1.0.0
```

- `git tag` lista etiquetas.
- `git tag NOMBRE` crea una etiqueta ligera en el commit actual.
- `git tag -a NOMBRE -m "mensaje"` crea una etiqueta anotada, recomendada para versiones.
- `git show NOMBRE` inspecciona la etiqueta y el commit asociado.
- `git push origin NOMBRE` publica una etiqueta concreta; las etiquetas no siempre se envían con el `push` normal.
- `git push origin --tags` publica todas las etiquetas locales.
- `git tag -d NOMBRE` borra una etiqueta local. Eliminar una etiqueta remota requiere una operación explícita y debe coordinarse.

### 14. Diagnóstico y herramientas avanzadas

```powershell
git reflog
git bisect start
git bisect bad
git bisect good HASH
git bisect reset
git clean -n
git clean -fd
git fsck
git count-objects -v
git archive --format=zip --output=proyecto.zip HEAD
```

- `git reflog` registra movimientos recientes de referencias locales. Puede ayudar a encontrar un commit que ya no aparece en una rama después de un reset o rebase; no es una copia de seguridad permanente.
- `git bisect` busca qué commit introdujo un fallo mediante búsqueda binaria: marca una versión mala (`bad`) y otra buena (`good`), prueba las versiones que propone Git y continúa marcando cada una. Ejecuta `git bisect reset` al terminar.
- `git clean -n` (**simulación**) muestra qué archivos sin seguimiento eliminaría. `git clean -fd` los borra; revisa primero la simulación, porque esos archivos no están guardados en commits.
- `git fsck` comprueba la integridad y puede encontrar objetos no referenciados.
- `git count-objects -v` informa sobre el almacenamiento de objetos.
- `git archive` genera un archivo con el contenido versionado de un commit.

Funciones útiles para proyectos que las utilizan:

```powershell
git submodule status
git submodule update --init --recursive
git worktree list
git worktree add ..\proyecto-hotfix hotfix
```

- Los **submódulos** registran otro repositorio dentro del proyecto. `git submodule update --init --recursive` inicializa y descarga también los submódulos anidados.
- Los **worktrees** permiten tener varias ramas en carpetas de trabajo distintas compartiendo el mismo repositorio local. `git worktree add RUTA RAMA` crea otro directorio; respeta las restricciones de Git sobre usar simultáneamente una misma rama.

No ejecutes `git gc` manualmente como solución a problemas normales: Git mantiene y optimiza su base de datos automáticamente. Tampoco edites archivos dentro de `.git` a mano.

### 15. Atajos de referencia

| Necesitas... | Comando |
| --- | --- |
| Ver si hay cambios pendientes | `git status` |
| Revisar cambios todavía no preparados | `git diff` |
| Revisar lo que entrará en el commit | `git diff --staged` |
| Preparar un archivo | `git add ARCHIVO` |
| Guardar un commit local | `git commit -m "mensaje"` |
| Ver el historial resumido | `git log --oneline --graph --decorate --all` |
| Crear y cambiar de rama | `git switch -c NOMBRE` |
| Cambiar de rama | `git switch NOMBRE` |
| Descargar referencias remotas sin integrar | `git fetch origin` |
| Publicar la rama actual | `git push` |
| Deshacer `git add` sin perder edición | `git restore --staged ARCHIVO` |
| Deshacer cambios locales no preparados | `git restore ARCHIVO` (descarta esos cambios) |
| Deshacer un commit compartido | `git revert HASH` |
| Consultar ayuda de un comando | `git help COMANDO` |

### 16. Ayuda completa incluida en Git

La lista anterior no sustituye el manual oficial de la versión instalada. Para encontrar comandos, opciones y ejemplos adicionales:

```powershell
git help -a
git help -g
git help status
git status -h
```

- `git help -a` lista los comandos disponibles.
- `git help -g` lista los conceptos y guías generales.
- `git help COMANDO` abre el manual del comando, por ejemplo `git help rebase`.
- `git COMANDO -h` muestra una ayuda breve cuando ese subcomando admite esa opción.

## Solución de problemas

- **No abre la interfaz web:** comprueba `docker compose ps` y `docker compose logs -f server`; verifica que el puerto configurado en `FORGEJO_HTTP_PORT` esté libre y que exista `.env`.
- **No conecta por SSH:** comprueba que Forgejo esté activo, que el puerto configurado en `FORGEJO_SSH_PORT` esté permitido por el firewall y que la URL SSH use ese puerto. Confirma también que añadiste la clave pública correcta a tu cuenta.
- **La aplicación no conecta con PostgreSQL:** consulta `docker compose logs -f db` y `docker compose logs -f server`. En esta red, el servidor debe conectarse a `db:5432`; `localhost` dentro del contenedor no apunta al servicio de base de datos.
- **No aparecen los datos previos:** verifica que estás ejecutando Compose desde la carpeta correcta y que existen `forgejo-data/` y `postgres-data/`.
